# EntraIDAppSecretsnCertsExpirationNotification

Use Power Automate to proactively notify of upcoming Azure AD / Entra ID application client secrets and certificate expirations 
 
Are you challenged with keeping up with all your Azure Active Directory Enterprise Application client secrets and certificates and their associated expiration dates? 
I sometimes work with customers who have a thousand or more Azure AD / Entra ID applications to manage. Unfortunately, trying to keep up with all the client secrets and certificates expiring on each one of these apps can be a futile experience.   

I came up with a solution to this problem by using Power Automate to proactively notify the Entra ID administrators of upcoming client secret and certificate expirations. This solution was a big help for customers with thousands of AAD apps to keep track of.  I owe a huge thanks to a friend and peer of mine, Norman Drews, for his CSS & HTML expertise. 

Here’s how I solved it using Power Automate: 
 
1.	Create (or use an existing) Azure AD app registration that has ONE of the following Application Permissions (starting from the least and ending with the most restrictive option) -  Application.Read.All, Application.ReadWrite.All, Directory.Read.All, or Directory.AccessAsUser.All.  
2.	Create a Scheduled Flow to run daily or weekly depending on how often you want to be alerted.   
3.	Initialize variable (String) – appId – this is the appID of the application. 
4.	Initialize variable (String) – displayName – this will be used for the display name of the application. 
5.	Initialize variable (String) – clientSecret – this needs to be set with the client secret of the Azure AD application created or chosen in step 1. 
6.	Initialize variable (String) – clientId – this needs to be set with the application (client) ID of the Azure AD application created or chosen in step 1. 
7.	Initialize variable (String) – tenantId – this needs to be set with the tenant ID of the Azure AD application created or chosen in step 1. 
8.	Initialize variable (Array) – passwordCredentials – this variable will be used to populate the client secrets of each Azure AD application. 
9.	Initialize variable (Array) – keyCredentials – this variable will be used to populate the certificate properties of each Azure AD application. 
10.	Initialize variable (String) – styles – this is some CSS styling to highlight Azure AD app secrets and expirations that are going to expire in 30 days (yellow) vs 15 days (red).  You can adjust these values accordingly to meet your needs. 
 
Content of this step: 
{ 
  "tableStyle": "style=\"border-collapse: collapse;\"", 
  "headerStyle": "style=\"font-family: Helvetica; padding: 5px; border: 1px solid black;\"", 
  "cellStyle": "style=\"font-family: Calibri; padding: 5px; border: 1px solid black;\"", 
  "redStyle": "style=\"background-color:red; font-family: Calibri; padding: 5px; border: 1px solid black;\"", 
  "yellowStyle": "style=\"background-color:yellow; font-family: Calibri; padding: 5px; border: 1px solid black;\"" 
} 
 
11.	Initialize variable (String) – html – this creates the table headings and rows that will be populated with each of the Azure AD applications and associated expiration info. 
 
Content of this step: 
 
<table @{variables('styles').tableStyle}><thead><th @{variables('styles').headerStyle}>Application ID</th><th @{variables('styles').headerStyle}>Display Name</th><th @{variables('styles').headerStyle}>Days until Expiration</th><th @{variables('styles').headerStyle}>Type</th><th @{variables('styles').headerStyle}>Expiration Date</th></thead><tbody> 
 
12.	Initialize variable (Float) – daysTilExpiration – this is the number of days prior to client secret or certificate expiration to use in order to be included in the report 
13.	We need to request an authentication token using our tenantId, clientId, and clientSecret variables. 
  
14.	The Parse JSON step will parse all the properties in the returned token request. 
The JSON schema to use is as follows: 
{ 
    "type": "object", 
    "properties": { 
        "token_type": { 
            "type": "string" 
        }, 
        "expires_in": { 
            "type": "integer" 
        }, 
        "ext_expires_in": { 
            "type": "integer" 
        }, 
        "access_token": { 
            "type": "string" 
        } 
    } 
} 
 
  
 
15.	Initialize variable (String) – NextLink – This is the graph API URI to request the list of Azure AD applications.  The $select only returns the appId, DisplayName, passwordCredentials, and keyCredentials, and since graph API calls are limited to 100 rows at a time, I bumped my $top up to 999 so it would use less API requests (1 per 1000 apps vs 10 per 1000 apps). 
https://graph.microsoft.com/v1.0/applications?$select=appId,displayName,passwordCredentials,keyCredentials&$top=999 
16.	Next, we enter the Do until loop. It will perform the loop until the NextLink variable is empty.  The NextLink variable will hold the @odata.nextlink property returned by the API call. When the API call retrieves all the applications in existence, there is no @odata.nextlink property.  If there are more applications to retrieve, the @odata.nextlink property will store a URL containing the link to the next page of applications to retrieve. The way to accomplish this is to click “Edit in advanced mode” and paste @empty(variables('NextLink')). 
  
17.	The next step in the Do until loop uses the HTTP action to retrieve the Azure AD applications list.  The first call will use the URL we populated this variable within step 15. 
18.	A Parse JSON step is added to parse the properties from the returned body from the API call. 

The content of this Parse JSON step is as follows:

{
    "type": "object",
    "properties": {
        "@@odata.context": {
            "type": "string"
        },
        "value": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "appId": {
                        "type": "string"
                    },
                    "displayName": {
                        "type": "string"
                    },
                    "passwordCredentials": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "customKeyIdentifier": {},
                                "displayName": {},
                                "endDateTime": {},
                                "hint": {},
                                "keyId": {},
                                "secretText": {},
                                "startDateTime": {}
                            },
                            "required": []
                        }
                    },
                    "keyCredentials": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "customKeyIdentifier": {},
                                "displayName": {},
                                "endDateTime": {},
                                "key": {},
                                "keyId": {},
                                "startDateTime": {},
                                "type": {},
                                "usage": {}
                            },
                            "required": []
                        }
                    }
                },
                "required": []
            }
        },
        "@@odata.nextLink": {
            "type": "string"
        }
    }
}

19.	A Get future time action will get a date in the future based on the number of days you’d like to start receiving notifications prior to expiration of the client secrets and certificates. 
20.	Next a foreach – apps loop will use the value array returned from the Parse JSON step of the API call to take several actions on each Azure AD application. 
  
21.	Set variable (String) – appId – uses the appId variable we initialized in step 3 to populate it with the application ID of the current application being processed. 
22.	Set variable (String) – displayName – uses the displayName variable we initialized in step 4 to populate it with the displayName of the application being processed. 
23.	Set variable (String) – passwordCredentials – uses the passwordCredentials variable we initialized in step 5 to populate it with the client secret of the application being processed. 
24.	Set variable (String) – keyCredentials – uses the keyCredentials variable we initialized in step 5 to populate it with the client secret of the application being processed. 
25.	A foreach will be used to loop through each of the client secrets within the current Azure AD application being processed. 
  
26.	The output from the previous steps to use for the foreach input is the passwordCreds variable.  
27.	A condition step is used to determine if the Future time from the Get future time step 19 is greater than the endDateTime value from the current application being evaluated. 
28.	If the future time isn’t greater than the endDateTime, we leave this foreach and go to the next one. 
29.	If the future time is greater than the endDateTime, we first convert the endDateTime to ticks. Ticks is a 100-nanosecond interval since January 1, 0001 12:00 AM midnight in the Gregorian calendar up to the date value parameter passed in as a string format. This makes it easy to compare two dates, which is accomplished using the expression ticks(item()?[‘endDateTime’]). 
30.	Next, use a Compose step to convert the startDateTime variable of the current time to ticks, which equates to ticks(utcnow()). 
31.	Next, use another Compose step to calculate the difference between the two ticks values, and re-calculate it using the following expression to determine the number of days between the two dates. 

div(div(div(mul(sub(outputs('EndTimeTickValue'),outputs('StartTimeTickValue')),100),1000000000) , 3600), 24) 

32.	Set the variable daystilexpiration to the output of the previous calculation. 
33.	Set variable (String) – html – creates the HTML table.  The content of this step is as follows: 
 
<tr><td @{variables('styles').cellStyle}><a href="https://ms.portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationMenuBlade/Credentials/appId/@{variables('appId')}/isMSAApp/">@{variables('appId')}</a></td><td @{variables('styles').cellStyle}>@{variables('displayName')}</td><td @{if(less(variables('daystilexpiration'),15),variables('styles').redStyle,if(less(variables('daystilexpiration'),30),variables('styles').yellowStyle,variables('styles').cellStyle))}>@{variables('daystilexpiration')} </td><td @{variables('styles').cellStyle}>Secret</td><td @{variables('styles').cellStyle}>@{formatDateTime(item()?['endDateTime'],'g')}</td></tr> 
 
34.	Another foreach will be used to loop through each of the certificates within the current Azure AD application being processed.  This is a duplication of steps 25 through 33 except that it uses the keyCredentials as its input, compares the future date against the currently processed certificate endDateTime, and the Set variable – html step is as follows: 
 
<tr><td @{variables('styles').cellStyle}><a href="https://ms.portal.azure.com/#blade/Microsoft_AAD_RegisteredApps/ApplicationMenuBlade/Credentials/appId/@{variables('appId')}/isMSAApp/">@{variables('appId')}</a></td><td @{variables('styles').cellStyle}>@{variables('displayName')}</td><td @{if(less(variables('daystilexpiration'), 15), variables('styles').redStyle, if(less(variables('daystilexpiration'), 30), variables('styles').yellowStyle, variables('styles').cellStyle))}>@{variables('daystilexpiration')} </td><td @{variables('styles').cellStyle}>Certificate</td><td @{variables('styles').cellStyle}>@{formatDateTime(item()?['endDateTime'], 'g')}</td></tr> 
 
  
 
35.	Immediately following the foreach – apps loop, as a final step in the Do while loop is a Set NextLink variable which will store the dynamic @odata.nextlink URL parsed from the JSON of the API call. 
36.	Append to variable (Array) – html – Immediately following the Do while loop ends, we close out the html body and table by appending <tbody></table> to the variable named html. 

 

37.	Finally, send the HTML in a Send an e-mail action, using the variable html for the body of the e-mail. 
   
And below is the resulting e-mail received when the flow runs on its scheduled time.  Included is a hyperlink for each application that takes you directly to where you need to update the client secret and/or certificates for each application within the Azure portal. 

  

