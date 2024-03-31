# Homelab


```sh


curl -sSLf https://get.k0s.sh | sudo sh
sudo k0s install controller --single
sudo k0s start
sleep 120
sudo cp /var/lib/k0s/pki/admin.conf .kube/config
chmod 600 .kube/config
kubectl create ns externaldns
kubectl create ns cert-manager
kubectl create ns argocd
kubectl apply -f https://raw.githubusercontent.com/anton264/homelab/main/publicip.yaml


cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: github-client-secret
  namespace: argocd
type: Opaque
stringData:
  dex.github.clientSecret: 0f458bed99dc52ef76e856a6cfc4419e1861b49a

EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: godaddy-api-key
  namespace: externaldns
type: Opaque
stringData:
  token: ${GODADDY-API-KEY}:${GODADDY-API-SECRET}
  godaddy-api-key: ${GODADDY-API-KEY}
  godaddy-api-secret: ${GODADDY-API-SECRET}

EOF

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: godaddy-api-key
  namespace: cert-manager
type: Opaque
stringData:
  token: ${GODADDY-API-KEY}:${GODADDY-API-SECRET}
  godaddy-api-key: ${GODADDY-API-KEY}
  godaddy-api-secret: ${GODADDY-API-SECRET}

EOF

kubectl apply -k https://github.com/anton264/homelab/argocd
sleep 120

argocd app create apps \
    --dest-namespace argocd \
    --dest-server https://kubernetes.default.svc \
    --repo https://github.com/anton264/homelab.git \
    --path appOfApp  
argocd app sync apps 
```


https://medium.com/@ipuustin/using-metallb-as-kubernetes-load-balancer-with-ubiquiti-edgerouter-7ff680e9dca3

configure
set protocols bgp 64512 parameters router-id 192.168.2.1
set protocols bgp 64512 neighbor 192.168.2.200 remote-as 64512
set protocols bgp 64512 maximum-paths ibgp 32
commit
save
exit