# Homelab

https://medium.com/@ipuustin/using-metallb-as-kubernetes-load-balancer-with-ubiquiti-edgerouter-7ff680e9dca3
```sh
configure
set protocols bgp 64512 parameters router-id 192.168.2.1
set protocols bgp 64512 neighbor 192.168.2.200 remote-as 64512
set protocols bgp 64512 maximum-paths ibgp 32
commit
save
exit
```
## K3s

wip, cloudflare restricted their api. Needs other provider
```sh
mkdir -p /etc/rancher/k3s/config.yaml.d
echo "disable:
  - traefik" > /etc/rancher/k3s/config.yaml.d/traefik.yaml
curl -sfL https://get.k3s.io | sh -
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/k3s.yaml
sudo chown $(whoami) $HOME/k3s.yaml
chmod 600 $HOME/k3s.yaml
export KUBECONFIG=$HOME/k3s.yaml
kubectl config set-context default --namespace=kube-system
kubectl get nodes

kubectl create ns externaldns
kubectl create ns cert-manager
kubectl create ns argocd
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/k3s/publicip.yaml

until kubectl get configmap public-ip-config -n externaldns -o jsonpath='{.data.public-ip}' | grep -P '^(?:\d{1,3}\.){3}\d{1,3}$'
do
  kubectl -n externaldns create job --from=cronjob/public-ip-cronjob firstjob
  kubectl -n externaldns wait --for=condition=complete job/firstjob
  kubectl -n externaldns delete job firstjob
done
kubectl -n externaldns get configmap public-ip-config -o jsonpath={.data.public-ip}

kubectl -n argocd create secret generic github-client-secret --from-literal=dex.github.clientSecret=${GITHUBCLIENTSECRET}
kubectl -n externaldns create secret generic cloudflare-api-key --from-literal=apiKey=${cloudflareAPItoken} 
kubectl -n cert-manager create secret generic cloudflare-api-token-secret --from-literal=api-token=${cloudflareAPItoken}


kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/k3s/argocd/argocd-cm.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/k3s/argocd/argocd-rbac-cm.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/k3s/argocd/argoingress.yaml
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/k3s/appOfApp/templates/all.yaml
until kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/k3s/appOfApp/templates/metallbobjects.yaml
do
  echo "Waiting for metallb.."
  sleep 5
done

sleep 180
kubectl -n argocd rollout restart deployment
kubectl -n argocd rollout restart statefulset
```

## Tear down
Uninstall k3s
```sh
/usr/local/bin/k3s-uninstall.sh
```
