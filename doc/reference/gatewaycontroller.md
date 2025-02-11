# GatewayController

The GatewayController is an implementation of
[Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/), it watches
Kubernetes Gateway, HTTPRoute, Service, and Secrets then translates them to
Easegress HTTP server and pipelines.

**Note** the currenct GatewayController is an experimental implementation, NOT
all the features of the `Kubernetes Gateway API` features are supported.

## Prerequisites

1. K8s cluster : **v1.23+**
2. Gateway API: following https://gateway-api.sigs.k8s.io/guides/#installing-gateway-api
   to install.

## Getting Started

### Role Based Access Control configuration

If your cluster is configured with RBAC, first you will need to authorize
Easegress GatewayController for using the Kubernetes API. Below is an example
configuration:

```yaml
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: easegress-gateway-controller
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["services", "secrets", "endpoints", "namespaces"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["gateway.networking.k8s.io"]
  resources: ["gatewayclasses", "httproutes", "gateways"]
  verbs: ["get", "watch", "list"]

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: easegress-gateway-controller
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: easegress-gateway-controller
subjects:
- kind: ServiceAccount
  name: easegress-gateway-controller
  namespace: default
roleRef:
  kind: ClusterRole
  name: easegress-gateway-controller
  apiGroup: rbac.authorization.k8s.io
```

Note the name of the ServiceAccount we just created is `easegress-gateway-controller`,
it will be used later.

### Config Map

Let's use ConfigMap to store Easegress server configuration and Easegress
gateway configuration. This ConfigMap is used later in Deployment.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: easegress-cm
  namespace: default
data:
  easegress-server.yaml: |
    name: gateway-easegress
    cluster-name: easegress-gateway-controller
    cluster-role: primary
    api-addr: 0.0.0.0:2381
    data-dir: /opt/easegress/data
    log-dir: /opt/easegress/log
    debug: false
  controller.yaml: |
    kind: GatewayController
    name: gateway-controller-example
    kubeConfig:
    masterURL:
    namespaces: []
```

The `easegress-server.yaml` creates Easegress instance named *easegress-gateway-controller*
and `controller.yaml` defines GatewayController object for Easegress.

### Deploy Easegress GatewayController

To deploy the GatewayController, we will create a Deployment and a Service as below:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: easegress-gateway
  name: easegress
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: easegress-gateway
  template:
    metadata:
      labels:
        app: easegress-gateway
    spec:
      serviceAccountName: easegress-gateway-controller
      containers:
      - args:
        - -c
        - |-
          /opt/easegress/bin/easegress-server \
            -f /opt/eg-config/easegress-server.yaml \
            --initial-object-config-files /opt/eg-config/controller.yaml \
            --initial-cluster $(EG_NAME)=http://localhost:2380
        command:
        - /bin/sh
        env:
        - name: EG_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.name
        image: megaease/easegress:latest
        imagePullPolicy: IfNotPresent
        name: easegress-primary
        resources:
          limits:
            cpu: 1200m
            memory: 2Gi
          requests:
            cpu: 100m
            memory: 256Mi
        volumeMounts:
        - mountPath: /opt/eg-config/easegress-server.yaml
          name: easegress-cm
          subPath: easegress-server.yaml
        - mountPath: /opt/eg-config/controller.yaml
          name: easegress-cm
          subPath: controller.yaml
        - mountPath: /opt/easegress/data
          name: gateway-data-volume
        - mountPath: /opt/easegress/log
          name: gateway-data-volume
      restartPolicy: Always
      volumes:
      - emptyDir: {}
        name: gateway-data-volume
      - configMap:
          defaultMode: 420
          items:
          - key: easegress-server.yaml
            path: easegress-server.yaml
          - key: controller.yaml
            path: controller.yaml
          name: easegress-cm
        name: easegress-cm
```

The GatewayController is created via the command line argument `initial-object-config-files`
of `easegress-server`. Note that Easegress logs and data are stored to *emptyDir* called
`gateway-data-volume` inside the pod as GatewayController is stateless so we can
restart new pods without preserving previous state.

### Deploy Backend Services

Apply below YAML configuration to Kubernetes:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deployment
spec:
  selector:
    matchLabels:
      app: products
      department: sales
  replicas: 2
  template:
    metadata:
      labels:
        app: products
        department: sales
    spec:
      containers:
      - name: hello-v1
        image: "us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0"
        env:
        - name: "PORT"
          value: "50001"
      - name: hello-v2
        image: "us-docker.pkg.dev/google-samples/containers/gke/hello-app:2.0"
        env:
        - name: "PORT"
          value: "50002"

---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  type: NodePort
  selector:
    app: products
    department: sales
  ports:
  - name: port-v1
    protocol: TCP
    port: 60001
    targetPort: 50001
  - name: port-v2
    protocol: TCP
    port: 60002
    targetPort: 50002
```

### Deploy Gateway API Objects

Please refer https://gateway-api.sigs.k8s.io/ for the detailed information
of Gateway Class, Gateway and HTTPRoute configuration. Up to now, the Easegress
GatewayController only support HTTP & HTTPS listeners.

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: GatewayClass
metadata:
  name: example-gateway-class
spec:
  controllerName: "megaease.com/gateway-controller"

---

apiVersion: gateway.networking.k8s.io/v1beta1
kind: Gateway
metadata:
  name: example-gateway
spec:
  gatewayClassName: example-gateway-class
  listeners:
  - protocol: HTTP
    port: 8080
    name: example-listener

---

apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: example-route
spec:
  parentRefs:
  - kind: Gateway
    name: example-gateway
    sectionName: example-listener
  rules:
  - matches:
    - path:
        value: /1
    backendRefs:
    - name: hello-service
      port: 60001

---

apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: example-route-2
spec:
  parentRefs:
  - kind: Gateway
    name: example-gateway
    sectionName: example-listener
  hostnames:
    - megaease.com
  rules:
  - matches:
    - path:
        value: /2
    backendRefs:
    - name: hello-service
      port: 60002
```

### Expose The Listener Port

Create service to forward the incoming traffic to Listener in Easegress.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: easegress-public
  namespace: default
spec:
  ports:
  - name: web
    protocol: TCP
    port: 8080
    nodePort: 30080
  selector:
    app: easegress-gateway
  type: NodePort
  ```

The port `web` is to receive external HTTP requests from port 30080 and forward
them to the HTTP server created according to the Listener configuration of
the Gateway.

### Verification

Once all configurations are applied, we can leverage the command below to access
both versions of the `hello` application:

```bash
$ curl http://127.0.0.1:30080/1
Hello, world!
Version: 2.0.0
Hostname: hello-deployment-7855bc9747-n4qsx

$ curl http://127.0.0.1:30080/2 -i
HTTP/1.1 404 Not Found
Date: Wed, 19 Jul 2023 06:59:07 GMT
Content-Length: 0
Connection: close

$ curl http://127.0.0.1:30080/2 -H Host:megaease.com
Hello, world!
Version: 2.0.0
Hostname: hello-deployment-7855bc9747-n4qsx
```
