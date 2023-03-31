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

![](Images\Picture1.png)
