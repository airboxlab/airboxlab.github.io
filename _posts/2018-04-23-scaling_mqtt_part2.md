---
layout: post
title:  "Scaling MQTT connections with RabbitMQ - Part II"
date:   2018-04-23 08:00:00
categories: "iot mqtt scalability rabbitmq haproxy"
comments: true
author: Antoine Galataud
published: true
---

<style type="text/css">
.callout {
    float: right;
    margin-left: 5px;
}
</style>

![firehose]({{ site.baseurl }}/assets/scale_mqtt_2/drink-out-of-a-hose.jpg){: .callout}

We’ve been running our messaging gateways with a more distributed approach as described in [](part I) for more than a year now, and wanted to share problems we faced and solutions we found.

First of all, let’s face it: it has been a shaky road. After an initial period of a few weeks where everything seemed to run smoothly under control, new incidents and outages started to show up intermittently. This was due to several problems I describe here.
<br/><br/>

#### AWS EC2 instances: choosing the right type

When we deployed our new version in May 2017, we chose cheap instance types like _t2_ for our proxies, with the idea of scaling them in mind. 

It revealed to be a bad choice: _t2_ network interfaces are flaky and we suffered several hiccups, with massive reconnection events as a result. They are also not well suited for medium to long burst periods, as cpu access is throttled after a few 10a of minutes running above credit line. And we needed that to deal with recovery during massive reconnection events.

Finally a bunch of weak instances compared to a fewer but stronger ones is strategically a non sense here: more maintenance, less predictability.

Our current gateways now use _m4_ instance family. Since that, network hiccups have disappeared.
<br/><br/>

#### HAProxy timeouts

At first we've let default or too high values for some we didn't know about. Problem with persistent MQTT connections and nextwork connections from resource-constrained hardware is that you may endup with a lot of idle, half-opened connections that could pile up and prevent legitimate connections to open.

Hopefull for us, HAProxy has a very flexible configuration. And an extensive documentation. It's possible to configure client, server and other type of timeouts. Here is an extract of one of our configs:

```cfg
defaults
    mode    tcp
    option  abortonclose
    timeout client      30s
    timeout client-fin  15s
    timeout connect     5s
    timeout server      30s
    timeout server-fin  15s
    timeout queue       30s
    timeout tunnel      30s
```

Some values may seem a bit aggressive but since MQTT clients are all configured to automatically reconnect, it's fine. 

One important thing to note is HAProxy _timeout tunnel_ supersedes _timeout client_ and _timeout server_, for persistent connections it's an important one to set. Also HAProxy doc recommends using _timeout client-fin_ and _timeout server-fin_ in conjunction, to close sockets in _FIN\_WAIT_ state faster. 

Finally, in order to maintain connections open, MQTT clients are using heartbeat messages with interval < 30s.
<br/><br/>

#### MQTT clients tuning

**Clean session & QoS 1**

_clean session_ is a MQTT flag that instructs server it can remove everything belonging to the client and connection is closed. We've used it to ensure we don't end up in situations where a RabbitMQ cluster node would refuse a connection because a MQTT queue was still present on an other node (we finally gave up using RabbitMQ clustering, but we may come back to it one day).

_QoS 1_ is the intermediate level of guarantee for MQTT messages delivery. It ensures messages are delivered *at least once* to the receiver (hence doesn't guarantee the same message isn't received several times).

Also, _TTL_ on messages is fairly low (never more than 30s) so they are lost when they aren't consumed fast enough. This to prevent filling up queues on server side.

In order to cope with this client & server technical constraints, we also added a couple of application-level safeguards:

- important messages are resent to receivers upon connection. Even in case of short disconnection
- idempotency and statelessness are enforced: state is preserved in the backend, which decides when and how to send messages. MQTT clients are designed to support receiving same message several times. RabbitMQ only passes messages, doesn't store them.

**Exponential backoff**

MQTT clients may disconnect for different reasons: flky network quality, server-side closed, you name it. MQTT library we wrote is made to take care of reconnecting automatically. In order to avoid self-inflicted DDoS, clients will retry indefinitely but with an exponentially-increasing delay between each attempt every time. 

As we were testing this, we noticed that this principle must be extended up to the reception of the first MQTT heartbeat answer from server: it's only at that moment we know server is fully operational.
<br/><br/>

#### RabbitMQ tuning

**no more clustering**

RabbitMQ clustering let's you form clusters of nodes with the benefit of possible fail-over in case of node failure, or queues high availability. However it's not well suited for load-balancing tens of thousands of connections.

First of all, communication overhead between nodes is significant and requires to keep nodes in the network to prevent latency. Second, recovery and upgrade are far more complex than with a collection of single nodes. 
Last but not least, RabbitMQ suffers a bug that's not directly linked to clustering, but gets worse because of it: in case of node shutdown or massive reconnection, thousands of auto-delete queues get evicted. This triggers a Mnesia (distributed DB RabbitMQ is using) lock contention and it can get up to several minutes for the node to recover from that. See [this dicussion on RabbitMQ users list](https://groups.google.com/d/msg/rabbitmq-users/hgRmhpL8Y6o/F7UoGyDsCwAJ) and related [bug opened](https://github.com/rabbitmq/rabbitmq-server/issues/1566) (not fixed in 3.6 yet).
<br/><br/>

#### OS tuning: sysctl, TCP stack & network interfaces

Our current architecture pairs several RabbitMQ nodes with 1 HAProxy instance. HAProxy resources consumption is pretty low for that type of workload so it's possible to achieve hundreds of thousands of connections with a single HAProxy instance.

In order to that, there's a major OS constraint that limits number of outbound connections for a specific IP. This is first controlled by `net.ipv4.ip_local_port_range` sysctl option (very low by default on most distributions). But the maximum is _65535_

There are several ways to go beyond, we chose to use AWS ENIs (Elastic Network Interface): our instance setup script allocates 4 ENIs then configures each network interface. Finally, it can be used n HAProxy configuration as following:

```
backend tcp-out-1883
    default-server inter 30s rise 2 fall 2
    timeout check 5s
    option tcp-check
    server server1-eni1 10.0.2.1:1883 source 10.0.1.1 check on-marked-down shutdown-sessions
    server server1-eni2 10.0.2.1:1883 source 10.0.1.2 check on-marked-down shutdown-sessions
    server server1-eni3 10.0.2.1:1883 source 10.0.1.3 check on-marked-down shutdown-sessions
    server server1-eni4 10.0.2.1:1883 source 10.0.1.4 check on-marked-down shutdown-sessions
    server server2-eni1 10.0.2.2:1883 source 10.0.1.1 check on-marked-down shutdown-sessions
    server server2-eni2 10.0.2.2:1883 source 10.0.1.2 check on-marked-down shutdown-sessions
    server server2-eni3 10.0.2.2:1883 source 10.0.1.3 check on-marked-down shutdown-sessions
    server server2-eni4 10.0.2.2:1883 source 10.0.1.4 check on-marked-down shutdown-sessions
```

where _10.0.2.X_ are RabbitMQ nodes and _10.0.1.X_ are HAProxy network interfaces.

There are also a couple other sysctl options that can be tuned. [RabbitMQ documentation](https://www.rabbitmq.com/networking.html) gives some recommendations.
<br/><br/>

#### Testing: a load testing framework

How can one validate settings at various levels without testing? For this purpose we created a small test framework that interacts with AWS EC2 and spawns desired number of instances, and execute several thousands of MQTT clients on each of them. MQTT client is using [Eclipse Paho](https://www.eclipse.org/paho/) library and is configurable (number of clients, ramp up period, interval between messages, QoS level, clean session).

This helped us validating client, server, OS, and software configurations, as well as finding the right amout of CPU/RAM required for the load.
<br/><br/>

#### Automating operations

It sounds obvious, but we didn't have an end-to-end automated way of recreating our messaging gateways infrastructure. We know have ~90% scripted.

It's of great help during testing phase, as we were able to simply trash a whole set of instances and configurations and restart from scratch. Now we use it for scaling operations.
<br/><br/>

#### High Availability with AWS Route53

This was one of our goals in first article I wrote: we were first relying on clustering for HA, but it was only able to deal with AWS Availability Zone failures. 

We implemented HA at a higher level, without RabbitMQ clustering. It's using AWS Route53 and a combination of geo-location, weight and healtcheck rules to decide where to send clients. This offers both intra and cross region failover.

The whole thing is well described in [this document](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-complex-configs.html)
<br/><br/>

## Final notes

We incrementally released this new version of our messaging infrastructure in December 2017 and January 2018. It proved to be a good move as we significantly recuded number of incidents and maintenance time, while improving scalability. There's still work to do though! 

Our next goals are about finalizing automation of gateway provisioning and setup. With that, we will be able to implement auto scaling: our workload doesn't vary much, but in case of instance failure, automatically spawning and configuring a new can be a major improvement.

