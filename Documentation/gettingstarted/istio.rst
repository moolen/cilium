.. only:: not (epub or latex or html)

    WARNING: You are looking at unreleased Cilium documentation.
    Please use the official rendered version released here:
    http://docs.cilium.io

***************************
Getting Started Using Istio
***************************

This document serves as an introduction to using Cilium to enforce
security policies in Kubernetes micro-services managed with Istio.  It
is a detailed walk-through of getting a single-node Cilium + Istio
environment running on your machine.

.. include:: gsg_requirements.rst

Step 2: Install Istio
=====================

Install the `Helm client <https://docs.helm.sh/using_helm/#installing-helm>`_.

Download `Istio version 1.0.2
<https://github.com/istio/istio/releases/tag/1.0.2>`_:

::

   $ export ISTIO_VERSION=1.0.2
   $ curl -L https://git.io/getLatestIstio | sh -
   $ export ISTIO_HOME=`pwd`/istio-${ISTIO_VERSION}
   $ export PATH="$PATH:${ISTIO_HOME}/bin"

Create a copy of Istio's Helm charts in order to customize them:

::

    $ cp -r ${ISTIO_HOME}/install/kubernetes/helm/istio istio-cilium-helm

Configure the Cilium-specific variant of Pilot to inject the
Cilium network policy filters into each Istio sidecar proxy:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/cilium-pilot.awk > cilium-pilot.awk

::

    $ awk -f cilium-pilot.awk \
          < istio-cilium-helm/charts/pilot/templates/deployment.yaml \
          > istio-cilium-helm/charts/pilot/templates/deployment.yaml.new && \
          mv istio-cilium-helm/charts/pilot/templates/deployment.yaml.new istio-cilium-helm/charts/pilot/templates/deployment.yaml

Configure the Istio's sidecar injection to setup the transparent proxy mode
(TPROXY) as required by Cilium's proxy filters:

::

    $ sed -e 's,#interceptionMode: .*,interceptionMode: TPROXY,' \
          < istio-cilium-helm/templates/configmap.yaml \
          > istio-cilium-helm/templates/configmap.yaml.new && \
          mv istio-cilium-helm/templates/configmap.yaml.new istio-cilium-helm/templates/configmap.yaml

Modify the Istio sidecar injection template to uses Cilium's proxy Docker
images and mount Cilium's API Unix domain sockets into each sidecar to allow
Cilium's Envoy filters to query the Cilium agent for policy configuration:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/cilium-kube-inject.awk > cilium-kube-inject.awk

::

    $ awk -f cilium-kube-inject.awk \
          < istio-cilium-helm/templates/sidecar-injector-configmap.yaml \
          > istio-cilium-helm/templates/sidecar-injector-configmap.yaml.new && \
          mv istio-cilium-helm/templates/sidecar-injector-configmap.yaml.new istio-cilium-helm/templates/sidecar-injector-configmap.yaml

Create an Istio deployment spec, which configures the Cilium-specific variant
of Pilot, and disables unused services:

::

    $ helm template istio-cilium-helm --name istio --namespace istio-system \
          --set pilot.image=docker.io/cilium/istio_pilot:1.0.2 \
          --set sidecarInjectorWebhook.enabled=false \
          --set global.controlPlaneSecurityEnabled=true \
          --set global.mtls.enabled=true \
          --set global.proxy.image=proxy_debug \
          --set ingress.enabled=false \
          --set egressgateway.enabled=false \
          > istio-cilium.yaml

.. TODO: Set global.controlPlaneSecurityEnabled=true and
   global.mtls.enabled=true when we stop seeing TLS connections getting
   forcefully closed by sidecar proxies sporadically.

Deploy Istio onto Kubernetes:

::

    $ kubectl create namespace istio-system
    $ kubectl create -f istio-cilium.yaml

Check the progress of the deployment (every service should have an
``AVAILABLE`` count of ``1``):

::

    $ kubectl get deployments -n istio-system
    NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    istio-citadel              1         1         1            1           1m
    istio-egressgateway        1         1         1            1           1m
    istio-galley               1         1         1            1           1m
    istio-ingressgateway       1         1         1            1           1m
    istio-pilot                1         1         1            1           1m
    istio-policy               1         1         1            1           1m
    istio-statsd-prom-bridge   1         1         1            1           1m
    istio-telemetry            1         1         1            1           1m
    prometheus                 1         1         1            1           1m

Once all Istio pods are ready, we are ready to install the demo
application.

Step 3: Deploy the Bookinfo Application V1
==========================================

Now that we have Cilium and Istio deployed, we can deploy version
``v1`` of the services of the `Istio Bookinfo sample application
<https://istio.io/docs/guides/bookinfo.html>`_.

The BookInfo application is broken into four separate microservices:

- *productpage*. The productpage microservice calls the details and
  reviews microservices to populate the page.
- *details*. The details microservice contains book information.
- *reviews*. The reviews microservice contains book reviews. It also
  calls the ratings microservice.
- *ratings*. The ratings microservice contains book ranking
  information that accompanies a book review.

In this demo, each specific version of each microservice is deployed
into Kubernetes using separate YAML files which define:

- A Kubernetes Service.
- A Kubernetes Deployment specifying the microservice's pods, specific
  to each service version.
- A Cilium Network Policy limiting the traffic to the microservice,
  specific to each service version.

.. image:: images/istio-bookinfo-v1.png
   :scale: 75 %
   :align: center

First create a policy to explicitly allow the sidecar proxies to access
the Istio services while the pods are initializing:

.. parsed-literal::

    $ kubectl create -f \ |SCM_WEB|\/examples/kubernetes-istio/istio-sidecar-init-policy.yaml
    ciliumnetworkpolicy "istio-sidecar" created

Create an Istio ingress gateway for the productpage service:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-productpage-ingress.yaml | \\
          istioctl create -f -
    Created config gateway/default/productpage at revision ...
    Created config virtual-service/default/productpage at revision ...

To package the Istio sidecar proxy and generate final YAML
specifications, run:

.. parsed-literal::

    $ for service in productpage-service productpage-v1 details-v1 reviews-v1; do \\
          curl -s \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-${service}.yaml | \\
          istioctl kube-inject -f - | \\
          kubectl create --validate=false -f - ; done
    service "productpage" created
    ciliumnetworkpolicy "productpage-v1" created
    deployment "productpage-v1" created
    service "details" created
    ciliumnetworkpolicy "details-v1" created
    deployment "details-v1" created
    service "reviews" created
    ciliumnetworkpolicy "reviews-v1" created
    deployment "reviews-v1" created

Check the progress of the deployment (every service should have an
``AVAILABLE`` count of ``1``):

::

    $ kubectl get deployments -n default
    NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    details-v1       1         1         1            1           6m
    productpage-v1   1         1         1            1           6m
    reviews-v1       1         1         1            1           6m

To obtain the URL to the frontend productpage service, run:

::

    $ export PRODUCTPAGE=`minikube service istio-ingressgateway -n istio-system --url | head -n 1`
    $ echo "Open URL: ${PRODUCTPAGE}/productpage"

Open that URL in your web browser and check that the application has
been successfully deployed.  It may take several seconds before all
services become accessible in the Istio service mesh, so you may have
have to reload the page.

Step 4: Canary and Deploy the Reviews Service V2
================================================

We will now deploy version ``v2`` of the ``reviews`` service.  In
addition to providing reviews from readers, ``reviews v2`` queries a
new ``ratings`` service for book ratings, and displays each rating as
1 to 5 black stars.

As a precaution, we will use Istio's service routing feature to canary
the ``v2`` deployment to prevent breaking the end-to-end application
completely if it is faulty.

Before deploying ``v2``, to prevent any traffic from being routed to
it for now, we will create this Istio route rules to route 100% of the
``reviews`` traffic to ``v1``:

.. literalinclude:: ../../examples/kubernetes-istio/route-rule-reviews-v1.yaml

.. image:: images/istio-bookinfo-reviews-v2-route-to-v1.png
   :scale: 75 %
   :align: center

Apply this route rule:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/route-rule-reviews-v1.yaml | \\
          istioctl create -f -
    Created config virtual-service/default/reviews at revision ...
    Created config destination-rule/default/reviews at revision ...

Deploy the ``ratings v1`` and ``reviews v2`` services:

.. parsed-literal::

    $ for service in ratings-v1 reviews-v2; do \\
          curl -s \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-${service}.yaml | \\
          istioctl kube-inject -f - | \\
          kubectl create --validate=false -f - ; done
    service "ratings" created
    ciliumnetworkpolicy "ratings-v1" created
    deployment "ratings-v1" created
    ciliumnetworkpolicy "reviews-v2" created   
    deployment "reviews-v2" created

Check the progress of the deployment (every service should have an
``AVAILABLE`` count of ``1``):

::

    $ kubectl get deployments -n default
    NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    details-v1       1         1         1            1           6m
    productpage-v1   1         1         1            1           6m
    ratings-v1       1         1         1            1           57s
    reviews-v1       1         1         1            1           6m
    reviews-v2       1         1         1            1           57s

Check in your web browser that no stars are appearing in the Book
Reviews, even after refreshing the page several times.  This indicates
that all reviews are retrieved from ``reviews v1`` and none from
``reviews v2``.

.. image:: images/istio-bookinfo-reviews-v1.png
   :scale: 50 %
   :align: center

The ``ratings-v1`` CiliumNetworkPolicy explicitly whitelists access
to the ``ratings`` API only from ``productpage`` and ``reviews v2``:

.. literalinclude:: ../../examples/kubernetes-istio/bookinfo-ratings-v1-policy.yaml

Check that ``reviews v1`` may not be able to access the ``ratings``
service, even if it were compromised or suffered from a bug, by
running ``curl`` from within the pod:

::

    $ export POD_REVIEWS_V1=`kubectl get pods -n default -l app=reviews,version=v1 -o jsonpath='{.items[0].metadata.name}'`
    $ kubectl exec ${POD_REVIEWS_V1} -c istio-proxy -ti -- curl --connect-timeout 5 --fail http://ratings:9080/ratings/0
    curl: (22) The requested URL returned error: 503 Service Unavailable
    command terminated with exit code 22

Update the Istio route rule to send 50% of ``reviews`` traffic to
``v1`` and 50% to ``v2``:

.. literalinclude:: ../../examples/kubernetes-istio/route-rule-reviews-v1-v2.yaml

.. image:: images/istio-bookinfo-reviews-v2-route-to-v1-and-v2.png
   :scale: 75 %
   :align: center

Apply this route rule:
		    
.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/route-rule-reviews-v1-v2.yaml | \\
          istioctl replace -f -
    Updated config virtual-service/default/reviews to revision ...

Check in your web browser that stars are appearing in the Book Reviews
roughly 50% of the time.  This may require refreshing the page for a
few seconds to observe.  Queries to ``reviews v2`` result in reviews
containing ratings displayed as black stars:

.. image:: images/istio-bookinfo-reviews-v2.png
   :scale: 50 %
   :align: center

Finally, update the route rule to send 100% of ``reviews`` traffic to
``v2``:

.. literalinclude:: ../../examples/kubernetes-istio/route-rule-reviews-v2.yaml

.. image:: images/istio-bookinfo-reviews-v2-route-to-v2.png
   :scale: 75 %
   :align: center

Apply this route rule:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/route-rule-reviews-v2.yaml | \\
          istioctl replace -f -
    Updated config virtual-service/default/reviews to revision ...

Refresh the product page in your web browser several times to verify
that stars are now appearing in the Book Reviews on every page
refresh.  All the reviews are now retrieved from ``reviews v2`` and
none from ``reviews v1``.

Step 5: Deploy the Product Page Service V2
==========================================

We will now deploy version ``v2`` of the ``productpage`` service,
which brings two changes:

- It is deployed with a more restrictive CiliumNetworkPolicy, which
  restricts access to a subset of the HTTP URLs, at Layer-7.
- It implements a new authentication audit log into Kafka.

.. image:: images/istio-bookinfo-productpage-v2-kafka.png
   :scale: 75 %
   :align: center

The policy for ``v1`` currently allows read access to the full HTTP
REST API, under the ``/api/v1`` HTTP URI path:

- ``/api/v1/products``: Returns the list of books and their details.
- ``/api/v1/products/<id>``: Returns details about a specific book.
- ``/api/v1/products/<id>/reviews``: Returns reviews for a specific
  book.
- ``/api/v1/products/<id>/ratings``: Returns ratings for a specific
  book.

Check that the full REST API is currently accessible in ``v1`` and
returns valid JSON data:

::

    $ export PRODUCTPAGE=`minikube service istio-ingressgateway -n istio-system --url | head -n 1`
    $ for APIPATH in /api/v1/products /api/v1/products/0 /api/v1/products/0/reviews /api/v1/products/0/ratings; do echo ; curl -s -S "${PRODUCTPAGE}${APIPATH}" ; echo ; done

    [{"descriptionHtml": "<a href=\"https://en.wikipedia.org/wiki/The_Comedy_of_Errors\">Wikipedia Summary</a>: The Comedy of Errors is one of <b>William Shakespeare's</b> early plays. It is his shortest and one of his most farcical comedies, with a major part of the humour coming from slapstick and mistaken identity, in addition to puns and word play.", "id": 0, "title": "The Comedy of Errors"}]

    {"publisher": "PublisherA", "language": "English", "author": "William Shakespeare", "id": 0, "ISBN-10": "1234567890", "ISBN-13": "123-1234567890", "year": 1595, "type": "paperback", "pages": 200}

    {"reviews": [{"reviewer": "Reviewer1", "rating": {"color": "black", "stars": 5}, "text": "An extremely entertaining play by Shakespeare. The slapstick humour is refreshing!"}, {"reviewer": "Reviewer2", "rating": {"color": "black", "stars": 4}, "text": "Absolutely fun and entertaining. The play lacks thematic depth when compared to other plays by Shakespeare."}], "id": "0"}

    {"ratings": {"Reviewer2": 4, "Reviewer1": 5}, "id": 0}

We realized that the REST API to get the book reviews and ratings was
meant only for consumption by other internal services, and will be
blocked from external clients using the updated Layer-7
CiliumNetworkPolicy in ``productpage v2``, i.e. only the
``/api/v1/products`` and ``/api/v1/products/<id>`` HTTP URLs will be
whitelisted:

.. literalinclude:: ../../examples/kubernetes-istio/bookinfo-productpage-v2-policy.yaml

Because ``productpage v2`` sends messages into Kafka, we must first
deploy a Kafka broker:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/kafka-v1-destrule.yaml | \\
          istioctl create -f -
    Created config destination-rule/default/kafka-disable-mtls at revision ...

.. TODO: Re-enable sidecar injection after we support Kafka with mTLS.
    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/kafka-v1.yaml | \\
          istioctl kube-inject -f - | \\
          kubectl create --validate=false -f -

.. parsed-literal::

    $ kubectl create -f \ |SCM_WEB|\/examples/kubernetes-istio/kafka-v1.yaml
    service "kafka" created
    ciliumnetworkpolicy "kafka-authaudit" created
    statefulset "kafka-v1" created
    ciliumnetworkpolicy "kafka-from-init" created

Wait until the ``kafka-v1-0`` pod is ready, i.e. until it has a
``READY`` count of ``1/1``:

::

    $ kubectl get pods -n default -l app=kafka
    NAME         READY     STATUS    RESTARTS   AGE
    kafka-v1-0   1/1       Running   0          21m

Create the ``authaudit`` Kafka topic, which will be used by
``productpage v2``:

::

    $ kubectl exec kafka-v1-0 -c kafka -- bash -c '/opt/kafka_2.11-0.10.1.0/bin/kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic authaudit --partitions 1 --replication-factor 1'
    Created topic "authaudit".

We are now ready to deploy ``productpage v2``.

Create the ``productpage v2`` service and its updated
CiliumNetworkPolicy and delete ``productpage v1``:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-productpage-v2.yaml | \\
          istioctl kube-inject -f - | \\
          kubectl create --validate=false -f -
    ciliumnetworkpolicy "productpage-v2" created
    deployment "productpage-v2" created

    $ kubectl delete -f \ |SCM_WEB|\/examples/kubernetes-istio/bookinfo-productpage-v1.yaml
    ciliumnetworkpolicy "productpage-v1" deleted
    deployment "productpage-v1" deleted

``productpage v2`` implements an authorization audit logging.  On
every user login or logout, it produces into Kafka topic ``authaudit``
a JSON-formatted message which contains the following information:

- event: ``login`` or ``logout``
- username
- client IP address
- timestamp

To observe the Kafka messages sent by ``productpage``, we will run an
additional ``authaudit-logger`` service.  This service fetches and
prints out all messages from the ``authaudit`` Kafka topic.  Start
this service:

.. parsed-literal::

    $ curl -s \ |SCM_WEB|\/examples/kubernetes-istio/authaudit-logger-v1.yaml | \\
          istioctl kube-inject -f - | \\
          kubectl apply --validate=false -f -
    deployment "authaudit-logger-v1" created

Check the progress of the deployment (every service should have an
``AVAILABLE`` count of ``1``):

::

    $ kubectl get deployments -n default
    NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
    authaudit-logger-v1   1         1         1            1           20s
    details-v1            1         1         1            1           22m
    productpage-v2        1         1         1            1           4m
    ratings-v1            1         1         1            1           19m
    reviews-v1            1         1         1            1           22m
    reviews-v2            1         1         1            1           19m

Check that the product REST API is still accessible, and that Cilium
now denies at Layer-7 any access to the reviews and ratings REST API:

::

    $ export PRODUCTPAGE=`minikube service istio-ingressgateway -n istio-system --url | head -n 1`
    $ for APIPATH in /api/v1/products /api/v1/products/0 /api/v1/products/0/reviews /api/v1/products/0/ratings; do echo ; curl -s -S "${PRODUCTPAGE}${APIPATH}" ; echo ; done

    [{"descriptionHtml": "<a href=\"https://en.wikipedia.org/wiki/The_Comedy_of_Errors\">Wikipedia Summary</a>: The Comedy of Errors is one of <b>William Shakespeare's</b> early plays. It is his shortest and one of his most farcical comedies, with a major part of the humour coming from slapstick and mistaken identity, in addition to puns and word play.", "id": 0, "title": "The Comedy of Errors"}]

    {"publisher": "PublisherA", "language": "English", "author": "William Shakespeare", "id": 0, "ISBN-10": "1234567890", "ISBN-13": "123-1234567890", "year": 1595, "type": "paperback", "pages": 200}

    Access denied


    Access denied

This demonstrated that requests to the
``/api/v1/products/<id>/reviews`` and
``/api/v1/products/<id>/ratings`` URIs now result in Cilium returning
``HTTP 403 Forbidden`` HTTP responses.

Every login and logout on the product page will result in a line in
this service's log:

::

    $ export POD_LOGGER_V1=`kubectl get pods -n default -l app=authaudit-logger,version=v1 -o jsonpath='{.items[0].metadata.name}'`
    $ kubectl logs ${POD_LOGGER_V1} -c authaudit-logger
    ...
    {"timestamp": "2017-12-04T09:34:24.341668", "remote_addr": "10.15.28.238", "event": "login", "user": "richard"}
    {"timestamp": "2017-12-04T09:34:40.943772", "remote_addr": "10.15.28.238", "event": "logout", "user": "richard"}
    {"timestamp": "2017-12-04T09:35:03.096497", "remote_addr": "10.15.28.238", "event": "login", "user": "gilfoyle"}
    {"timestamp": "2017-12-04T09:35:08.777389", "remote_addr": "10.15.28.238", "event": "logout", "user": "gilfoyle"}

As you can see, the user-identifiable information sent by
``productpage`` in every Kafka message is sensitive, so access to this
Kafka topic must be protected using Cilium.  The CiliumNetworkPolicy
configured on the Kafka broker enforces that:

- only ``productpage v2`` is allowed to produce messages into the
  ``authaudit`` topic;
- only ``authaudit-logger`` can fetch messages from this topic;
- no service can access any other topic.

.. literalinclude:: ../../examples/kubernetes-istio/kafka-v1-policy.yaml

Check that Cilium prevents the ``authaudit-logger`` service from
writing into the ``authaudit`` topic (enter a message followed by
ENTER, e.g. ``test message``):

::

    $ export POD_LOGGER_V1=`kubectl get pods -n default -l app=authaudit-logger,version=v1 -o jsonpath='{.items[0].metadata.name}'`
    $ kubectl exec ${POD_LOGGER_V1} -c authaudit-logger -ti -- /opt/kafka_2.11-0.10.1.0/bin/kafka-console-producer.sh --broker-list=kafka:9092 --topic=authaudit
    test message
    [2017-12-07 02:13:47,020] ERROR Error when sending message to topic authaudit with key: null, value: 12 bytes with error: (org.apache.kafka.clients.producer.internals.ErrorLoggingCallback)
    org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [authaudit]

This demonstrated that Cilium sent a response with an authorization
error for any ``Produce`` request from this service.

Create another topic named ``credit-card-payments``, meant to transmit
highly-sensitive credit card payment requests:

::

    $ kubectl exec kafka-v1-0 -c kafka -- bash -c '/opt/kafka_2.11-0.10.1.0/bin/kafka-topics.sh --zookeeper localhost:2181/kafka --create --topic credit-card-payments --partitions 1 --replication-factor 1'
    Created topic "credit-card-payments".

Check that Cilium prevents the ``authaudit-logger`` service from
fetching messages from this topic:

::

    $ export POD_LOGGER_V1=`kubectl get pods -n default -l app=authaudit-logger,version=v1 -o jsonpath='{.items[0].metadata.name}'`
    $ kubectl exec ${POD_LOGGER_V1} -c authaudit-logger -ti -- /opt/kafka_2.11-0.10.1.0/bin/kafka-console-consumer.sh --bootstrap-server=kafka:9092 --topic=credit-card-payments
    [2017-12-07 03:08:54,513] WARN Not authorized to read from topic credit-card-payments. (org.apache.kafka.clients.consumer.internals.Fetcher)
    [2017-12-07 03:08:54,517] ERROR Error processing message, terminating consumer process:  (kafka.tools.ConsoleConsumer$)
    org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [credit-card-payments]
    Processed a total of 0 messages

This demonstrated that Cilium sent a response with an authorization
error for any ``Fetch`` request from this service for any topic other
than ``authaudit``.

Step 6: Clean Up
================

You have now installed Cilium and Istio, deployed a demo app, and
tested both Cilium's L3-L7 network security policies and Istio's
service route rules.  To clean up, run:

::

    $ minikube delete

After this, you can re-run the tutorial from Step 0.
