## 

# Labo 03 - Load balancing

# Authors : Adrien Barth, Jeremy Zerbib

## Introduction

The goal of this lab is to be familiar with a load-balancing tool like `HAProxy`. 

We did all the asked tasks in this lab, going from installing the tools to tweaking the configuration files and playing with them.

The goals in details were : 

- Deploy a web application in a two-tier architecture for scalability
- Configure a load balancer
- Performance-test a load-balanced web application

## Task 1: Install the tools

**Explain how the load balancer behaves when you open and refresh the URL <http://192.168.42.42> in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.**

When we first connect to the URL <http://192.168.42.42>, we can see this page appearing : 

![First try](./assets/img/rapport/Part1_Q1_without_refresh.png)

Then, we refresh the page and we see that the page loading is the following : 

![](./assets/img/rapport/Part1_Q1_with_refresh.png)

The load-balancing is of the type *Round-Robin*. Basically, what is happening is that the proxy is redirecting the client once on one server, once on the other. We can see that happening because of the change of the `tag`, the `host`, the `ip` and the `id` fields. 

The basic functionality of a Round-Robin is explained below : 

![http://careerkafe.blogspot.com/2013/11/f5-big-ip-ltm-load-balancing-methos.html](./assets/img/rapport/load_balancing_RR.png) 

The proxy creates a queue with the servers lined up. It goes from one server to another and queues the used server back to its original spot.

**Explain what should be the correct behavior of the load balancer for session management.**

The correct behavior for the session management would be to keep a given client connected to a server. It should be obvious that a client wants to keep his session opened while reloading a page. For example, if a clients aims to keep his number of connection to a server accurate, he might want to be connected every time to the same server.

**Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2. Here is an example:** 

![Sequence diagram for part 1](assets/img/seq-diag-1.png)

As the proxy runs a *Round-Robin* configuration, the communications go like this  : 

![answer](./assets/img/rapport/seq-diag-1.png)

We can see that the balancing operates in a way that during the first request, the client goes to the server *S1* and sets up an ID for each request to the client. The client stores the token ID but the server does not know what to do with it, so it applies its *Round-Robin* queue algorithm. 

Therefore, the client is redirected to the server *S2* and as the server does not know what to do with the ID token, it will set up a new ID.

This mechanism will go on as long as we keep refreshing the page. We can see that ID is never the same. 

**Provide a screenshot of the summary report from JMeter.**

![](./assets/img/rapport/jmeter.png)

**Run the following command:**

```
$ docker stop s1
```

**Clear the results in JMeter and re-run the test plan. Explain what is happening when only one node remains active. Provide another sequence diagram using the same model as the previous one.**

![jmeter_without_s1](./assets/img/rapport/jmeter_without_s1.png)

Basically, what is happening is that the client tries to get to *S1*. As S1 is not available, it goes to *S2* one time out of two. We can see we try to `GET` 2000 times but get a hit only half of those tries.  The ID stays the same throughout the test because the server knows the first ID and therefore no need to create a new one. An user can potentially keep his session alive.

The sequence diagram shows what we is happening.

![sq-2](./assets/img/rapport/seq-diag-2.png)

## Task 2: Sticky sessions

**There is different way to implement the sticky session. One  possibility is to use the SERVERID provided by HAProxy. Another way is  to use the NODESESSID provided by the application. Briefly explain the  difference between both approaches (provide a sequence diagram with  cookies to show the difference).**

- **Choose one of the both stickiness approach for the next tasks.**

The difference between the *SERVERID* and the *NODESESSID* methods lives in the fact that the first is on a proxy side and the latter is on the client's side.  Indeed, the cookie we want to create, is produced by the proxy in the *SERVERID*'s case and on client's side for the *NODESESSID*.

In the first case, the proxy will produce the *ID* and sticks to the packet, called SERVERID, only if the user did not start communicating with such cookie.

In the last case, the proxy will use the cookie that the client produces and sticks a prefix to it, which will be cleaned prior to transmitting the request to the server.

- SERVERID

![seq-3-SERVERID](./assets/img/rapport/seq-diag-3.png)



- NODESESSID

![seq3-NODESESSID](/home/jeremy/Bureau/HEIG-VD/Semestre5/AIT/Labos/labo3/Teaching-HEIGVD-AIT-2019-Labo-Load-Balancing/assets/img/rapport/seq-diag-4.png)

`For the remaining of this task, we will be using the *SERVERID* stickiness method.`

**Provide the modified `haproxy.cfg` file with a short explanation of the modifications you did to enable sticky session management.**

You can see [here](https://www.haproxy.com/blog/load-balancing-affinity-persistence-sticky-sessions-what-you-need-to-know/) how we got the configuration file edited in order to set up the configuration file that produces the cookie server side.  

To summarize everything, here is how we do it. 

We must edit the configuration file located in : `ha/config/haproxy.cfg`. 

After the `backend nodes` label, right after the line `option forwardfor`,  we added this line `cookie SERVERID insert indirect nocache`.

Then, we added at the end of the line `server s1 ${WEBAPP_1_IP}:3000 check`  `cookie s1` and the same for `s2`. 

What it does is basically adding a *cookie* named *SERVERID* if it does not exist. Then while connecting to the server, the load-balancer will check if there is a cookie available for the session.

If there is, then nothing is created and the server follows the session.

If there isn't a cookie setup, then the proxy will create a cookie and create a new session.

```bash
# Global configuration for HAProxy
# http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#3
global
    # Bind UNIX socket to get various stats
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#stats
    stats socket /var/run/haproxy.sock mode 600 level admin

    # Bind TCP port to get various stats
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#stats
    stats socket ipv4@0.0.0.0:9999 level admin

    # Define the timeout on the stats
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#3.1-stats%20timeout
    stats timeout 30s

    # Configure the way the logging is done
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#log
    log 127.0.0.1 local1 notice

# Configure defaults for all the proxies configuration (applied for all the next sections in the configuration)
# http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4
defaults
    # Enable logging
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-log
    log     global

    # The default mode for all the services
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-bind
    mode    http

    # Enable the logging of HTTP requests
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-option%20httplog
    option  httplog

    # Enable the logging of null connections
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-option%20dontlognull
    option  dontlognull

    # Configure the timeout to connect to a server
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-timeout%20connect
    timeout connect 5000

    # Configure the timeout before cutting the connection of a client
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-timeout%20client
    timeout client  50000

    # Same kind of configuration for the servers side
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-timeout%20server
    timeout server  50000

# Open the metrics HAProxy page on the port 1936 on any network interface on the host
# http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4
listen stats *:1936
    # Enable HAProxy to serve stats about himself and the nodes
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-stats%20enable
    stats enable

    # Define the URI to access the stats
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-stats%20uri
    stats uri /

    # Avoid leaking more info than necessary with hiding the version of HAProxy
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-stats%20hide-version
    stats hide-version

# Define the frontend configuration. In fact, that's the part that configure how HAProxy will handle
# the requests from the outside world:
# http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4
frontend localnodes
    # Bind the port 80 to listen incoming outside connections (from the outside world)
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-bind
    bind *:80

    # Define which protocol is enabled on the binded ports.
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-mode
    mode http

    # Use the backend configuration references by the backend name section in this configuration
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-default_backend
    default_backend nodes

# Define the backend configuration. In fact, that's the part that configure what is not directly
# accessible from the outside of the network.
# http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4
backend nodes
    # Define the protocol accepted
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-mode
    mode http

    # Define the way the backend nodes are checked to know if they are alive or down
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-option%20httpchk
    option httpchk HEAD /

    # Define the balancing policy
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#balance
    balance roundrobin

    # Automatically add the X-Forwarded-For header
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-option%20forwardfor
    # https://en.wikipedia.org/wiki/X-Forwarded-For
    option forwardfor

    cookie SERVERID insert indirect nocache

    # With this config, we add the header X-Forwarded-Port
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-http-request
    http-request set-header X-Forwarded-Port %[dst_port]

    # Define the list of nodes to be in the balancing mechanism
    # http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-server
    server s1 ${WEBAPP_1_IP}:3000 check cookie s1
    server s2 ${WEBAPP_2_IP}:3000 check cookie s2

# Other links you will need later for this lab
#
# About cookies: http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#4-cookie
```

**Explain what is the behavior when you open and refresh the URL http://192.168.42.42 in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.**

After editing the configuration file, we can see the expected behavior through those screenshots : 

![payload_received](./assets/img/rapport/payload.png)



![cookie_received_1](./assets/img/rapport/cookie_received_1.png)

We can see here that the first time we sent the request, `sessionview` has a value of 1, that we receive only the `NODESESSID` cookie. 

![cookie_sent](./assets/img/rapport/cookie_sent_1.png)

We can also see that we send the new cookie, `SERVERID` with the request. The value is set to *S2* as we started on the server *S2* at the beginning of our request.

After reloading the page, we get this : 

![payload_after_reload](./assets/img/rapport/payload_reload.png)

The cookie received is the following : 

![cookie_received](./assets/img/rapport/cookie_received_2.png)

We see that the session ID is saved thanks to the `SERVERID`. 

The session can stick if I do not change the user doing the request.

**Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2. We also want to see what is happening when a second browser is used.**

![](./assets/img/rapport/seq-diag-5.png)

For this part we can see the exchange between all the parts of the circuit in the sequence diagram above. 

When a user uses a second browser, what happens is the following : 

- First the user connect to *S1* on the first browser.
- The proxy goes to the first server in the *Round-Robin* queue. (*S1* in our case)
- The server responds with the *NodeSessionID* appropriate for the session.
- The proxy creates a new cookie, *SERVERID*, and sticks it to the session.
- While the user connects from the same browser, the session will stick.
- If the user opens a new browser, or a private navigation window on the same browser, the session will change.

**Provide a screenshot of JMeter's summary report. Is there a difference with this run and the run of Task 1?**

![](./assets/img/rapport/Task2_JMETER.png)

There is a difference between this situation and the one in *Task 1*.  It lies in the fact that only on server is reached in this situation because of the cookie we set. The proxy makes sure that the session sticks throughout the thread's life. That's the reason why we never reach the *S1* server.

- **Clear the results in JMeter.**
- **Now, update the JMeter script. Go in the HTTP Cookie Manager and ~~uncheck~~verify that the box `Clear cookies each iteration?` is unchecked.**
- **Go in `Thread Group` and update the `Number of threads`. Set the value to 2.**

**Provide a screenshot of JMeter's summary report. Give a short explanation of what the load balancer is doing.**

![](./assets/img/rapport/Task2_JMETER_2Threads.png)

The load balancer is doing its job, by the looks of it.

We can that the first request is reaching the *S2* server first then the second thread is redirected to *S1*. The *Round-Robin* algorithm is doing what it is supposed to do and the load is equally balanced between the two servers.

## Task 3 : Drain mode

**Take a screenshot of the Step 5 and tell us which node is answering.**

![](./assets/img/rapport/Task3_HAProxy.png)

Given the info on the browser, the node *S2* is reached every time and we can see that, given the cookie we set up prior to this task, that we stay in this node.

![](./assets/img/rapport/Task3_proof_S2.png)



**Based on your previous answer, set the node in DRAIN mode. Take a screenshot of the HAProxy state page.**

![](./assets/img/rapport/Task3_Drain_HAProxy.png)

![](./assets/img/rapport/Task3_Drain_proof.png)

We can see that we are in `drain` mode on the *S2* node.

**Refresh your browser and explain what is happening. Tell us if you stay on the same node or not. If yes, why? If no, why?**

![](./assets/img/rapport/Task3_refresh_drain_mode.png)

After a few refresh, we can see that we stayed on the same node. The fact that we put the `Drain` mode on means that we are blocking any connection on another server but any user can establish a new session on this server. It blocks any load balancing but still allow any health-checks or a new persistent connection. (cf. slides from the course on `High performance systems` slide 40).

**Open another browser and open `http://192.168.42.42`. What is happening?**

We are going to the other server, *S1*.

![](./assets/img/rapport/Task3_new_browser.png)

**Clear the cookies on the new browser and repeat these two steps multiple times. What is happening? Are you reaching the node in DRAIN mode?**

![](./assets/img/rapport/Task3_clear_cookie.png)

![](./assets/img/rapport/Task3_clear_cookie_2.png)

![](./assets/img/rapport/Task3_clear_cookie_3.png)

After multiple tries, we can see that we are always reaching the node *S1*, the on that is not on `DRAIN` mode, and we cannot reach the drained node. We can see that the session ids are different every time. The server has no way of knowing that the user is the same so that's why it assigns a new ID every time.

**Reset the node in READY mode. Repeat the three previous steps and explain what is happening. Provide a screenshot of HAProxy's stats page.**

![](./assets/img/rapport/Task3_ready_HAProxy.png)

The node is reset to `READY`which means normal mode according to the slides on `High performance systems` slide 40 from the course material. 

Therefore, the behavior is the one we expected.

First, we connect to a node, *S2* in our case, and if refresh the page a few times, we stay on it, thanks to the cookie set up in task 2.

![](./assets/img/rapport/Task3_ready.png)

Then, we open another browser and go to the *S1* node.

![](./assets/img/rapport/Task3_ready_other_browser_1.png)

We can see that we are redirected to the other node, as a normal *Round-Robin* algorithm would do. But we might need something more to convince you.

So we open a third browser and connect to server.

![](./assets/img/rapport/Task3_ready_other_browser_2.png)

We see that we are redirected to *S2*.

**Finally, set the node in MAINT mode. Redo the three same steps and explain what is happening. Provide a screenshot of HAProxy's stats page.**

![](./assets/img/rapport/Task3_maint_haproxy.png)

We can see in the above screenshot that the *S2* node is set up in `MAINT` mode and below that when we connect to the server, it cannot reach the *S2*  node.

![](./assets/img/rapport/Task3_maint.png)

Even when we try on another browser.

![](./assets/img/rapport/Task3_maint_browser_1.png)

We can see a different ID but accessing the same server.

## Task 4 : Round robin in degraded mode.

**We based our task on a previous version of the questions so we set the requests to execute 10 threads.**
**We did this because if we didn't we couldn't see the load-balancing operation with one thread**

***Remark*: Make sure you have the cookies are kept between two requests.**

**Be sure the delay is of 0 milliseconds is set on `s1`. Do a run to have base data to compare with the next experiments.**

![](./assets/img/rapport/Task4_set_0.png)

![](./assets/img/rapport/Task4_Jmeter_1.png)

**Set a delay of 250 milliseconds on `s1`. Relaunch a run with the JMeter script and explain what it is happening?**

![](./assets/img/rapport/Task4_set_250.png)

After increasing the delay between each answer on *S1*, we can see that it takes a significant amount of time to finish the requests, as on *S2*  the time seems to be almost the same.

![](./assets/img/rapport/Task4_JMeter_250.png)

**Set a delay of 2500 milliseconds on `s1`. Same than previous step.**

![](./assets/img/rapport/Task4_set_2500.png)

The *S1* node is not even attained by the users because of the timeout. Therefore, only *S2*  is shown in the result. 

![](./assets/img/rapport/Task4_JMeter_2500.png)

**In the two previous steps, are there any error? Why?**

`JMeter` does not prompt any error.  

What happens is that the load balancer, `HAProxy`, analyses the delay and thinks that the *S1* is down. Therefore, JMeter does not think that the node has a problem but it receives the information that the node is down. Therefore, it is a normal behavior.

![](./assets/img/rapport/Task4_Jmeter_proof.png)

As we can see, the node is found to be down

**Update the HAProxy configuration to add a weight to your nodes. For that, add `weight [1-256]` where the value of weight is between the two values (inclusive). Set `s1` to 2 and `s2` to 1. Redo a run with 250ms delay.**

![](./assets/img/rapport/Task4_weights_proof.png)

![](./assets/img/rapport/Task4_delay_proof.png)

We set a delay of 250 ms for the two nodes.

 ![](./assets/img/rapport/Tasks4_JMETER_with_cookies.png)

The balance is not respected at all. As the weight condition is greater in the *S1* node, it is normal for a load balancer to put more requests directed towards the one that can hold more. As we set a delay on the two nodes, the execution is even longer than before. 

**Now, what happened when the cookies are cleared between each requests and the delay is set to 250ms ? We expect just one or two sentence to  summarize your observations of the behavior with/without cookies.**

![](./assets/img/rapport/Task4_weight_Jmeter.png)

We can see that the *S1* node is more reached than *S2*. The reason for that is that the weight of the node is heavier in *S1* and cookies are reseting between each requests. 

We can see that the balance is almost evenly shared with the cookie being reseted between every request. 

## Task 5 : Balancing strategies

**Briefly explain the strategies you have chosen and why you have chosen them.**  

The two strategies we chose are : 

- `static-rr`
- `leastconn`

We chose those two strategies because : 

-  `static-rr` is almost the same as the one we saw during this lab and we wanted to know what the difference might be. It lies in the fact that you cannot perform any changes on the fly of the configuration. 
-  `leastconn` seems interesting in our case because it should behave the same way that `round-robin` did. The algorithm is also dynamic so we should be able to change weights on the fly. Furthermore, it is not recommended to use it in short sessions like `HTTP`. It is more suitable for `LDAP` connections, for example. The fact that the load might not be evenly distributed in this case is a possibility which we wanted to experiment.

**Provide evidences that you have played with the two strategies (configuration done, screenshots, ...)**  

- `static-rr` : We configured the load balancing in `static-rr`. ![](./assets/img/rapport/Task5_Config_static.png)
Then we launched the docker containers in order to test if the configuration was up. 
As the behavior is pretty similar we had to test that if we modified on the fly the configuration and see if the changes were effective.
First we checked the configuration : 
![](./assets/img/rapport/Task5_static_prior_remove.png)
We can see that the configuration is up and running in `static-rr`.  
Then we log into the HAProxy's Docker and modify the configuration.
![](./assets/img/rapport/Task5_docker_static.png)
![](./assets/img/rapport/Task5_static_remove_node.png)
We removed a node from the load-balancing configuration and we waited a few minutes before reloading the configuration page.
After reloading it, we checked if the node *S2* was removed or not. 
![](./assets/img/rapport/Task5_static_after_remove.png)
We can see that *S2* is still present in the config.  
- `leastconn` : We configured the load balancing in `leastconn` mode.
![](./assets/img/rapport/Task5_least.png) 
We removed the cookie for the test. 
We'll set them back later on. 
We can see that the configuration is up and running : 
![](./assets/img/rapport/Task5_config_least.png)
Then we connect for the first time. 
![](./assets/img/rapport/Task5_least_firstConn.png)  
Then the second.  
![](./assets/img/rapport/Task5_least_second_conn.png)

We can see that the behavior is, expected the same as the `roundrobin` mode because we have two nodes.

Let's see what happens if we enable the cookies !
![](./assets/img/rapport/Task5_leastconn_with_cookies.png)

 We then proceed to connect to the main page and refresh two times.

 ![](./assets/img/rapport/Task5_leastconn_with_cookies_first_conn.png)
 ![](./assets/img/rapport/Task5_leastconn_with_cookies_second.png)
 ![](./assets/img/rapport/Task5_leastconn_with_cookies_third.png)

 We can see that the cookie is always taken into account. 
 It overlaps the configuration as in `round robin` mode. 


**Compare the both strategies and conclude which is the best for this lab (not necessary the best at all).**  
For this lab, we think that the best mode would be `leatconn` as it is essentially the same behavior as `round-robin`.
`static-rr` is too constraint of a mode to make change on the fly.

## Conclusion

To conclude this lab, we now know how to use and configure the `HAProxy` tool. We know which load-balancing strategy is the best to use and in which case to use it.