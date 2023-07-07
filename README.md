# multicluster-gateway-proxy-sandwich
testing out GKE Gateway controller in front of nginx ingress controller in front of a web service

in this demo i have two GKE autopilot clusters (and corresponding kubecontexts) set up:
- autopilot-cluster-us-central1
- autopilot-cluster-us-east4 

GKE, even with the `HttpLoadBalancing` add-on enabled, allows for running additional Ingress controllers. See [here](https://cloud.google.com/kubernetes-engine/docs/how-to/custom-ingress-controller) for more details. We'll use the `ingressClassName` approach.

We'll use [whereami](https://github.com/GoogleCloudPlatform/kubernetes-engine-samples/tree/main/whereami) as the sample application, which is a JSON web service that returns information about the pod that's responding to the request.

### steps 

```
# install nginx ingress on each cluster 
# NOTE: this creates a public-facing K8s service called `ingress-nginx-controller` in the `ingress-nginx` namespace, which means you now have a public-facing way to get into your services you may not want
# NOTE: you should change this service to `clusterIP` instead of `LoadBalancer` - we'll do this in a minute
kubectl --context=autopilot-cluster-us-central1 apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
kubectl --context=autopilot-cluster-us-east4 apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# hack alert!!! we're going to patch the service type of `ingress-nginx-controller` to `clusterIP` - you should probably do this using something like Kustomize 
kubectl --context=autopilot-cluster-us-central1 -n ingress-nginx patch svc ingress-nginx-controller -p '{"spec": {"type": "ClusterIP"}}'
kubectl --context=autopilot-cluster-us-east4 -n ingress-nginx patch svc ingress-nginx-controller -p '{"spec": {"type": "ClusterIP"}}'

# set up our target web service (in this case, `whereami`)
kubectl --context=autopilot-cluster-us-central1 create namespace whereami-nginx-demo
kubectl --context=autopilot-cluster-us-east4 create namespace whereami-nginx-demo

# create whereami-nginx-demo base
mkdir whereami-nginx-demo
mkdir whereami-nginx-demo/base
cat <<EOF > whereami-nginx-demo/base/kustomization.yaml 
resources:
  - github.com/GoogleCloudPlatform/kubernetes-engine-samples/whereami/k8s
EOF

mkdir whereami-nginx-demo/variant
cat <<EOF > whereami-nginx-demo/variant/cm-flag.yaml 
apiVersion: v1
kind: ConfigMap
metadata:
  name: whereami
data:
  BACKEND_ENABLED: "False" # assuming you don't want a chain of backend calls
  METADATA:        "whereami-nginx-demo"
EOF

cat <<EOF > whereami-nginx-demo/variant/service-type.yaml 
apiVersion: "v1"
kind: "Service"
metadata:
  name: "whereami"
spec:
  type: ClusterIP
EOF

cat <<EOF > whereami-nginx-demo/variant/kustomization.yaml 
nameSuffix: "-nginx-demo"
namespace: whereami-nginx-demo
commonLabels:
  app: whereami-nginx-demo
resources:
- ../base
patches:
- path: cm-flag.yaml
  target:
    kind: ConfigMap
- path: service-type.yaml
  target:
    kind: Service
EOF

kubectl --context=autopilot-cluster-us-central1 apply -k whereami-nginx-demo/variant
kubectl --context=autopilot-cluster-us-east4 apply -k whereami-nginx-demo/variant

# create ingress spec for exposing the `whereami-nginx-demo` service
cat <<EOF > whereami-nginx-demo/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  namespace: whereami-nginx-demo
  #annotations:
    #nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: whereami-nginx-demo
            port:
              number: 80
EOF

kubectl --context=autopilot-cluster-us-central1 apply -f whereami-nginx-demo/ingress.yaml
kubectl --context=autopilot-cluster-us-east4 apply -f whereami-nginx-demo/ingress.yaml

# create service exports and gateway for nginx, to be consumed by the GCLB
cat <<EOF > nginx_svc_export.yaml 
kind: ServiceExport
apiVersion: net.gke.io/v1
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
EOF

kubectl --context=autopilot-cluster-us-central1 apply -f nginx_svc_export.yaml
kubectl --context=autopilot-cluster-us-east4 apply -f nginx_svc_export.yaml

# create multi-cluster gateway
cat <<EOF > gateway.yaml
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1beta1
metadata:
  name: external-http-nginx
  namespace: ingress-nginx
spec:
  gatewayClassName: gke-l7-global-external-managed-mc
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      kinds:
      - kind: HTTPRoute
EOF

kubectl --context=autopilot-cluster-us-central1 apply -f gateway.yaml

cat << EOF > default-httproute.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: default-httproute
  namespace: ingress-nginx
spec:
  parentRefs:
  - name: external-http-nginx
    namespace: ingress-nginx
  rules:
  - backendRefs:
    - group: net.gke.io
      kind: ServiceImport
      name: ingress-nginx-controller
      port: 80
EOF

kubectl --context=autopilot-cluster-us-central1 apply -f default-httproute.yaml

# get GCLB IP address
export GCLB_IP=$(kubectl get gateway -n ingress-nginx external-http-nginx -o jsonpath='{.status.addresses[0].value}')

# call the service
curl $GCLB_IP -s | jq "."
```