# OpenWhisk building block - Message Hub Trigger
Create Message Hub data processing apps with Apache OpenWhisk on IBM Bluemix. This tutorial takes less than 5 minutes to complete. After this, move on to more complex serverless applications such as those tagged [_openwhisk-hands-on-demo_](https://github.com/search?q=topic%3Aopenwhisk-hands-on-demo+org%3AIBM&type=Repositories).

![Sample Architecture](https://openwhisk-ui-prod.cdn.us-south.s-bluemix.net/openwhisk/ngow-public/img/getting-started-messagehub.svg)

If you're not familiar with the OpenWhisk programming model [try the action, trigger, and rule sample first](https://github.com/IBM/openwhisk-action-trigger-rule). [You'll need a Bluemix account and the latest OpenWhisk command line tool](https://github.com/IBM/openwhisk-action-trigger-rule/blob/master/docs/OPENWHISK.md).

This example shows how to create an action that consumes Message Hub (Apache Kafka) messages (records).

1. [Configure Message Hub](#1-configure-message-hub)
2. [Create OpenWhisk actions](#2-create-openwhisk-actions)
3. [Clean up](#3-clean-up)

# 1. Configure Message Hub
Log into Bluemix, provision a [Message Hub](https://console.ng.bluemix.net/catalog/services/message-hub) instance, and name it `openwhisk-kafka`. On the "Manage" tab of the Message Hub console create a topic named "cats-topic". Set the corresponding names as environment variables in a terminal window:

```bash
export KAFKA_INSTANCE="openwhisk-kafka"
export KAFKA_TOPIC="cats-topic"
```

We will make use of the built-in [OpenWhisk Kafka package](https://github.com/apache/incubator-openwhisk-package-kafka#producing-messages-to-message-hub), which contains a set of actions and feeds that integrate with both Apache Kafka and IBM Message Hub (based on Kafka).

On Bluemix, this package can be automatically configured with the credentials and connection information from the Message Hub instance we provisioned above. We make it available by refreshing our list of packages.

```bash
# Ensures the IBM Message Hub credentials are available to OpenWhisk.
wsk package refresh
```

Triggers can be explicitly fired by a user or fired on behalf of a user by an external event source, such as a feed. Use the code below to create a trigger to fire events when messages are received using the "messageHubFeed" provided in the Message Hub package.

```bash
# Create trigger to fire events when messages (records) are received
wsk trigger create message-received-trigger \
  --feed Bluemix_${KAFKA_INSTANCE}_Credentials-1/messageHubFeed \
  --param isJSONData true \
  --param topic "$KAFKA_TOPIC"
```

# 2. Create OpenWhisk actions
Create a file named `process-message.js`. This file will define an OpenWhisk action written as a JavaScript function. This function will print out messages that are received from Kafka. For this example, we are expecting a stream of messages that contain a `cat` object with `name` and `color` fields.

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

To do this, we are going to create a rule, which maps triggers to actions. Once this rule is created, the `process-message` action will be executed whenever the `message-received-trigger` is fired in response to new messages being written to the Kafka stream.

```bash
wsk rule create log-message-rule message-received-trigger process-message
```

## Enter data to fire a change
Begin streaming the OpenWhisk activation log in a second terminal window.
```bash
wsk activation poll
```

Now send a message to Message Hub using the message producer action back in the original window.
```bash
echo '{"cats": [{"name": "Tahoma", "color": "Tabby"}, {"name": "Tarball", "color": "Black"}] }' > records.json
DATA=$( base64 records.json )

wsk action invoke Bluemix_${KAFKA_INSTANCE}_Credentials-1/messageHubProduce \
  --param topic $KAFKA_TOPIC \
  --param value "$DATA" \
  --param base64DecodeValue true
```

View the OpenWhisk log to look for the change notification. You should see activation records for the producing action, the rule, the trigger, and the consuming action.

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
```

# Troubleshooting
Check for errors first in the OpenWhisk activation log. Tail the log on the command line with `wsk activation poll` or drill into details visually with the [monitoring console on Bluemix](https://console.ng.bluemix.net/openwhisk/dashboard).

If the error is not immediately obvious, make sure you have the [latest version of the `wsk` CLI installed](https://console.ng.bluemix.net/openwhisk/learn/cli). If it's older than a few weeks, download an update.
```bash
wsk property get --cliversion
```

# License
[Apache 2.0](LICENSE.txt)
