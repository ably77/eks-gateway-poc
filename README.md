**NOTE**
This runbook assumes that you already have Gloo Mesh installed as described in `main`. Please refer to the `main` branch for instructions to install Gloo Platform if it has not already been deployed.

![Gloo Mesh Enterprise](images/gloo-mesh-enterprise.png)
# <center>Gloo Mesh Workshop</center>



## Table of Contents
* [Introduction](#introduction)
* [Lab 0 - Prerequisites](#lab-0---prerequisites-)
* [Lab 1 - Setting up your Environment Variables](#lab-1---setting-up-your-environment-variables-)
* [Lab 2 - Deploy Revision based Istio using Helm](#lab-2---deploy-revision-based-istio-using-helm-)
* [Lab 3 - Reconfigure Previous Labs](#lab-3---reconfigure-previous-labs-)
* [Lab 4 - Using Passthrough External Auth](#lab-4---using-passthrough-external-auth-)
* [Lab 5 - Enable JWT Validation at the Gateway](#lab-5---enable-jwt-validation-at-the-gateway-)

* [Load Test 1](#load-test-1-)



## Introduction <a name="introduction"></a>

[Gloo Mesh Enterprise](https://www.solo.io/products/gloo-mesh/) is a management plane which makes it easy to operate [Istio](https://istio.io) on one or many Kubernetes clusters deployed anywhere (any platform, anywhere).

### Istio support

The Gloo Mesh Enterprise subscription includes end to end Istio support:

- Upstream first
- Specialty builds available (FIPS, ARM, etc)
- Long Term Support (LTS) N-4 
- Critical security patches
- Production break-fix
- One hour SLA Severity 1
- Install / upgrade
- Architecture and operational guidance, best practices

### Gloo Mesh overview

Gloo Mesh provides many unique features, including:

- multi-tenancy based on global workspaces
- zero trust enforcement
- global observability (centralized metrics and access logging)
- simplified cross cluster communications (using virtual destinations)
- advanced gateway capabilities (oauth, jwt, transformations, rate limiting, web application firewall, ...)

![Gloo Mesh graph](images/gloo-mesh-graph.png)

### Want to learn more about Gloo Mesh

You can find more information about Gloo Mesh in the official documentation:

[https://docs.solo.io/gloo-mesh/latest/](https://docs.solo.io/gloo-mesh/latest/)

## Lab 0 - Prerequisites <a name="lab-0---prerequisites-"></a>

### HIGHLY RECOMMENDED: Read Before Starting the Labs Below:
Before you start running through the Labs below, it is highly recommended to read the About and Concepts sections linked below. Here you will begin to learn the high level value add that Gloo Mesh and Gloo Gateway brings to your Kubernetes architecture. Understanding of the concepts and architecture of Gloo Gateway will help us greatly as we move along the hands-on labs.

[Gloo Mesh Docs - Concepts](https://docs.solo.io/gloo-mesh-enterprise/main/concepts/)

[Gloo Mesh Docs - About Gloo Gateway](https://docs.solo.io/gloo-gateway/latest/concepts/)

[Gloo Mesh Docs - Routing requests Overview](https://docs.solo.io/gloo-mesh-enterprise/main/routing/)

[Gloo Mesh Docs - Gloo Mesh Policies](https://docs.solo.io/gloo-mesh-enterprise/main/policies/)

[Gloo Mesh Docs - Workspaces](https://docs.solo.io/gloo-mesh-enterprise/main/concepts/multi-tenancy/)

### Prerequisites
This POC runbook assumes the following:
- 1x clusters deployed on EKS (m5.2xlarge instance size)
- AWS NLB Controller deployed on cluster
- Gloo Platform components have already been deployed per instructions on `main` branch
  - gateways workspace created
  - httpbin workspace created
- Httpbin app is deployed (will reconfigure)
- Httpbin routing is already configured (Lab 8 in `main`)

## Lab 1 - Setting up your Environment Variables <a name="lab-1---deploy-a-kind-cluster-"></a>
Set the GLOO_MESH_LICENSE_KEY environment variable before starting:
```bash
export GLOO_MESH_LICENSE_KEY="<INSERT_LICENSE_KEY_HERE>"
```

Set the context environment variables:
```bash
export CLUSTER1=cluster1
```

You also need to rename the Kubernete contexts of each Kubernetes cluster to match `cluster1`

Here is an example showing how to rename a Kubernetes context:
```
kubectl config rename-context <context to rename> <new context name>
```

Run the following command to make `cluster1` the current cluster.
```bash
kubectl config use-context ${CLUSTER1}
```

> If you prefer to use the existing context name, just set the variables as so:
> ```
> export CLUSTER1=<cluster1_context>
> ```
>
> Note: these variables may need to be set in each new terminal used


## Additional Cleanup Steps for policies from istio-takeover branch

If you have already walked through the `istio-takeover` branch in its entirety, you can clean up the environment policies with the following commands

```bash
# grpc labs
kubectl --context ${CLUSTER1} delete -n currency RateLimitClientConfig currency
kubectl --context ${CLUSTER1} delete -n gloo-mesh-addons RateLimitServerConfig currency
kubectl --context ${CLUSTER1} delete -n currency RateLimitPolicy currency
kubectl --context ${CLUSTER1} delete -n currency RateLimitServerSettings rate-limit-server
kubectl --context ${CLUSTER1} delete -n currency RouteTable currency
kubectl --context ${CLUSTER1} delete -n currency ExtAuthPolicy currency-extauth
kubectl --context ${CLUSTER1} delete -n currency Deployment currencyservice
kubectl --context ${CLUSTER1} delete -n currency Service currencyservice
kubectl --context ${CLUSTER1} delete -n currency ServiceAccount currency
kubectl --context ${CLUSTER1} delete -n currency WorkspaceSettings currency
kubectl --context ${CLUSTER1} delete -n currency Workspace currency

# main route table in istio-system
kubectl --context ${CLUSTER1} delete -n istio-system RouteTable main

# httpbin labs
kubectl --context ${CLUSTER1} delete -n httpbin RouteTable httpbin
kubectl --context ${CLUSTER1} delete -n httpbin ExtAuthPolicy passthrough-auth
kubectl --context ${CLUSTER1} delete -n httpbin ExtAuthServer cluster1-ext-auth-server
kubectl --context ${CLUSTER1} delete -f example-config/extauth-w-sidecar-2.2.6.yaml
kubectl --context ${CLUSTER1} delete -n httpbin RateLimitClientConfig httpbin
kubectl --context ${CLUSTER1} delete -n gloo-mesh-addons RateLimitServerConfig httpbin
kubectl --context ${CLUSTER1} delete -n httpbin RateLimitPolicy httpbin
kubectl --context ${CLUSTER1} delete -n httpbin RateLimitServerSettings rate-limit-server
kubectl --context ${CLUSTER1} delete -n httpbin ConfigMap httpbin-opa
kubectl --context ${CLUSTER1} delete -n httpbin ExtAuthPolicy httpbin-opa
kubectl --context ${CLUSTER1} delete -n httpbin ExtAuthPolicy httpbin-extauth
kubectl --context ${CLUSTER1} delete -n httpbin Secret oidc-client-secret
```

## Lab 2 - Deploy Revision based Istio using Helm <a name="lab-2---deploy-revision-based-istio-using-helm-"></a>

**NOTE**
This runbook assumes that you already have Gloo Mesh installed as described in `main`. Please refer to the `main` branch for instructions to install Gloo Platform if it has not already been deployed.

In this version of the runbook we are going to deploy Istio using Helm.

First of all, let's Download the Istio release 1.14.5:
```bash
export ISTIO_VERSION=1.14.5
curl -L https://istio.io/downloadIstio | sh -
```

Then, you need to create the `istio-gateways` namespace
```bash
kubectl --context ${CLUSTER1} create ns istio-gateways
```

Now, let's deploy the Istio control plane on the cluster:
```bash
helm --kube-context=${CLUSTER1} upgrade --install istio-1.14.5 ./istio-1.14.5/manifests/charts/istio-control/istio-discovery -n istio-system --values - <<EOF
revision: 1-14
global:
  meshID: mesh1
  multiCluster:
    clusterName: cluster1
  network: network1
  hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
  tag: 1.14.5-solo
meshConfig:
  trustDomain: cluster1
  accessLogFile: /dev/stdout
  enableAutoMtls: true
  defaultConfig:
    envoyAccessLogService:
      address: gloo-mesh-agent.gloo-mesh:9977
    proxyMetadata:
      ISTIO_META_DNS_CAPTURE: "true"
      ISTIO_META_DNS_AUTO_ALLOCATE: "true"
pilot:
  env:
    PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES: "false"
EOF
```

After that, you can deploy the gateway(s):
```bash
kubectl --context ${CLUSTER1} label namespace istio-gateways istio.io/rev=1-14

helm --kube-context=${CLUSTER1} upgrade --install istio-ingressgateway ./istio-1.14.5/manifests/charts/gateways/istio-ingress -n istio-gateways --values - <<EOF
global:
  hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
  tag: 1.14.5-solo
gateways:
  istio-ingressgateway:
    name: istio-ingressgateway
    namespace: istio-gateways
    labels:
      istio: solo-ingressgateway
    injectionTemplate: gateway
    type: LoadBalancer
    ports:
    - name: http2
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
    serviceAnnotations:
      meta.helm.sh/release-name: istio-ingressgateway
      meta.helm.sh/release-namespace: istio-gateways
      # uncomment below if using NLB controller
      #service.beta.kubernetes.io/aws-load-balancer-backend-protocol: TCP
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "15021"
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: tcp
      #service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
      #service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
      #service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
      #service.beta.kubernetes.io/aws-load-balancer-type: nlb-ip
      # uncomment below if using classic LB controller
      service.beta.kubernetes.io/aws-load-balancer-type: external
      service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
      service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
EOF
```

Run the following command until all the Istio Pods are ready:
```bash
kubectl --context ${CLUSTER1} get pods -n istio-system && kubectl --context ${CLUSTER1} get pods -n istio-gateways
```

When they are ready, you should get this output:
```
NAME                      READY   STATUS    RESTARTS   AGE
istiod-5c669bcf6f-2hn6c   1/1     Running   0          3m7s
NAME                                     READY   STATUS    RESTARTS   AGE
istio-ingressgateway-744fcf4fb-5dc7q     1/1     Running   0          2m44s
```

Set the environment variable for the service corresponding to the Istio Ingress Gateway of the cluster(s):

```bash
export ENDPOINT_HTTP_GW_CLUSTER1=$(kubectl --context ${CLUSTER1} -n istio-gateways get svc -l istio=solo-ingressgateway -o jsonpath='{.items[0].status.loadBalancer.ingress[0].*}'):80
export ENDPOINT_HTTPS_GW_CLUSTER1=$(kubectl --context ${CLUSTER1} -n istio-gateways get svc -l istio=solo-ingressgateway -o jsonpath='{.items[0].status.loadBalancer.ingress[0].*}'):443
export HOST_GW_CLUSTER1=$(echo ${ENDPOINT_HTTP_GW_CLUSTER1} | cut -d: -f1)
```

If you are using k3d LB integration you can just set the following instead
```
export ENDPOINT_HTTP_GW_CLUSTER1="localhost:80"
export ENDPOINT_HTTPS_GW_CLUSTER1="localhost:443"
```

Check that the variables have correct values:
```bash
echo $ENDPOINT_HTTP_GW_CLUSTER1
echo $ENDPOINT_HTTPS_GW_CLUSTER1
echo $HOST_GW_CLUSTER1
```

## reconfigure workspaces

We will need to reconfigure our gateways workspace to use the `istio-gateways` namespace instead of `istio-system`

```bash
kubectl apply --context ${CLUSTER1} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: Workspace
metadata:
  name: gateways
  namespace: gloo-mesh
spec:
  workloadClusters:
  - name: cluster1
    namespaces:
    - name: istio-gateways
    - name: gloo-mesh-addons
EOF
```

Then, we will update our `WorkspaceSettings` Kubernetes object so that it is also a part of the `istio-gateways` namespace. We can also delete the old `WorkspaceSettings`

```bash
kubectl delete --context ${CLUSTER1} workspacesettings -n istio-system gateways

kubectl apply --context ${CLUSTER1} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: gateways
  namespace: istio-gateways
spec:
  importFrom:
  - workspaces:
    - selector:
        allow_ingress: "true"
    resources:
    - kind: SERVICE
    - kind: ALL
      labels:
        expose: "true"
  exportTo:
  - workspaces:
    - selector:
        allow_ingress: "true"
    resources:
    - kind: SERVICE
EOF
```

## Lab 3 - Reconfigure Previous Labs <a name="lab-3---reconfigure-previous-labs-"></a>

Since we already have the `httpbin` app deployed and configured by our existing Istio, we going to re-deploy the httpbin application to use our newly provisioned `1-14` revisioned Istio.

This involves a few steps:
- Re-label the namespace to have an istio revision label `istio.io/rev=1-14`
- Restart httpbin deployment to pick up a new sidecar from the revision-based Istio control plane

Run the following commands to deploy the httpbin app named `in-mesh` on `cluster1`. 
```bash
kubectl --context ${CLUSTER1} label namespace httpbin istio-injection-
kubectl --context ${CLUSTER1} label namespace httpbin istio.io/rev=1-14 --overwrite

kubectl --context ${CLUSTER1} rollout restart deploy/in-mesh -n httpbin
```

You can follow the progress using the following command:

```
kubectl --context ${CLUSTER1} -n httpbin get pods
```

```
% kubectl --context ${CLUSTER1} -n httpbin get pods
NAME                       READY   STATUS        RESTARTS   AGE
in-mesh-977b6c948-bvhqx    2/2     Running       0          6s
in-mesh-84b6978978-6kpt2   2/2     Terminating   0          6m51s
```


Now we can reconfigure our VirtualGateway to use our new gateway label. Additionally we will need to apply our gateway TLS secret in the `istio-gateways` namespace as well

Let's first create a private key and a self-signed certificate if you haven't done so already

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
   -keyout tls.key -out tls.crt -subj "/CN=*"
```

Then, you have to store them in a Kubernetes secrets running the following commands:

```bash
kubectl --context ${CLUSTER1} -n istio-gateways create secret generic tls-secret \
--from-file=tls.key=tls.key \
--from-file=tls.crt=tls.crt
```

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: VirtualGateway
metadata:
  name: north-south-gw
  namespace: istio-gateways
spec:
  workloads:
    - selector:
        labels:
          istio: solo-ingressgateway
        cluster: cluster1
  listeners: 
    - http: {}
      port:
        number: 80
# ---------------- Redirect to https --------------------
      httpsRedirect: true
# -------------------------------------------------------
    - http: {}
# ---------------- SSL config ---------------------------
      port:
        number: 443
      tls:
        mode: SIMPLE
        secretName: tls-secret
# -------------------------------------------------------
      allowedRouteTables:
        - host: '*'
EOF
```

And finally let's reconfigure our main `RouteTable` to be a part of `istio-gateways` as well

```bash
kubectl --context ${CLUSTER1} delete routetable -n istio-system main

kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: main
  namespace: istio-gateways
spec:
  hosts:
    - '*'
  virtualGateways:
    - name: north-south-gw
      namespace: istio-gateways
      cluster: cluster1
  workloadSelectors: []
  http:
    - name: root
      matchers:
      - uri:
          prefix: /
      delegate:
        routeTables:
          - labels:
              expose: "true"
EOF
```

Finally, re-apply the httpbin `RouteTable` if you deleted it earlier when cleaning up

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  http:
    - name: httpbin
      labels:
        auth: passthrough
        transform: body
      matchers:
      - uri:
          exact: /
      - uri:
          prefix: /get
      - uri:
          prefix: /anything
      - uri:
          prefix: /callback
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Re-discover our new gateway endpoint by locating the endpoint of our gateway which has the label `istio=solo-ingressgateway` in the `istio-gateways` namespace

```bash
export ENDPOINT_HTTP_GW_CLUSTER1=$(kubectl --context ${CLUSTER1} -n istio-gateways get svc -l istio=solo-ingressgateway -o jsonpath='{.items[0].status.loadBalancer.ingress[0].*}'):80
export ENDPOINT_HTTPS_GW_CLUSTER1=$(kubectl --context ${CLUSTER1} -n istio-gateways get svc -l istio=solo-ingressgateway -o jsonpath='{.items[0].status.loadBalancer.ingress[0].*}'):443
export HOST_GW_CLUSTER1=$(echo ${ENDPOINT_HTTP_GW_CLUSTER1} | cut -d: -f1)
```

You can now access the `httpbin` application securely through the browser from our newly created gateway. Note that the IP or URL has changed in the process of migrating over

Get the URL to access the `httpbin` service using the following command:
```
echo "https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get"
```

## Lab 4 - Using Passthrough External Auth <a name="lab-4---using-passthrough-external-auth-"></a>

This example will be used to demonstrate how we can leverage [Passthrough External Auth](https://docs.solo.io/gloo-gateway/latest/policies/external-auth/passthrough/) to authenticate requests with an external auth server, in this case an OPA deployment.

Benefits of passthrough external auth: 
With passthrough external auth, you can integrate with existing auth implementations, while still being able to use other Gloo Gateway external auth implementations, such as OIDC and API key auth.

The [Passthrough External Auth](https://docs.solo.io/gloo-gateway/latest/policies/external-auth/passthrough/) example provides a sample `extauth-grpcservice` that is configured as a `Deployment`. A high level architecture diagram of the flow looks like this

![](images/extauth/passthrough1b.png)

In our lab example, we will continue to use this similar architecture used in the docs, but in this lab we will additionally demonstrate how to manipulate requests and responses to meet our traffic shaping requirements using `TransformationPolicy`

A high level architecture diagram of the flow looks like this

![](images/extauth/passthrough2b.png)

First, let's clean up any policies that may already exist from previous labs
```bash
kubectl --context ${CLUSTER1} -n httpbin delete ExtAuthPolicy httpbin-extauth
kubectl --context ${CLUSTER1} -n httpbin delete ExtAuthPolicy httpbin-opa
kubectl --context ${CLUSTER1} -n httpbin delete ExtAuthPolicy httpbin-keycloak-extauth
```

If we have already deployed the `ext-auth-server` as a part of Gloo Mesh Addons deployment, please update the image below using the following commands. If you  have not yet deployed the Gloo Mesh Addons, you can still continue to use the same commands below.

```bash
kubectl --context ${CLUSTER1} create ns gloo-mesh-addons
kubectl --context ${CLUSTER1} label namespace gloo-mesh-addons istio-injection-
kubectl --context ${CLUSTER1} label namespace gloo-mesh-addons istio.io/rev=1-14 --overwrite

helm upgrade --install gloo-mesh-agent-addons gloo-mesh-agent/gloo-mesh-agent \
--namespace gloo-mesh-addons \
--kube-context=${CLUSTER1} \
--version 2.2.6 \
--values - <<EOF
glooMeshAgent:
  enabled: false
rate-limiter:
  enabled: true
  rateLimiter:
    resources:
      requests:
        cpu: 750m
        memory: 1500Mi
ext-auth-service:
  enabled: true
  extAuth:
    image:
      pullPolicy: IfNotPresent
      registry: registry.hub.docker.com
      repository: ably77/ext-auth-service
      tag: amd64-ext-auth-service-0.35.0-poc
    resources:
      requests:
        cpu: 1500m
        memory: 500Mi
EOF

kubectl --context ${CLUSTER1} -n gloo-mesh-addons rollout restart deploy/ext-auth-service
kubectl --context ${CLUSTER1} -n gloo-mesh-addons rollout restart deploy/rate-limiter
kubectl --context ${CLUSTER1} -n gloo-mesh-addons rollout restart deploy/redis
```

For our testing, based on the provided throughput we should scale the replicas of our `ext-auth-service` to 3

```bash
kubectl --context ${CLUSTER1} -n gloo-mesh-addons scale deploy/ext-auth-service --replicas 3
```

Check to see that the `ext-auth-service` is deployed

```bash
kubectl --context ${CLUSTER1} get pods -n gloo-mesh-addons
```

output should look like this

```bash
% kubectl --context ${CLUSTER1} get pods -n gloo-mesh-addons
NAME                                READY   STATUS    RESTARTS   AGE
redis-6bb84c5647-nzd4v              2/2     Running   0          58s
ext-auth-service-7979d7685f-9lw2k   2/2     Running   0          58s
rate-limiter-f9c6598c5-tlgbc        2/2     Running   0          58s
ext-auth-service-7979d7685f-fqqnv   2/2     Running   0          13s
ext-auth-service-7979d7685f-2gpk7   2/2     Running   0          13s
```

### Deploying OPA

Next we will deploy OPA as our authorization server

```bash
kubectl apply --context ${CLUSTER1} -f - <<EOF
apiVersion: v1
data:
  policy.rego: cGFja2FnZSByYmFjCgppbXBvcnQgZnV0dXJlLmtleXdvcmRzLmlmCgpkZWZhdWx0IGFsbG93IDo9IGZhbHNlCgojIERlY29kZSBKV1QgZnJvbSBpbnB1dC5qd3QKY2xhaW1zIDo9IHBheWxvYWQgaWYgewoJdiA6PSBpbnB1dC5qd3QKCVtfLCBwYXlsb2FkLCBfXSA6PSBpby5qd3QuZGVjb2RlKHYpCn0KCiMgYWxsb3cgaWYgZGVjb2RlZCBKV1QgY29udGFpbnMgdGhlIHJpZ2h0IGdyb3VwIGNsYWltCmFsbG93IGlmIGNsYWltcy5ncm91cHMgPT0gWyJhZG1pbi1ncm91cCJdCgoK
kind: Secret
metadata:
  name: opa-policy
  namespace: gloo-mesh-addons
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  name: opa
  namespace: gloo-mesh-addons
  labels:
      app: opa
spec:
  ports:
  - port: 8181
    protocol: TCP
  selector:
      app: opa
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: opa
  namespace: gloo-mesh-addons
  labels:
    app: opa
spec:
  replicas: 1
  selector:
    matchLabels:
      app: opa
  template:
    metadata:
      labels:
        app: opa
    spec:
      containers:
        - name: opa
          image: openpolicyagent/opa:latest-envoy
          securityContext:
            runAsUser: 1111
          volumeMounts:
          - readOnly: true
            mountPath: /policy
            name: opa-policy
          args:
          - "run"
          - "--server"
          - "--set=default_decision=v1/data/rbac/allow"
          - "--addr=0.0.0.0:8181"
          - "--diagnostic-addr=0.0.0.0:8282"
          - "--log-level=debug"
          - "--set=decision_logs.console=true"
          - "--set=log-format=json-pretty"
          - "--ignore=.*"
          - "/policy/policy.rego"
          ports:
          - containerPort: 8181
          resources:
            requests:
              cpu: "2000m"
              memory: "2Gi"
          livenessProbe:
            httpGet:
              path: /health?plugins
              scheme: HTTP
              port: 8282
            initialDelaySeconds: 5
            periodSeconds: 5
          readinessProbe:
            httpGet:
              path: /health?plugins
              scheme: HTTP
              port: 8282
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: proxy-config
          configMap:
            name: proxy-config
        - name: opa-policy
          secret:
            secretName: opa-policy
EOF
```

The `opa-policy` secret contains a simple rego policy

```bash
package rbac

import future.keywords.if

default allow := false

# Decode JWT from input.jwt
claims := payload if {
	v := input.jwt
	[_, payload, _] := io.jwt.decode(v)
}

# allow if decoded JWT contains the right group claim
allow if claims.groups == ["admin-group"]
```

Check to see if the OPA server has been deployed

```bash
kubectl --context ${CLUSTER1} get pods -n gloo-mesh-addons
```

The output should look similar to below:

```bash
% kubectl --context ${CLUSTER1} get pods -n gloo-mesh-addons   
NAME                               READY   STATUS    RESTARTS   AGE
rate-limiter-64b64b779c-xrtsn      2/2     Running   0          29m
redis-578865fd78-rgjqm             2/2     Running   0          29m
ext-auth-service-76d8457d9-d69k9   2/2     Running   0          11m
opa-7f845fd897-t95l2               2/2     Running   0          20s
```

In order for us to leverage the `TransformationPolicy` built into Gloo Platform, we will need to apply an internal `RouteTable` for our OPA server

```bash
kubectl apply --context ${CLUSTER1} -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: opa-rt
  namespace: gloo-mesh-addons
spec:
  hosts:
    - 'opa.gloo-mesh-addons.svc.cluster.local'
  http:
    - name: opa
      matchers:
      - uri:
          prefix: /
      forwardTo:
        destinations:
        - ref:
            name: opa
            namespace: gloo-mesh-addons
          port:
            number: 8181
EOF
```

Now we can apply the required transformations in our diagram:

First the transformation from client > OPA passthrough input
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: TransformationPolicy
metadata:
  name: client-transformation
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        transform: body
  config:
    phase:
      preAuthz:
        priority: -5
    request:
      injaTemplate:
        body:
          text: '{"input": { "action": "{{ action }}", "context": { "resourceId": "{{ resourceId }}" }, "jwt": "{{ jwt }}"}}'
        extractors:
          resourceId:
            header: ':path'
            regex: '.*'
            subgroup: 0
          action:
            header: ':method'
            regex: '.*'
            subgroup: 0
          jwt:
            header: 'jwt'
            regex: '.*'
            subgroup: 0
          
EOF
```

We will need one other transformation for our POC to complete the expected OPA workflow. The current HTTP Passthrough implementation assumes the request is authorized if the server returns a OK (200) status code. When using HTTP Passthrough auth to an OPA container/sidecar using port 8181 (http mode) OPA will return a HTTP/1.1 200 OK status response to the ext-auth-server and authorize the request regardless of the OPA validation output result in the response body i.e. `{"result":true/false}`

In order to follow this OPA model, we can use the transformation capability again to modify the :status pseudo-header based on the `{"result":true/false}`

This E/W capability is currently planned for Gloo Platform 2.4.0, but in the meanwhile we can leverage the underlying Envoy Filter like so to complete our transformation workflow

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: transformation
  namespace: gloo-mesh-addons
spec:
  configPatches:
  - applyTo: HTTP_ROUTE
    match:
      context: SIDECAR_OUTBOUND
      routeConfiguration:
        vhost:
          route:
            name: opa-opa-rt.gloo-mesh-addons.cluster1
    patch:
      operation: MERGE
      value:
        typedPerFilterConfig:
          io.solo.transformation:
            '@type': type.googleapis.com/udpa.type.v1.TypedStruct
            typeUrl: envoy.api.v2.filter.http.RouteTransformations
            value:
              transformations:
              - responseMatch:
                  responseTransformation:
                    transformationTemplate:
                      headers:
                        :status:
                          text: '{% if default(result, "true") %}200{% else %}403{% endif %}'
                stage: 1
  - applyTo: HTTP_FILTER
    match:
      context: SIDECAR_OUTBOUND
      listener:
        filterChain:
          filter:
            name: envoy.filters.network.http_connection_manager
            subFilter:
              name: envoy.filters.http.router
    patch:
      operation: INSERT_BEFORE
      value:
        name: io.solo.transformation
        typedConfig:
          '@type': type.googleapis.com/udpa.type.v1.TypedStruct
          typeUrl: envoy.api.v2.filter.http.FilterTransformations
          value:
            stage: 1
  workloadSelector:
    labels:
    # we need this only on the client
      app: ext-auth-service
EOF
```

Here you can see the transformation logic being applied between ext-auth-service and OPA.

```bash
value:
  transformations:
  - responseMatch:
      responseTransformation:
        transformationTemplate:
          headers:
            :status:
              text: '{% if default(result, "true") %}200{% else %}403{% endif %}'
    stage: 1
```

Make sure our `ExtAuthServer` CRD is configured with `passthroughBody: true` which will allow us to passthrough the body input to OPA

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: ExtAuthServer
metadata:
  name: cluster1-ext-auth-server
  namespace: httpbin
spec:
  destinationServer:
    ref:
      cluster: cluster1
      name: ext-auth-service
      namespace: gloo-mesh-addons
    port:
      name: grpc
  requestBody: 
    maxRequestBytes: 4096
EOF
```

Next, we can apply our Passthrough Auth policy
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: passthrough-auth
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        auth: passthrough
  config:
    glooAuth:
      configs:
      - passThroughAuth:
          http:
            # internal opa route table
            url: http://opa.gloo-mesh-addons.svc.cluster.local/v1/data/rbac/allow
            request:
              passThroughBody: true
    server:
      name: cluster1-ext-auth-server
      namespace: httpbin
      cluster: cluster1
EOF
```

Lastly, we need to update our route table with our ExtAuthPolicy route label `auth: passthrough` and `transform: body` which will then apply our Passthrough `ExtAuthPolicy` as well as `TransformationPolicy` to our route

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: httpbin
  namespace: httpbin
  labels:
    expose: "true"
spec:
  http:
    - name: httpbin
      labels:
        auth: passthrough
        transform: body
      matchers:
      - uri:
          exact: /
      - uri:
          prefix: /get
      - uri:
          prefix: /anything
      - uri:
          prefix: /callback
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Now we should be able to test our passthrough ext auth by providing a valid `jwt` header in our request

First without - output should be `401`

```bash
curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get
```

output:

```bash
HTTP/2 401
date: Thu, 20 Apr 2023 00:09:07 GMT
server: istio-envoy
```

Then with - output should be `200`

```bash
curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get --header "jwt: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJncm91cHMiOlsiYWRtaW4tZ3JvdXAiXX0.8V_AFuGdtFk3FyDKbAjDPX5zoxh7RP0TeMyVg2ZUClg"
```

output:

```bash
HTTP/2 200 
server: istio-envoy
date: Tue, 09 May 2023 00:40:00 GMT
content-type: application/json
content-length: 824
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 5
```

If you take a look at the logs of our OPA deployment, we can see the `true/false` policy validation happening

Run the following command to view the logs

```bash
kubectl --context ${CLUSTER1} logs -n gloo-mesh-addons deploy/opa -f
```

Output should look similar to below

```bash
% kubectl --context ${CLUSTER1} logs -n gloo-mesh-addons deploy/opa

{"client_addr":"127.0.0.6:36771","level":"info","msg":"Sent response.","req_id":100,"req_method":"POST","req_path":"/v1/data/rbac/allow","resp_body":"{\"decision_id\":\"4c67c514-bf38-49dc-b205-0e45487a26a7\",\"result\":false}\n","resp_bytes":70,"resp_duration":0.447475,"resp_status":200,"time":"2023-05-09T00:42:11Z"}

{"client_addr":"127.0.0.6:36771","level":"info","msg":"Sent response.","req_id":105,"req_method":"POST","req_path":"/v1/data/rbac/allow","resp_body":"{\"decision_id\":\"bca7be21-8477-46e5-ae71-1497b45cf6d3\",\"result\":true}\n","resp_bytes":69,"resp_duration":0.404343,"resp_status":200,"time":"2023-05-09T00:42:46Z"}
```

Congrats, now we have just completed the full required flow of our OPA implementation by leveraging the capabilities of the platform!

## Lab 5 - Enable JWT Validation at the Gateway <a name="lab-5---enable-jwt-validation-at-the-gateway-"></a>

In this lab, we will enable JWT validation at the gateway using the Istio `RequestAuthentication` and `AuthorizationPolicy` CRDs. This lab uses the test token [JWT test](https://raw.githubusercontent.com/istio/istio/release-1.17/security/tools/jwt/samples/demo.jwt) and [JWKS endpoint](https://raw.githubusercontent.com/istio/istio/release-1.17/security/tools/jwt/samples/jwks.json) from the Istio code base. Note that the JWT must correspond to the JWKS endpoint you want to use. 

### Enable Permissive JWT Validation

First we can apply the `RequestAuthentication` policy. The policy below shows two methods of providing the jwks public key set

- jwks: JSON Web Key Set of public keys to validate signature of the JWT
- jwksUri: URL of the provider’s public key set to validate signature of the JWT.

Note: Only one of `jwksUri` and `jwks` should be used.

The following example uses the `jwksUri` method. However if you would like to test the `jwks` method, you can just uncomment the required parameters below and re-apply

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: "gateway-jwt-validation"
  namespace: istio-gateways
spec:
  selector:
    matchLabels:
      istio: solo-ingressgateway
  jwtRules:
  - issuer: "testing@secure.istio.io"
    # NOTE: only one of jwks or jwksUri can be used
    # uncomment to provide jwks inline
    #jwks: |
    #  { "{ "keys":[ {"e":"AQAB","kid":"DHFbpoIUqrY8t2zpA2qXfCmr5VO5ZEr4RzHU_-envvQ","kty":"RSA","n":"xAE7eB6qugXyCAG3yhh7pkDkT65pHymX-P7KfIupjf59vsdo91bSP9C8H07pSAGQO1MV_xFj9VswgsCg4R6otmg5PV2He95lZdHtOcU5DXIg_pbhLdKXbi66GlVeK6ABZOUW3WYtnNHD-91gVuoeJT_DwtGGcp4ignkgXfkiEm4sw-4sfb4qdt5oLbyVpmW6x9cfa7vs2WTfURiCrBoUqgBo_-4WTiULmmHSGZHOjzwa8WtrtOQGsAFjIbno85jp6MnGGGZPYZbDAa_b3y5u-YpW7ypZrvD8BgtKVjgtQgZhLAGezMt0ua3DRrWnKqTZ0BJ_EyxOGuHJrLsn00fnMQ"}]}" }]}
    # uncomment to provide jwksUri
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.17/security/tools/jwt/samples/jwks.json"
EOF
```

If you provide a token in the authorization header, its implicitly default location, Istio validates the token using the public key set, and rejects requests if the bearer token is invalid. However, take note that requests without tokens are accepted. To observe this behavior, retry the request without a token, with a bad token, and with a valid token:

Example curl command with no token - output should be `200`

```bash
curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get
```

output:

```
HTTP/2 200 
server: istio-envoy
date: Wed, 19 Apr 2023 18:31:29 GMT
content-type: application/json
content-length: 503
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 7
```

Example curl command with a bad token - output should be `401`

```bash
curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get --header "Authorization: Bearer badtoken"
```

output:

```
HTTP/2 401 
www-authenticate: Bearer realm="https://httpbin-local.glootest.com/get", error="invalid_token"
content-length: 79
content-type: text/plain
date: Wed, 19 Apr 2023 18:31:58 GMT
server: istio-envoy
```

Example curl command with valid token - output should be `200`

```bash
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.17/security/tools/jwt/samples/demo.jwt -s)

curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get --header "Authorization: Bearer $TOKEN"
```

output:

```
HTTP/2 200 
server: istio-envoy
date: Wed, 19 Apr 2023 18:32:18 GMT
content-type: application/json
content-length: 503
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 6
```

By default, Istio runs these Authentication policy checks in permissive mode. Meaning you can send a request if you provide a valid token or provide no token at all. It helps you in the gradual migration process when you are moving to an Istio-based system. Not blocking your entire operation by being too strict.

### Enable Strict JWT Validation

To reject requests without valid tokens, add an `AuthorizationPolicy` with a rule specifying a `DENY` action for requests without request principals. Request principals are available only when valid JWT tokens are provided. The following rule therefore denies requests without valid tokens.

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: "gateway-require-jwt"
  namespace: istio-gateways
spec:
  selector:
    matchLabels:
      istio: solo-ingressgateway
  action: DENY
  rules:
  - from:
    - source:
        notRequestPrincipals: ["*"]
EOF
```

Now let's try our test again

Example curl command with no token - output should now be `403` because no token was presented

```bash
curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get
```

output:

```
HTTP/2 403 
content-length: 19
content-type: text/plain
date: Wed, 19 Apr 2023 19:09:22 GMT
server: istio-envoy
```

Example curl command with a bad token - output should be `401`

```bash
curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get --header "Authorization: Bearer badtoken"
```

output:

```
HTTP/2 401 
www-authenticate: Bearer realm="https://httpbin-local.glootest.com/get", error="invalid_token"
content-length: 79
content-type: text/plain
date: Wed, 19 Apr 2023 18:31:58 GMT
server: istio-envoy
```

Example curl command with valid token - output should be `200`

```bash
TOKEN=$(curl https://raw.githubusercontent.com/istio/istio/release-1.17/security/tools/jwt/samples/demo.jwt -s)

curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get --header "Authorization: Bearer $TOKEN"
```

output:

```
HTTP/2 200 
server: istio-envoy
date: Wed, 19 Apr 2023 18:32:18 GMT
content-type: application/json
content-length: 503
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 6
```

## Load Test 1 <a name="load-test-1-"></a>

Example load gen client based on [Bombardier](https://github.com/codesenberg/bombardier)

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: bombardier
  name: bombardier
  namespace: istio-gateways
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bombardier
  template:
    metadata:
      labels:
        app: bombardier
    spec:
      nodeSelector:
        solo-poc: "loadgen"
      tolerations:
        - key: cloud.google.com/solo-poc
          operator: Equal
          value: "loadgen"
          effect: NoSchedule  
      containers:
      - name: bombardier
        image: alpine/bombardier
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            cpu: "3500m"
            memory: "5Gi"
        command: ["/bin/sh"]
        # https with jwt - 5kb header payload
        args: ["-c", "for run in $(seq 1 1000); do bombardier -c 100 -d 60s -r 750 -p i,p,r -k -l -H 'jwt: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJncm91cHMiOlsiYWRtaW4tZ3JvdXAiXX0.8V_AFuGdtFk3FyDKbAjDPX5zoxh7RP0TeMyVg2ZUClg' -H 'payload: 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' https://34.139.255.28/get; done"]
EOF
```

Results from Solo test:
```bash
Statistics        Avg      Stdev        Max
  Reqs/sec       750.04      93.48    1220.42
  Latency        9.84ms     1.36ms    33.44ms
  Latency Distribution
     50%     9.72ms
     75%    10.87ms
     90%    12.03ms
     95%    12.85ms
     99%    15.66ms
  HTTP codes:
    1xx - 0, 2xx - 45006, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:     8.10MB/s
Bombarding https://34.139.255.28:443/get for 1m0s using 100 connection(s)

Statistics        Avg      Stdev        Max
  Reqs/sec       750.04      92.71    1281.55
  Latency        9.91ms     1.38ms    35.38ms
  Latency Distribution
     50%     9.72ms
     75%    10.90ms
     90%    12.10ms
     95%    12.98ms
     99%    15.89ms
  HTTP codes:
    1xx - 0, 2xx - 45006, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:     8.10MB/s
Bombarding https://34.139.255.28:443/get for 1m0s using 100 connection(s)

Statistics        Avg      Stdev        Max
  Reqs/sec       750.06      95.47    1222.89
  Latency        9.94ms     1.37ms    41.26ms
  Latency Distribution
     50%     9.77ms
     75%    10.92ms
     90%    12.16ms
     95%    13.02ms
     99%    15.83ms
  HTTP codes:
    1xx - 0, 2xx - 45006, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:     8.10MB/s
Bombarding https://34.139.255.28:443/get for 1m0s using 100 connection(s)

Statistics        Avg      Stdev        Max
  Reqs/sec       750.03      94.26    1212.49
  Latency        9.89ms     1.39ms    36.66ms
  Latency Distribution
     50%     9.73ms
     75%    10.87ms
     90%    12.05ms
     95%    12.90ms
     99%    15.84ms
  HTTP codes:
    1xx - 0, 2xx - 45006, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:     8.10MB/s
Bombarding https://34.139.255.28:443/get for 1m0s using 100 connection(s)

Statistics        Avg      Stdev        Max
  Reqs/sec       750.01      95.63    1117.19
  Latency       10.01ms     1.45ms    41.98ms
  Latency Distribution
     50%     9.84ms
     75%    10.99ms
     90%    12.17ms
     95%    13.07ms
     99%    16.52ms
  HTTP codes:
    1xx - 0, 2xx - 45006, 3xx - 0, 4xx - 0, 5xx - 0
    others - 0
  Throughput:     8.10MB/s
Bombarding https://34.139.255.28:443/get for 1m0s using 100 connection(s)
```