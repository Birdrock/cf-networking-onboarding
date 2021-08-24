User Workflow: HTTP Routes

## Assumptions
- You have a CF deployed
- You have two [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) apps pushed, named appA and appB

## What?

**Routes** are the URLs that can be used to access CF apps.
**Route Mappings** are the join table between routes (URLs) and the apps they send traffic to. Apps can have many routes.
And routes can send traffic to many apps. So Route Mappings is a many-to-many mapping.

## How?

📝 **Create a route that maps to two apps**
0. By default `cf push` creates a route. Look at all of the routes for all of your apps.
 ```
 cf apps
 ```
0. Use curl to hit appA.
 ```
 curl APP_A_URL
 ```
 It should respond with something like `{"ListenAddresses":["127.0.0.1","10.255.116.44"],"Port":8080}`
 We'll get into the listen addresses later, but for now the most important thing to know is that the 10.255.X.X address is the overlay IP address. This IP is unique per app instance.
0. Create your own route (`cf map-route --help`) and map it to both appA **AND** appB.
0. Curl your new route over and over again `watch "curl -sS MY-NEW-ROUTE`".

### Expected Result
You have a route that maps to both appA and appB. See that the overlay IP changes, showing that you are routed evenly(ish) between all the apps mapped to the routes.

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
L: user-workflow
---

Route Propagation - Part 0 - Creating a Route Overview

## What
In the previous story you used the CF CLI to create and map routes. But what happens under the hood to make all of this work? (hint: iptables is involved)
There are two main data flows for routes, (1) when an app dev pushes a new app with a route and (2) when an internet user connects to an app using the route.

Let's focus on the first case. Here is what happens under the hood:

Each step marked with a ✨ will be explained in more detail in a story in this track.
**When an app dev pushes a new app with a route**
1. ✨ The app dev pushes an app with a route using the CF CLI.
1. Cloud Controller receives this request and sends this information to Diego.
1. Diego schedules the container create on a specific Diego Cell.
1. Garden creates the container for your app.
1. ✨ Diego deploys a sidecar envoy inside of the app container, which will proxy traffic to your app.
1. ✨ When the container is being set up, iptables rules are created on the Diego Cell to send traffic that is intended for the app to the sidecar proxy.
1. ✨ When the app is created, Diego sends the route information to the Route Emitter. The Route Emitter sends the route information to GoRouter via NATS.
1. ✨ The GoRouter keeps a mapping of routes -> ip:ports in a routes table, which is consulted when someone curls the route.

## How
The following stories will look at how many components (Cloud Controller, Diego BBS, Route Emitter, Nats, GoRouter, DNAT Rules, Envoy) work together to make routes work.

0. 🤔 Step through steps above and follow along on [the HTTP Routing section of this diagram](https://miro.com/app/board/o9J_kyWPVPM=/?moveToWidget=3074457346471397934).

### Expected Result
You can talk about route propagation at a high level.

## Logistics
In the next few stories, you are going to need to remember values from one story to another, there will be a space provided at the bottom of each story for your to record these values so you can store them.
It can be annoying to scroll up and down in the story as you use the values, so it could be helpful to store these values in a doc outside of tracker.

## Resources for the entire route propagation track
**Cloud Controller**
[Cloud Controller V2 API docs](https://apidocs.cloudfoundry.org)
[Cloud Controller V3 API docs](http://v3-apidocs.cloudfoundry.org)

**Diego**
[cfdot docs](https://github.com/cloudfoundry/cfdot)
[diego design notes](https://github.com/cloudfoundry/diego-design-notes#what-are-all-these-repos-and-what-do-they-do)
[diego bbs API docs](https://github.com/cloudfoundry/bbs/tree/master/doc)

**NATs**
[NATS message bus repo](https://github.com/nats-io/gnatsd)
[NATS ruby gem repo](https://github.com/nats-io/ruby-nats)

**GoRouter**
[GoRouter routing table docs](https://github.com/cloudfoundry/gorouter#the-routing-table)
[Detailed Diagram of several Route Related User Flows](https://realtimeboard.com/app/board/o9J_kyWPVPM=/)

**Iptables**
[iptables man page](http://ipset.netfilter.org/iptables.man.html)
[Aidan's iptables in CF ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)

**Route Integrity**
[Route Integrity/Misrouting Docs](https://docs.cloudfoundry.org/concepts/http-routing.html#-preventing-misrouting)

**Envoy**
[What is Envoy?](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
---

Route Propagation - Part 1 - Cloud Controller

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- I recommend deleting all other apps

## What
The Cloud Controller API (CC API) maintains the database of all apps, domains, routes, and route mappings.
However Cloud Controller does not keep track of *where* those apps are deployed. Nor does CC track the IPs and ports each
route should, well, *route* to. That's the job of the router, often called GoRouter.

CC keeps track of the desired state. The user wants a route called MY_ROUTE that sends traffic to appA.
But the user doesn't (shouldn't) care about the logistics needed to make that route happen. That is the responsibility of other components.

Let's look at what information Cloud Controller *does* keep track of.

## How

0. 🤔 Map a route to appA. Let's call this route APP_A_ROUTE. I recommend _deleting_ all other routes.

0. 🤔 Look at the domains, routes, destinations (route mappings), and apps via the Cloud Controller's API.
    To look at all the domains you can curl using `cf curl /v3/domains`. Use the [API docs](https://v3-apidocs.cloudfoundry.org/) to figure out the endpoints for the other resources.

This is all of the information that CC contains about routes. Note there are no IPs anywhere. Note that all of these routes are for CF apps, none of them are for CF components.

### Expected Result
You can view data from CC about the route APP_A_ROUTE that you created.

## Recorded Values
Record the following values that you generated or discovered during this story.
```
APP_A_ROUTE=<value>
```

## Resources
[Cloud Controller API docs](https://v3-apidocs.cloudfoundry.org/)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
---

Route Propagation - Part 2 - Diego BBS

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE
- You have completed the previous stories in this track

## What

**Diego** is an umbrella term for many components that work together to make CF container orchestration happen. These jobs are bundled together in [diego-release](https://github.com/cloudfoundry/diego-release/tree/develop/jobs).

**BBS** stands for Bulletin Board System. This is the database that the Diego components use to keep track of DesiredLRPs and ActualLRPs.

**LRPs**, or Long Running Processes, represent work that a client (ie Cloud Controller) wants Diego to keep running indefinitely. Apps are the primary example of LRPs. Diego will try to
keep them running as best it can. When an app stops or fails, it will attempt to restart it and keep it running.

**Desired LRPs** represent what a client (ie Cloud Controller) wants "the world" to look like (for example, how many instances of which apps). They contain no information
about where LRPs should be run, because the user shouldn't care.

**Actual LRPs** represent what Diego is currently *actually* running. Actual LRPs contain information about which Diego Cell the LRP is running on and which port maps to the LRP.

For this story, let's look at the data stored in the BBS and see what information it has about appA. Diego will go on to send this information via the Route Emitter to GoRouter, so GoRouter knows where to send network traffic to.

## How

📝 **Look at actualLRPS**
0. Grab the guid for appA. You'll need it in a moment. Let's call it APP_A_GUID.
 ```
 cf app appA --guid
 ```
0. Ssh onto the Diego Cell vm where appA is running and become root. You can find where appA is running by running the following command:
 ```
 cf curl /v3/processes/<app-guid>/stats
 ```
0. Use the [cfdot CLI](https://github.com/cloudfoundry/cfdot) to query BBS for actualLRPs. Cfdot is a helpful CLI for using the BBS API.
 It's a great tool for debugging on the Diego Cell.
 ```
 cfdot actual-lrps | jq .
 ```
0. Search through the actual LRPs for APP_A_GUID. It should match the beginning of a process guid. You'll find an entry for each instance of appA that is running.
0. Let's dissect and store the most important information (for us) about appA:
   ```
   {
     "process_guid": "ab2bd185-9d9a-4628-9cd8-626649ec5432-cb50adac-6861-4f03-92e4-9fcc1a204a1e",
     "index": 0,
     "cell_id": "d8d4f5fe-36f2-4f50-8c4a-8df293f6bc5b",
     "address": "10.0.1.12",                  <------ The cell's IP address where this app instance is running, also sometimes called the host IP. Let's call this DIEGO_CELL_IP.
       "ports": [
         {
           "container_port": 8080,            <------ The port the app is listening on inside of its container. 8080 is the default value. Let's call this CONTAINER_APP_PORT.
           "host_port": 61012,                <------ The port on the Diego Cell where traffic to your app is sent to before it is forwarded to the overlay address and the container_port. Let's call this DIEGO_CELL_APP_PORT.
           "container_tls_proxy_port": 61001, <------ The port inside of the app container that envoy is listening on for HTTPS traffic. This is the default value (currently unchangeable). Let's call this CONTAINER_ENVOY_PORT.
           "host_tls_proxy_port": 61014,      <------ The port on the Diego Cell where traffic to your app's envoy sidecar is sent to before it is forwarded to the overlay address and the container_tls_proxy_port. Let's call this DIEGO_CELL_ENVOY_PORT
         },
         {
           "container_port": 2222,            <------ The port exposed on the app container for sshing onto the app container
           "host_port": 61013,                <------ The port on the Diego Cell where ssh traffic to your app container is sent to before it is forwarded to the overlay address and the ssh container_port
           "container_tls_proxy_port": 61002, <------ The ssh port inside of the app container that envoy is listening on for ssh traffic. This is the default value (currently unchangeable).
           "host_tls_proxy_port": 61015       <------ The port on the Diego Cell where ssh traffic to your app's envoy sidecar is sent to before it is forwarded to the overlay address and the ssh container_tls_proxy_port
         }
       ],
     "instance_address": "10.255.116.6",      <------ The overlay IP address of this app instance, let's call this the OVERLAY_IP
     "state": "RUNNING",
      ...
   }
   ```
0. Use the cfdot CLI to query BBS for desiredLRPs.

❓What information is provided for desiredLRPs, but not for actualLRPs?
❓What information is provided for actualLRPs, but not for desiredLRPs?
❓How does this match with the definition of desired and actual LRPs in the "what" section above?

### Expected Result
Get information from BBS about the desiredLRP and actualLRP for appA. Use cfdot CLI to discover the following values and record them.

## Recorded Values
Record the following values that you generated or discovered during this story.
```
APP_A_GUID=<value>
DIEGO_CELL_IP=<value>
CONTAINER_APP_PORT=<value>
DIEGO_CELL_APP_PORT=<value>
CONTAINER_ENVOY_PORT=<value>
DIEGO_CELL_ENVOY_PORT=<value>
OVERLAY_IP=<value>
```

## Resources
[cfdot docs](https://github.com/cloudfoundry/cfdot)
[diego design notes](https://github.com/cloudfoundry/diego-design-notes#what-are-all-these-repos-and-what-do-they-do)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
L: questions
---

Route Propagation - Part 3 - Route Emitter and NATS

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE
- You have completed the previous stories in this track

## Recorded values from previous stories
```
APP_A_ROUTE=<value>
APP_A_GUID=<value>
DIEGO_CELL_IP=<value>
CONTAINER_APP_PORT=<value>
DIEGO_CELL_APP_PORT=<value>
CONTAINER_ENVOY_PORT=<value>
DIEGO_CELL_ENVOY_PORT=<value>
OVERLAY_IP=<value>
```

## What

There is one Route Emitter per Diego Cell and its job is to... emit routes. According to the ever helpful
[Diego Design Notes](https://github.com/cloudfoundry/diego-design-notes) the Route Emitter "monitors DesiredLRP state
and ActualLRP state via the BBS. When a change is detected, the Route Emitter emits route registration and unregistration
messages to the GoRouter via the NATS message bus." Even when no change is detected, the Route Emitter will periodically
emit the entire routing table as a kind of heartbeat.

For this story, let's look at the messages that the Route Emitter is publishing via NATS. Subscribing to these NATs messages
can be a helpful debugging technique.

## How

📝 **subscribe to NATs messages**
0. Bosh ssh onto the Diego Cell where your app is running and become root
0. Download the NATS cli
    ```
    wget https://github.com/nats-io/natscli/releases/download/0.0.25/nats-0.0.25-linux-amd64.zip
    unzip nats-0.0.25-linux-amd64.zip
    chmod +x nats-0.0.25-linux-amd64
    mv nats-0.0.25-linux-amd64/nats /usr/bin
    ```
0. Get NATS username, password, and server address
    ```
    cat /var/vcap/jobs/route_emitter/config/route_emitter.json | jq . | grep nats
    ```
0. Use the nats gem to connect to nats: `nats sub "*.*" -s nats://NATS_USERNAME:NATS_PASSWORD@NATS_ADDRESS --tlsca NATS_CA_CERT_FILE --tlscert NATS_CLIENT_CERT_FILE --tlskey NATS_CLIENT_KEY_FILE`. The `"*.*"` means that you are subscribing to all NATs messages. 
    The Route Emitter registers routes every 20 seconds (by default) so that the GoRouter (which subscribes to these messages) has the most up-to-date information about which IPs map to which apps and routes. Depending on how many routes there are, this might be a lot of information.

0. Find the NATs message for APP_A_ROUTE.
 ```
 $ nats sub "*.*" -s nats://NATS_USERNAME:NATS_PASSWORD@NATS_ADDRESS  --tlsca NATS_CA_CERT_FILE --tlscert NATS_CLIENT_CERT_FILE --tlskey NATS_CLIENT_KEY_FILE | grep APP_A_ROUTE
 ```

0. When you successfully connect to nats, plus a few seconds of waiting, you should see a message that contains information about the route you created. It will look something like this and contain APP_A_ROUTE:
 ```
   [#32] Received on [router.register] :
{
    "host": "10.0.1.12",
    "port": 61012,
    "tls_port": 61014,
    "uris": [
        "proxy.meow.cloche.c2c.cf-app.com"     <--- This should match APP_A_ROUTE
      ],
    "app": "6856799f-aebf-4e2b-81a5-28c74dfb6162",
     "private_instance_id": "a0d2b217-fa7d-4ac1-65a2-7b19",
     "private_instance_index": "0",
    "server_cert_domain_san": "a0d2b217-fa7d-4ac1-65a2-7b19",
    "tags": {
         "component": "route-emitter"
     }
}
 ```

❓Do the values in the NATS message match the values you recorded previously from BBS? Which ones are present? Which ones aren't there?
❓How does it compare to the information in Cloud Controller?


### Expected Result
Inspect NATs messages. Look at what route information is sent to the GoRouter.

## Resources
[NATS message bus repo](https://github.com/nats-io/gnatsd)
[NATS ruby gem repo](https://github.com/nats-io/ruby-nats)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
L: questions
---

Route Propagation - Part 4 - GoRouter

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE
- You have completed the previous stories in this track

## Recorded values from previous stories
```
APP_A_ROUTE=<value>
APP_A_GUID=<value>
DIEGO_CELL_IP=<value>
CONTAINER_APP_PORT=<value>
DIEGO_CELL_APP_PORT=<value>
CONTAINER_ENVOY_PORT=<value>
DIEGO_CELL_ENVOY_PORT=<value>
OVERLAY_IP=<value>
```

## What
So the Route Emitter emits routes via the NATS message Bus. GoRouter subscribes to those messages and keeps a route table that it uses to route network traffic bound for CF apps and CF components.

Let's take a look at that route table.
## How

📝 **look at route table**
0. Bosh ssh onto the router vm and become root.
0. Get the username and password for the routing api
 ```
 head /var/vcap/jobs/gorouter/config/gorouter.yml
 ```
0. Get the routes table
 ```
 curl "http://USERNAME:PASSWORD@localhost:8080/routes" | jq .
 ```
0. Scroll through and look at the routes.
  ❓How does this differ from the route information you saw in Cloud Controller?
   For example, you should see routes for CF components, like UAA and doppler.
   This because the GoRouter is in charge of routing traffic to CF apps *AND* to CF components.
0. Find APP_A_ROUTE in the list of routes. Let's dissect the most important bits.
    ```
    "proxy.meow.cloche.c2c.cf-app.com": [   <------ The name of the route! This should match APP_A_ROUTE
        {
          "address": "10.0.1.12:61014",     <------ This is where GoRouter will send traffic for this route. This should match DIEGO_CELL_IP:DIEGO_CELL_ENVOY_PORT
          "tls": true                       <------ This means Route Integrity is turned on, so the GoRouter will use send traffic to this app over TLS
        }
      ]
    ```
    See how the traffic is being sent to `10.0.1.12:61014` or DIEGO_CELL_IP:DIEGO_CELL_ENVOY_PORT?
    This means all traffic is being sent to the sidecar envoy via TLS, this is because route integrity is enabled.
    ❓What port do you think would be listed here if route integrity was not enabled?

### Expected Result
Access the route table on the router vm. Inspect app routes and CF component routes.

See that the GoRouter sends traffic for this route to DIEGO_CELL_IP:DIEGO_CELL_ENVOY_PORT.

The route has now been propagated all the way to the Gorouter! In the next stories we will learn what happens when somone uses that route.
## Resources

[GoRouter routing table docs](https://github.com/cloudfoundry/gorouter#the-routing-table)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
L: questions
---

Incoming HTTP Requests - Part 0 - HTTP Traffic Overview

## What
In the previous stories you learned what happens when an app dev pushes a new app with a route. In this story you will learn an at a high level what happens when an end user connects to an app using the route.

Each step marked with a ✨ will be explained in more detail in a story in this track.
**When an internet user sends traffic to your app**
1. The user visits your route in the browser or curls it via the command line.
1. The traffic first hits a load balancer in front of the CF Foundation.
1. The load balancer sends it to one of the GoRouters.
1. ✨ The GoRouter consults the route table and sends it to the listed IP and port. If Route Integrity is enabled, it sends this traffic via TLS. (This was explored in the previous story!)
1. ✨ The traffic makes its way to the correct Diego Cell, where it hits iptables DNAT rules that reroutes the traffic to the sidecar envoy for the app.
1. ✨ The Envoy terminates the TLS from the GoRouter and then sends the traffic on to the app.

## How
The following stories will look at how many components (Cloud Controller, Diego BBS, Route Emitter, Nats, GoRouter, DNAT Rules, Envoy) work together to make routes work.

0. 🤔 Step through steps above and follow along on [the HTTP Routing section of this diagram](https://realtimeboard.com/app/board/o9J_kyWPVPM=/)

### Expected Result
You can talk about HTTP network traffic flow with fellow CF engineers.

## Logistics
In the next few stories, you are going to need to remember values from one story to another, there will be a space provided at the bottom of each story for your to record these values so you can store them.

## Resources for the entire route propagation track
**Cloud Controller**
[Cloud Controller V2 API docs](https://apidocs.cloudfoundry.org)
[Cloud Controller V3 API docs](http://v3-apidocs.cloudfoundry.org)

**Diego**
[cfdot docs](https://github.com/cloudfoundry/cfdot)
[diego design notes](https://github.com/cloudfoundry/diego-design-notes#what-are-all-these-repos-and-what-do-they-do)
[diego bbs API docs](https://github.com/cloudfoundry/bbs/tree/master/doc)

**NATs**
[NATS message bus repo](https://github.com/nats-io/gnatsd)
[NATS ruby gem repo](https://github.com/nats-io/ruby-nats)

**GoRouter**
[GoRouter routing table docs](https://github.com/cloudfoundry/gorouter#the-routing-table)
[Detailed Diagram of several Route Related User Flows](https://realtimeboard.com/app/board/o9J_kyWPVPM=/)

**Iptables**
[iptables man page](http://ipset.netfilter.org/iptables.man.html)
[Aidan's iptables in CF ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)

**Route Integrity**
[Route Integrity/Misrouting Docs](https://docs.cloudfoundry.org/concepts/http-routing.html#-preventing-misrouting)

**Envoy**
[What is Envoy?](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
---
Incoming HTTP Requests - Part 1 - see what's listening with netstat

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE
- You have completed the previous stories in this track

## Recorded values from previous stories
```
APP_A_ROUTE=<value>
APP_A_GUID=<value>
DIEGO_CELL_IP=<value>
CONTAINER_APP_PORT=<value>
DIEGO_CELL_APP_PORT=<value>
CONTAINER_ENVOY_PORT=<value>
DIEGO_CELL_ENVOY_PORT=<value>
OVERLAY_IP=<value>
```

## What
Netstat is a tool that can show information about network connections, routing tables, and network interface statistics.
In the previous story we saw that the GoRouter sent traffic for APP_A_ROUTE to DIEGO_CELL_IP:DIEGO_CELL_ENVOY_PORT.
Let's use netstat to see what is listening at on the Diego Cell and specifically at DIEGO_CELL_IP:DIEGO_CELL_ENVOY_PORT.

## How
📝 **look at open ports on a Diego Cell**
1. Ssh onto the Diego Cell where appA is deployed and become root.
1. Use netstat to look at open ports
 ```
 netstat -tulp
 # -t  <---- show tcp sockets
 # -u  <---- show udp sockets
 # -l  <---- display listening sockets
 # -p  <---- display PID/program name for sockets
 ```
  You should recognize the program names in the far right column. Most of them are the long running cf component processes.

1. Find the local address for the Route Emitter. What port is it running on? Does that match what is in the [spec file](https://github.com/cloudfoundry/diego-release/blob/develop/jobs/route_emitter/spec)?

1. Search for DIEGO_CELL_ENVOY_PORT in the output. Can you find it?

### Expected Result
You won't see the DIEGO_CELL_ENVOY_PORT anywhere in the netstat output because nothing is *actually* running there.
But if there's nothing running there, how does the traffic reach the app? Would you believe that iptables are involved?
Check out the next story to learn more :)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
---

Incoming HTTP Requests - Part 2 - DNAT Rules

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE
- You have completed the previous stories in this track
- It will help if you have completed the "iptables-primer" track, but it is not required.

## Recorded values from previous stories
```
APP_A_ROUTE=<value>
APP_A_GUID=<value>
DIEGO_CELL_IP=<value>
CONTAINER_APP_PORT=<value>
DIEGO_CELL_APP_PORT=<value>
CONTAINER_ENVOY_PORT=<value>
DIEGO_CELL_ENVOY_PORT=<value>
OVERLAY_IP=<value>
```

## What
In a previous story you saw that the GoRouter sent traffic for APP_A_ROUTE to DIEGO_CELL_IP:DIEGO_CELL_ENVOY_PORT.
But in the previous story, you saw nothing was actually listening at that port on the Diego Cell. So how does the network traffic hit the app?
With the help of iptables rules of course! Everything always comes back to iptables rules.

Nothing is actually listening on the Diego Cell at port DIEGO_CELL_ENVOY_PORT. Instead, all packets that are sent there hit iptables rules that then redirect them to... somewhere. Let's find out!

Let's not brute force looking through every iptables rule. Instead, let's reason about what chain and table most likely contain these iptables rules.
Hint, these rules translate ingress network traffic sent from GoRouter.

## How

🤔 **Find those iptables rules**
Aidan Obley made a great diagram showing the different types of traffic in CF and which iptables chains they hit in what order.
We are currently concerned with ingress traffic, which is represented by the orange line.

0. Look at the diagram. Which chain does the ingress traffic hit first?
    ![traffic-flow-through-iptables-on-cf-diagram](https://storage.googleapis.com/cf-networking-onboarding-images/traffic-flow-through-iptables-on-cf.png)

0. Based on the previous diagram, the ingress traffic hits the prerouting chain first. Look at the diagram below and do some research to learn more about the raw, conn_tracking, mangle, and nat tables.
    Which table should contain the rules to redirect our traffic to a new address?
    ![iptables tables and chains diagram](https://storage.googleapis.com/cf-networking-onboarding-images/iptables-tables-and-chains-diagram.png)

    NAT stands for Network Address Translation. That sounds like what we want.  So let's look at iptables rules for the nat table on the prerouting chain.

0. Ssh onto the Diego Cell where your app is running and become root.
0. Run `iptables -S -t nat`
    You should see some custom chains attached to the PREROUTING chain. There will be one custom chain per app running on this Diego Cell.  They will look something like this.
    ```
    -A PREROUTING -j netin--a0d2b217-fa7d-4ac1-65
    -A PREROUTING -j netin--317736ed-70ac-4087-74
    ...
    ```
0. You should also see 4 rules that contain the OVERLAY_IP for appA. If you look closely you'll see that the ports in the iptables rules match the ports we saw when inspecting the actual LRPs.
    Which port represents what?
    ```
    -A netin--a0d2b217-fa7d-4ac1-65 -d 10.0.1.12/32 -p tcp -m tcp --dport 61012 -j DNAT --to-destination 10.255.116.6:8080
    -A netin--a0d2b217-fa7d-4ac1-65 -d 10.0.1.12/32 -p tcp -m tcp --dport 61013 -j DNAT --to-destination 10.255.116.6:2222
    -A netin--a0d2b217-fa7d-4ac1-65 -d 10.0.1.12/32 -p tcp -m tcp --dport 61014 -j DNAT --to-destination 10.255.116.6:61001
    -A netin--a0d2b217-fa7d-4ac1-65 -d 10.0.1.12/32 -p tcp -m tcp --dport 61015 -j DNAT --to-destination 10.255.116.6:61002
    ```

0. For appA, find the rule that will match with the traffic the GoRouter sends to DIEGO_CELL_IP:DIEGO_CELL_ENVOY_PORT. It should look something like this...
    ![example DNAT rule with explanation](https://storage.googleapis.com/cf-networking-onboarding-images/example-DNAT-rule-with-explanation.png)

    In summary, when the GoRouter sends network traffic to 10.0.1.12:61014 (DIEGO_CELL_IP:DIEGO_CELL_ENVOY_PORT) it gets redirected to 10.255.116.6:61001 (OVERLAY_IP:CONTAINER_ENVOY_PORT).
    But, looking at the information we learned about the actual LRP, the app isn't even listening on 10.255.116.6:61001, envoy is.
    When will the traffic finally reach the app!?!?

### Expected Result
Inspect the iptables rules that DNAT the traffic from the GoRouter and send it to the correct sidecar envoy.

## Resources
[iptables man page](http://ipset.netfilter.org/iptables.man.html)
[Aidan's iptables in CF ppt](https://docs.google.com/presentation/d/1qLkNu633yLHP5_S_OqOIIBETJpW3erk2QuGSVo71_oY/edit#slide=id.p)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
---

Incoming HTTP Requests - Part 3 - Envoy Primer

## Assumptions
- None

## What

Before you go further it will help if you read a quick primer on Envoy.

## How

1. 📚 Read Julia Evan's blog post ["Some Envoy Basics"](https://jvns.ca/blog/2018/10/27/envoy-basics/).
 Unfortunately, the envoy in the docker container referenced in the blog post doesn't work anymore. However, just reading the post is enough to get a nice overview.

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
---

Incoming HTTP Requests - Part 4 - Route Integrity and Envoy

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE
- You have completed the previous stories in this track

## Recorded values from previous stories
```
APP_A_ROUTE=<value>
APP_A_GUID=<value>
DIEGO_CELL_IP=<value>
CONTAINER_APP_PORT=<value>
DIEGO_CELL_APP_PORT=<value>
CONTAINER_ENVOY_PORT=<value>
DIEGO_CELL_ENVOY_PORT=<value>
OVERLAY_IP=<value>
```

## What
A **proxy** is a process that sits in-between the client and the server and intercepts traffic before forwarding it on to the server. Proxies can add extra functionality, like caching or SSL termination.

In this case, Envoy is a sidecar proxy (Envoy can be other types of proxies too, but forget about that for now). The sidecar Envoy is only present when Route Integrity is turned on (which is done by default).

Route Integrity is when the GoRouter sends all app traffic via TLS. As part of the TLS handshake, the GoRouter validates the certificate's SAN against the ID found in its route table to make sure it is connecting to the intended app instance. This makes communication more secure and prevents stale routes in the route table from causing misrouting, which is a large security concern. Read more about how Route Integrity prevents misrouting [here](https://docs.cloudfoundry.org/concepts/http-routing.html#-preventing-misrouting).

The Envoy sidecar is the process that actually terminates the TLS traffic from the GoRouter making Route Integrity possible. Then the Envoy proxies it onto...can it be? finally?! YES! THE APP!!

Let's look at how the Envoy sidecar is configured to proxy traffic to the app.
## How

📝 **Look at envoy config**
0. Ssh onto AppA
0. Hit the Envoy `help` endpoint: `curl localhost:61003/help`
    These are all of the endpoints you can hit. Try `/clusters` what do you see?
0. Run `curl localhost:61003/config_dump`. This gives you all of the information about how the Envoy is configured.
0. Search for the CONTAINER_ENVOY_PORT, in the example it is 61001. This is where the DNAT rules forwarded the traffic to, as we saw in the last story. Find a listener called `listener-8080` that looks similar to the following: 
    ```
     "listeners": [
      {
       "name": "listener-8080",                                                    <---- The name of the listener
       "address": {
        "socket_address": { "address": "0.0.0.0", "port_value": 61001}             <---- This listener is listening on port 61001. That's the CONTAINER_ENVOY_PORT we know and love!
       },
       "filter_chains": [
        {
         "tls_context": { "require_client_certificate": true },                    <---- This means Route Integrity is turned on
         "filters": [
          {
           "name": "envoy.tcp_proxy",
           "config": { "stat_prefix": "0-stats", "cluster": "0-service-cluster" }  <---- This is the name of the cluster where Envoy will forward traffic that is sent to the CONTAINER_ENVOY_PORT, let's call this CLUSTER-NAME
          }
         ]
        }
       ]
      }
     ]
    ```
0. In the same config_dump output, find the cluster, CLUSTER-NAME, that is referenced above. It should look something like this:
    ```
     "clusters": [
      {
       "name": "0-service-cluster",                                          <---- This is the name of the cluster, CLUSTER-NAME
       "hosts": [
        {
         "socket_address": { "address": "10.255.116.6", "port_value": 8080}  <---- This is the port that the app is listening on inside of the container, should match OVERLAY_IP and CONTAINER_APP_PORT
        }
       ]
      }
    ]
    ```

So the traffic gets sent to the OVERLAY_IP:CONTAINER_ENVOY_PORT, then the envoy forwards it on to OVERLAY_IP:CONTAINER_APP_PORT!

We made it! We finally made it to the end! Everything is set up and someone can use that route you made!

### Expected Result
Look at the Envoy's 8080 listener and related cluster and see how network traffic is sent to the app.

## Resources
[Route Integrity/Misrouting Docs](https://docs.cloudfoundry.org/concepts/http-routing.html#-preventing-misrouting)
[What is Envoy?](https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
---

Incoming HTTP Requests - Part 5 - Access logs

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE
- You have completed the previous stories in this track

## Recorded values from previous stories
```
APP_A_ROUTE=<value>
APP_A_GUID=<value>
DIEGO_CELL_IP=<value>
CONTAINER_APP_PORT=<value>
DIEGO_CELL_APP_PORT=<value>
CONTAINER_ENVOY_PORT=<value>
DIEGO_CELL_ENVOY_PORT=<value>
OVERLAY_IP=<value>
```

## What
In the previous stories you followed the path of a request from a client to an app deployed on Cloud Foundry.

For every successful* response that Gorouter returns to a client it logs an access log. Here successful means "that there was a response from the app with any status code". 

These access logs can be very helpful for debugging. One common situation is that a customer sees via metrics that they are getting lots of 502s. But what apps are returning 502s? Let's look at the access logs to find out!

## How

📝 **Look at access logs**
1. Ssh onto the Router VM and become root
1. Tail the access log `/var/vcap/sys/log/gorouter/access.log`
1. In another terminal curl APP_A_ROUTE.

You should see something that looks like this: 
```
APP_A_ROUTE - [2021-02-18T21:22:32.355501523Z] "GET / HTTP/1.1" 200 0 62 "-" "curl/7.54.0" "35.191.2.80:51628" "10.0.1.11:61002" x_forwarded_for:"142.105.202.35, 35.227.211.74, 35.191.2.80" x_forwarded_proto:"http" vcap_request_id:"6cb8b8de-bc85-4479-5f90-1c3a52d88d84" response_time:0.011962 gorouter_time:0.000467 app_id:"cabd9e08-384e-4689-b868-1ba3d5d838bf" app_index:"0" x_cf_routererror:"-" x_b3_traceid:"9d30688835227cf3" x_b3_spanid:"9d30688835227cf3" x_b3_parentspanid:"-" b3:"9d30688835227cf3-9d30688835227cf3"
```

❓Can you find the status code in the access log?
❓Can you find the x-cf-routererror in the access log? 

1. 📚Read about the [X-CF-RouterError here](https://docs.cloudfoundry.org/adminguide/troubleshooting-router-error-responses.html#gorouter-specific-response-headers) and learn how it can be used for debugging. 

1. Look at the app logs for APP_A.
  ❓Can you find a log line that looks like the access log line? 
  ❓What additional information does the app log contain?

### Expected Result
You have found the access log from your curl in the log file on the router VM and in the app logs.

## Resources
* [X-CF-RouterError docs](https://docs.cloudfoundry.org/adminguide/troubleshooting-router-error-responses.html#gorouter-specific-response-headers)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
L: questions
---
Debugging tip: skip the lb and send traffic straight to gorouter

## Assumptions
- You have a OSS CF deployed
- You have one [proxy](https://github.com/cloudfoundry/cf-networking-release/tree/develop/src/example-apps/proxy) app pushed and called appA
- You have one route mapped to appA called APP_A_ROUTE
- You have completed the previous stories in this track

## What
In this story we are going to learn how to remove the LB (load balancer) from the data flow.

Here is simplified diagram of the data flow of an http route: 
```
+----+    +----------+         +-------+     +-----+
| LB +--->+ Gorouter +-------->+ Envoy +---->+ App |
+----+    +----------+         +-------+     +-----+
```

When to do this: 
* when you are having problems connecting to an app and you want to start picking off items on by one that are _not_ the problem.
* when one particular gorouter is having problems and you want to send traffic to just that gorouter.
* when you are debugging and want to point your traffic at a particular gorouter so you can find the logs easier.

## How

📝**Send HTTP traffic using LB**
1. Curl the route for your app!
  ```
  curl APP_A_ROUTE -v
  ```
1. Save this output.

❓Do you see a host header on the request? How did that get there?

📝**Send HTTP traffic without using LB**
1. Get the IP for your router VM.
1. Send the traffic to the Gorouter IP and set the route in the host header:
  ```
  curl GOROUTER_IP -H "Host: APP_A_ROUTE" -v
  ```
1. Huh. That timed out and failed. 
1. Ssh onto any bosh VM and try again. 

❓Why did it fail the first time and succeed when you were ssh-ed onto any bosh VM?
❓How does the output for this curl differ from the one you did in the first section?

### Expected Result
You were able to send traffic to a specific gorouter bypassing the LB.

## Resources
- [Host Header Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Host)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
L: questions
---
So how many data flow options are there?

## Assumptions
* You have completed all the previous stories in this track.

## What 

When we talk about the data flow from client to Cloud Foundry app, we often draw it like this: 
```
+----+    +----------+         +-------+     +-----+
| LB +--->+ Gorouter +-------->+ Envoy +---->+ App |
+----+    +----------+         +-------+     +-----+
```

But that diagram is very general. Often that is okay because the details don't always matter. But sometimes the details _do_ matter (like with HTTP/2).

In this story we will look at some more specific data flow diagrams for Cloud Foundry.

## How

First we need to understand what L4 (TCP) LBs and L7 (HTTPS) LBs _are_. 
1. 📚 Read [this medium article about "TCP vs HTTP(S) Load Balancing."](https://medium.com/martinomburajr/distributed-computing-tcp-vs-http-s-load-balancing-7b3e9efc6167)
1. 📚 Read [these cloudfoundry docs on TLS Termination Options for HTTP Routing](https://docs.pivotal.io/application-service/2-10/adminguide/securing-traffic.html).
1. Look at the following diagrams and think about the following questions for each: 
  ❓What connections (the arrows between boxes) are encrypted? Which are not? 
  ❓The big question: How will this work with HTTP/2?


**With a L4 LB in front** 
```
+-------+    +----------+         +-------+     +-----+
| L4 LB +--->+ Gorouter +-------->+ Envoy +---->+ App |
+-------+    +----------+         +-------+     +-----+
```

**With a L7 LB in front** 
```
+-------+    +----------+         +-------+     +-----+
| L7 LB +--->+ Gorouter +-------->+ Envoy +---->+ App |
+-------+    +----------+         +-------+     +-----+
```

**With an HAProxy in front** 
```
+----------+    +----------+         +-------+     +-----+
| HA Proxy +--->+ Gorouter +-------->+ Envoy +---->+ App |
+----------+    +----------+         +-------+     +-----+
```

**With an L4 LB and an HAProxy in front** 
```
+-------+    +----------+    +----------+         +-------+     +-----+
| L4 LB +--->| HA Proxy +--->+ Gorouter +-------->+ Envoy +---->+ App |
+-------+    +----------+    +----------+         +-------+     +-----+
```

**With an L7 LB and an HAProxy in front** 
```
+-------+    +----------+    +----------+         +-------+     +-----+
| L7 LB +--->| HA Proxy +--->+ Gorouter +-------->+ Envoy +---->+ App |
+-------+    +----------+    +----------+         +-------+     +-----+
```

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
L: questions
---
HTTP Routes Wrap Up

## Assumptions
- You have completed the previous stories in this track

## What
Let's review!

## How
1. Go to a whiteboard and have one pair diagram what happens when an app dev creates and maps a new route.
1. Go to a whiteboard and have the other pair diagram what happens when a person on the internet makes an HTTP request to a CF route.

### Expected Result
You know everything about routes. (Just kidding... maybe?)

🙏 _If this story needs to be updated: please, please, PLEASE submit a PR. Amelia will be eternally grateful. How? Open [this file in GitHub](https://github.com/pivotal/cf-networking-program-onboarding/edit/master/routes.prolific.md). Search for the phrase you want to edit. Make the fix!_

L: http-routes
---
[RELEASE] HTTP Routes ⇧
L: http-routes
