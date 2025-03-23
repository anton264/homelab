# Homelab


```sh


curl -sSLf https://get.k0s.sh | sudo sh
sudo k0s install controller --single
sudo k0s start
until sudo cp /var/lib/k0s/pki/admin.conf .kube/config && chmod 600 .kube/config && k get nodes | grep ${HOSTNAME}; do echo "Waiting for k0s"; sleep 5; done

kubectl create ns externaldns
kubectl create ns cert-manager
kubectl create ns argocd
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/main/publicip.yaml


until kubectl get configmap public-ip-config -n externaldns -o jsonpath='{.data.public-ip}' | grep -P '^(?:\d{1,3}\.){3}\d{1,3}$'
do
  kubectl -n externaldns create job --from=cronjob/public-ip-cronjob firstjob
  kubectl -n externaldns wait --for=condition=complete job/firstjob
  kubectl -n externaldns delete job firstjob
done
k -n externaldns get configmap public-ip-config -o jsonpath={.data.public-ip}

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
kubectl -n externaldns create secret generic cloudflare-api-key --from-literal=apiKey=JtSjaqubq-Fu7cN7hMsChR9Mpr0p6wZZpPi6c8hy --from-literal=email=4mgty4bv5h@privaterelay.appleid.com
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: externaldns
type: Opaque
stringData:
  token: ${cloudflareAPIKEY}:${cloudflareAPISECRET}
  cloudflare-api-key: ${cloudflareAPIKEY}
  cloudflare-api-secret: ${cloudflareAPISECRET}

EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: cert-manager
type: Opaque
stringData:
  token: ${cloudflareAPIKEY}:${cloudflareAPISECRET}
  cloudflare-api-key: ${cloudflareAPIKEY}
  cloudflare-api-secret: ${cloudflareAPISECRET}

EOF

kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/main/argocd/argocd-cm.yaml
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/main/argocd/argocd-rbac-cm.yaml
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/main/argocd/argocdingress.yaml
kubectl apply -f https://github.com/anton264/homelab/appOfApp/templates/all.yaml
until kubectl apply -f https://github.com/anton264/homelab/appOfApp/templates/metallbobjects.yaml
do
  echo "Waiting for metallb.."
  sleep 5
done


sleep 180
k -n argocd rollout restart deployment
k -n argocd rollout restart statefulset

```


https://medium.com/@ipuustin/using-metallb-as-kubernetes-load-balancer-with-ubiquiti-edgerouter-7ff680e9dca3

configure
set protocols bgp 64512 parameters router-id 192.168.2.1
set protocols bgp 64512 neighbor 192.168.2.200 remote-as 64512
set protocols bgp 64512 maximum-paths ibgp 32
commit
save
exit

### Teardown

```sh
sudo k0s stop ; sudo k0s reset; sudo k0s reset; sudo reboot
```
## K3s

wip, cloudflare restricted their api. Needs other provider
```sh
curl -sfL https://get.k3s.io | sh -
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/k3s.yaml
sudo chown $(whoami) $HOME/k3s.yaml
export KUBECONFIG=$HOME/k3s.yaml
kubectl config set-context default --namespace=kube-system
kubectl get nodes


kubectl create ns externaldns
kubectl create ns cert-manager
kubectl create ns argocd
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/main/publicip.yaml

until kubectl get configmap public-ip-config -n externaldns -o jsonpath='{.data.public-ip}' | grep -P '^(?:\d{1,3}\.){3}\d{1,3}$'
do
  kubectl -n externaldns create job --from=cronjob/public-ip-cronjob firstjob
  kubectl -n externaldns wait --for=condition=complete job/firstjob
  kubectl -n externaldns delete job firstjob
done
kubectl -n externaldns get configmap public-ip-config -o jsonpath={.data.public-ip}

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
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: externaldns
type: Opaque
stringData:
  token: ${cloudflareAPIKEY}:${cloudflareAPISECRET}
  cloudflare-api-key: ${cloudflareAPIKEY}
  cloudflare-api-secret: ${cloudflareAPISECRET}
EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: cert-manager
type: Opaque
stringData:
  token: ${cloudflareAPIKEY}:${cloudflareAPISECRET}
  cloudflare-api-key: ${cloudflareAPIKEY}
  cloudflare-api-secret: ${cloudflareAPISECRET}
EOF

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/main/argocd/argocd-cm.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/main/argocd/argocd-rbac-cm.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/main/argocd/argoingress.yaml
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/main/appOfApp/templates/all.yaml
until kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/refs/heads/main/appOfApp/templates/metallbobjects.yaml
do
  echo "Waiting for metallb.."
  sleep 5
done

sleep 180
kubectl -n argocd rollout restart deployment
kubectl -n argocd rollout restart statefulset
```
