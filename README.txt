# Create cluster
# https://kind.sigs.k8s.io/docs/user/quick-start/
kind create cluster --config djokica.yaml

# Install INGRESS - Nginx 
# https://kind.sigs.k8s.io/docs/user/ingress/
# kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

kubectl apply -f 1-Ingress/deployNginxIngres.yaml

# !!!!!!!!!!!!!!!!!!!!!!!
# LOAD quay.io/ IMAGES FIRST
# !!!!!!!!!!!!!!!!!!!!!!E
# Install MetalLB - load balancer
# https://kind.sigs.k8s.io/docs/user/loadbalancer/
kubectl apply -f 2-MetalLB/namespace.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)" 
# kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/metallb.yaml
kubectl apply -f 2-MetalLB/metallb.yaml
# Setup MetalLB address pool via config map
# docker network inspect -f '{{.IPAM.Config}}' kind
kubectl apply -f 2-MetalLB/metallb-configmap.yaml

# Install ArgoCD
# https://argo-cd.readthedocs.io/en/stable/getting_started/
kubectl create namespace argocd
# kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n argocd -f 3-ArgoCD/install.yaml
# Expose API server
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
# Do port forward 
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login using argocd client
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# Username: admin , password see above
argocd login <ARGOCD_SERVER>
argocd login 127.0.0.1:8080
argocd account update-password

# Create ingress /argocd - NOT WORKING??? Need to check https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/
# kubectl apply -f 3-ArgoCD/IngressArgoCD.yaml

# As workaround for Ingress, do Port-forward to the ArgoCD server - Works 
kubectl port-forward svc/argocd-server -n argocd 8888:443 --address='0.0.0.0' &

# !!!!!!!!!  TO MAKE STARTUP FASTER !!!!!!
kind load docker-image getraining-docker-snapshot-local.docker.fis.dev/nodetest:20211201_13_137  --name djokica

# !!!!!!!!!  CREATE SECRET !!!!!!!! 
kubectl create secret docker-registry regcred --docker-server='getraining-docker-snapshot-local.docker.fis.dev'  --docker-username='e1083457' --docker-password='*******' --docker-email='aleksandar.misovic@fisglobal.com'

# KIND Delete cluster
kind delete cluster --name djokica

