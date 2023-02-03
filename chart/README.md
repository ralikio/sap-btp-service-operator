# sap-btp-operator Helm Chart

## Overview  

This is a custom version of the sap-btp-operator Helm chart.

The upstream version of the sap-btp-operator Helm chart has a dependency on the Jetstack cert-manager. This custom version makes [jetstack/cert-manager](https://github.com/jetstack/cert-manager) optional and adds the possibility to use a custom caBundle or [gardener/cert-management](https://github.com/gardener/cert-management).

## Prerequeisites

* Kubernetes 1.16+
* Helm 3+

## Install Chart

<details>
<summary>With fixed caBundle</summary>

helm install sap-btp-operator . \
    --atomic \
    --create-namespace \
    --namespace=sap-btp-operator \
    --set manager.secret.clientid="<fill in>" \
    --set manager.secret.clientsecret="<fill in>" \
    --set manager.secret.url="<fill in>" \
    --set manager.secret.tokenurl="<fill in>" \
    --set cluster.id="<fill in>"

 </details>

<details>
<summary>With custom caBundle</summary>

helm install sap-btp-operator . \
    --atomic \
    --create-namespace \
    --namespace=sap-btp-operator \
    --set manager.secret.clientid="<fill in>" \
    --set manager.secret.clientsecret="<fill in>" \
    --set manager.secret.url="<fill in>" \
    --set manager.secret.tokenurl="<fill in>" \
    --set manager.certificates.selfSigned.caBundle="${CABUNDLE}" \
    --set manager.certificates.selfSigned.crt="${SERVERCRT}" \
    --set manager.certificates.selfSigned.key="${SERVERKEY}" \
    --set cluster.id="<fill in>"

 </details>

<details>
<summary>With jetstack/cert-manager</summary>

helm install sap-btp-operator . \
    --atomic \
    --create-namespace \
    --namespace=sap-btp-operator \
    --set manager.secret.clientid="<fill in>" \
    --set manager.secret.clientsecret="<fill in>" \
    --set manager.secret.url="<fill in>" \
    --set manager.secret.tokenurl="<fill in>" \
    --set manager.certificates.certManager=true \
    --set cluster.id="<fill in>"

  </details>

<details>
<summary>With gardener/cert-management</summary>

helm template sap-btp-operator . \
    --atomic \
    --create-namespace \
    --namespace=sap-btp-operator \
    --set manager.secret.clientid="<fill in>" \
    --set manager.secret.clientsecret="<fill in>" \
    --set manager.secret.url="<fill in>" \
    --set manager.secret.tokenurl="<fill in>" \
    --set manager.certificates.certManagement.caBundle="${CABUNDLE}" \
    --set manager.certificates.certManagement.crt=${CACRT} \
    --set manager.certificates.certManagement.key=${CAKEY} \
    --set cluster.id="<fill in>"

  </details>

## Change the original chart to sap-btp-operator Helm chart

1. Move Secrets into `webhook.yml` and define certificates:
```yaml
{{- $cn := printf "sap-btp-operator-webhook-service"  }}
{{- $ca := genCA (printf "%s-%s" $cn "ca") 3650 }}
{{- $altName1 := printf "%s.%s" $cn .Release.Namespace }}
{{- $altName2 := printf "%s.%s.svc" $cn .Release.Namespace }}
{{- $cert := genSignedCert $cn nil (list $altName1 $altName2) 3650 $ca }}
{{- if not .Values.manager.certificates }}
apiVersion: v1
kind: Secret
metadata:
  name: webhook-server-cert
  namespace: {{.Release.Namespace}}
type: kubernetes.io/tls
data:
  tls.crt: {{ b64enc $cert.Cert }}
  tls.key: {{ b64enc $cert.Key }}
---
apiVersion: v1
kind: Secret
metadata:
  name: sap-btp-service-operator-tls
  namespace: {{ .Release.Namespace }}
type: kubernetes.io/tls
data:
  tls.crt: {{ b64enc $cert.Cert }}
  tls.key: {{ b64enc $cert.Key }}
---
{{- end}}
{{- if .Values.manager.certificates.selfSigned }}
apiVersion: v1
kind: Secret
metadata:
  name: webhook-server-cert
  namespace: {{.Release.Namespace}}
type: kubernetes.io/tls
data:
  tls.crt: "{{ .Values.manager.certificates.selfSigned.crt }}"
  tls.key: "{{ .Values.manager.certificates.selfSigned.key }}"
---
{{- end}}
```
2. Add the `caBundle` definition in both webhooks:
```
{{- if not .Values.manager.certificates }}
caBundle: {{ b64enc $ca.Cert }}
{{- end }}
```

### Overrides
While rendering Kubernetes resource files by Helm, the following values overrides are applied.

```yaml
manager:
  annotations:
    sidecar.istio.io/inject: "false"
  req_cpu_limit: 10m
  replica_count: 1
  enable_leader_election: false
  certificates:
    certManager: false
  kubernetesMatchLabels:
    enabled: true
```

## Publish new version of the chart
1. Download the original chart from Helm repository.  
   i. Configure the Helm repository
   ```
    helm repo add sap-btp-operator https://sap.github.io/sap-btp-service-operator
   ```
   ii. Pull the chart
    ```
    helm pull sap-btp-operator/sap-btp-operator
    ```
   iii. Specify the version if needed
    ```
    helm pull sap-btp-operator/sap-btp-operator --version v0.2.0
    ```

   iv. Unpack the downloaded tar and apply necessary changes.

2. Create a package.
    ```
    helm package chart 
    ```
3.  Release the chart on GitHub.  
Create a GitHub release and upload the generated Helm chart (tgz).