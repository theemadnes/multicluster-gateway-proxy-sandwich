# multicluster-gateway-proxy-sandwich
testing out GKE Gateway controller in front of nginx ingress controller in front of a web service

in this demo i have two GKE autopilot clusters (and corresponding kubecontexts) set up:
- autopilot-cluster-us-central1
- autopilot-cluster-us-east4 

GKE, even with the `HttpLoadBalancing` add-on enabled, allows for running additional Ingress controllers. See [here](https://cloud.google.com/kubernetes-engine/docs/how-to/custom-ingress-controller) for more details. We'll use the `ingressClassName` approach.

### steps 

```
# install nginx ingress on each cluster 
kubectl --context=autopilot-cluster-us-central1 apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
kubectl --context=autopilot-cluster-us-east4 apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

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

```