**NOTE**
This runbook assumes that you already have Gloo Mesh installed as described in `main`. Please refer to the `main` branch for instructions to install Gloo Platform if it has not already been deployed.

![Gloo Mesh Enterprise](images/gloo-mesh-enterprise.png)
# <center>Gloo Mesh Workshop</center>



## Table of Contents
* [Introduction](#introduction)
* [Lab 0 - Prerequisites](#lab-0---prerequisites-)
* [Lab 1 - Setting up your Environment Variables](#lab-1---setting-up-your-environment-variables-)
* [Lab 2 - Deploy Revision based Istio using Gloo Mesh Lifecycle Manager](#lab-2---deploy-revision-based-istio-using-gloo-mesh-lifecycle-manager-)
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


## Lab 2 - Deploy Revision based Istio using Gloo Mesh Lifecycle Manager <a name="lab-2---deploy-revision-based-istio-using-gloo-mesh-lifecycle-manager-"></a>

**NOTE**
This runbook assumes that you already have Gloo Mesh installed as described in `main`. Please refer to the `main` branch for instructions to install Gloo Platform if it has not already been deployed.

We are going to deploy Istio using Gloo Mesh Lifecycle Manager.

First of all, let's create Kubernetes services for the gateways:

```bash
kubectl --context ${CLUSTER1} create ns istio-gateways
kubectl --context ${CLUSTER1} label namespace istio-gateways istio.io/rev=1-17 --overwrite

cat << EOF | kubectl --context ${CLUSTER1} apply -f -
apiVersion: v1
kind: Service
metadata:
  labels:
    app: istio-ingressgateway
    istio: solo-ingressgateway
  name: istio-ingressgateway
  namespace: istio-gateways
spec:
  ports:
  - name: http2
    port: 80
    protocol: TCP
    targetPort: 8080
  - name: https
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    app: istio-ingressgateway
    istio: solo-ingressgateway
    revision: 1-17
  type: LoadBalancer

EOF
```

Note: If using the NLB controller, additional annotations may be required in the service above. Otherwise, a Classic Load Balancer will be created
```
annotations:
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
```

It allows us to have full control on which Istio revision we want to use.

Then, we can tell Gloo Mesh to deploy the Istio control plane using `IstioLifecycleManager`

```bash
cat << EOF | kubectl --context ${CLUSTER1} apply -f -

apiVersion: admin.gloo.solo.io/v2
kind: IstioLifecycleManager
metadata:
  name: cluster1-installation
  namespace: gloo-mesh
spec:
  installations:
    - clusters:
      - name: cluster1
        defaultRevision: true
      revision: 1-17
      istioOperatorSpec:
        profile: minimal
        hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
        tag: 1.17.1-solo
        namespace: istio-system
        values:
          global:
            meshID: mesh1
            multiCluster:
              clusterName: cluster1
            network: cluster1
        meshConfig:
          accessLogFile: /dev/stdout
          defaultConfig:        
            proxyMetadata:
              ISTIO_META_DNS_CAPTURE: "true"
              ISTIO_META_DNS_AUTO_ALLOCATE: "true"
        components:
          pilot:
            k8s:
              env:
                - name: PILOT_ENABLE_K8S_SELECT_WORKLOAD_ENTRIES
                  value: "false"
          ingressGateways:
          - name: istio-ingressgateway
            enabled: false
EOF
```

Afterwards we can deploy the Istio Ingress Gateway using `GatewayLifecycleManager`. Take note of the additional `nodeSelector` and `toleration` configuration to pin the gateway to a specific node pool as well as the set resource requests for our gateway

```bash
cat << EOF | kubectl --context ${CLUSTER1} apply -f -
apiVersion: admin.gloo.solo.io/v2
kind: GatewayLifecycleManager
metadata:
  name: cluster1-ingress
  namespace: gloo-mesh
spec:
  installations:
    - clusters:
      - name: cluster1
        activeGateway: false
      gatewayRevision: 1-17
      istioOperatorSpec:
        profile: empty
        hub: us-docker.pkg.dev/gloo-mesh/istio-workshops
        tag: 1.17.1-solo
        values:
          gateways:
            istio-ingressgateway:
              customService: true
        components:
          ingressGateways:
            - name: istio-ingressgateway
              namespace: istio-gateways
              enabled: true
              label:
                istio: solo-ingressgateway
              k8s:
                nodeSelector:
                  solo-poc: "gateway"
                tolerations:
                  - key: cloud.google.com/solo-poc
                    operator: Equal
                    value: "gateway"
                    effect: NoSchedule  
                resources:
                  requests:
                    cpu: 7000m
                    memory: 3Gi
                  limits:
                    cpu: 7800m
                    memory: 4Gi
EOF
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

Since we already have the `httpbin` app deployed and configured by our existing Istio, we going to re-deploy the httpbin application to use our newly provisioned `1-17` revisioned Istio.

This involves a few steps:
- Re-label the namespace to have an istio revision label `istio.io/rev=1-17`
- Restart httpbin deployment to pick up a new sidecar from the revision-based Istio control plane

Run the following commands to deploy the httpbin app named `in-mesh` on `cluster1`. 
```bash
kubectl --context ${CLUSTER1} label namespace httpbin istio-injection-
kubectl --context ${CLUSTER1} label namespace httpbin istio.io/rev=1-17 --overwrite

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
kubectl --context ${CLUSTER1} label namespace gloo-mesh-addons istio-injection-
kubectl --context ${CLUSTER1} label namespace gloo-mesh-addons istio.io/rev=1-17 --overwrite

helm upgrade --install gloo-mesh-agent-addons gloo-mesh-agent/gloo-mesh-agent \
--namespace gloo-mesh-addons \
--kube-context=${CLUSTER1} \
--version 2.2.6 \
--values - <<EOF
glooMeshAgent:
  enabled: false
rate-limiter:
  enabled: false
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
      istio: ingressgateway
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
curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get"
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
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: ingressgateway
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
curl -kI https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get"
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