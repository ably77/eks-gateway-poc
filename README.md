![Gloo Mesh Enterprise](images/gloo-mesh-enterprise.png)
# <center>Gloo Mesh Workshop</center>



## Table of Contents
* [Introduction](#introduction)
* [Lab 0 - Prerequisites](#lab-0---prerequisites-)
* [Lab 1 - Setting up your Environment Variables](#lab-1---setting-up-your-environment-variables-)
* [Lab 2 - Deploy and register Gloo Mesh](#lab-2---deploy-and-register-gloo-mesh-)
* [Lab 3 - Deploy Istio using Gloo Mesh Lifecycle Manager](#lab-3---deploy-istio-using-gloo-mesh-lifecycle-manager-)
* [Lab 4 - Deploy Gloo Mesh Addons](#lab-4---deploy-gloo-mesh-addons-)
* [Lab 5 - Create the gateways workspace](#lab-5---create-the-gateways-workspace-)
* [Lab 6 - Deploy the httpbin demo app](#lab-6---deploy-the-httpbin-demo-app-)
* [Lab 7 - Create the httpbin workspace](#lab-7---create-the-httpbin-workspace-)
* [Lab 8 - Expose the httpbin service](#lab-8---expose-the-httpbin-service-)
* [Lab 9 - Securing Application access with ExtAuthPolicy](#lab-9---securing-application-access-with-extauthpolicy-)
* [Lab 10 - Integrating with OPA](#lab-10---integrating-with-opa-)
* [Lab 11 - Apply rate limiting to the Gateway](#lab-11---apply-rate-limiting-to-the-gateway-)
* [Lab 12 - Exploring Istio, Envoy Proxy Config, and Metrics](#lab-12---exploring-istio-envoy-proxy-config-and-metrics-)
* [Lab 13 - Exploring the Opentelemetry Pipeline](#lab-13---exploring-the-opentelemetry-pipeline-)
* [Lab 14 - Leveraging the Latency EnvoyFilter for additional performance metrics from our gateway](#lab-14---leveraging-the-latency-envoyfilter-for-additional-performance-metrics-from-our-gateway-)
* [Lab 15 - Upgrade Istio using Gloo Mesh Lifecycle Manager](#lab-15---upgrade-istio-using-gloo-mesh-lifecycle-manager-)
* [Lab 16 - Configure a mutual TLS ingress gateway](#lab-16---configure-a-mutual-tls-ingress-gateway-)
* [Lab 17 - Install and Configure Keycloak](#lab-17---install-and-configure-keycloak-)
* [Lab 18 - Integrate Okta with Keycloak](#lab-18---integrate-okta-with-keycloak-)
* [Lab 19 - Enable JWT Validation at the Gateway](#lab-19---enable-jwt-validation-at-the-gateway-)
* [Lab 20 - Using Passthrough External Auth](#lab-20---using-passthrough-external-auth-)
* [Lab 21 - Routing to a GRPC Service](#lab-21---routing-to-a-grpc-service-)
* [Lab 22 - Securing GRPC Service with ExtAuthPolicy](#lab-22---securing-grpc-service-with-extauthpolicy-)
* [Lab 23 - Apply rate limiting to the GRPC service](#lab-23---apply-rate-limiting-to-the-grpc-service-)



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


## Lab 2 - Deploy and register Gloo Mesh <a name="lab-2---deploy-and-register-gloo-mesh-"></a>

First of all, let's install the `meshctl` CLI:

```bash
export GLOO_MESH_VERSION=v2.4.0-rc1
curl -sL https://run.solo.io/meshctl/install | sh -
export PATH=$HOME/.gloo-mesh/bin:$PATH
```

Run the following commands to deploy the Gloo Mesh management plane:

```bash
helm repo add gloo-mesh-enterprise https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-enterprise 
helm repo update
kubectl --context ${CLUSTER1} create ns gloo-mesh 
helm upgrade --install gloo-mesh-enterprise gloo-mesh-enterprise/gloo-mesh-enterprise \
--namespace gloo-mesh --kube-context ${CLUSTER1} \
--version=2.4.0-rc1 \
--set glooMeshMgmtServer.ports.healthcheck=8091 \
--set legacyMetricsPipeline.enabled=false \
--set metricsgateway.enabled=true \
--set metricsgateway.service.type=ClusterIP \
--set glooMeshUi.serviceType=ClusterIP \
--set mgmtClusterName=${CLUSTER1} \
--set global.cluster=${CLUSTER1} \
--set licenseKey=${GLOO_MESH_LICENSE_KEY}
kubectl --context ${CLUSTER1} -n gloo-mesh rollout status deploy/gloo-mesh-mgmt-server
```
Then, you need to set the environment variable to tell the Gloo Mesh agents how to communicate with the management plane:

```bash
export ENDPOINT_GLOO_MESH=gloo-mesh-mgmt-server:9900
export HOST_GLOO_MESH=$(echo ${ENDPOINT_GLOO_MESH} | cut -d: -f1)
export ENDPOINT_METRICS_GATEWAY=gloo-metrics-gateway:4317
```

Check that the variables have correct values:
```
echo $HOST_GLOO_MESH
echo $ENDPOINT_GLOO_MESH
```

Finally, you need to register the cluster(s).

Here is how you register the first one:

```bash
helm repo add gloo-mesh-agent https://storage.googleapis.com/gloo-mesh-enterprise/gloo-mesh-agent
helm repo update

kubectl apply --context ${CLUSTER1} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: KubernetesCluster
metadata:
  name: cluster1
  namespace: gloo-mesh
spec:
  clusterDomain: cluster.local
EOF

kubectl --context ${CLUSTER1} create ns gloo-mesh

helm upgrade --install gloo-mesh-agent gloo-mesh-agent/gloo-mesh-agent \
  --namespace gloo-mesh \
  --kube-context=${CLUSTER1} \
  --set relay.serverAddress=${ENDPOINT_GLOO_MESH} \
  --set relay.authority=gloo-mesh-mgmt-server.gloo-mesh \
  --set rate-limiter.enabled=false \
  --set ext-auth-service.enabled=false \
  --set glooMeshPortalServer.enabled=false \
  --set cluster=cluster1 \
  --set metricscollector.enabled=true \
  --set metricscollector.config.exporters.otlp.endpoint=\"${ENDPOINT_METRICS_GATEWAY}\" \
  --version 2.4.0-rc1
```

Note that the registration can also be performed using `meshctl cluster register`.

You can check the cluster(s) have been registered correctly using the following commands:

```
meshctl --kubecontext ${CLUSTER1} check
```

```
pod=$(kubectl --context ${CLUSTER1} -n gloo-mesh get pods -l app=gloo-mesh-mgmt-server -o jsonpath='{.items[0].metadata.name}')
kubectl --context ${CLUSTER1} -n gloo-mesh debug -q -i ${pod} --image=curlimages/curl -- curl -s http://localhost:9091/metrics | grep relay_push_clients_connected
```

You should get an output similar to this:

```
# HELP relay_push_clients_connected Current number of connected Relay push clients (Relay Agents).
# TYPE relay_push_clients_connected gauge
relay_push_clients_connected{cluster="cluster1"} 1
```

## Lab 3 - Deploy Istio using Gloo Mesh Lifecycle Manager <a name="lab-3---deploy-istio-using-gloo-mesh-lifecycle-manager-"></a>

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

Then, we can tell Gloo Mesh to deploy the Istio control planes and the gateways in the cluster(s)

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

## Lab 4 - Deploy Gloo Mesh Addons <a name="lab-4---deploy-gloo-mesh-addons-"></a>

To use the Gloo Mesh Gateway advanced features (external authentication, rate limiting, ...), you need to install the Gloo Mesh addons.

First, you need to create a namespace for the addons, with Istio injection enabled:

```bash
kubectl --context ${CLUSTER1} create namespace gloo-mesh-addons
kubectl --context ${CLUSTER1} label namespace gloo-mesh-addons istio.io/rev=1-17 --overwrite
```

Then, you can deploy the addons on the cluster(s) using Helm:

```bash
helm upgrade --install gloo-mesh-agent-addons gloo-mesh-agent/gloo-mesh-agent \
--namespace gloo-mesh-addons \
--kube-context=${CLUSTER1} \
--version 2.4.0-rc1 \
--values - <<EOF
glooMeshAgent:
  enabled: false
rate-limiter:
  enabled: true
ext-auth-service:
  enabled: true
EOF
```

This is how to environment looks like now:

![Gloo Mesh Workshop Environment](images/arch/arch-1a.png)

## Lab 5 - Create the gateways workspace <a name="lab-5---create-the-gateways-workspace-"></a>

We're going to create a workspace for the team in charge of the Gateways.

The platform team needs to create the corresponding `Workspace` Kubernetes objects in the Gloo Mesh management cluster.

Let's create the `gateways` workspace which corresponds to the `istio-gateways` and the `gloo-mesh-addons` namespaces on the cluster(s):

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

Then, the Gateway team creates a `WorkspaceSettings` Kubernetes object in one of the namespaces of the `gateways` workspace (so the `istio-gateways` or the `gloo-mesh-addons` namespace):

```bash
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

The Gateway team has decided to import the following from the workspaces that have the label `allow_ingress` set to `true` (using a selector):
- all the Kubernetes services exported by these workspaces
- all the resources (RouteTables, VirtualDestination, ...) exported by these workspaces that have the label `expose` set to `true`


## Lab 6 - Deploy the httpbin demo app <a name="lab-6---deploy-the-httpbin-demo-app-"></a>

We're going to deploy the httpbin application to demonstrate several features of Gloo Mesh.

You can find more information about this application [here](http://httpbin.org/).

Run the following commands to deploy the httpbin app named `in-mesh` on `cluster1`. 

```bash
kubectl --context ${CLUSTER1} create ns httpbin
kubectl --context ${CLUSTER1} label namespace httpbin istio.io/rev=1-17 --overwrite

kubectl --context ${CLUSTER1} apply -n httpbin -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: in-mesh
---
apiVersion: v1
kind: Service
metadata:
  name: in-mesh
  labels:
    app: in-mesh
    service: in-mesh
spec:
  ports:
  - name: http
    port: 8000
    targetPort: 80
  selector:
    app: in-mesh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: in-mesh
spec:
  replicas: 1
  selector:
    matchLabels:
      app: in-mesh
      version: v1
  template:
    metadata:
      labels:
        app: in-mesh
        version: v1
        # uncomment for pod label injection
        #istio.io/rev: 1-17
    spec:
      serviceAccountName: in-mesh
      containers:
      - image: docker.io/kennethreitz/httpbin
        imagePullPolicy: IfNotPresent
        name: in-mesh
        ports:
        - containerPort: 80
EOF
```


You can follow the progress using the following command:

```
kubectl --context ${CLUSTER1} -n httpbin get pods
```

```
NAME                           READY   STATUS    RESTARTS   AGE
in-mesh-5d9d9549b5-qrdgd       2/2     Running   0          11s
```

## Lab 7 - Create the httpbin workspace <a name="lab-7---create-the-httpbin-workspace-"></a>

We're going to create a workspace for the team in charge of the httpbin application.

The platform team needs to create the corresponding `Workspace` Kubernetes objects in the Gloo Mesh management cluster.

Let's create the `httpbin` workspace which corresponds to the `httpbin` namespace on `cluster1`:

```bash
kubectl apply --context ${CLUSTER1} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: Workspace
metadata:
  name: httpbin
  namespace: gloo-mesh
  labels:
    allow_ingress: "true"
spec:
  workloadClusters:
  - name: cluster1
    namespaces:
    - name: httpbin
EOF
```

Then, the Httpbin team creates a `WorkspaceSettings` Kubernetes object in one of the namespaces of the `httpbin` workspace:

```bash
kubectl apply --context ${CLUSTER1} -f- <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: httpbin
  namespace: httpbin
spec:
  importFrom:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
  exportTo:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
      labels:
        app: in-mesh
    - kind: ALL
      labels:
        expose: "true"
EOF
```

The Httpbin team has decided to export the following to the `gateway` workspace (using a reference):
- the `in-mesh` Kubernetes service
- all the resources (RouteTables, VirtualDestination, ...) that have the label `expose` set to `true`

## Lab 8 - Expose the httpbin service <a name="lab-8---expose-the-httpbin-service-"></a>

In this step, we're going to expose the `httpbin` service through the Ingress Gateway using Gloo Mesh.

The Gateway team must create a `VirtualGateway` to configure the Istio Ingress Gateway in cluster1 to listen to incoming requests.

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
      allowedRouteTables:
        - host: '*'
EOF
```

Then, the Gateway team should create a parent `RouteTable` to configure the main routing.

```bash
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

In this example, you can see that the Gateway team is delegating the routing details to the `httpbin` workspace. The teams in charge of these workspaces can expose their services through the gateway.

The Gateway team can use this main `RouteTable` to enforce a global policy, but also to have control on which hostnames and paths can be used by each application team.

Now the httpbin team can create a `RouteTable` to determine how they want to handle the traffic.

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
      matchers:
      - uri:
          exact: /
      - uri:
          prefix: /get
      - uri:
          prefix: /anything
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

You should now be able to access the `httpbin` application through the browser.

Get the URL to access the `httpbin` service using the following command:
```
echo "http://${ENDPOINT_HTTP_GW_CLUSTER1}/get"
```

Gloo Mesh translates the `VirtualGateway` and `RouteTable` into the corresponding Istio objects (`Gateway` and `VirtualService`).

Now, let's secure the access through TLS.

Let's first create a private key and a self-signed certificate:

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

Finally, the Gateway team needs to update the `VirtualGateway` to use this secret:

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

You can now access the `httpbin` application securely through the browser.
Get the URL to access the `httpbin` service using the following command:
```
echo "https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get"
```

This diagram shows the flow of the request (through the Istio Ingress Gateway):

![Gloo Mesh Gateway](images/arch/arch-1b.png)


## Lab 9 - Securing Application access with ExtAuthPolicy <a name="lab-9---securing-application-access-with-extauthpolicy-"></a>
In this step, we're going to secure the access to the `httpbin` service using OAuth. Integrating an app with extauth consists of a few steps:
```
- create app registration in your OIDC
- configuring a Gloo Mesh `ExtAuthPolicy` and `ExtAuthServer`
- configuring the `RouteTable` with a specified label (i.e. `route_name: "httpbin-all"`)
```

### In your OIDC Provider
Once the app has been configured in the external OIDC, we need to create a Kubernetes Secret that contains the OIDC client-secret. Please provide this value input before running the command below:
```bash
# example oidc client secret that works with okta
export HTTPBIN_CLIENT_SECRET="3cMS-DdmZH0UoHHRWgfWlNzHjTjty56DPaB66Mv2"
```

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: oidc-client-secret
  namespace: gloo-mesh
type: extauth.solo.io/oauth
data:
  client-secret: $(echo -n ${HTTPBIN_CLIENT_SECRET} | base64)
EOF
```

Set the callback URL in your OIDC provider to map to our httpbin app
```bash
export APP_CALLBACK_URL="https://$(kubectl --context ${CLUSTER1} -n istio-gateways get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].*}')"

echo $APP_CALLBACK_URL
```

Lastly, replace the `OICD_CLIENT_ID` and `ISSUER_URL` values below with your OIDC app settings
```bash
# example oidc client secret that works with okta
export OIDC_CLIENT_ID="0oa6qvzybcVCK6PcS5d7"
export ISSUER_URL="https://dev-22653158.okta.com/oauth2/default"
```

Let's make sure our variables are set correctly:
```bash
echo $OIDC_CLIENT_ID
echo $ISSUER_URL
```

### Create ExtAuthPolicy
Now we will create an `ExtAuthPolicy`, which is a CRD that contains authentication information. 
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: httpbin-extauth
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        route_name: "httpbin"
  config:
    server:
      name: cluster1-ext-auth-server
      namespace: httpbin
      cluster: cluster1
    glooAuth:
      configs:
      - oauth2:
          oidcAuthorizationCode:
            appUrl: ${APP_CALLBACK_URL}
            callbackPath: /callback
            clientId: ${OIDC_CLIENT_ID}
            clientSecretRef:
              name: oidc-client-secret
              namespace: gloo-mesh
            issuerUrl: ${ISSUER_URL}
            session:
              failOnFetchFailure: true
              redis:
                cookieName: oidc-session
                options:
                  host: redis.gloo-mesh-addons:6379
                allowRefreshing: true
              cookieOptions:
                maxAge: "90"
            scopes:
            - email
            - profile
            headers:
              idTokenHeader: Jwt
EOF
```

After that, you need to create an `ExtAuthServer`, which is a CRD that define which extauth server to use: 
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
EOF
```

Finally, you need to update the `RouteTable` to use this `AuthConfig`:
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
        route_name: "httpbin"
        ratelimited: "true"
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      - uri:
          prefix: /callback
      - uri:
          prefix: /logout
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

Now when you access your httpbin app through the browser, it will be protected by the OIDC provider login page.
```
echo "${APP_CALLBACK_URL}/get"
```

### User Credentials
If you are using the example client config above, below are a few users that you can validate with
- Username: jdoe@solo.io // Password: gloo-dev
- Username: test@solo.io // Password: gloo-public
- Username: jdoe@gmail.com // Password: gloo-public

## Lab 10 - Integrating with OPA <a name="lab-10---integrating-with-opa-"></a>

### OPA inputs
You can also perform authorization using OPA. Gloo Mesh's OPA integration populates an input document to use in your OPA policies which allows you to easily write rego policy
- `input.check_request` - By default, all OPA policies will contain an Envoy Auth Service CheckRequest. This object contains all the information Envoy has gathered of the request being processed. See the Envoy docs and proto files for AttributeContext for the structure of this object.
- `input.http_request` - When processing an HTTP request, this field will be populated for convenience. See the Envoy HttpRequest docs and proto files for the structure of this object.
- `input.state.jwt` - When the OIDC auth plugin is utilized, the token retrieved during the OIDC flow is placed into this field. 

## Lab
In this lab, we will make use of the `input.http_request` parameter in our OPA policies to decode the `jwt` token retrieved in the last lab and create policies using the claims available

Instead of coupling the `oauth2` config with the `opa` config in a single `ExtAuthPolicy`, here we will separate the app to decouple the APIs from apps

## High Level Workflow
First we will create a new `ExtAuthPolicy` object to add the OPA filter. Note the use of the `applyToDestinations` in this `ExtAuthPolicy` instead of `applyToRoutes`. This matcher allows us to specify a destination where we want to apply our policy to, rather than on a Route. Note that the below uses a direct reference, but label matchers would work as well.

Lets apply the following policy
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: httpbin-opa
  namespace: httpbin
spec:
  applyToDestinations:
  - selector:
      name: in-mesh
      namespace: httpbin
      workspace: httpbin
  config:
    server:
      name: cluster1-ext-auth-server
      namespace: httpbin
      cluster: cluster1
    glooAuth:
      configs:
      - opaAuth:
          modules:
          - name: httpbin-opa
            namespace: httpbin
          query: "data.test.allow == true"
EOF
```

### Enforce @solo.io username login by inspecting the JWT email payload
For our first use-case we will decode the JWT token passed through extauth for the `email` payload, and enforce that a user logging in must end with `@solo.io` as the username with OPA

First, you need to create a `ConfigMap` with the policy written in rego. 
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package test

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
    }
EOF
```

Now we should see success when logging in with a username that ends with `@solo.io` but will encounter a `403 Error - You don't have authorization to view this page` when using a username that ends with anything else (`@gmail.com` for example)

### Map to other claims in JWT
If you decode the JWT provided (using jwt.io for example), we can see other claims available
```
{
  "sub": "00u5c8ipkj3HyHoS25d7",
  "name": "jdoe",
  "email": "jdoe@solo.io",
}
```

We can modify our rego rule to apply policy to map to `name` instead of the `email` claim
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package test

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["name"], "jdoe")
    }
EOF
```

Now you should be able to access the app logging in with jdoe user, but would get a 403 if you had a different `name` claim. In an incognito brower you can test this by logging into the `test@solo.io` user with the password `gloo-public`

This is because the test@solo.io user's claims would look something like this:
```
{
  "sub": "00u6v09p4fVhmvMEN5d7",
  "name": "solouser io",
  "email": "test@solo.io",
}
```

Let's revert back to the original policy for now
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package test

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
    }
EOF
```

### Use OPA to enforce a specific HTTP method
Let's continue to expand on our example by enforcing different HTTP methods in our rego policy
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package test

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
        any({input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
EOF
```

If you refresh the browser where the `@solo.io` user is logged in, we should now see a `403 Error - You don't have authorization to view this page`. This is because we are not allowing the `GET` method for either of those matches in our OPA policy

Let's fix that
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package test

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
EOF
```

Now we should be able to access our app again.

### Enforce paths with OPA
Let's continue to expand on our example by enforcing a specified path for our users

Here we will modify our rego rule so that users with the `email` claim containing `@solo.io` can access the `/get` endpoint as well as any path with the prefix `/anything`, while users with the `email` claim containing `@gmail.com` can only access specifically the `/anything/opa-protected` endpoint
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: httpbin-opa
  namespace: httpbin
data:
  policy.rego: |-
    package test

    default allow = false

    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@solo.io")
        any({input.http_request.path == "/get",
        startswith(input.http_request.path, "/anything")
    })
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
    allow {
        [header, payload, signature] = io.jwt.decode(input.http_request.headers.jwt)
        endswith(payload["email"], "@gmail.com")
        input.http_request.path == "/anything/opa-protected"
        any({input.http_request.method == "GET",
             input.http_request.method == "POST",
             input.http_request.method == "PUT",
             input.http_request.method == "DELETE",
    })
    }
EOF
```
If you refresh the browser where the `@solo.io` user is logged in, we should be able to access the `/get` endpoint as well as any path with the prefix `/anything`. Try and access `/anything/foo` for example - it should work.

If you refresh the browser where the `@gmail.com` user is logged in, we should now see a `403 Error - You don't have authorization to view this page` if you access anything other than the `/anything/opa-protected` endpoint

### cleanup extauthpolicy for next labs
This step is not required to continue with the labs, but if you need to clean up the extauth policy, simply just remove it with the command below
```
kubectl --context ${CLUSTER1} -n httpbin delete ExtAuthPolicy httpbin-opa
```

This diagram shows the flow of the request (with the Istio ingress gateway leveraging the `extauth` Pod to authorize the request):

![Gloo Mesh Dashboard OIDC](images/arch/arch-1c.png)

## Lab 11 - Apply rate limiting to the Gateway <a name="lab-11---apply-rate-limiting-to-the-gateway-"></a>

In this lab, lets explore adding rate limiting to our httpbin route

In this step, we're going to apply rate limiting to the Gateway to only allow 5 requests per minute

First, we need to create a `RateLimitClientConfig` object to define the descriptors:

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: RateLimitClientConfig
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: httpbin
  namespace: httpbin
spec:
  raw:
    rateLimits:
      - actions:
          - genericKey:
              descriptorValue: counter
EOF
```

Then, we need to create a `RateLimitServerConfig` object to define the limits based on the descriptors:

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: RateLimitServerConfig
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: httpbin
  namespace: gloo-mesh-addons
spec:
  destinationServers:
  - port:
      name: grpc
    ref:
      cluster: cluster1
      name: rate-limiter
      namespace: gloo-mesh-addons
  raw:
    descriptors:
      - key: generic_key
        rateLimit:
          requestsPerUnit: 5
          unit: MINUTE
        value: counter
EOF
```

After that, we need to create a `RateLimitPolicy` object to define the descriptors:

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: RateLimitPolicy
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: httpbin
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        ratelimited: "true"
  config:
    ratelimitClientConfig:
      cluster: cluster1
      name: httpbin
      namespace: httpbin
    ratelimitServerConfig:
      cluster: cluster1
      name: httpbin
      namespace: gloo-mesh-addons
    serverSettings:
      cluster: cluster1
      name: rate-limit-server
      namespace: httpbin
EOF
```

We also need to create a `RateLimitServerSettings`, which is a CRD that define which extauth server to use: 

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: RateLimitServerSettings
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: rate-limit-server
  namespace: httpbin
spec:
  destinationServer:
    port:
      name: grpc
    ref:
      cluster: cluster1
      name: rate-limiter
      namespace: gloo-mesh-addons
EOF
```

Now refresh the httpbin web page multiple times. You should see a 429 error after 5 refreshes

Note: If you scroll up, notice that we had already preloaded this route table with the `ratelimited: "true"` label in an earlier lab. When we applied our rate limiting policy, Gloo Platform automatically picked up on this label and applied our RL configuration for us. Below is the `RouteTable` again for reference:
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
        route_name: "httpbin"
        ratelimited: "true"
      matchers:
      - uri:
          exact: /get
      - uri:
          prefix: /anything
      - uri:
          prefix: /callback
      - uri:
          prefix: /logout
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

This diagram shows the flow of the request (with the Istio ingress gateway leveraging the `rate limiter` Pod to determine if the request should be allowed):

![Gloo Mesh Gateway Rate Limiting](images/arch/arch-1d.png)


### cleanup

Let's apply the original `RouteTable` yaml:

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
      matchers:
      - uri:
          exact: /get
      forwardTo:
        destinations:
        - ref:
            name: in-mesh
            namespace: httpbin
          port:
            number: 8000
EOF
```

And also delete the different objects we've created:

```bash
kubectl --context ${CLUSTER1} -n httpbin delete ratelimitpolicy httpbin
kubectl --context ${CLUSTER1} -n httpbin delete ratelimitclientconfig httpbin
kubectl --context ${CLUSTER1} -n gloo-mesh-addons delete ratelimitserverconfig httpbin
kubectl --context ${CLUSTER1} -n httpbin delete ratelimitserversettings rate-limit-server
```

## Lab 12 - Exploring Istio, Envoy Proxy Config, and Metrics <a name="lab-12---exploring-istio-envoy-proxy-config-and-metrics-"></a>

## Get an overview of your mesh
```
istioctl proxy-status
```

Example output
```
% istioctl proxy-status
NAME                                                          CLUSTER      CDS        LDS        EDS        RDS          ECDS         ISTIOD                         VERSION
ext-auth-service-7fccf5b78f-mb6wb.gloo-mesh-addons            cluster1     SYNCED     SYNCED     SYNCED     SYNCED       NOT SENT     istiod-1-17-94858bc8-9chtf     1.17.1-solo
in-mesh-5978df87cc-mfmtx.httpbin                              cluster1     SYNCED     SYNCED     SYNCED     SYNCED       NOT SENT     istiod-1-17-94858bc8-9chtf     1.17.1-solo
istio-ingressgateway-1-17-79b44d8bb-vth6t.istio-gateways      cluster1     SYNCED     SYNCED     SYNCED     SYNCED       NOT SENT     istiod-1-17-94858bc8-9chtf     1.17.1-solo
rate-limiter-66676f8d5b-wrcd7.gloo-mesh-addons                cluster1     SYNCED     SYNCED     SYNCED     SYNCED       NOT SENT     istiod-1-17-94858bc8-9chtf     1.17.1-solo
redis-669c97869d-hjtfp.gloo-mesh-addons                       cluster1     SYNCED     SYNCED     SYNCED     SYNCED       NOT SENT     istiod-1-17-94858bc8-9chtf     1.17.1-solo
```

## Retrieve diffs between Envoy and Istiod
```
istioctl proxy-status deploy/in-mesh -n httpbin
```

## grab envoy stats of sidecar using istioctl
```
istioctl experimental envoy-stats <pod> --namespace <namespace> 

istioctl experimental envoy-stats deploy/<deployment_name> --namespace <namespace> 
```

For example try this on the httpbin application
```
istioctl experimental envoy-stats deploy/in-mesh --namespace httpbin
```

### output in prometheus format
Add the `--output prom` flag to output metrics in prometheus format
```
istioctl experimental envoy-stats deploy/in-mesh --namespace httpbin --output prom
```

## Get all Envoy proxy config
```
istioctl proxy-config all -n <namespace> <pod> -o <output>
```

Example:
```
istioctl proxy-config all -n httpbin deploy/in-mesh
```

### Retrieve just the endpoint configuration
```
istioctl proxy-config endpoint -n httpbin deploy/in-mesh
```

## Inspect bootstrap configuration
```
istioctl proxy-config bootstrap -n istio-gateways deploy/istio-ingressgateway-1-17
```

## Create an Istio Bug Report
```
istioctl bug-report
```
See output named `bug-report.tar.gz`

## Lab 13 - Exploring the Opentelemetry Pipeline <a name="lab-13---exploring-the-opentelemetry-pipeline-"></a>

As a part of the Gloo Platform Helm setup, we installed the Gloo OpenTelemetry Pipeline. A high level architecture of the pipeline looks like this:

![otel pipeline](images/observability/metrics-architecture-otel.png)

- Gloo metrics collector agents are deployed as a daemonset in all Gloo workload clusters. The collector agents scrape metrics from workloads in your cluster, such as the Gloo agents, the Istio control plane istiod, or the Istio-injected workloads. The agents then enrich and convert the metrics. For example, the ID of the source and destination workload is added to the metrics so that you can filter the metrics for the workload that you are interested in.
- The collector agents send the scraped metrics to the Gloo metrics gateway in the Gloo management cluster via gRPC push procedures.
- The Prometheus server scrapes the metrics from the Gloo metrics gateway.


To view the scrape config, take a look at the `gloo-metrics-collector-config` configmap
```
kubectl --context ${CLUSTER1} get configmap gloo-metrics-collector-config -n gloo-mesh -o yaml
```

To view metrics that are being sent to our metrics gateway component port-forward to the service at port 9091 with the command below
```
kubectl --context ${CLUSTER1} -n gloo-mesh \
    port-forward deploy/gloo-metrics-gateway 9091
```

Navigate to http://localhost:9091/metrics to view the metrics that have been collected by the oTel pipeline


### Using Prometheus to view oTel observability metrics
By default, Gloo Platform configures the Prometheus Reciever for the oTel collector on each node as well as the Exporter on the Metrics Gateway to send metrics to the Prometheus service in the `gloo-mesh` namespace. The Gloo Mesh UI uses these metrics to populate it's service graph

You can port-forward to the Prometheus service using the command below:
```
kubectl --context ${CLUSTER1} port-forward svc/prometheus-server -n gloo-mesh 9090:80
```
Navigate to https://localhost:9090 to access Prometheus UI

In the Prometheus UI, if you navigate to the Status dropdown > Targets you should see the otel-collector details

You can also query for metrics within the Prometheus UI, for example try to query the `istio_requests_total`

![prometheus](images/observability/prometheus-ui.png)

## Lab 14 - Leveraging the Latency EnvoyFilter for additional performance metrics from our gateway <a name="lab-14---leveraging-the-latency-envoyfilter-for-additional-performance-metrics-from-our-gateway-"></a>

The Solo Istio images are packaged with an additional proxy latency filter that measures the latency incurred by the filter chain in a histogram. We can use this latency filter by deploying an `EnvoyFilter` in Istio

Here we are going to aply the latency filter on our gateway at port 443
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: latency 
  namespace: istio-system
spec:
  configPatches:
  - applyTo: HTTP_FILTER
    match:
      context: ANY
      listener:
        portNumber: 443
        filterChain:
          filter:
            name: "envoy.filters.network.http_connection_manager"
            subFilter:
              name: "envoy.filters.http.router"
    patch:
      operation: INSERT_FIRST
      value:
        name: io.solo.filters.http.proxy_latency
        typed_config:
          "@type": type.googleapis.com/udpa.type.v1.TypedStruct
          type_url: "type.googleapis.com/envoy.config.filter.http.proxylatency.v2.ProxyLatency"
          value:
            request: "FIRST_INCOMING_FIRST_OUTGOING"
            response: "FIRST_INCOMING_FIRST_OUTGOING"
            measure_request_internally: true
            emit_dynamic_metadata: true

EOF
```

After you let some traffic flow through the gateway, you will be able to see proxy latency histograms corresponding to each service in the proxy envoy/prometheus stats. To get to the stats page, use this port-forward command below
```
kubectl --context ${CLUSTER1} port-forward deploy/istio-ingressgateway-1-17 -n istio-gateways 15000:15000
```

Now navigate to localhost:15000 in your browser. Scroll down to the stats/prometheus to view the proxy Prometheus stats. If you search for `latency` you should be able to find newly emitted metrics measuring proxy latency
```
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="0.5"} 75858
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="1"} 75858
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="5"} 75913
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="10"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="25"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="50"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="100"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="250"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="500"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="1000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="2500"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="5000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="10000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="30000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="60000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="300000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="600000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="1800000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="3600000"} 75916
envoy_cluster_proxy_latency_rq_proxy_latency_bucket{cluster_name="outbound|8000||in-mesh.httpbin.svc.cluster.local",le="+Inf"} 75916
```

Note that `le` refers to milliseconds, and the counts are cumulative. So using the example above I can derive the following:
- 75916 total count of requests
- 75858 requests took under 0.5ms latency
- 75913 were less than 5ms
- All were less than 10ms

## Lab 15 - Upgrade Istio using Gloo Mesh Lifecycle Manager <a name="lab-15---upgrade-istio-using-gloo-mesh-lifecycle-manager-"></a>

Set the variables corresponding to the old and new revision tags:

```bash
export OLD_REVISION=1-17
export NEW_REVISION=1-17
```

We are going to upgrade Istio using Gloo Mesh Lifecycle Manager.

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
      revision: ${OLD_REVISION}
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
    - clusters:
      - name: cluster1
        defaultRevision: false
      revision: ${NEW_REVISION}
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
            envoyAccessLogService:
              address: gloo-mesh-agent.gloo-mesh:9977
            proxyMetadata:
              ISTIO_META_DNS_CAPTURE: "true"
              ISTIO_META_DNS_AUTO_ALLOCATE: "true"
            proxyStatsMatcher:
                inclusionRegexps:
                  - ".*proxy_latency.*"
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
      gatewayRevision: ${OLD_REVISION}
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
    - clusters:
      - name: cluster1
        activeGateway: false
      gatewayRevision: ${NEW_REVISION}
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
EOF
```

Run the following command to check the status of the upgrade(s):

```sh
kubectl --context ${CLUSTER1} -n gloo-mesh get istiolifecyclemanager cluster1-installation -o yaml
kubectl --context ${CLUSTER1} -n gloo-mesh get gatewaylifecyclemanager cluster1-ingress -o yaml
```

When the upgrade is completed, the state at the end of the objects will be `HEALTHY`.

Now, let's restart all the Istio Pods to use the new revision:

```bash
kubectl --context ${CLUSTER1} get ns -l istio.io/rev=${OLD_REVISION} -o json | jq -r '.items[].metadata.name' | while read ns; do
  kubectl --context ${CLUSTER1} label ns ${ns} istio.io/rev=${NEW_REVISION} --overwrite
  kubectl --context ${CLUSTER1} -n ${ns} rollout restart deploy
done
kubectl --context ${CLUSTER1} -n httpbin patch deploy in-mesh --patch "{\"spec\": {\"template\": {\"metadata\": {\"labels\": {\"istio.io/rev\": \"${NEW_REVISION}\" }}}}}"
```

Test that you can still access the `in-mesh` service through the Istio Ingress Gateway corresponding to the old revision using the command below:

```bash
curl -k "https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get" -I
```

You should get a response similar to the following one:

```
HTTP/2 200 
server: istio-envoy
date: Wed, 24 Aug 2022 14:58:22 GMT
content-type: application/json
content-length: 670
access-control-allow-origin: *
access-control-allow-credentials: true
x-envoy-upstream-service-time: 7
```

All good, so we can now switch to the Istio gateways corresponding to the new revision:

```bash
kubectl --context ${CLUSTER1} -n istio-gateways patch svc istio-ingressgateway --patch "{\"spec\": {\"selector\": {\"revision\": \"${NEW_REVISION}\" }}}"
```

Test that you can still access the `in-mesh` service:

```bash
curl -k "https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get" -I
```

You should get a response similar to the following one:

```
HTTP/2 200 
server: istio-envoy
date: Wed, 24 Aug 2022 14:58:22 GMT
content-type: application/json
content-length: 670
access-control-allow-origin: *
access-control-allow-credentials: true
```

Now that everything is working well with the new version, we can uninstall the previous version:

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
      revision: ${NEW_REVISION}
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
        activeGateway: true
      gatewayRevision: ${NEW_REVISION}
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
EOF
```

Run the following command:

```bash
kubectl --context ${CLUSTER1} -n istio-system get pods && kubectl --context ${CLUSTER1} -n istio-gateways get pods
```

You should get the following output:

```
NAME                           READY   STATUS    RESTARTS   AGE
istiod-1-17-577c47cf5c-m6mf4   1/1     Running   0          3m16s
NAME                                          READY   STATUS    RESTARTS   AGE
istio-ingressgateway-1-17-d64766d6b-bmpft    1/1     Running   0          100s
```

It confirms that only the new version is running.


## Lab 16 - Configure a mutual TLS ingress gateway <a name="lab-16---configure-a-mutual-tls-ingress-gateway-"></a>

Istio can be configured to verify downstream client certificates. This is done automatically if the gateway TLS secret used also contains a root CA. The server uses the CA certificate to verify its clients

Let's generate a new set of TLS certs. For our example, we will create them in a new `mtls-gateway` directory

```
mkdir mtls-gateway

# tls cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
   -keyout mtls-gateway/tls.key -out mtls-gateway/tls.crt -subj "/CN=*"

# mtls cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
   -keyout mtls-gateway/mtls.key -out mtls-gateway/mtls.crt -subj "/CN=*"
```

Check that the new TLS certs were created

```
% ls mtls-gateway 
mtls.crt        mtls.key        tls.crt         tls.key
```

Now we can create a new Kubernetes secret named `mtls-secret` using these certificates

```
kubectl create -n istio-gateways secret generic mtls-secret \
  --from-file=tls.key=mtls-gateway/tls.key \
  --from-file=tls.crt=mtls-gateway/tls.crt \
  --from-file=ca.crt=mtls-gateway/mtls.crt
```

Update our existing VirtualGateway to use this `mtls-secret`, and set the `tls.mode` to `MUTUAL` to validate incoming client mTLS

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
      httpsRedirect: true
    - http: {}
      port:
        number: 443
# ---------------- mTLS config ---------------------------
      tls:
        mode: MUTUAL
        secretName: mtls-secret
# -------------------------------------------------------
      allowedRouteTables:
        - host: '*'
EOF
```

Now attempt to send an HTTPS request using only TLS and see how it fails:

```
% curl -kv https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get --cert mtls-gateway/tls.crt --key mtls-gateway/tls.key
```

You should see an error similar to below:
```
*   Trying 127.0.0.1:443...
* Connected to localhost (127.0.0.1) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (OUT), TLS handshake, Client hello (1):
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Request CERT (13):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Certificate (11):
* (304) (OUT), TLS handshake, CERT verify (15):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*
*  start date: Apr 10 15:52:40 2023 GMT
*  expire date: Apr  9 15:52:40 2024 GMT
*  issuer: CN=*
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x11800ca00)
> GET /get HTTP/2
> Host: httpbin-local.glootest.com
> user-agent: curl/7.79.1
> accept: */*
> 
* LibreSSL SSL_read: error:1404C418:SSL routines:ST_OK:tlsv1 alert unknown ca, errno 0
* Failed receiving HTTP2 data
* LibreSSL SSL_write: SSL_ERROR_SYSCALL, errno 0
* Failed sending HTTP2 data
* Connection #0 to host localhost left intact
curl: (56) LibreSSL SSL_read: error:1404C418:SSL routines:ST_OK:tlsv1 alert unknown ca, errno 0
```

Now, attempt to curl again but this time provide the `mtls.crt` and `mtls.key` client certificate and private key

```
curl -kv https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get --cert mtls-gateway/mtls.crt --key mtls-gateway/mtls.key
```

This should succeed
```
% curl -kv https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get --cert mtls.crt --key mtls.key
*   Trying 127.0.0.1:443...
* Connected to localhost (127.0.0.1) port 443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*  CAfile: /etc/ssl/cert.pem
*  CApath: none
* (304) (OUT), TLS handshake, Client hello (1):
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Request CERT (13):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Certificate (11):
* (304) (OUT), TLS handshake, CERT verify (15):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256
* ALPN, server accepted to use h2
* Server certificate:
*  subject: CN=*
*  start date: Apr 10 15:52:40 2023 GMT
*  expire date: Apr  9 15:52:40 2024 GMT
*  issuer: CN=*
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
* Using HTTP2, server supports multiplexing
* Connection state changed (HTTP/2 confirmed)
* Copying HTTP/2 data in stream buffer to connection buffer after upgrade: len=0
* Using Stream ID: 1 (easy handle 0x130012e00)
> GET /get HTTP/2
> Host: httpbin-local.glootest.com
> user-agent: curl/7.79.1
> accept: */*
> 
* Connection state changed (MAX_CONCURRENT_STREAMS == 2147483647)!
< HTTP/2 200 
< server: istio-envoy
< date: Mon, 10 Apr 2023 16:59:05 GMT
< content-type: application/json
< access-control-allow-origin: *
< access-control-allow-credentials: true
< x-envoy-upstream-service-time: 9
< content-length: 1974
< 
{
  "args": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "httpbin-local.glootest.com", 
    "User-Agent": "curl/7.79.1", 
    "X-B3-Parentspanid": "bda0797f91efc059", 
    "X-B3-Sampled": "0", 
    "X-B3-Spanid": "2ca32dfa3081f9e6", 
    "X-B3-Traceid": "cf8ef719219f3093bda0797f91efc059", 
    "X-Envoy-Attempt-Count": "1", 
    "X-Envoy-Internal": "true", 
    "X-Forwarded-Client-Cert": "Hash=12918ade28133e2af82c13758baba46a3ebbfb86693fa5ae0d9f4c4198942bc9;Cert=\"-----BEGIN%20CERTIFICATE-----%0AMIIC%2BTCCAeGgAwIBAgIUY0DWKWCz5iYfcmbjnwX1MIwaj7wwDQYJKoZIhvcNAQEL%0ABQAwDDEKMAgGA1UEAwwBKjAeFw0yMzA0MTAxNTUyNDNaFw0yNDA0MDkxNTUyNDNa%0AMAwxCjAIBgNVBAMMASowggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCT%0AD6kdIWO1ZUPKLzpAJ3KXrYl%2F6CDZDfHYb5iCf%2BkdwymlJ%2FL3FqNBk6ufg27N69OW%0AK2N%2FkHa707E6%2FFysH4qX%2FbJ2nCC4CDxiTU972y56sduHkpv7%2F%2FpkgUD2Cwir0j%2FH%0AOrmsEo60XZq4hsFVX7XMs4MojhW1IojVvcANj5x8wV16%2Bs6Pfbstw5dtf86zVRqT%0AoOXLnlE%2FMZ0LjFg%2FJxetEwD9RpMXIZp44uriPk4Ja0Q0RJ2hzBhbPah8J77FcuC2%0AAii1OLQwGHgcCQQQ%2FLKs1ZRO07WAqgGibQytCGGWScxX2cNW3%2BgslvbEVAno2XGG%0ABGdTq30DhL7u%2FCILB6XnAgMBAAGjUzBRMB0GA1UdDgQWBBRD7km0jt7S1Ry%2FZ9T2%0Azl8KJ%2F1bmTAfBgNVHSMEGDAWgBRD7km0jt7S1Ry%2FZ9T2zl8KJ%2F1bmTAPBgNVHRMB%0AAf8EBTADAQH%2FMA0GCSqGSIb3DQEBCwUAA4IBAQBnU5hnCnghSQHFOiBWYcmGRdkU%0APtoXTM6KVIxzS%2F11DokIfiIp9mVbUpx3i%2BYSKY8NQ6%2BkyNkWQkrWaVDumn%2Fk6Q0s%0A%2F2uJ3H6L6Wv0xavudG7Jz5iqiTTJG%2FEBfEhaXffyHxjsq69wu4a3GAQ4WPvA%2BnhX%0AY%2FQBLlP%2BCbtrFBFlx%2F0TLyPnSJ2YvuGe53FV3341r1N9%2BlM6%2FrC6xVzHHWt0u%2BVO%0AS3qFbTTWKFWqFD1u6KiOzm6DrIOJCdej7X1ZHNYKx4WsPQ6fpZlzYECPsx%2FbnS7F%0AN6Nb1AtqxEdqE6gVAzg7UX9uuNAi8VbQbwxddvxpk5LtahZ1b3lWjsPYVLnR%0A-----END%20CERTIFICATE-----%0A\";Subject=\"CN=*\";URI=,By=spiffe://mgmt/ns/httpbin/sa/in-mesh;Hash=ed5b55e8df6f9845f52506d6e760d242f475ff84a0a551d97a988073b54868a2;Subject=\"\";URI=spiffe://mgmt/ns/istio-gateways/sa/istio-ingressgateway-1-17"
  }, 
  "origin": "10.42.0.1", 
  "url": "https://httpbin-local.glootest.com/get"
}
* Connection #0 to host localhost left intact
```

To revert back to our original TLS gateway configuration you can apply the config below

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
      httpsRedirect: true
    - http: {}
      port:
        number: 443
# ---------------- SSL config ---------------------------
      tls:
        mode: SIMPLE
        secretName: tls-secret
# -------------------------------------------------------
      allowedRouteTables:
        - host: '*'
EOF
```

## Lab 17 - Install and Configure Keycloak <a name="lab-17---install-and-configure-keycloak-"></a>

First you can use the following script to deploy your own OIDC provider `keycloak` which will allow you to secure your website with a user/pass login

```bash
./keycloak/setup.sh
```

The output should look similar to below:

```bash
% ./keycloak/setup.sh
namespace/keycloak created
service/keycloak created
deployment.apps/keycloak created
deployment.apps/keycloak condition met
{"ingress":[{"ip":"172.24.0.3"}]}
serviceaccount/keycloak-setup created
rolebinding.rbac.authorization.k8s.io/keycloak-setup-admin created
pod/keycloak-setup created
pod/keycloak-setup condition met
<...omitted...>

Client ID: 7158eaae-caf6-4e53-adad-bc2e744da531
Client Secret: cfb0cade-8791-4636-8830-c507f0476fc9
secret/oauth created
configmap/keycloak-info created
ClientId: 7158eaae-caf6-4e53-adad-bc2e744da531
```

Set the following outputs as variables:

```bash
# set this
export KEYCLOAK_CLIENT_SECRET="<INPUT_CLIENT_SECRET>"

export KEYCLOAK_CLIENTID=$(kubectl get configmap -n gloo-mesh keycloak-info -o json | jq -r '.data."client-id"')
export APP_CALLBACK_URL="https://localhost"
```

Check that the variables have correct values:
```
echo $KEYCLOAK_CLIENTID
echo $KEYCLOAK_CLIENT_SECRET
echo $APP_CALLBACK_URL
```

Now lets create the client secret as a kubernetes secret

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-client-secret
  namespace: gloo-mesh
type: extauth.solo.io/oauth
data:
  client-secret: $(echo -n ${KEYCLOAK_CLIENT_SECRET} | base64)
EOF
```

After that, you need to create an `ExtAuthServer`, which is a CRD that define which extauth server to use. If you have been following previous labs this step may have already been completed

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
EOF
```

Now apply the extauth policy

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: httpbin-keycloak-extauth
  namespace: httpbin
spec:
  applyToRoutes:
  - route:
      labels:
        oidc: "keycloak"
  config:
    server:
      name: cluster1-ext-auth-server
      namespace: httpbin
      cluster: cluster1
    glooAuth:
      configs:
      - oauth2:
          oidcAuthorizationCode:
            appUrl: https://localhost
            callbackPath: /callback
            clientId: ${KEYCLOAK_CLIENTID}
            clientSecretRef:
              name: keycloak-client-secret
              namespace: gloo-mesh
            issuerUrl: "http://keycloak.keycloak:9000/auth/realms/master/"
            session:
              failOnFetchFailure: true
              redis:
                cookieName: keycloak-oidc-session
                options:
                  host: redis.gloo-mesh-addons:6379
                allowRefreshing: true
              cookieOptions:
                maxAge: "90"
                notSecure: true
            scopes:
            - email
            - profile
            headers:
              idTokenHeader: Jwt
EOF
```

Add callback redirect in keycloak UI:

Keycloak has been exposed by a LoadBalancer at http://localhost:9000/auth

![](images/keycloak/keycloak1.png)

Login to the adminstration console using `admin/admin`

In the left sidebar, click on Clients and select the ClientID that we defined above. In this example it is `7158eaae-caf6-4e53-adad-bc2e744da531`

![](images/keycloak/keycloak2.png)

Add a Valid Redirect URI to this Client. If using K3d, this would be `https://localhost/callback` and then hit save at the bottom of the screen

![](images/keycloak/keycloak3.png)

Modify /etc/hosts on your local machine:

In K3d, in order for the OIDC flow to resolve correctly when running locally in Docker we will need to modify our `/etc/hosts` file to include a new entry. Note you may need `sudo` privileges on your local machine to do so.
```
127.0.0.1 localhost keycloak.keycloak
```

Next will need to update our route table with our callback path and ExtAuthPolicy route label if we havent done so already

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
        oidc: keycloak
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

Now when you access your httpbin app through the browser, it will be protected by the `keycloak` OIDC provider login page.
```
echo "https://${ENDPOINT_HTTPS_GW_CLUSTER1}/get"
```

![](images/keycloak/keycloak4.png)

User Credentials:
Below are a few users that you can validate with
- Username: gloo-mesh // Password: solo.io

![](images/keycloak/keycloak5.png)


## Lab 18 - Integrate Okta with Keycloak <a name="lab-18---integrate-okta-with-keycloak-"></a>

Now that we have Keycloak set up, we can also integrate it with the Okta OIDC Provider. 

This [Doc](https://maybeitdepends.com/keycloak-integration-with-okta) was very helpful as a reference for these steps. 

Configure the Okta side:
First we will configure a new App Registration in Okta. The instructions in this [Doc](https://maybeitdepends.com/keycloak-integration-with-okta) will guide you to do a few things
```
- Create a new Okta Application Integration
- Find your authorization server URL from Okta in the Security > API tab (i.e. `https://dev-12345678.okta.com/oauth2/default/.well-known/oauth-authorization-server`)
```

Configure the Keycloak side:
Now that you have your Okta app configured and have identified the Okta authorization server URL, we can just simply import the configuration.

In the Keycloak Administrator Console, select Identity Providers in the sidebar and add the OpenID Connect v1.0 Provider

![](images/keycloak/keycloak6.png)

Provide an alias for your Identity Provider such as `oidc-okta` for example. Take note of the `Redirect URI`. We will need to copy this into our Okta application Sign-in Redirect URIs in the last step

![](images/keycloak/keycloak7.png)


Then finally at the bottom of the Add Identity Providers page we can Import an External IDP Config by providing our Okta Authorization Server URL found in an earlier step. This should automatically import a few entries in the form

![](images/keycloak/keycloak8.png)

Then we need to provide the Okta Application's `Client Authentication`, `Client ID`, and `Client Secret` to finish up the OpenID Connect Config

![](images/keycloak/keycloak9.png)

Finally, back in Okta we will need to add the Identity Provider Redirect URI to our Okta Application

![](images/keycloak/keycloak10.png)

Now if you refresh the httpbin application in the browser or open it in Incognito mode we should see that Keycloak has now been integrated with Okta as an additional OIDC Provider

![](images/keycloak/keycloak11.png)

Selecting the `oidc-okta` provider should take you to an Okta Login. Note that when running locally, we may have issues with this integration resolving to `localhost` without some workarounds.


## Lab 19 - Enable JWT Validation at the Gateway <a name="lab-19---enable-jwt-validation-at-the-gateway-"></a>

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

## Lab 20 - Using Passthrough External Auth <a name="lab-20---using-passthrough-external-auth-"></a>

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
kubectl --context ${CLUSTER1} create namespace gloo-mesh-addons
kubectl --context ${CLUSTER1} label namespace gloo-mesh-addons istio.io/rev=1-17 --overwrite

helm upgrade --install gloo-mesh-agent-addons gloo-mesh-agent/gloo-mesh-agent \
--namespace gloo-mesh-addons \
--kube-context=${CLUSTER1} \
--version 2.4.0-rc1 \
--values - <<EOF
glooMeshAgent:
  enabled: false
rate-limiter:
  enabled: true
ext-auth-service:
  enabled: true
  extAuth:
    image:
      pullPolicy: IfNotPresent
      registry: registry.hub.docker.com
      repository: ably77/ext-auth-service
      tag: amd64-ext-auth-service-0.35.0-poc
EOF
```

Check to see that the `ext-auth-service` is deployed

```bash
kubectl --context ${CLUSTER1} get pods -n gloo-mesh-addons
```

output should look like this

```bash
% kubectl --context ${CLUSTER1} get pods -n gloo-mesh-addons
NAME                               READY   STATUS    RESTARTS   AGE
rate-limiter-64b64b779c-xrtsn      2/2     Running   0          19m
redis-578865fd78-rgjqm             2/2     Running   0          19m
ext-auth-service-76d8457d9-d69k9   2/2     Running   0          92s
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



## Lab 21 - Routing to a GRPC Service <a name="lab-21---routing-to-a-grpc-service-"></a>

In this lab we are going to explore routing to a GRPC service through the gateway. In this example we will use a GRPC service `currency` that converts one money amount to another currency

Since we can treat this as a new app/team we can create a new `Workspace` and `WorkspaceSettings` for our currency service

First we need to create the `currency` namespace and label it for istio injection
```
kubectl --context ${CLUSTER1} create namespace currency
kubectl --context ${CLUSTER1} label namespace currency istio.io/rev=1-17 --overwrite
```

Next we can configure the Workspace

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: Workspace
metadata:
  name: currency
  namespace: gloo-mesh
  labels:
    allow_ingress: "true"
spec:
  workloadClusters:
  - name: cluster1
    namespaces:
    - name: currency
EOF
```

Then we can deploy the WorkspaceSettings

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: admin.gloo.solo.io/v2
kind: WorkspaceSettings
metadata:
  name: currency
  namespace: currency
spec:
  importFrom:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
  exportTo:
  - workspaces:
    - name: gateways
    resources:
    - kind: SERVICE
      labels:
        app: currencyservice
    - kind: ALL
      labels:
        expose: "true"
EOF
```

Next we can deploy the `currency` GRPC service into the `currency` namespace

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: currency
  namespace: currency
---
    apiVersion: v1
    kind: Service
    metadata:
      name: currencyservice
      namespace: currency
      labels:
        app: currencyservice
    spec:
      selector:
        app: currencyservice
      ports:
      - name: grpc
        port: 7000
        targetPort: 7000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: currencyservice
  namespace: currency
spec:
  selector:
    matchLabels:
      app: currencyservice
  template:
    metadata:
      labels:
        app: currencyservice
      annotations:
        proxy.istio.io/config: '{ "holdApplicationUntilProxyStarts": true }'
        sidecar.istio.io/inject: "true"
    spec:
      serviceAccountName: currency
      terminationGracePeriodSeconds: 5
      containers:
      - name: server
        image: gcr.io/solo-test-236622/currencyservice:solo-build
        ports:
        - name: grpc
          containerPort: 7000
        env:
        - name: PORT
          value: "7000"
        - name: DISABLE_TRACING
          value: "1"
        - name: DISABLE_PROFILER
          value: "1"
        - name: DISABLE_DEBUGGER
          value: "1"
        readinessProbe:
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:7000"]
        livenessProbe:
          exec:
            command: ["/bin/grpc_health_probe", "-addr=:7000"]
EOF
```

If you have not already done so from Lab 6, apply the parent `RouteTable` to configure the main routing 

```bash
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

Now we can deploy a RouteTable for our currency service

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: currency
  namespace: currency
  labels:
    expose: "true"
spec:
  http:
    - name: currency
      labels:
        route_name: currency
      matchers:
      - uri:
          prefix: /hipstershop.CurrencyService/Convert
      forwardTo:
        destinations:
          - ref:
              name: currencyservice
              namespace: currency
              cluster: cluster1
            port:
              number: 7000
EOF
```

Test that the route is serving traffic. In order to properly test, you will need `grpcurl` installed and it can be downloaded here: [grpcurl installation](https://github.com/fullstorydev/grpcurl#installation)

```bash
grpcurl --insecure --proto example-config/grpc/online-boutique.proto -d '{ "from": { "currency_code": "USD", "nanos": 44637071, "units": "31" }, "to_code": "JPY" }' ${ENDPOINT_HTTPS_GW_CLUSTER1} hipstershop.CurrencyService/Convert
```

Output should look similar to below:
```bash
% grpcurl --insecure --proto example-config/grpc/online-boutique.proto -d '{ "from": { "currency_code": "USD", "nanos": 44637071, "units": "31" }, "to_code": "JPY" }' ${ENDPOINT_HTTPS_GW_CLUSTER1} hipstershop.CurrencyService/Convert
{
  "currencyCode": "JPY",
  "units": "3471",
  "nanos": 67780486
}
```

## Lab 22 - Securing GRPC Service with ExtAuthPolicy <a name="lab-22---securing-grpc-service-with-extauthpolicy-"></a>

Prerequisites to complete this lab:
- Labs 1-9 Completed
- Lab 21

Now that we have exposed our GRPC application using a RouteTable, let's continue by securing this service with an `ExtAuthPolicy`. This is pretty simple as we have already completed a lot of the steps in Lab 9 and can re-use some of these components

```
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: security.policy.gloo.solo.io/v2
kind: ExtAuthPolicy
metadata:
  name: currency-extauth
  namespace: currency
spec:
  applyToRoutes:
  - route:
      labels:
        route_name: "currency"
  config:
    server:
      name: cluster1-ext-auth-server
      namespace: httpbin
      cluster: cluster1
    glooAuth:
      configs:
      - oauth2:
          oidcAuthorizationCode:
            appUrl: ${APP_CALLBACK_URL}
            callbackPath: /callback
            clientId: ${OIDC_CLIENT_ID}
            clientSecretRef:
              name: oidc-client-secret
              namespace: gloo-mesh
            issuerUrl: ${ISSUER_URL}
            session:
              failOnFetchFailure: true
              redis:
                cookieName: oidc-session
                options:
                  host: redis.gloo-mesh-addons:6379
                allowRefreshing: true
              cookieOptions:
                maxAge: "90"
            scopes:
            - email
            - profile
            headers:
              idTokenHeader: Jwt
EOF
```

Now let's try to curl our GRPC service again. Since we are not passing in a valid token, this request should fail

```bash
grpcurl --insecure --proto example-config/grpc/online-boutique.proto -d '{ "from": { "currency_code": "USD", "nanos": 44637071, "units": "31" }, "to_code": "JPY" }' ${ENDPOINT_HTTPS_GW_CLUSTER1} hipstershop.CurrencyService/Convert
```

Output should look similar to below:

```bash
% grpcurl --insecure --proto example-config/grpc/online-boutique.proto -d '{ "from": { "currency_code": "USD", "nanos": 44637071, "units": "31" }, "to_code": "JPY" }' ${ENDPOINT_HTTPS_GW_CLUSTER1} hipstershop.CurrencyService/Convert
ERROR:
  Code: Unknown
  Message:
```

If we provide a valid access token in the curl request, it should succeed

Set your access token variable. A simple way to get this would be to grab it from our `httpbin` app that we enabled with ext auth earlier. Navigate and login to your httpbin app through the browser

```bash
echo "${APP_CALLBACK_URL}/get"
```

You can use the JWT provided in the `jwt` header as this is from the same OIDC application

```bash
{
  "args": {}, 
  "headers": {
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7", 
    "Accept-Encoding": "gzip, deflate, br", 
    "Accept-Language": "en-US,en;q=0.9", 
    "Cache-Control": "max-age=0", 
    "Cookie": "oidc-session=TIPANHW4ITKNEJOFXJVHINHMRWER3ZMYZZYOULXZGOHKTYEZK5HGOCTHCNGQDAGRVY5EBLBYALBTFBHGNTBZIVB546NKZ6MH5TIFKOA=", 
    "Host": "localhost", 
    "Jwt": "eyJraWQiOiJ4dm1UNTJKLXlyUHM1eG9rR1J2NDAxZzQ4dkxkZkFBRUtZX2NMLWdLTGo0IiwiYWxnIjoiUlMyNTYifQ.eyJzdWIiOiIwMHUxbThjeDNlM2RTUlRSdzVkNyIsIm5hbWUiOiJhbGV4IiwiZW1haWwiOiJhbGV4Lmx5QHNvbG8uaW8iLCJ2ZXIiOjEsImlzcyI6Imh0dHBzOi8vZGV2LTIyNjUzMTU4Lm9rdGEuY29tL29hdXRoMi9kZWZhdWx0IiwiYXVkIjoiMG9hNnF2enliY1ZDSzZQY1M1ZDciLCJpYXQiOjE2ODMzMjMzNDcsImV4cCI6MTY4MzMyNjk0NywianRpIjoiSUQuTnhfcmQ3bTB1b29qOVluRWVQWW94cDdpX1FNRDJpbGE1dUktRVR2bTBvOCIsImFtciI6WyJwd2QiXSwiaWRwIjoiMDBvMW04Y3d4cUtrbXM1b2M1ZDciLCJwcmVmZXJyZWRfdXNlcm5hbWUiOiJhbGV4Lmx5QHNvbG8uaW8iLCJhdXRoX3RpbWUiOjE2ODMzMjExODUsImF0X2hhc2giOiJYLThDTHVVRE1iV2I4NFBDd1pKVDh3IiwiZ3JvdXBzIjpbIkV2ZXJ5b25lIiwib3JnMS1hZG1pbnMiLCJ0aHJvd2F3YXktYWRtaW4iXX0.G-AjlzDiWiFmSdZr947Fm1g_e2DZVKxYbu80O26XksAbAORtF_-qZgTVbHT_m-YsK4I9fyTR5NgdFiTtvVf5ROTlPEYTgOzkguxG8et4K6F4Lu8lqCKWrT8k3RYm9-eatctgekXICCGLsPNAwLMZHAMnfaoaGZ6i9iNTiTe0cFVBUNOzDU5cTUrquv7FMBytJZ70wsX1LbyXYNkOV8FFgmEsVHuSQyg3irZl3XO5vQr6Ra-TzO3s44ueT_xMveU10cOMnkzu3t2GRZ96EcOEd-kuFTL_frUfZLKL5wM-d8FKjAboqZsx5ny0uziEKg75EGoaGdBKGT0g4KlMTaOM0Q"
<...>
}
```

Set your access token below:
```bash
ACCESS_TOKEN="<insert jwt token here>"
```

Now retry the curl command with your access token provided

```bash
grpcurl -H "Authorization: Bearer ${ACCESS_TOKEN}" --insecure --proto example-config/grpc/online-boutique.proto -d '{ "from": { "currency_code": "USD", "nanos": 44637071, "units": "31" }, "to_code": "JPY" }' ${ENDPOINT_HTTPS_GW_CLUSTER1} hipstershop.CurrencyService/Convert
```

Output should look similar to below:

```bash
% grpcurl -H "Authorization: Bearer ${ACCESS_TOKEN}" --insecure --proto example-config/grpc/online-boutique.proto -d '{ "from": { "currency_code": "USD", "nanos": 44637071, "units": "31" }, "to_code": "JPY" }' ${ENDPOINT_HTTPS_GW_CLUSTER1} hipstershop.CurrencyService/Convert
{
  "currencyCode": "JPY",
  "units": "3471",
  "nanos": 67780486
}
```

Congrats! You have now just secured your GRPC service using ExtAuthPolicy!

## Lab 23 - Apply rate limiting to the GRPC service <a name="lab-23---apply-rate-limiting-to-the-grpc-service-"></a>

Prerequisites to complete this lab:
- Labs 1-9 Completed
- Lab 11
- Lab 21

Similar to Lab 11, we can set up rate-limiting for our GRPC service. This time, we can set our rate limit for this GRPC service to 10 requests per minute

```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: RateLimitClientConfig
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: currency
  namespace: currency
spec:
  raw:
    rateLimits:
      - actions:
          - genericKey:
              descriptorValue: counter
---
apiVersion: admin.gloo.solo.io/v2
kind: RateLimitServerConfig
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: currency
  namespace: gloo-mesh-addons
spec:
  destinationServers:
  - port:
      name: grpc
    ref:
      cluster: cluster1
      name: rate-limiter
      namespace: gloo-mesh-addons
  raw:
    descriptors:
      - key: generic_key
        rateLimit:
          requestsPerUnit: 10
          unit: MINUTE
        value: counter
---
apiVersion: trafficcontrol.policy.gloo.solo.io/v2
kind: RateLimitPolicy
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: currency
  namespace: currency
spec:
  applyToRoutes:
  - route:
      labels:
        ratelimited: "true"
  config:
    ratelimitClientConfig:
      cluster: cluster1
      name: currency
      namespace: currency
    ratelimitServerConfig:
      cluster: cluster1
      name: currency
      namespace: gloo-mesh-addons
    serverSettings:
      cluster: cluster1
      name: rate-limit-server
      namespace: currency
---
apiVersion: admin.gloo.solo.io/v2
kind: RateLimitServerSettings
metadata:
  labels:
    workspace.solo.io/exported: "true"
  name: rate-limit-server
  namespace: currency
spec:
  destinationServer:
    port:
      name: grpc
    ref:
      cluster: cluster1
      name: rate-limiter
      namespace: gloo-mesh-addons
EOF
```

Then we can update our `RouteTable` with the `ratelimited: true` label
```bash
kubectl --context ${CLUSTER1} apply -f - <<EOF
apiVersion: networking.gloo.solo.io/v2
kind: RouteTable
metadata:
  name: currency
  namespace: currency
  labels:
    expose: "true"
spec:
  http:
    - name: currency
      labels:
        route_name: currency
        ratelimited: "true"
      matchers:
      - uri:
          prefix: /hipstershop.CurrencyService/Convert
      forwardTo:
        destinations:
          - ref:
              name: currencyservice
              namespace: currency
              cluster: cluster1
            port:
              number: 7000
EOF
```

Now retry the curl command with your access token provided, it should error after 10 req/min
```
for i in {1..11}; do
  grpcurl -H "Authorization: Bearer ${ACCESS_TOKEN}" --insecure --proto example-config/grpc/online-boutique.proto -d '{ "from": { "currency_code": "USD", "nanos": 44637071, "units": "31" }, "to_code": "JPY" }' ${ENDPOINT_HTTPS_GW_CLUSTER1} hipstershop.CurrencyService/Convert
  sleep 1
done
```

Note: if you did not complete Lab 22, you can remove the token from your curl request. Rate limiting is not dependent on the ext auth policy to function

Output should look similar to below:

```bash
% grpcurl -H "Authorization: Bearer ${ACCESS_TOKEN}" --insecure --proto example-config/grpc/online-boutique.proto -d '{ "from": { "currency_code": "USD", "nanos": 44637071, "units": "31" }, "to_code": "JPY" }' ${ENDPOINT_HTTPS_GW_CLUSTER1} hipstershop.CurrencyService/Convert
ERROR:
  Code: Unavailable
  Message:
```

Congrats! You have now applied rate limiting to your GRPC service!