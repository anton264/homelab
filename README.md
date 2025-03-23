# Homelab


```sh


curl -sSLf https://get.k0s.sh | sudo sh
sudo k0s install controller --single -c /etc/k0s/k0s.yaml
sudo k0s start
until sudo cp /var/lib/k0s/pki/admin.conf .kube/config && chmod 600 .kube/config && k get nodes | grep ${HOSTNAME}; do echo "Waiting for k0s"; sleep 5; done
helm repo add cilium https://helm.cilium.io/
helm repo update
helm install cilium cilium/cilium --version 1.15.4 \
  --namespace kube-system
  
kubectl rollout status deployment -n kube-system coredns

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

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: godaddy-api-key
  namespace: externaldns
type: Opaque
stringData:
  token: ${GODADDYAPIKEY}:${GODADDYAPISECRET}
  godaddy-api-key: ${GODADDYAPIKEY}
  godaddy-api-secret: ${GODADDYAPISECRET}

EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: godaddy-api-key
  namespace: cert-manager
type: Opaque
stringData:
  token: ${GODADDYAPIKEY}:${GODADDYAPISECRET}
  godaddy-api-key: ${GODADDYAPIKEY}
  godaddy-api-secret: ${GODADDYAPISECRET}

EOF

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/main/argocd/argocd-cm.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/main/argocd/argocd-rbac-cm.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/main/argocd/argoingress.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/anton264/homelab/main/appOfApp/templates/all.yaml

until kubectl -n cert-manager get deployment cert-manager
do
  k -n argocd rollout restart deployment
  echo "Waiting 2 minutes for apps to start.."
  sleep 120
done
kubectl rollout status deployment -n metallb-system metallb-controller
kubectl rollout status daemonset -n metallb-system metallb-speaker
until kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/main/appOfApp/templates/metallbobjects.yaml
do
  echo "Waiting for metallb.."
  sleep 5
done

kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/main/clusterissuer.yaml
sleep 20
kubectl -n argocd wait --for=condition=ready --timeout=300s certificate/argocd-server-tls
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