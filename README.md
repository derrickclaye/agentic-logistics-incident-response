# agentic-logistics-incident-response

## System Overview

PepsiCo’s logistics operations required an intelligent automation system capable of dynamically calculating delay costs based on customer contracts, identifying the most cost-effective rerouting options while accounting for delivery constraints, and seamlessly coordinating execution with external logistics providers and customer communications — all without manual intervention.

To address this, I designed and implemented an automated supply chain incident processing system that analyzes the financial impact of truck breakdowns, determines optimal rerouting strategies, and orchestrates execution with external logistics partners through AI agents and ServiceNow workflow automation.

## System Components

1. ServiceNow Scoped Application (2 tables, 2 AI Agents, 1 Use Case)

2. n8n Orchestration Agent: Responsible for receiving logistical remediation data from ServiceNow and invoking the appropriate MCP Clients.

3. 2 AI Agents created in ServiceNow AI Agent Studio using ServiceNow's proprietary NOW LLM

4. Schneider MCP Server: Exposes the tool **execute_route(route_id, truck_id, chosen_option)** that is utilized by the n8n Orchestration Agent. The purpose of this tool is to update Schneider on the newly updated route to take.

5. Retailer MCP Server (ex. Wholefoods): Exposes the tool **notify_delivery_delay(route_id, truck_id, chosen_option)** that is utilized by the n8n Orchestration Agent. The purpose of this tool is to update the retailer of the delay details and the updated route Schneider is taking.

6. ServiceNow MCP Server: The purpose of this MCP server is to update the Delivery Delay table. It adds records to the table when a truck delay is reported from Schneider. It also updates the status of a record when Schneider initiates execution of the new route and the retailer is made aware of the new route change by exposing the too **update_execution_status(route_id, status)**.

## Implementation Steps

1. The first step was to create a scoped application within ServiceNow. I decided to create a scoped application vs a global one because it was imperative that none of the data or application artifacts were compromised.

2. Within the scoped application I created two tables. The first table was the Delivery Delay table. This table was designed to contain the records of the logistical interruptions - which in this case was delivery truck breakdowns. The data each record contains is:

	- route_id (Integer, Primary Key)
	- truck_id (Integer)
	- customer_id (Integer, Default: 1)
	- problem_description (String, 4000)
	- proposed_routes (String, 4000) - JSON format with route options
	- calculated_impact (String, 4000) - JSON format with financial analysis
	- chosen_option (String, 4000) - Selected route details
	- status (String, 16) - Workflow progression: pending/calculated/approved/dispatched
	- assigned_to (Reference to User) - Critical: Used for trigger execution context and permissions
	- incident_sys_id (String, 32) - Links to associated incident records

When a new record is added to the table, the route_id, truck_id, customer_id, status, and proposed_routes are already populated. The status field and the rest of the fields are populated/updated throughout automated process. 

The second table was the Supply Agreements table. This table was designed to contain the records of the retailers currently under contract with PepsiCo. The data each record contains is:

	- customer_id (Integer, Primary Key)
	- customer_name (String, 100)
	- deliver_window_hours (Integer) - Contractual delivery timeframe
	- stockout_penalty_rate (Integer) - Cost per hour of delay in dollars

The deliver_window_hours represents the time threshold agreed upon between PepsiCo and the retailer where PepsiCo won't be charged if the delivery is delayed.

The stockout_penalty_rate represents the fee per hour that PepsiCo has to pay the retailer if the delivery is delayed beyond the agreed time threshold.

3. Within ServiceNow AI Agent Studio I created 2 AI Agents. 

The first agent was the **Route Financial Analysis Agent**. The role of this agent is to calculate the cost incurred by logistical interruptions and provide a comprehensive  summary that outlines its findings as well as update/create the appropriate records documenting its assessment. In order to carry out its responsibilities I created the following tools for it to use:

	- Retrieve Delivery Delay Details (Record Operation Tool) - Fetches the record on the delivery delay table based on the provided sys id.

	- Retrieve Supply Agreement (Record Operation Tool) - Uses the customer id obtained from the delivery delay record to find the corresponding retailer record.

	- Create Incident Record (Record Operation Tool) - Creates an incident record documenting the delivery delay details.

	- Update Delivery Delay Record (Record Operation Tool) - Updates the calculated_impact and status fields on the delivery delay record.

	- Calculate Financial Impact (Script Tool) - Calculates the financial impact of each proposed route by determining how much the estimated time exceeds the allotted time (delivery_window_hours) then multiplying that by the stockout_penalty_rate. To ensure maximum accuracy every time duration was converted to minutes.

The second agent was the **Route Decision Agent**. The role of this agent is to select the optimal route and coordinate external execution of the n8n orchestration agent. In order to carry out its responsibilities I created the following tools for it to use:

	- Retrieve Financial Impact (Record Operation Tool) - Retrieves the calculated_impact field from the Delivery Delay record.

	- Determine Optimal Option - (Script Tool) - Based on the financial impacts of each proposed route, the route with the lowest financial impact is selected. In the event that multiple routes incur no financial penalty or the same financial penalty, then with the shorter estimated time of arrival is selected. After the optimal option is determine the script updates the Delivery Delay record with that selection. It also updates the Impact and Urgency of the Incident Record that was created by the Route Financial Analysis Agent based on the severity of the optimal options financial severity. 

	- Trigger n8n (Script Tool) - This tool acts as a webhoook to send information from ServiceNow to the n8n Orchestration Agent. The data sent over is the route_id, truck_id and chosen_option. 

4. The system is designed to trigger automatically whenever a new record with the status field set to 'pending' is added to the Delivery Delay table. In order to accomplish this I leveraged a feature in ServiceNow's AI Agent Studio called Use Cases. A Use Case allows me to define a scenario that 1 or multiple AI Agents can handle. The two agents outlined above are the agents assigned to this Use Case. In order for the Use Case trigger to work, the **Assigned To** user on that Delivery Delay record must have the necessary permissions.

5. Leveraging the n8n platform I created a **Payload Preparation and MCP Orchestration Agent**. This agent is triggered by webhoook (which is called in the n8n trigger tool within the Route Decision Agent). After the agent is triggered the first step uses an AI Agent node which receives the data sent from ServiceNow and prepares the payloads each MCP Client requires. **Amazon Bedrock** provided the LLM connected to this node, which is **gpt-oss-20b** - a light weight model that uses an MoE architecture. After that node is a code node that just cleans up the payloads ensuring correct formatting. Next is another AI Agent node. This node has 3 MCP Client tools attached to it - each representing one of the MCP servers mentioned above in the System Components section. Since it is very important that this node calls each MCP client accordingly with their associated payloads, I used the **gpt-oss-120b** model. That particular model has very strong reasoning capabilities, is particularly good at function/tool calling and uses an MoE architecture, therefore during inference only the parameters that belong to the chosen experts are activated. So I figured those benefits outweighed the tradeoffs (very large, higher latency). 

## Optimizations

1. For the purpose of this project I was only working with one retailer, but in reality there would be dozens, if not hundreds of retailers that PepsiCo ships prepuces to. In turn, that means the n8n Data Preparation and MCP Orchestration Agent would consume a lot more retail MCPs. If this was the case then the agent would have to have some sort of context that would allow for it to use the appropriate MCP tool. Taking that into consideration I modified the n8n trigger script to also send over the customer_id along with the route_id, truck_id and chosen_option. This way, I could map the customer_id to the MCP Client node that belonged to that respective Retailer. 

2. On the ServiceNow instance I created a new category choice for Incident records named "logistics" which would allow for a more accurately configured incident record.

3. On the ServiceNow instance I created a new group - "PepsiCo Logistics" for the specific purpose of assigning that group the created incident records. 



