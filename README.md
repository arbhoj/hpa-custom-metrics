# Autoscale Kubernetes pods using HPA & Custom-Metrics
The purpose of this repo is to provide an example for autoscaling a kubernetes pod using [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) and [Custom-Metrics](https://github.com/kubernetes-sigs/custom-metrics-apiserver). It specifically demonstrates the strength and flexibility of this feature to autoscale pods based on ingress requests received by a particular service behind an ingress being served by [Traefik](https://traefik.io) ingress controller. 

## Pre-reqs:
- Kubernetes cluster 1.24.x+ with the following components
    - Traefik 2.x
    - Prometheus-Stack
    - kube-prometheus-stack 46.8.0
    - prometheus-adapter 4.2.1
>[DKP](https://d2iq.com/kubernetes-platform) comes pre-built with all these components so start a [self hosted 30 day trial](https://d2iq.com/self-hosted-free-trial) today!

## Steps:

### Step 1: Deploy a workload (deployment/service/ingress etc.) to the any namespace.

```
export NAMESPACE=autoscale
export SERVICE_NAME=scaletest
export SERVICE_PORT=80
export APP_NAME=scaletest
export CLUSTER_VIP=<cluster-ingress-vip>

kubectl create ns ${NAMESPACE}

kubectl apply -f - <<EOF
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mainpage
  namespace: ${NAMESPACE}
data:
  index.html: |
    <html>
        <body>
            <h1>D2iQ Kubernetes Platform (DKP) - The leading independent kubernetes platform!</h1>
            <img src="https://github.com/arbhoj/sample-k8s-app/raw/master/images/D2iq_large.jpeg" width:100px;height:200px>
        </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    configmap.reloader.stakater.com/reload: mainpage
  labels:
    app: ${APP_NAME}
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - image: library/nginx:1.17-alpine
        name: nginx
        volumeMounts:
        - mountPath: /usr/share/nginx/html/index.html
          name: mainpage
          subPath: index.html
      volumes:
      - name: mainpage
        configMap:
          name: mainpage
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: ${APP_NAME}
  name: ${SERVICE_NAME}
  namespace: ${NAMESPACE}
spec:
  ports:
  - port: ${SERVICE_PORT}
  selector:
    app: ${APP_NAME}
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: kommander-traefik
    traefik.ingress.kubernetes.io/router.middlewares: "${NAMESPACE}-${APP_NAME}-stripprefixes@kubernetescrd"
    traefik.ingress.kubernetes.io/router.tls: "true"
  name: ${APP_NAME}
  namespace: ${NAMESPACE}
spec:
  rules:
  - http:
      paths:
      - backend:
          service:
            name: ${APP_NAME}
            port:
              number: 80
        path: /${APP_NAME}
        pathType: ImplementationSpecific
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: ${APP_NAME}-stripprefixes
  namespace: ${NAMESPACE}
spec:
  stripPrefix:
    prefixes:
    - /${APP_NAME}
EOF
```


### Step 2: Send some traffic to the ingress endpoint
```
curl https://${CLUSTER_VIP}/${APP_NAME} -k
```

### Step 3: Create a PrometheusRule to generate custom-metric for ingress in this namespace.
> Note:
- This only has to be done once for a namespace
- The traefik metric “traefik_service_requests_total” that is being used as the base metric for this custom-metric, puts kommander as the value for the namespace label irrespective of where the workload is created. Hence, the overriding the namespace  label in the rules section of the PrometheusRule to set it to sclae  (i.e. the namespace where the workload resides. This is important as without it the metric will get associated with the kommander namespace instead of scale

```
kubectl apply -f - <<EOF
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: ingress-request-per-second-${NAMESPACE}
  namespace: ${NAMESPACE}
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: k8s-ingress.rules
    rules:
    - expr: |-
        sum by(service) (rate(traefik_service_requests_total{}[1m]))
      record: ingress_request_per_second
      labels:
        namespace: ${NAMESPACE}
EOF
```

### Step 4:  Wait for the rule to get activated and verify

This could take up to a minute. This can be verified from the prometheus ui by traversing to Status > Rules . Then check if the custom-metric is populated

```
k get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/${NAMESPACE}/services/${NAMESPACE}-${SERVICE_NAME}-${SERVICE_PORT}@kubernetes/ingress_request_per_second"
```

### Step 5: Create HPA based on the custom metric
```
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ingress-hpa-${SERVICE_NAME}
  namespace: ${NAMESPACE}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${APP_NAME}
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Object
    object:
      metric:
        name: ingress_request_per_second
      describedObject:
        apiVersion: v1
        kind: Service
        name: ${NAMESPACE}-${SERVICE_NAME}-${SERVICE_PORT}@kubernetes
      target:
        type: Value
        value: 0.1k
EOF
```

### Step 6: Very HPA is successfully created
> Note: Wait for a minute or so before checking and ensure there is no failure in the Events field. Also the Deployment pods:  field will be updated from zero to the actual count of current and desired pods (i.e. 1 for both in this example)

```
kubectl describe hpa -n ${NAMESPACE}  
```

### Step 7: Test HPA
Send heavy load on the ingress using a tool like “hey” and see if the HPA kicks in and scales the number of pods

```
e.g. hey -n 20000 https://${CLUSTER_VIP}/${APP_NAME}
```

That's it. All it took was a simple configuration to scale based on the amount of requests coming in for a particular service behing an ingress. The same concept can be used for different ingress controllers and other custom-metrics. 


## Cleanup:
```
kubectl -n ${NAMESPACE} delete hpa ingress-hpa-${SERVICE_NAME}
kubectl -n ${NAMESPACE} delete prometheusrule ingress-request-per-second-${NAMESPACE}
kubectl -n ${NAMESPACE} delete deploy ${APP_NAME}
kubectl -n ${NAMESPACE} delete service ${SERVICE_NAME}
kubectl -n ${NAMESPACE} delete configmap mainpage
kubectl -n ${NAMESPACE} delete middleware ${APP_NAME}-stripprefixes
```

