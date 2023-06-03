## PIP Alerting Framework
The below Architecture was Designed and Developed by myself in my time at Perenti. For obvious reasons, I am unable to provide any of the source code or any other confidential information such as the service display names.

This piece of documentation provides insight into the Perenti Integration Platform Alerting Framework. It provides a high level design which illustrates the different entry points into the workflow.

### High Level Design
![PIP Alerting Framework](https://github.com/AidenOliver1/Alerting_Framework/assets/94108662/e7025637-a009-4e42-87aa-5161b5a04e9f)


#### Narrative
There are 4 major entry points into the workflow of the framework. These include:
1. Logic App #2 (Automatic Schedule Trigger)
2. Logic App #1 (Event Trigger to Web-Hook)
3. Logic App #4 (Manual HTTP Trigger)
4. Logic App #3 (Manual HTTP Trigger)

Once an entry point logic apps get triggered, the payload will go through multiple stages of transformation and data quality checks before it eventually sends an email and/or a POST request to the ServiceNow API to raise a ticket.
 
##### Entry Point - 1:
1. The 'Logic App #2' schedule get's triggered once a day. The logic then initiates authentication values and sends a GET request to the SAP CPI API. The  returned content then gets passed through a for-each loop. Each json Alert gets transformed using a Liquid file (Located in an Integration Account) to an acceptable payload. 'Logic App #3'  then gets called.
2. The 'Logic App #3' transforms the data further to then be parsed to an API endpoint in a API Management instance, via a HTTP request. This then calls a function app.
3. The function is C# code which processes the payload and converts it into a friendly schema and upserts it as an event in 'Event Hub #1'.
4. A Stream Analytic Job Query is constantly looking at event hub 'Event Hub #1' and moving any events which return True to the following logic to 'Event Hub #2':
`SELECT
    *
INTO
    errorlog
FROM
    unilog
where 
    Severity = 'Error' or Severity = 'Critical' or Severity = 'Warning'`
5. The Logic App #6' then has an event-based trigger which points at the 'Event Hub #2' event hub. Once this logic app is triggered, it will transform the payload and do the following:

| Alert Type | Send Email ( Y / N ) | Raise Ticket ( Y / N )
|---|---|---|
| CPI iFlow errors | Y | Y |
| CPI Stalled iFlows | Y | Y |
##### Entry Point - 2:
1. The KeyVault instance has an event-based webhook which sends off a HTTP POST Request to 'Logic App #1' whenever a Key / Secret / Certificate changes status to either of the following:
i. NewVersionCreated
ii. NearExpiry
iii. Expired 
2. 'Logic App #1' then transforms the payload using a liquid file located in an Integration Account and then calls 'Logic App #3'.
3. 'Logic App #3' transforms the data further to then be parsed to an API endpoint in an API Management instance, via a HTTP request. This then calls a function app
4. The function is C# code which processes the payload and converts it into a friendly schema and upserts it as an event in 'Event Hub #1'.
5. A Stream Analytic Job Query is constantly looking at event hub 'Event Hub #1' and moving any events which return True to the following logic to 'Event Hub #2':
`SELECT
    *
INTO
    errorlog
FROM
    unilog
where 
    Severity = 'Error' or Severity = 'Critical' or Severity = 'Warning'`
6. The Logic App #6' then has an event-based trigger which points at the 'Event Hub #2' event hub. Once this logic app is triggered, it will transform the payload and do the following:

| Status Type | Send Email ( Y / N ) | Raise Ticket ( Y / N )
|---|---|---|
| NewVersionCreated | Y | N |
| NearExpiry | Y | N |
| Expired | Y | Y |

##### Entry Point - 3:
1. 'Logic App #4' entry point is widely used and typically the preferred method of alerting. It has a HTTP trigger which is currently getting called from Integration Services, Azure Monitor Alerts and more! 
2. Once the logic app gets called, it transforms the payload using a liquid transform template located in a storage account. Based off the returned data of that payload, it will either raise a ticket in ServiceNow or just send off an email.

| Alert Type | Send Email ( Y / N ) | Raise Ticket ( Y / N )
|---|---|---|
| Integration Service RunsFailed | Y | Y |
| APIM FailedRequests | Y | N |
| APIM ResponseTime | Y | N |
 
##### Entry Point - 4:
1. 'Logic App #3' is also a very common workflow to generate alerts for Integration Services. It has a HTTP trigger which is called with a generic JSON Schema. This then calls a function app.
2. The function is C# code which processes the payload and converts it into a friendly schema and upserts it as an event in 'Event Hub #1'.
3. A Stream Analytic Job Query is constantly looking at event hub 'Event Hub #1' and moving any events which return True to the following logic to 'Event Hub #2':
`SELECT
    *
INTO
    errorlog
FROM
    unilog
where 
    Severity = 'Error' or Severity = 'Critical' or Severity = 'Warning'`
4. The Logic App #6' then has an event-based trigger which points at the 'Event Hub #2' event hub. Once this logic app is triggered, it will transform the payload and do the following:

| Alert Type | Send Email ( Y / N ) | Raise Ticket ( Y / N )
|---|---|---|
| Integration Service RunsFailed | Y | Y |

### Azure Resources
| Environment	| Resource Type	| Resource |
|---|---|---|
| Production | Logic App | 'Logic App #2' | 
| Production | Logic App | 'Logic App #3' | 
| Production | Logic App | 'Logic App #4' |  
| Production | Logic App | 'Logic App #1' |  
| Production | Logic App | 'Logic App #6' | 
| Production | Logic App | 'Logic App #5' | 
| Production | Event Hub Namespace | - |
| Production | API Management Services | - | 
| Production | Function | - | 
| Production | Stream Analytics Job | - | 
| Production | Azure Monitor (Alert Rules) | - |
| Production | Storage Account | - |
| Production | Key Vault | - |

### Alert Configuration

Alerts are generated for the following scenarios

||Alert Type|Email Notification| Incident Assignment Group
|---|---|---|---|
|1. | CPI iFlow errors | - | -
|2. | CPI Stalled iFlows | - | -
|3. | APIM ResponseTime Alerts | - | -
|4. | APIM FailedRequest Alerts | - | -
|5. | KeyVault NearExpiry Alerts | - | -
|6. | KeyVault Expired Alerts | - | -
|7. | KeyVault NewVersionCreated | - | -
|8. | LogicApp RunsFailed Alerts | - | -














