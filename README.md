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

kubectl -n externaldns create job --from=cronjob/public-ip-cronjob firstjob
kubectl -n externaldns wait --for=condition=complete job/firstjob
kubectl -n externaldns delete job firstjob
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