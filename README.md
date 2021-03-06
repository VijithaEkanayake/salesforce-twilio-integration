[![Build Status](https://travis-ci.org/ballerina-guides/salesforce-twilio-integration.svg?branch=master)](https://travis-ci.org/ballerina-guides/salesforce-twilio-integration)

# Salesforce-Twilio Integration

[Salesforce](https://www.salesforce.com) is the world’s #1 CRM platform that employees can access entirely over the Internet. [Twilio](https://www.twilio.com/) is a cloud communications platform for building SMS, Voice, and Messaging applications on an API built for global scale. To understand how you can use Twilio for sending messages, let's consider a real-world use case of a service promotional SMS sending system to a selected group of Leads.

> This guide walks you through a typical cross-platform integration that uses Ballerina to send customized SMS messages via Twilio to a set of Leads that are taken from Salesforce.

The following are the sections available in this guide.

- [What you'll build](#what-youll-build)
- [Prerequisites](#prerequisites)
- [Developing the Program](#developing-the-program)
- [Testing](#testing)
- [Deployment](#deployment)

## What you'll build

In this particular use case, Salesforce gives the relational contact details of the selected Leads and Twilio is used to contact them via SMS to send promotional messages for the respective user group. This represents a typical cross-platform integration that a marketing or promotion manager might require.

You can use Ballerina Salesforce connector to get the interested leads with their names and phone numbers (by sending an SOQL query) and Ballerina Twilio connector to send an SMS to those relevant phone numbers.
  
![alt text](https://github.com/erandiganepola/salesforce-twilio-integration/blob/master/Salesforce%20-%20Twilio%20integration.svg)

## Prerequisites

* JDK 1.8 or later
* [Ballerina Distribution](https://github.com/ballerina-platform/ballerina-lang/blob/master/docs/quick-tour.md)
* A Text Editor or an IDE
* [Salesforce Connector](https://github.com/wso2-ballerina/package-salesforce) and the [Twilio Connector](https://github.com/wso2-ballerina/package-twilio) will be downloaded from `Ballerina Central` when running the Ballerina file.

### Before you begin

##### Understand the package structure

Ballerina is a complete programming language that can have any custom project structure as you wish. Although the language allows you to have any package structure, use the following simple package structure for this project.

```
salesforce-twilio-integration 
  └── sms-sender
  |    └── test
  |          └── sms_sender_test
  |    └── constants.bal
  |    └── Package.md
  |    └── sms_sender.bal
  └── ballerina.conf
  └── README.md
```

Change the configurations in the `ballerina.conf` file. Replace "" with your data.

##### ballerina.conf
```
TWILIO_ACCOUNT_SID=""
TWILIO_AUTH_TOKEN=""
TWILIO_FROM_MOBILE=""
TWILIO_MESSAGE=""

SF_URL=""
SF_ACCESS_TOKEN=""
SF_CLIENT_ID=""
SF_CLIENT_SECRET=""
SF_REFRESH_TOKEN=""
SF_REFRESH_URL=""

```

Let's first see how to add the Salesforce configurations and Twilio configurations for the application written in Ballerina language.

#### Setup Salesforce configurations
Create a Salesforce account and create a connected app by visiting [Salesforce](https://www.salesforce.com). Obtain the following parameters:

* Base URl (Endpoint)
* Client Id
* Client Secret
* Access Token
* Refresh Token
* Refresh URL

For more information on obtaining OAuth2 credentials, visit [Salesforce help documentation](https://help.salesforce.com/articleView?id=remoteaccess_authenticate_overview.htm).

Set Salesforce credentials in `ballerina.conf` (requested parameters are `SF_URL`, `SF_ACCESS_TOKEN`, `SF_CLIENT_ID`,
`SF_CLIENT_SECRET`, `SF_REFRESH_TOKEN`, and `SF_REFRESH_URL`).

`sms_sender.bal` file shows how to create the Salesforce Client endpoint.

```ballerina
endpoint sf:Client salesforceClient {
    clientConfig:{
        url: ""
        auth:{
            scheme:http:OAUTH2,
            accessToken: "",
            refreshToken: "",
            clientId: "",
            clientSecret: "",
            refreshUrl: ""
        }
    }
};
```

#### Setup Twilio configurations
Create a [Twilio](https://www.twilio.com/) account and obtain the following parameters:

* Account SId
* Auth Token

For more information on obtaining the above parameters, see [Create a Twilio Authy app](https://www.twilio.com/console/authy/applications).

Set Twilio credentials in `ballerina.conf` (required parameters are `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_FROM_MOBILE`, and `TWILIO_MESSAGE`).

The `sms_sender.bal` file shows how to create the Twilio Client endpoint.

```ballerina
endpoint twilio:Client twilioClient {
    accountSId: "",
    authToken: ""
};
```
  
> IMPORTANT: These access tokens and refresh tokens can be used to make API requests on your own account's behalf. Do not share these credentials.

## Developing the Program

You can use SOQL queries to get SObject data. In this example a `SELECT` query is used to get interested Leads' information.

The `getLeadsData()` function takes a query string as the parameter and returns a map that consists of Leads' phone number as the key and name as the value.

```ballerina
function getLeadsData(string leadQuery) returns (map, boolean) {
    log:printDebug("Salesforce Connector -> Getting query results");
    map leadsMap;
    var response = salesforceClient->getQueryResult(leadQuery);
    match response {
        json jsonRes => {
            addRecordsToMap(jsonRes, leadsMap);
            while (jsonRes.nextRecordsUrl != null) {
                log:printDebug("Found new query result set!");
                string nextQueryUrl = jsonRes.nextRecordsUrl.toString();
                response = salesforceClient->getNextQueryResult(nextQueryUrl);
                match response {
                    json jsonNextRes => addRecordsToMap(jsonNextRes, leadsMap);
                    sf:SalesforceConnectorError err => {
                        log:printDebug("Salesforce Connector -> Failed to get leads data");
                        log:printError(err.message);
                        return (leadsMap, false);
                    }
                }
            }
        }
        sf:SalesforceConnectorError err => {
            log:printDebug("Salesforce Connector -> Failed to get leads data");
            log:printError(err.message);
            return (leadsMap, false);
        }
    }
    return (leadsMap, true);
}
```

The `sendTextMessage()` function takes the `from-mobile number`, `to-mobile` number, and the `message` that is sent as parameters and sends the request to the Twilio connector. This is done to send the message to the relevant phone number.

Function returns `true` if message sending gets successful (if it gets SID as return). If the SID is an empty string
or the result is an error, function returns `false`.

```ballerina
function sendTextMessage(string fromMobile, string toMobile, string message) returns boolean {
    var details = twilioClient->sendSms(fromMobile, toMobile, message);
    match details {
        twilio:SmsResponse smsResponse => {
            if (smsResponse.sid != EMPTY_STRING) {
                log:printDebug("Twilio Connector -> SMS successfully sent to " + toMobile);
                return true;
            }
        }
        twilio:TwilioError err => {
            log:printDebug("Twilio Connector -> SMS failed sent to " + toMobile);
            log:printError(err.message);
        }
    }
    return false;
}
```
Inside the `sendSmsToLeads()` function, it takes the Lead's data by calling the `getLeadsData()` function. The result map is iterated and phone numbers that are not null are taken. Customized messages are prepared and sent to relevant Leads' phone numbers. The function returns `true` if at least one message is being sent to a Lead. If not, this returns `false`.

```ballerina
function sendSmsToLeads(string sfQuery) returns boolean {
    (map, boolean) leadsResponse = getLeadsData(sfQuery);
    map leadsDataMap;
    boolean isSuccess;
    (leadsDataMap, isSuccess) = leadsResponse;

    if (isSuccess){
        string messageBody = config:getAsString(TWILIO_MESSAGE);
        string fromMobile = config:getAsString(TWILIO_FROM_MOBILE);
        foreach k, v in leadsDataMap {
            string result = <string>v;
            string message = "Hi " + result + NEW_LINE_CHARACTER + messageBody;
            isSuccess = sendTextMessage(fromMobile, k, message);
        }
    }
    return isSuccess;
}
```

Inside the main function, it calls to `sendSmsToLeads()` by passing the requested query.
Result status can be checked with the `boolean` value.

Inside the main function, it calls the `sendSmsToLeads()` function by passing the requested query. The result status can be checked with the `boolean` value.

```ballerina
function main(string... args) {
    log:printDebug("Salesforce-Twilio Integration -> Sending promotional SMS to leads of Salesforce");
    string sampleQuery = "SELECT Name, Phone, Country FROM Lead WHERE Country = 'LK'";
    boolean result = sendSmsToLeads(sampleQuery);
    if (result) {
        log:printDebug("Salesforce-Twilio Integration -> Promotional SMS sending process successfully completed!");
    } else {
        log:printDebug("Salesforce-Twilio Integration -> Promotional SMS sending process failed!");
    }
}
```

## Testing

You can use `Testerina` to test Ballerina implementations. Run the `sms_sender_test.bal` file using the `ballerina run sms-sender` command to execute the test function.

```ballerina
@test:Config
function testSendSmsToLeads() {
    log:printDebug("Salesforce-Twilio Integration -> Sending promotional SMS to leads of Salesforce");
    string sampleQuery = "SELECT Name, Phone, Country FROM Lead WHERE Country = 'LK'";
    boolean result = sendSmsToLeads(sampleQuery);
    if (result) {
        log:printDebug("Salesforce-Twilio Integration -> Promotional SMS sending process successfully completed!");
    } else {
        log:printDebug("Salesforce-Twilio Integration -> Promotional SMS sending process failed!");
    }
}
```

* You receive an SMS with the relevant numbers as the result.

#### Sample Result SMS
```
Hi Carmen

Enjoy discounts up to 25% by downloading our new Cloud Platform before 31st May'18! T&C Apply.
```
#### Terminal Output 

You receive logs with Twilio SID number as shown below if it is successful. If not successful, you see logs with error messages.

```bash
...
2018-04-12 13:27:13,869 INFO  [src] - Salesforce Connector -> Getting query results... 
2018-04-12 13:27:15,718 INFO  [src] - Twilio Connector => Sending messages... 
2018-04-12 13:27:17,061 INFO  [src] - SM08134284d310461aa7dd4b20d8d2a7b5 
2018-04-12 13:27:17,378 INFO  [src] - SM1f40e267c0c2489a9a8ae2172665647f 
...

```

## Deployment

#### Deploying locally
You can deploy the services that you developed above in your local environment. You can create the Ballerina executable archives (.balx) first and run them in your local environment as follows.

**Building**

```
<SAMPLE_ROOT_DIRECTORY>$ ballerina build salesforce-twilio-integration/
```

After build is successful, there will be a .balx file inside the target directory. That executable can be executed as follows.

**Running**

```
<SAMPLE_ROOT_DIRECTORY>$ ballerina run <Exec_Archive_File_Name>
```
