# AWS Lambda function to send GuardDuty alerts through Slack

## Overview

This AWS Lambda function is designed to integrate with Amazon GuardDuty, a threat detection service, and send alerts to a Slack channel when security events are detected. The Lambda function processes GuardDuty events, evaluates their severity, and formats the information into a Slack message.

## Function Components

### 1. Importing Modules

The function uses the following Node.js modules:

```javascript
const AWS = require('aws-sdk');
const url = require('url');
const https = require('https');
```

These modules are necessary for making HTTPS requests and handling URLs.

### 2. Configuration Variables

The function relies on the following environment variables:

- `webHookUrl`: Slack webhook URL for sending messages.
- `slackChannel`: Slack channel where alerts will be posted.
- `minSeverityLevel`: Minimum severity level for sending alerts.

### 3. `postMessage` Function

This function sends a JSON message to the configured Slack webhook URL.

```javascript
function postMessage(message, callback) {
    const body = JSON.stringify(message);
    const options = url.parse(webHookUrl);
    options.method = 'POST';
    options.headers = {
        'Content-Type': 'application/json',
        'Content-Length': Buffer.byteLength(body),
    };

    const postReq = https.request(options, (res) => {
        const chunks = [];
        res.setEncoding('utf8');
        res.on('data', (chunk) => chunks.push(chunk));
        res.on('end', () => {
            if (callback) {
                callback({
                    body: chunks.join(''),
                    statusCode: res.statusCode,
                    statusMessage: res.statusMessage,
                });
            }
        });
    });

    postReq.write(body);
    postReq.end();
}
```

### 4. `processEvent` Function

This is the main function that processes GuardDuty events and constructs Slack messages.

```javascript
function processEvent(event, callback) {
    const message = event;
    const consoleUrl = `https://console.aws.amazon.com/guardduty`;
    const finding = message.detail.type;
    
    //...
}
```

#### 4.1. Skipping Unwanted Alerts

The function checks if the GuardDuty event type is 'Recon:EC2/PortProbeUnprotectedPort' and skips the alert if true.

```javascript
if (finding === 'Recon:EC2/PortProbeUnprotectedPort') {
        console.info('Skipping alert for Recon:EC2/PortProbeUnprotectedPort');
        return callback(null);
    }
```

#### 4.2. Extracting Event Details

Extracts relevant details from the GuardDuty event, such as type, description, time, account ID, region, etc.

```javascript
const findingDescription = message.detail.description;
    const findingTime = message.detail.updatedAt;
    const findingTimeEpoch = Math.floor(new Date(findingTime) / 1000);
    const account = message.detail.accountId;
    const region = message.region;
    const messageId = message.detail.id;
    const lastSeen = `<!date^${findingTimeEpoch}^{date} at {time} | ${findingTime}>`;
```

#### 4.3. Determining Severity and Color

Evaluates the severity of the event and assigns a corresponding color for the Slack message.

```javascript
var color = '#7CD197';
var severity = '';
var skip = false;

if (message.detail.severity < 4.0) {
        if (minSeverityLevel !== 'LOW') {
            skip = true;
        }
        severity = 'Low';
        color = '#e2d43b';
    } else if (message.detail.severity < 7.0) {
        if (minSeverityLevel !== 'LOW' && minSeverityLevel !== 'MEDIUM') {
            skip = true;
        }
        severity = 'Medium';
        color = '#ff8c00';
    } else if (message.detail.severity >= 7.0) {
        severity = 'High';
        color = '#ad0614';
    } else {
        skip = true;
    }
```

#### 4.4. Constructing Slack Message Attachment

Builds a structured attachment for the Slack message containing information about the event.

```javascript
const attachment = [{
        "fallback": finding + ` - ${consoleUrl}/home?region=` +
            `${region}#/findings?search=id%3D${messageId}`,
        "pretext": `*Finding in ${region} for Acct: ${account}`,
        "title": `${finding}`,
        "title_link": `${consoleUrl}/home?region=${region}#/findings?search=id%3D${messageId}`,
        "text": `${findingDescription}`,
        "fields": [
            {"title": "Severity","value": `${severity}`, "short": true},
            {"title": "Region","value": `${region}`,"short": true},
            {"title": "Last Seen","value": `${lastSeen}`, "short": true}
        ],
        "mrkdwn_in": ["pretext"],
        "color": color
    }];
```

#### 4.5. Sending Slack Message

Sends the constructed Slack message if it meets the severity criteria.

```javascript
const slackMessage = {
        channel: slackChannel,
        text : '',
        attachments : attachment,
        username: 'GuardDuty',
        'mrkdwn': true,
        icon_url: 'https://raw.githubusercontent.com/aws-samples/amazon-guardduty-to-slack/master/images/gd_logo.png'
    };

    if (!skip) {
        postMessage(slackMessage, (response) => {
            if (response.statusCode < 400) {
                console.info('Message posted successfully');
                callback(null);
            } else if (response.statusCode < 500) {
                console.error(`Error posting message to Slack API: ${response.statusCode} - ${response.statusMessage}`);
                callback(null);
            } else {
                callback(`Server error when processing message: ${response.statusCode} - ${response.statusMessage}`);
            }
        });
    }
```

### 5. AWS Lambda Handler

The AWS Lambda handler invokes the `processEvent` function when triggered.

```javascript
exports.handler = (event, context, callback) => {
    processEvent(event, callback);
};
```

## Usage

1. **Environment Variables**: Set the required environment variables (`webHookUrl`, `slackChannel`, `minSeverityLevel`) in the AWS Lambda console.

2. **Integration with GuardDuty**: Configure this Lambda function as an event handler for GuardDuty findings in the AWS Lambda console.

3. **Slack Integration**: Ensure that the Slack channel and webhook URL are correctly configured for receiving alerts.

## Conclusion

This Lambda function enhances security monitoring by providing real-time alerts for GuardDuty findings in a Slack channel, allowing for timely response to potential security threats.