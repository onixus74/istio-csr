<p align="center"><img src="https://github.com/jetstack/cert-manager/blob/master/logo/logo.png" width="250x" /></p>
</a>
<a href="https://godoc.org/github.com/cert-manager/istio-csr"><img src="https://godoc.org/github.com/cert-manager/istio-csr?status.svg"></a>
<a href="https://goreportcard.com/report/github.com/cert-manager/istio-csr"><img alt="Go Report Card" src="https://goreportcard.com/badge/github.com/cert-manager/istio-csr" /></a></p>

# istio-csr

istio-csr is an agent that allows for [istio](https://istio.io) workload and
control plane components to be secured using
[cert-manager](https://cert-manager.io). Certificates facilitating mTLS, inter
and intra cluster, will be signed, delivered and renewed using [cert-manager
issuers](https://cert-manager.io/docs/concepts/issuer).

⚠️ Currently supports istio versions v1.7+

⚠️ Currently supports cert-manager versions v1.3+

---

## Installation

### Installing cert-manager

Firstly, [cert-manager must be
installed](https://cert-manager.io/docs/installation/) in your cluster.

### Issuer or ClusterIssuer
An Issuer or ClusterIssuer must be configured which will be used to sign istio
certificates against.

Example Issuer configuration can be found in
[`./hack/demo/cert-manager-bootstrap-resources.yaml`](./hack/demo/cert-manager-bootstrap-resources.yaml).
It is advised to not use the CA Issuer type in production environments.

> It is important to use an issuer type that is able to sign istio mTLS workload
> certificates (SPIFFE URI SANs) and
> [istiod serving certificates](./deploy/charts/istio-csr/templates/certificate.yaml).
> ACME issuers will not work.

### Installing istio-csr

Next, install istio-csr into the cluster, configured to use the Issuer or
ClusterIssuer installed earlier. If using an Issuer rather than the
ClusterIssuer type, the Issuer must reside in the same namespace as that
configured by `--certificate-namespace`, `istio-system` by default.

> ⚠️ It is highly recommended that the root CA certificates are statically
> defined in istio-csr. If they are not, istio-csr will "discover" the root CA
> certificates when requesting its serving certificate, effectively making
> istio-csr TOFU.

#### Discover root CAs installation

```terminal
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
$ kubectl create namespace istio-system
$ helm install -n cert-manager cert-manager-istio-csr jetstack/cert-manager-istio-csr
```

#### Load root CAs from file ca.pem (Preferred)

```terminal
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update
$ kubectl create namespace istio-system
$ kubectl create secret generic istio-root-ca --from-file=ca.pem=ca.pem -n cert-manager
$ helm install -n cert-manager cert-manager-istio-csr jetstack/cert-manager-istio-csr \
  --set "app.tls.rootCAFile=/var/run/secrets/istio-csr/ca.pem" \
  --set "volumeMounts[0].name=root-ca" \
  --set "volumeMounts[0].mountPath=/var/run/secrets/istio-csr" \
  --set "volumes[0].name=root-ca" \
  --set "volumes[0].secret.secretName=istio-root-ca"
```

All helm value options can be found [here](./deploy/charts/istio-csr/README.md).

### Installing istio

If you are running Openshift, prepare the cluster for Istio.
Follow instructions from Istio [platform setup
guide](https://istio.io/latest/docs/setup/platform-setup/openshift/).

Finally, install istio. Istio must be installed using the IstioOperator
configuration changes within
[`./hack/istio-config-x.yaml`](./hack/istio-config-1.10.0.yaml). Later versions
of istio share the same config.

For OpenShift set the profile as `--set profile=openshift`.

These config options are required in order for the CA Server to be disabled in
istiod, ensure istio workloads request certificates from istio-csr, and the
istiod certificates and keys are mounted from the Certificate created when
installing istio-csr.


## How

The cert-manager istio agent implements the gRPC istio certificate service which
authenticates, authorizes, and signs incoming certificate signing requests from
istio workloads. This matches the behaviour of istiod in a typical installation,
however enables these certificates to be signed through cert-manager.
