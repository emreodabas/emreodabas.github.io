---

layout: post
title: You're up and running!
order: 1
---

Web Socket Behavior on GKE
==========================

[![Emre Odabas](https://miro.medium.com/fit/c/56/56/0*0soFcQXWNTTComIj.jpg)](https://medium.com/@emreodabas_20110?source=post_page-----7d9a22ef9b13--------------------------------)[

Emre Odabas

](https://medium.com/@emreodabas_20110?source=post_page-----7d9a22ef9b13--------------------------------)[

Sep 24, 2020·2 min read

](https://medium.com/@emreodabas_20110/web-socket-behavior-on-gke-7d9a22ef9b13?source=post_page-----7d9a22ef9b13--------------------------------)

In our project, We (with [Tümay Çeber](https://medium.com/u/11eadbe145ba?source=post_page-----7d9a22ef9b13--------------------------------) ) intend to investigate web socket behavior working on Kubernetes (GKE) while pods terminating. The graceful shutdown ability of pods needs to be evaluated for WebSocket management. We desire to increase the stability of application and user experience.

We aim to find answers for the following questions while pod terminating;

*   What happens to alive WebSocket/HTTP connections?
*   Could the client create a new WebSocket/HTTP connection?
*   Could the server still send any message to clients?
*   What could we do in a graceful shutdown state?

Preparation
===========

We developed a server application for handling WebSocket connections. This app received text messages and respond with reverted text via WebSocket. Also, send connection alive info to clients periodically (5 sec). We implement an endpoint for graceful shutdown ability. This endpoint return 15 seconds delayed response with HTTP 200 OK. Also, we developed a health endpoint for the liveness probe. The application code is reachable from this [link](https://github.com/emreodabas/websocket-graceful).

After that, we [dockerized](https://hub.docker.com/repository/docker/emreodabas/ws-test) our application and push the image to Docker Hub. We [deployed](https://github.com/emreodabas/websocket-graceful/blob/master/ws-test.yaml) our application to Kubernetes which working like the below design. Also, a chrome extension [WebSocket client tool](https://chrome.google.com/webstore/detail/websocket-test-client/fgponpodhbmadfljofbimhhlengambbn) was selected for the connection test.

<img alt="" class="ef es eo ex w" src="https://miro.medium.com/max/1400/1\*VbDUNzQZugIBim7nx74qag.png" width="700" height="308" srcSet="https://miro.medium.com/max/552/1\*VbDUNzQZugIBim7nx74qag.png 276w, https://miro.medium.com/max/1104/1\*VbDUNzQZugIBim7nx74qag.png 552w, https://miro.medium.com/max/1280/1\*VbDUNzQZugIBim7nx74qag.png 640w, https://miro.medium.com/max/1400/1\*VbDUNzQZugIBim7nx74qag.png 700w" sizes="700px" role="presentation"/>

Test Scenarios
==============

We connect to the WebSocket server via Websocket Test Client extension.

After the connection open, we checked the server works as described above. In this scenario, Google Load Balancer (GLB) and ingress upgrade connection to Websocket as needed. Also, we could access static files via the browser. Both HTTP and WebSocket connections are achieved.

*   On the first step, we delete the pod and trigger the preStop endpoint successfully. Our graceful shutdown endpoint paused pod termination 15 seconds. After pod terminating send to the pod, Kubernetes automatically start to create a new pod and not waiting for the grace period of the pod.
*   While pod in terminating state, we could not create any new HTTP connection but we could reuse our upgraded WebSocket connections. We could not reach the static file and create a new WebSocket connection to that pod.
*   We could send the request and get a response reverted text via WebSocket while terminating the state. We could also receive auto alive information from the client.

Decision
========

After our test results, we decide to use a graceful shutdown ability for managing in progress jobs in the pod. We could wait for activities done and persist states after that disconnect related WebSockets with clients. Clients will reconnect to a new instance and their flows could work as usual. With these features, clients will not affect any server downtime.

Takeaways
=========

*   Websockets still get requests while graceful shutdown in GKE
*   Websocket connections need to be handled manually for graceful shutdown
*   Implement client code with auto-connect/retry connect strategy
