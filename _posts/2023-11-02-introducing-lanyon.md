---
layout: post
title: A Basic Mosquitto Pipeline
---

*WARNING: the entirity of this implementation is for development purposes only and has not been tested for production systems. it is purely meant as a guideline to document some basic research findings and should NOT be used as-is for any real-world applications.*
<br/><br/>
## Motivation


This is a basic implementation of a Mosquitto based MQTT pipeline. MQTT is a lightweight M2M protocol designed with IoT applications in mind. For example, if we want to read the state of an assembly line and trigger changes to the system, we might use an MQTT based pipeline to do this.

What can we do with an MQTT based broker/client approach:

1. scalability: we might have several thousand sensors and actuators across our line that need to communicate with our central server cluster
2. quality of service: depending on importance, we want the flexibility of a fire-and-forget, an atleast-once or even an exactly-once guarantee of message delivery
3. persistent connections: edge devices might intermittently lose connection - we need sessions to persist between client and broker if the connection fails
4. publish-subscribe contract: 
    - trigger actions often take time to implement - unlike a RESTful call, we want to decouple actions from requests and allow them to execute asynchronously
    - sometimes, multiple clients/machines care about an upstream message - all of these should be able to subscribe to messages/topics that they care about (many-to-many) 
5. last-will-and-testament: in the event of an edge device failure, we want the network to allow this client to gracefully disconnect from the pipeline
6. heartbeat: in general, we want a low-power/bandwidth approach to evaluating the state of the entire network in near real-time through the broker

For more information, check out HiveMQ's [great writeup on what MQTT is and its benefits](https://www.hivemq.com/blog/).

## Code

This code went through cursory testing on a RPi 4 (SO DONT RELY ON IT IN A PRODUCTION ENVIRONMENT)
It builds on top of the Python Paho MQTT Client (on Python 3)

We want each client to be able to perform some basic tasks:
    - securely connect to a remote broker (TLSv1.2)
    - perform some desired action for callbacks: `on_connect`, `on_disconnect`
    - provide an API to publish a message to the broker

- the `Base` class below is responsible for connecting, maintaining and disconnecting a broker connection:

```python
import ssl
import time
import paho.mqtt.client as mqtt
from dotenv import load_dotenv

'''
define broker env vars somewhere safe!
load dotenv code here into variables (marked in CAPS below)
'''

class Base:

    def __init__(self, client_id="test_client", host=HOST, port=PORT):
        self.client_id = client_id
        self.host = host
        self.port = port

    def init_client(self):
        client = mqtt.Client(self.client_id, self.host, self.port)

        # bind callbacks
        client.on_connect       = self.on_connect
        client.on_disconnect    = self.on_disconnect
        client.on_log           = self.on_log

        self.client = client
        
    def connect(self):
        if (self.host == HOST):
            self.client.tls_set(
                ca_certs=ROOT_CERT, 
                tls_version=ssl.PROTOCOL_TLSv1_2
            )

            self.client.username_pw_set(BROKER_USER, BROKER_PW)

        self.client.connect(self.host, self.port, 60)
        self.client.loop_start()

    def disconnect(self):
        self.client.disconnect()

    def current_time(self):
        return time.strftime('%H:%M:%S', time.localtime())

    ########## PUBLISHER METHODS #############

    def publish(self, topic="test", payload=None):
        payload = payload or f"dummy publish at {self.current_time()}"
        self.client.publish(topic, payload)

    ########## CALLBACKS #############

    def on_connect(self, client, userdata, flags, rc):
        # pdb.set_trace()
        if(rc == 0):
            print(f"connected OK: {self.host}:{self.port}")
        else:
            print("bad connection. status: ", rc)

    def on_disconnect(self, client, userdata, rc):
        print(f"rc: {rc}: {self.client_id} is disconnected")
        self.client.loop_stop()

    def on_log(self, client, userdata, level, buf):
        print("log: ", buf)
```

We also want to leverage a parallel process on the Pi to perform a periodic publish
    - in this case, we call this a `Ping` action but keep in mind that the Mosquitto broker will performing heartbeat checks implicitly
    - the ping action is purely done for demonstration here

```python 
import sys
import time
from base import Base

class PingHandler(Base):
    def __init__(self, client_id=None):
        super().__init__(client_id)

        self.init_client()
        self.connect()
        self.ping()
    
    def ping(self):
        while(True):
            self.publish(
                "heartbeat", 
                f"PING: {self.client_id} at {self.current_time()}..."
            )

            time.sleep(2)

if __name__ == "__main__":
    handler = PingHandler(sys.argv[1])
    time.sleep(1)
    handler.disconnect()
```

Finally, we provide the added MQTT client pubsub functionality:
    - provide an API to subscribe to a topic through a function call
    - after subscribing to a topic, we might want a client to trigger a callback upon receiving a specific message over that topic
    - in the `main` function, we are also able to trigger callbacks between different clients by running these scripts on multiple RPis at once

```python
import pdb
import sys
import time
from base import Base

class PubSubHandler(Base):
    def __init__(self, client_id=None):
        super().__init__(client_id)

        self.init_client()
        self.bind_pubsub_callbacks()
        self.connect()
        self.listen()

    def bind_pubsub_callbacks(self):
        super().init_client()
        self.client.on_message = self.on_message
        self.client.message_callback_add(
            self.client_id, 
            self.callback
        )

    def listen(self):
        self.client.subscribe(self.client_id)

    ##### CALLBACKS ######

    def on_subscribe(client, userdata, mid, granted_qos):
        print("subscription successful OK.")

    def on_message(self, client, userdata, message):
        print(
            "catchall: MSG RCVD: " , 
            str(message.payload.decode("utf-8"))
        )

        print("message topic: ", message.topic)
        print("message qos=", message.qos)
        print("message retain flag=", message.retain)

    def callback(self, client, userdata, message):
        # print("specific: message received: " , 
            # str(message.payload.decode("utf-8"))
        # )
        
        # print("message topic: ", message.topic)
        # print("message qos=", message.qos)
        payload = f"CB_RES: {self.client_id} at {self.current_time()}"
        self.publish(f"cb_res", payload)

if __name__ == "__main__":
    handler = PubSubHandler(client_id=sys.argv[1])
    time.sleep(1)

    #### work ####
    for i in range(0,10):
        handler.publish(
            handler.client_id, 
            f"CB_REQ:{handler.client_id} at {handler.current_time()}"
        )

        time.sleep(1)
    #### work ####

    handler.disconnect() # loop_stop executed in the connect method

```

## Standing Up a Secure Mosquitto Broker

I will provide a high level overview of the steps to run a secure instance of the broker through an AWS EC2 instance:

#### Setup the EC2 Instance 

1. follow the usual steps on the AWS documentation to stand up an EC2 instance of AWS
    - expose ports 22, 80, 443, 8883 and 1883 (and custom ICMP to echo request for pinging for debugging purposes)

2. use an existing domain name (or purchase one if you don't already have one) and go through the domain ownership verification steps
3. generate a Route 53 record that maps the domain to the public IP of the EC2 instance for DNS requests
4. you should now be able to SSH into the EC2 instance

#### Setup Mosquitto on EC2 Instance

1. setup a firewall to expose the MQTT unsecured port (1883) and HTTP (using ufw to allow 1883, 8883 and http)
2. install mosquitto and generate TLS certs (use certbot)
3. assign the requisite permissions ot the user to access the certs
4. configure user/password authentication for the broker
5. configure mosquitto to use TLS through our new certificates
6. you can now ping the broker locally and remotely

## Thanks for reading!
