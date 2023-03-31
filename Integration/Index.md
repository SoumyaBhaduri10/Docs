![logo](images\DXC-logo)

Checks performed before pulling the CI information:

- set logs folder path

- set log file paths

- set proxy server and mail parameters 
- Fetch Credentials from CyberArk. Refer here for the document to fetch the credentials from CyberArk.

Steps performed to pull the CI Information:

- Fetch the Customer name from Aruba_Customers.csv

- Send HTTPS request to fetch Org Id by Customer Name from CMDB+

- Send HTTPS request to fetch Orchestrator URL from CMDB+

- Send HTTPS request to get all Edges from CMDB+

- Send HTTPS request to get Edge location from CMDB+

- Store Edge details in csv file
  
  
![image](images/Picture3)

  **Figure 4 - List of discovered Edges**

We are pulling the device list from the CMDB+ and the alarms as well as performance metrics are retrieved from Aruba orchestrator itself.
Benefits of Pulling data from CMDB+ instead of Aruba Orchestrator:
-	We can reduce the dependency on Aruba orchestrator.
-	We can reduce the load on orchestrator as we are not pulling Device list each time, this results in less time complexity and fast retrieving of metrics.
-	Keeps track of Network Configuration Items, Services and Customer Information required for support.
-	Trusted source for all Monitoring, Reporting and Ticketing systems.
-	Document management with automatic versioning.
-	Turnover to production tool for orderly turnover of network CI to a delivery team.
-	It allows not only to filter on a given object (infra, contact, contract, …) but also on related objects (organization, site, …) and linked objects (contracts linked to CI, CI linked to solution, …).

##  2.4   Performance Metrics Data Collection and Alerting

To collect performance data for alerting, another PowerShell script, which will be residing on server (Integration Manager). Script connects to Aruba through Rest API, fetches the Edge Performance data and stores to CSV file. Script compares this data with Threshold CSV and determine severity of alert then post it to Platform X Event API.

![logo](images\DXC-logo)

Navigate to /opt/tools/integration-manager/aruba/bin directory and execute the below script. 



```
pwsh sdwan_aruba_metrics.ps1
```

Following input files will be used for alerting based on collected performance data –

| Input | Description  | Function|
| :-------- | :------- | :------ |
| ci_sync.csv | This file is output of action performed under **section 2.4** | Collects recent edge inventory |
| threshold.csv | It contains performance metric threshold values | Threshold for metrics like CPU & Memory for all the edges/CIs |
| counter_sync.csv | Counter for 3 cycle incident creation (CPU and memory) | Script will keep updating the counter value in every execution and on 3rd counter event will be created based on CPU and memory |
| counter_sync_bandwidth.csv | Counter for 3 cycle incident creation (Bandwidth) | Script will keep updating the counter value in every execution and on 3rd counter event will be created based on Bandwidth |


Threshold metrics and values:

The threshold.csv file consists of following fields:
-	Customer_Name: It indicates the name of the customer.
-	Event_Name: It indicates events which may be CPU, Memory, Tx and Rx Percentage
-	Appliance_Name: Appliance name is mentioned as * which means All the appliances.
-	Major_Threshold: Major Threshold indicates the maximum load that is acceptable and used to compare the current threshold of the device
 Eg: In our case the Major_Threshold is set to 80 depends on Event.
-	Error_Threshold: Error Threshold indicates the unhealthy status of the device and needs to be checked.

    Eg: In our case the Error_Threshold is set to 90 depends on Event.





## 2.5	Scheduling And Execution of Main Code 

## 2.5.1	Fault Management Script

The purpose of these scripts ( sdwan_aruba_active_alarms .ps1, sdwan_aruba_closed_alarms.ps1) is to fetch Aruba events, create an incident on Platform X and auto resolve the incident when the event on Aruba is resolved.
This script is scheduled via cron job which will trigger the script every 2 minutes.


```
 pwsh sdwan_aruba_active_alarms.ps1
```

### **2.5.1.1	Steps performed by the script:**

First, execute the script sdwan_aruba_active_alarms.ps1
-	Script logging is started.
-	In the first section of the script, paths for the input files are specified along with Proxy server and API server name for PlatformX.
-	Next, Fetch the PlatformX username and password from CyberArk safe.
Refer here for the document to fetch the credentials from CyberArk.
-	Fetch the customer’s name from Aruba_Customers.csv file
-	Fetch list of Appliance IDs from ci_sync.csv file 
-	Fetch list of incidents from incident_logs.csv file 
-	As we are only picking the devices which are production and pre-production 
If CI_sync.csv file is empty
-	Then print that the CI_sync.csv is empty and make an entry in the log file saying “CI_Sync.csv file is empty”.
Else
-	Fetch Aruba orchestrator Url from CI sync file and Send HTTPS request to cyberArk to fetch Aruba Credentials for the desired customer and retrieve Aruba Token from it.
-	Check device reachability for each appliance
	  
**Steps to check Device Reachability:**
-	Pull the Device list from CMDB+
-	Loop through the device id’s and for each device id make the api calls to the appliance.
https://<ArubaAPIServer>:443/gms/rest/reachability/appliance/<appliance-id>?source=menu_rest_apis_id
-	Store the status code for the api call we made
-	If device is not reachable, create incident if key does not present in inc logs and add the details in incident csv file.
-	If device is now reachable, close incident if key present in inc logs and state is open and update incident state in csv file
-	Fetch appliance related active alarms occurred in past 5 minutes
-	Fetch orchestrator level active alarms occurred in past 5 minutes 

**To Create incidents for Appliance Related Alarms**

-	Check if current alarm type ID is listed in the alarm whitelist csv file
-	Filter out alarm: "Many tunnels to remote sites are down" 
-	Move forward with only the alarms listed in alarm whitelist csv file and alarms with severity major & critical 
-	Map alarm details as per Platform X incident format  
-	Create Incident body based upon the mapping 
-	Check if alarm log csv file exists 
-	If alarm log file exists 

o	Check if alarm ID already exist in csv file 

o	Creating incident for new alarm when alarm type is as specified in csv file 

-	If alarm log file does not exist  

o	creating incident for new alarm when alarm type is as specified in csv file    

**To create incidents for Orchestrator Related Alarms:**

-	Check if current alarm type ID is listed in the alarm whitelist csv file 
-	Move forward with only the alarms listed in alarm whitelist csv file and alarms with severity Major & critical 
-	Map alarm details as per Platform X incident format  
-	Create Incident body based upon the mapping 
-	Check if alarm log csv file exists 
-	If alarm log file exists

![logo](images\DXC-logo)


o	Check if alarm ID already exist in csv file

o	Creating incident for new alarm when alarm type is as specified in csv file 

-	If alarm log file does not exist  

o	Create incident for new alarm when alarm type is as specified in csv file      

-	If any error occurs while script is being executed an email is sent specifying the error message.
-	Store the event details in aruba_alarm_logs.csv
-	Script status in binary and latest script execution timestamp is stored in **active_alarm_script_status.csv** (this file will be used in Self-Monitoring of Scripts – Section 2.8)
-	Script logging is stopped.
-	This script will generate 3 output files:

o	/opt/tools/integration-manager/aruba/conf/aruba_alarm_logs.csv 

o	/opt/tools/integration-manager/aruba/Logs/_sdwan_aruba_active_alarms.log

o	/opt/tools/integration-manager/aruba/conf/active_alarm_script_status.csv


![image](images\Picture2)


**Figure 5 - Alarm Logs**

![logo](images\DXC-logo)

| Platform X Incident Fields |  Aruba Event Fields |
:-------- | :------------------------ | 
|  Severity |          Major/Critical/Minor |
| Title |  AlarmName |
|Node | EdgeName |
| RelatedCIHints|  EdgeName|
|EventSourceSendingServer |ArubaAPIServer  |
| EventSourceExternalID|ArubaAPIServer |
|EventSourceCreatedTime | EventTime|
|Category | SD-WAN|
| Application| Aruba EdgeConnect Orchestrator|
|Object |SD-WAN Aruba |
| BusinessService|Standard Management LAN/WAN |
| SupportGroup| Network NSRA DNOC India|
| IncCategory|      Network|
|IncSubCategory |WAN Failure |
|             key|      Alarm ID + Alarm Type ID|
|     Site Location |           Edge Device Address|
|       autoclosekey| Alarm ID + Alarm Type ID|
|   Customer Name |                Customer Name|
|     Site Location |            Edge Device Address |
|               Aruba EdgeConnect URL|Aruba Orchestrator URL |

Now, execute the script **sdwan_aruba_closed_alarms.ps1**

This script is scheduled via cron job which will trigger the script every **2 minutes**.

```
pwsh sdwan_aruba_closed_alarms.ps1
```

While closing the alarm, 

o	Fetch appliance related closed alarms

o	Fetch orchestrator level closed alarms

o	Check if alarm log csv file exists


![logo](images\DXC-logo)

If exists:

o	Mapping of alarm details as per Platform X incident format

o	Close incident if alarm ID found in aruba_alarm_logs.csv file

o	This script will generate 2 output files and update one file.

o	/opt/tools/integration-manager/aruba//Logs/_sdwan_aruba_closed_alarms.log

o	/opt/tools/integration-manager/aruba/conf/closed_alarm_script_status.csv

o	Update the status of alarm in aruba_alarm_logs.csv file

-	Incident created in ServiceNow


![](images\Picture1)

**Figure 6 – ServiceNow Incident for Tunnel Down [from DevQA environment]**

## **2.5.2	Performance Management Script:**

The purpose of this script(sdwan_aruba_metrics.ps1) is to fetch Aruba performance data in every 5 minutes and based on threshold, create an incident, update the severity of incident on Platform X and auto resolve the incident when the event in Aruba is resolved.

This script is scheduled via cron job which will trigger the script every **5 minutes**.

```
pwsh sdwan_aruba_metrics.ps1
```
![logo](images\DXC-logo)


![image](images\Picture4)
![image](images\Picture5)








                     






