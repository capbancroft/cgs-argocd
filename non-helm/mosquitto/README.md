# Dynamic Security

As part of our work publishing argo-workflow notifications to front end clients, we need to allow access to the MQTT broker from inside the cluster. The default behavior for a broker configured to operate without credentials is to listen on localhost (127.0.0.1). This necessitates configuring some for of security.  There are several options available as documented in the notes on [dynamic-security](https://mosquitto.org/documentation/dynamic-security/).

## deployment session

To configure the broker for dynamic security, we need to add two *plugin* lines to the configuration. The first is to specify the dynamic link library for the plugin, and the second of which is the location for a *dynamic-security.json* . . . 

Given the layout already used in the container image, this is simply the following.

```
    # dynamic authentication
    plugin /usr/lib/mosquitto_dynamic_security.so
    plugin /mosquitto/config/dynamic-security.json
```

We are able to generate an initial file using the following command.

```
mosquitto_ctrl dynsec init /mosquito/dynamic-security.json admin-user
```

The operation of this command asks to set the credentials, which were generated by the following command.

```
echo mosquitto | md5sum | cut -f 1 -d ' '
```

The resulting initial configuration file was captured here as `dynamic-security.json` . . . 

## security configuration

The default configuration for access control list behavior is [documented](https://mosquitto.org/documentation/dynamic-security/) to be the following.

* publishClientSend: deny
* publishClientReceive: allow
* subscribe: deny
* unsubscribe: allow

### Broker side

These settings for these defaults can be seen (verified) using *mosquitto_ctrl* 
```
mosquitto_ctrl -u admin-user dynsec getDefaultACLAccess 
```
The defaults for those two access controls which deny actions can be changed as follows.  
```
mosquitto_ctrl -u admin-user dynsec setDefaultACLAccess publishClientSend allow
mosquitto_ctrl -u admin-user dynsec setDefaultACLAccess subscribe allow
```

## client side

We have a small deployment manifest *inflate-mqtt.yaml* which can be applied to the cluster into the default namespace, which will allow us to test naming across namespaces.
```
kubectl apply -f inflate-mqtt.yaml
```
Note, one must be in the *kubernetes/base/karpenter* directory to run the prior command.

from that pod, I can publish messages.
```
/ # mosquitto_pub -h mosquitto-mqtts.bda.svc.cluster.local -t worlds -m "hello"
/ # mosquitto_pub -h mosquitto-mqtts.bda -t worlds -m "there"
```

## subscriber running on broker side

Subscribing to the *worlds* topic gets the messages.

```
$ mosquitto_sub -h 172.20.175.242 -t worlds
hello
there
```

## broker logs

Smoke test / sanity check can be supported by checking the logs.

```
% kubectl logs -n bda mosquitto-846cf97cd6-bx6zr 
1711557845: mosquitto version 2.0.18 starting
1711557845: Config loaded from /mosquitto/config/mosquitto.conf.
1711557845: Loading plugin: /usr/lib/mosquitto_dynamic_security.so
1711557845: Opening ipv4 listen socket on port 1883.
1711557845: Opening ipv6 listen socket on port 1883.
1711557845: Opening websockets listen socket on port 9001.
1711557845: mosquitto version 2.0.18 running
1711558164: New connection from 10.144.134.202:54572 on port 1883.
```

# TODO: automate configuration of default control settings !