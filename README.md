# mqtt_client supporting flutter
[![Build Status](https://travis-ci.org/shamblett/mqtt_client.svg?branch=master)](https://travis-ci.org/shamblett/mqtt_client)

A server side MQTT client for Dart.

The client is an MQTT v3(3.1 and 3.1.1) implementation supporting subscription/publishing at all QOS levels,
keep alive and synchronous connection. The client is designed to take as much MQTT protocol work
off the user as possible, connection protocol is handled automatically as are the message exchanges needed
to support the different QOS levels and the keep alive mechanism. This allows the user to concentrate on
publishing/subscribing and not the details of MQTT itself.

An example of usage can be found in the examples directory, this example is runnable.  An example is also provided
showing how to use the client to connect to the mqtt-bridge of Google's IoT-Core suite. This demonstrates
how to use secure connections and switch MQTT protocols. The test directory also contains standalone runnable scripts for subscription and publishing.

The client supports both normal and secure TCP connections and server side websocket connections.

The code is a port from the C# [nMQTT](https://www.openhub.net/p/nMQTT) client library to Dart.

use in pubspec.yaml:
```
mqtt_client:
  git:
    url: https://github.com/benbehringer/mqtt_client.git
```

when using the example, remember to add uuid as a dependency (in pubspec.yaml):
```
uuid: "^1.0.1"
```

works with:
```
Flutter 0.5.6 • channel dev • https://github.com/flutter/flutter.git
Framework • revision 472bbccf75 (2 weeks ago) • 2018-06-26 17:01:46 -0700
Engine • revision 6fe748490d
Tools • Dart 2.0.0-dev.63.0.flutter-4c9689c1d2
```

main.dart

```
import 'package:flutter/material.dart';
import 'package:uuid/uuid.dart';
import 'package:mqtt_client/mqtt_client.dart' as mqtt;
import 'package:observable/observable.dart' as observe;

void main() => runApp(new MyApp());

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return new MaterialApp(
      title: 'MQTT Demo',
      theme: new ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: new MyHomePage(title: 'MQTT Demo'),);
  }
}

class MyHomePage extends StatefulWidget {
  MyHomePage({Key key, this.title}) : super(key: key);
  final String title;

  @override
  _MyHomePageState createState() => new _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  final String broker = "iot.eclipse.org";
  final String topic = "mqtt/flutter/test";
  final String id = new Uuid().v1().toString().substring(0,10);
  String _mqttMessage = " ";

  void mqttTest() async {
    mqtt.MqttClient client = new mqtt.MqttClient(broker, id);
    client.logging(false);
    client.keepAlivePeriod = 30;
    // Add the unsolicited disconnection callback
    client.onDisconnected = () => print("Disconnected!!!!");

    final mqtt.MqttConnectMessage connMess = new mqtt.MqttConnectMessage()
        .withClientIdentifier(id)
        .keepAliveFor(30);

    client.connectionMessage = connMess;

    print("Trying to connect Mosquitto client...");

    try {
      await client.connect();
    } catch (Exception) {
      print("could not connect");
      return;
    }

    if (client.connectionState == mqtt.ConnectionState.connected) {
      print("Mosquitto client connected");
    } else {
      print(
          "ERROR Mosquitto client connection failed - disconnecting, state is ${client
              .connectionState}");
      return;
    }

    final observe.ChangeNotifier<observe.ChangeRecord> cn = client.subscribe(topic, mqtt.MqttQos.exactlyOnce).observable;
    cn.changes.listen((List<observe.ChangeRecord> c) {
      mqtt.MqttReceivedMessage myMessage = c[0] as mqtt.MqttReceivedMessage;
      final mqtt.MqttPublishMessage recMess = myMessage.payload as mqtt.MqttPublishMessage;
      final String pt =
      mqtt.MqttPublishPayload.bytesToStringAsString(recMess.payload.message);
      print("EXAMPLE::Change notification:: payload is <$pt> for topic <$topic>");
      setState(() {
        _mqttMessage = pt;
      });
    });


    // Use the payload builder rather than a raw buffer
    final mqtt.MqttClientPayloadBuilder builder =
    new mqtt.MqttClientPayloadBuilder();
    builder.addString("This is the message our subscriber should receive once.");
    client.publishMessage(topic, mqtt.MqttQos.exactlyOnce, builder.payload);
  }


  @override
  void initState() {
    mqttTest();
  }

  @override
  Widget build(BuildContext context) {
    return new Scaffold(
        appBar: new AppBar(
          title: new Text(widget.title),
        ),
        body:
           new Column(
             mainAxisAlignment: MainAxisAlignment.start,
             crossAxisAlignment: CrossAxisAlignment.start,
             children: <Widget>[
              new Text("Broker: $broker", style: new TextStyle(fontSize: 11.0)),
              new Text("Topic: $topic", style: new TextStyle(fontSize: 11.0)),
              new Text(" "),
              new Text("Publish: mosquitto_pub -h iot.eclipse.org -t mqtt/flutter/test -m \"publish this\"", style: new TextStyle(fontSize: 11.0)),
              new Text(" "),
              new Text("Received message:", style: new TextStyle(fontSize: 11.0)),
              new Text(_mqttMessage, style: new TextStyle(fontWeight: FontWeight.bold)),
            ],)
        );
  }
}
```

test:
```
mosquitto_pub -h iot.eclipse.org -t mqtt/flutter/test -m "This is my message"
```

