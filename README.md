# Escape from Certificate Hell: Managing Trust in Your Kubernetes Cluster

## Setup kind cluster

[Install podman-desktop](https://podman-desktop.io/)

![Start KinD cluster](image.png)

## Deploy our App

`kubectl apply -f https://raw.githubusercontent.com/mpsOxygen/cdnro-workshop/refs/heads/main/deployment_and_service.yaml`

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

## Install trust-manager

`helm repo add jetstack https://charts.jetstack.io --force-update`

```
helm upgrade trust-manager jetstack/trust-manager \
  --install \
  --namespace cert-manager \
  --wait
```

## Install Kyverno

`helm repo add kyverno https://kyverno.github.io/kyverno/`

`helm repo update`

```
helm install \
  kyverno kyverno/kyverno \
  -n kyverno \
  --create-namespace
```


## Create self signed issuer

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

## Create self signed CA

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-ca
  namespace: cert-manager
spec:
  commonName: MyCA
  duration: 43800h
  isCA: true
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  secretName: selfsigned-ca
  ```

## Create CA issuer

```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: cluster-issuer-ca
  namespace: cert-manager
spec:
  ca:
    secretName: selfsigned-ca
```

## Create trust bundle

```
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: example-bundle
spec:
  sources:
  - useDefaultCAs: true
  - secret:
      name: "selfsigned-ca"
      key: "ca.crt"
  target:
    # Sync the bundle to a ConfigMap called `my-org.com` in every namespace which
    # has the label "linkerd.io/inject=enabled"
    # All ConfigMaps will include a PEM-formatted bundle, here named "root-certs.pem"
    # and in this case we also request binary formatted bundles in JKS and PKCS#12 formats,
    # here named "bundle.jks" and "bundle.p12".
    configMap:
      key: "root-certs.pem"
    additionalFormats:
      jks:
        key: "bundle.jks"
      pkcs12:
        key: "bundle.p12"
    namespaceSelector:
      matchLabels:
        cloudnativedays.ro/inject: "enabled"
```

## Label Namespace for bundle injection

`kubectl label ns default "cloudnativedays.ro/inject=enabled"`

## Create certificate for app

```
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: server-cert
  namespace: default
spec:
  # Secret names are always required.
  secretName: server-tls

  privateKey:
    algorithm: RSA
    encoding: PKCS1
    size: 2048

  duration: 2160h # 90d
  renewBefore: 360h # 15d

  isCA: false
  usages:
    - server auth
    - client auth

  subject:
    organizations:
      - Cloud Native Days RO

  # Avoid using commonName for DNS names in end-entity (leaf) certificates. Unless you have a specific
  # need for it in your environment, use dnsNames exclusively to avoid issues with commonName.
  # Usually, commonName is used to give human-readable names to CA certificates and can be avoided for
  # other certificates.
  commonName: server

  # At least one of commonName (possibly through literalSubject), dnsNames, uris, emailAddresses, ipAddresses or otherNames is required.
  dnsNames:
    - server
    - server.default
    - server.default.svc.cluster.local

  # Issuer references are always required.
  issuerRef:
    name: cluster-issuer-ca
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: ClusterIssuer
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
kubectl run test-no-bundle --rm -i --tty --image nicolaka/netshoot
```

#### Test curl with error

```
curl https://server

curl https://server.default

curl https://server.default.svc.cluster.local
```

### Run pod with bundle

```
kubectl run test-with-bundle --rm -i --tty --image nicolaka/netshoot --annotations="inject-certs=enabled"
```

#### Test curl NO error

```
curl https://server

curl https://server.default

curl https://server.default.svc.cluster.local
```