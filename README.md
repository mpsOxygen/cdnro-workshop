# Escape from Certificate Hell: Managing Trust in Your Kubernetes Cluster

## Setup kind cluster

[Install podman-desktop](https://podman-desktop.io/)

![Start KinD cluster](image.png)

## Deploy our App

`kubectl apply -f https://raw.githubusercontent.com/mpsOxygen/cdnro-workshop/refs/heads/main/manifests/deployment_and_service.yaml`

## Install cert-manager

`helm repo add jetstack https://charts.jetstack.io --force-update`

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.1 \
  --set crds.enabled=true
```

## Create self signed issuer

![test](manifests/certmanager_ClusterIssuer_selfsinged.yaml)

## Create self signed CA

![](manifests/certmanager_Certificate_selfsigned.yaml)

## Create CA issuer

![](manifests/certmanager_ClusterIssuer_ca.yaml)

## Create certificate for app

![](manifests/certmanager_Certificate_server.yaml)

## Install trust-manager

`helm repo add jetstack https://charts.jetstack.io --force-update`

```
helm upgrade trust-manager jetstack/trust-manager \
  --install \
  --namespace cert-manager \
  --wait
```

## Create trust bundle

![](manifests/trustmanager_Bundle.yaml)

## Label Namespace for bundle injection

`kubectl label ns default "cloudnativedays.ro/inject=enabled"`


## Install Kyverno

`helm repo add kyverno https://kyverno.github.io/kyverno/`

`helm repo update`

```
helm install \
  kyverno kyverno/kyverno \
  -n kyverno \
  --create-namespace
```

## Create Kyverno policy for TrustBundle mounting

```
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-certificates-volume
  annotations:
    policies.kyverno.io/title: Add Certificates as a Volume
    policies.kyverno.io/category: Sample
    policies.kyverno.io/subject: Pod,Volume
    pod-policies.kyverno.io/autogen-controllers: DaemonSet,Deployment,Job,StatefulSet
    policies.kyverno.io/description: >-
      In some cases you would need to trust custom CA certificates for all the containers of a Pod.
      It makes sense to be in a ConfigMap so that you can automount them by only setting an annotation.
      This policy adds a volume to all containers in a Pod containing the certificate if the annotation
      called `inject-certs` with value `enabled` is found.
spec:
  background: false
  rules:
    - name: add-ssl-certs
      match:
        any:
          - resources:
              kinds:
                - Pod
      preconditions:
        all:
          - key: '{{request.object.metadata.annotations."inject-certs" || ""}}'
            operator: Equals
            value: enabled
          - key: "{{request.operation || 'BACKGROUND'}}"
            operator: AnyIn
            value:
              - CREATE
              - UPDATE
      mutate:
        foreach:
          - list: request.object.spec.containers
            patchStrategicMerge:
              spec:
                containers:
                  - name: "{{ element.name }}"
                    volumeMounts:
                      - mountPath: /etc/ssl/certs
                        name: etc-ssl-certs
                        readOnly: true
                volumes:
                  - configMap:
                      items:
                        - key: root-certs.pem
                          path: ca-certificates.crt
                      name: example-bundle
                      optional: false
                    name: etc-ssl-certs
```

## Run pods to test

### Run pod without bundle

```
kubectl run client-no-bundle --rm -i --tty --image nicolaka/netshoot
```

#### Test curl with error

```
curl https://server

curl https://server.default

curl https://server.default.svc.cluster.local
```

### Run pod with bundle

```
kubectl run client-with-bundle --rm -i --tty --image nicolaka/netshoot --annotations="inject-certs=enabled"
```

#### Test curl NO error

```
curl https://server

curl https://server.default

curl https://server.default.svc.cluster.local
```