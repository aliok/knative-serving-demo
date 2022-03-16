# Knative Serving Demo

## Prerequisites

* Docker
* `kind`: https://kind.sigs.k8s.io/
* `kubectl`: https://kubernetes.io/docs/tasks/tools/
* `stern`: https://github.com/wercker/stern
* `hey`: https://github.com/rakyll/hey

## Prepare

Start your cluster:

```bash
kind create cluster
```

Install Knative Serving:

```bash
./hack/01-kn-serving.sh
```

## Run the demo

Create the namespace and the Knative Serving service:
```bash
kubectl apply -f config/01-namespace.yaml
kubectl apply -f config/02-service.yaml
```

To invoke the service, we need to see the address of the Knative Service we just created.
To do so, note the URL by running the command below:

```bash
kubectl get ksvc -n my-namespace request-logger

NAME      URL                                       LATESTCREATED   LATESTREADY     READY   REASON
request-logger   http://request-logger.my-namespace.example.com   request-logger-00001   request-logger-00001   True
```

By default, `kind` uses `example.com` as the root domain. So, the service's URL is http://request-logger.my-namespace.example.com.

We now need to send requests to Knative ingress, Kourier. However, exposing it on `kind` is hard.
So, we'll use port forwarding.

```bash
kubectl port-forward -n kourier-system service/kourier 12345:80
```

In a separate terminal, run this command to invoke the service:
```bash
curl -H "Host: request-logger.my-namespace.example.com" 0.0.0.0:12345
```

This application does not produce any HTTP response, but it logs the request to the standard output.

To see the output, do this:

```bash
request-logger-00001-deployment-cd56f756f-9hqwg user-container =======================
request-logger-00001-deployment-cd56f756f-9hqwg user-container Request headers:
request-logger-00001-deployment-cd56f756f-9hqwg user-container { host: 'request-logger.my-namespace.example.com',
request-logger-00001-deployment-cd56f756f-9hqwg user-container   'user-agent': 'curl/7.66.0',
request-logger-00001-deployment-cd56f756f-9hqwg user-container   accept: '*/*',
request-logger-00001-deployment-cd56f756f-9hqwg user-container   forwarded: 'for=10.244.0.10;proto=http',
request-logger-00001-deployment-cd56f756f-9hqwg user-container   'k-proxy-request': 'activator',
request-logger-00001-deployment-cd56f756f-9hqwg user-container   'x-forwarded-for': '10.244.0.10, 10.244.0.5',
request-logger-00001-deployment-cd56f756f-9hqwg user-container   'x-forwarded-proto': 'http',
request-logger-00001-deployment-cd56f756f-9hqwg user-container   'x-request-id': '302c54b4-84ab-4fe8-a5e0-45668d289a74' }
request-logger-00001-deployment-cd56f756f-9hqwg user-container
request-logger-00001-deployment-cd56f756f-9hqwg user-container Request body - raw:
request-logger-00001-deployment-cd56f756f-9hqwg user-container {}
request-logger-00001-deployment-cd56f756f-9hqwg user-container
request-logger-00001-deployment-cd56f756f-9hqwg user-container Request body - to string:
request-logger-00001-deployment-cd56f756f-9hqwg user-container [object Object]
request-logger-00001-deployment-cd56f756f-9hqwg user-container =======================
request-logger-00001-deployment-cd56f756f-9hqwg user-container
request-logger-00001-deployment-cd56f756f-9hqwg user-container SLEEP 100 ms
```

Source code for the request-logger application is here: https://github.com/aliok/knative-serverless-event-processing-demo/tree/main/request-logger

See the resources created:
```bash

❯ kubectl get -n my-namespace ksvc
NAME      URL                                       LATESTCREATED   LATESTREADY     READY   REASON
request-logger   http://request-logger.my-namespace.example.com   request-logger-00001   request-logger-00001   True

❯ kubectl get -n my-namespace revision
NAME            CONFIG NAME   K8S SERVICE NAME   GENERATION   READY   REASON
request-logger-00001   request-logger       request-logger-00001      1            True

❯ kubectl get -n my-namespace route
NAME      URL                                       READY   REASON
request-logger   http://request-logger.my-namespace.example.com   True

❯ kubectl get -n my-namespace configuration
NAME      LATESTCREATED   LATESTREADY     READY   REASON
request-logger   request-logger-00001   request-logger-00001   True
```

Now let's change the `LATENCY` env var.

```bash
kubectl apply -f config/03-service-updated.yaml
```
Now your call will wait a bit more before returning.

Let's inspect the resources again:


```bash
❯ kubectl get -n my-namespace ksvc
NAME             URL                                              LATESTCREATED          LATESTREADY            READY   REASON
request-logger   http://request-logger.my-namespace.example.com   request-logger-00002   request-logger-00002   True

❯ kubectl get -n my-namespace revision                                                                                                                                                                                                                                                                               ─╯
NAME                   CONFIG NAME      K8S SERVICE NAME       GENERATION   READY   REASON
request-logger-00001   request-logger   request-logger-00001   1            True
request-logger-00002   request-logger   request-logger-00002   2            True

❯ kubectl get -n my-namespace route
NAME             URL                                              READY   REASON
request-logger   http://request-logger.my-namespace.example.com   True

❯ kubectl get -n my-namespace configuration
NAME             LATESTCREATED          LATESTREADY            READY   REASON
request-logger   request-logger-00002   request-logger-00002   True
```

Scaling to N:

```bash
hey -c 50 -z 10s -host request-logger.my-namespace.example.com http://localhost:12345

```

You will see a lot of pods created.

## Clean up

```bash
kubectl delete -f config/02-service.yaml
kubectl delete -f config/01-namespace.yaml

kind delete cluster
```
