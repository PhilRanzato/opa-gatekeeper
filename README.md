# opa-gatekeeper

## Installation

Create a CA

```shell
mkdir opa
cd opa
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -days 100000 -out ca.crt -subj "/CN=admission_ca"
cat >server.conf <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, serverAuth
EOF
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj "/CN=opa.opa.svc" -config server.conf
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 100000 -extensions v3_req -extfile server.conf
kubectl create ns opa
kubectl create secret tls opa-server --cert=server.crt --key=server.key -n opa
```

Apply the admission controller

```shell
kubectl apply -f admission-controller.yaml -n opa
```

Label kube-system and opa namespace

```shell
kubectl label ns kube-system openpolicyagent.org/webhook=ignore
kubectl label ns opa openpolicyagent.org/webhook=ignore
```

Apply webhook configuration

```shell
kubectl apply -f webhook-configuration.yaml -n opa
```

Check logs 

```shell
kubectl logs -l app=opa -c opa
```

## Testing - Ingress whitelisting

```shell
cd testing/ingress-whitelisting
kubectl create configmap ingress-whitelist --from-file=ingress-whitelist.rego -n opa
kubectl apply -f qa-namespace.yaml
kubectl apply -f prod-namespace.yaml
kubectl apply -f ingress-ok.yaml -n production
kubectl apply -f ingress-bad.yaml -n qa
```
