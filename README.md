# OpenWhisk building block - Message Hub Trigger
Create Message Hub data processing apps with Apache OpenWhisk on IBM Bluemix. This tutorial takes less than 5 minutes to complete. After this, move on to more complex serverless applications such as those tagged [_openwhisk-hands-on-demo_](https://github.com/search?q=topic%3Aopenwhisk-hands-on-demo+org%3AIBM&type=Repositories).

If you're not familiar with the OpenWhisk programming model [try the action, trigger, and rule sample first](https://github.com/IBM/openwhisk-action-trigger-rule). [You'll need a Bluemix account and the latest OpenWhisk command line tool](https://github.com/IBM/openwhisk-action-trigger-rule/blob/master/docs/OPENWHISK.md).

This example shows how to create an action that consumes Message Hub (Apache Kafka) messages (records).

1. [Configure Message Hub](#1-configure-message-hub)
2. [Create OpenWhisk actions](#2-create-openwhisk-actions)
3. [Clean up](#3-clean-up)

# 1. Configure Message Hub
Log into Bluemix, provision a [Message Hub](https://console.ng.bluemix.net/catalog/services/message-hub) instance, and name it `kafka-broker`. On the "Manage" tab of the Message Hub console create a topic named "cats-topic". Extract the API key and the REST URL endpoint from the "Service Credentials" tab in Bluemix and use them in place of the `API_KEY` and `KAFKA_REST_URL` variables below.

In this demo, we will make use of the Kafka package, which contains a set of OpenWhisk actions and feeds that integrate with Kafka. Use the OpenWhisk CLI to bind the Kafka package using your credentials. Binding a package allows you to set the default parameters that are inherited by every action and feed in the package.

```bash
# Bind Message Hub (Kafka) service as a package in OpenWhisk
wsk package refresh # This may show an error, it's safe to ignore
wsk package create kafka
wsk package bind kafka kafka-binding \
  --param api_key ${API_KEY} \
  --param kafka_rest_url ${KAFKA_REST_URL} \
  --param topic cats-topic
```

Triggers are a named channel for a class of event and can be explicitly fired by a user or fired on behalf of a user by an external event source, such as a feed. Use the code below to create a trigger to fire events when messages are received using the "messageHubFeed" feed provided in the Kafka package we just bound.

```bash
# Create trigger to fire events when messages (records) are received
wsk trigger create message-received-trigger \
   --feed /_/Bluemix_kafka-broker_Credentials-1/messageHubFeed \
   --param isJSONData true \
   --param topic cats-topic
```

# 2. Create OpenWhisk actions
Create a file named `process-message.js`. This file will define an OpenWhisk action written as a JavaScript function. This function will print out messages that are received on the Kafka queue. For this example, we are expecting messages that contain a `cat` with fields `name` and `color`.
```javascript
function main(params) {

  console.log(params);

  return new Promise(function(resolve, reject) {
    if (!params.messages || !params.messages[0] || !params.messages[0].value) {
      reject("Invalid arguments. Must include 'messages' JSON array with 'value' field");
    }
    var msgs = params.messages;
    var cats = [];
    for (var i = 0; i < msgs.length; i++) {
      var msg = msgs[i];
      for (var j = 0; j < msg.value.cats.length; j++) {
        var cat = msg.value.cats[j];
        console.log('A ' + cat.color + ' cat named ' + cat.name + ' was received.');
        cats.push(cat);
      }
    }
    resolve({
      "cats": cats
    });
  });

}
```

## Create action and map to trigger

Create an OpenWhisk action from the JavaScript function that we just created.
```bash
wsk action create process-message process-message.js
```

OpenWhisk actions are stateless code snippets that can be invoked explicitly or in response to an event. In this demo, we are going to configure this action to be invoked in response to events fired by the `message-received-trigger`.

To do this, we are going to create a rule, which maps triggers with actions. Once this rule is created, the `process-message` action will be executed whenever the `message-received-trigger` is fired in response to new messages being written to the Kafka queue.

```bash
wsk rule create log-message-rule message-received-trigger process-message
```


## Enter data to fire a change
Begin streaming the OpenWhisk activation log.
```bash
wsk activation poll
```

In another terminal window, send a message to Kafka using its REST API.
```bash
echo '{"cats": [{"name": "Tahoma", "color": "Tabby"}, {"name": "Tarball", "color": "Black"}]}' > cats.json
PAYLOAD=$( base64 cats.json )
DATA='{"records":[{"value":"'${PAYLOAD}'"}]}'
curl -X POST -H "Content-Type: application/vnd.kafka.binary.v1+json" \
  -H "X-Auth-Token: $API_KEY" \
  --data "$DATA" \
  "$KAFKA_REST_URL/topics/cats-topic"
```

View the OpenWhisk log to look for the change notification.

# 3. Clean up
## Remove the rule, trigger, action, and package

```bash
# Remove rule
wsk rule disable log-message-rule
wsk rule delete log-message-rule

# Remove trigger
wsk trigger delete message-received-trigger

# Remove actions
wsk action delete process-message

# Remove package
wsk package delete kafka-binding
```

# Troubleshooting
Check for errors first in the OpenWhisk activation log. Tail the log on the command line with `wsk activation poll` or drill into details visually with the [monitoring console on Bluemix](https://console.ng.bluemix.net/openwhisk/dashboard).

If the error is not immediately obvious, make sure you have the [latest version of the `wsk` CLI installed](https://console.ng.bluemix.net/openwhisk/learn/cli). If it's older than a few weeks, download an update.
```bash
wsk property get --cliversion
```

# License
[Apache 2.0](LICENSE.txt)
