# Homelab

https://medium.com/@ipuustin/using-metallb-as-kubernetes-load-balancer-with-ubiquiti-edgerouter-7ff680e9dca3
```sh
configure
set protocols bgp 64512 parameters router-id 192.168.2.1
set protocols bgp 64512 neighbor 12.12.12.150 remote-as 64512
set protocols bgp 64512 maximum-paths ibgp 32
commit
save
exit
```

## K3s


```sh
sudo mkdir -p /etc/rancher/k3s/config.yaml.d
echo -e "disable:\n  - traefik" | sudo tee /etc/rancher/k3s/config.yaml.d/traefik.yaml > /dev/null
export API_SERVER_IP=10.1.1.10
export API_SERVER_PORT=6443
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--flannel-backend=none --disable-network-policy --disable-kube-proxy' sh -
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/k3s.yaml
sudo chown $(whoami) $HOME/k3s.yaml
cp $HOME/k3s.yaml $HOME/.kube/config
chmod 600 $HOME/k3s.yaml
export KUBECONFIG=$HOME/k3s.yaml
kubectl config set-context default --namespace=kube-system
kubectl get nodes
kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') bgp-policy=a
helm repo add cilium https://helm.cilium.io
helm repo update
helm upgrade --install cilium cilium/cilium --version 1.18.5 --namespace kube-system --values https://raw.githubusercontent.com/240snusn/homelab/refs/heads/main/appOfApp/cilium-values.yaml
kubectl create ns externaldns
kubectl create ns cert-manager
kubectl create ns argocd
kubectl apply -f https://raw.githubusercontent.com/240snusn/homelab/main/publicip/publicip.yaml

until kubectl get configmap public-ip-config -n externaldns -o jsonpath='{.data.public-ip}' | grep -P '^(?:\d{1,3}\.){3}\d{1,3}$'
do
  kubectl -n externaldns create job --from=cronjob/public-ip-cronjob firstjob
  kubectl -n externaldns wait --for=condition=complete job/firstjob
  kubectl -n externaldns delete job firstjob
done
kubectl -n externaldns get configmap public-ip-config -o jsonpath={.data.public-ip}

kubectl create ns renovate
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: github-client-secret
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
type: Opaque
stringData:
  dex.github.clientSecret: ${GITHUBCLIENTSECRET}
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
stringData:
  apiKey: ${cloudflareAPItoken} 
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: externaldns
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager
type: Opaque
stringData:
  api-token: ${cloudflareAPItoken}
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: renovate-secrets
  namespace: renovate
type: Opaque
stringData:
  RENOVATE_TOKEN: ${GITHUBCOMTOKEN}
EOF


kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/240snusn/homelab/refs/heads/main/appOfapp.yaml

sleep 180
kubectl -n argocd rollout restart deployment
kubectl -n argocd rollout restart statefulset
```

## Tear down
Uninstall k3s
```sh
/usr/local/bin/k3s-uninstall.sh
```
