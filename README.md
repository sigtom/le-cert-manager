# Deploying Let's Encrypt certs with cert-manager and automating with Openshift GitOps

## Introduction

Hello, fellow OpenShift admins! This post will guide you through a critical aspect of OpenShift administration: setting up TLS certificates for your cluster using the cert-manager operator, Let's Encrypt, and Cloudflare's DNS challenge. Our approach will start with Let's Encrypt's staging servers, then progress to their production servers. Once we confirm that everything is working as expected, we'll automate the entire process using OpenShift GitOps (ArgoCD).

## Prerequisites

Before diving into the setup, make sure you have the following:

- An OpenShift 4.10+ cluster up and running.
- A registered domain name with DNS managed by Cloudflare.
- Cloudflare API Key for your domain.
- Installed OpenShift GitOps (ArgoCD).

## Step 1: Installing Cert-Manager

The first step is to install the cert-manager operator.  We are using the community version of the operator, as it supports Cloudflare and using the DNS challenge. We need to create a namespace `cert-mgr-ns.yaml` and a subscription `subscription.yaml`:


`cert-mgr-ns.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
```

Reviewing the file above, we are creating a Namespace, and the name will be `cert-manager`.

`subscription.yaml`

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/cert-manager.openshift-operators: ""
  name: cert-manager
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: cert-manager
  source: community-operators
  sourceNamespace: openshift-marketplace
```

The above file tells the cluster to create a subscription for the `cert-manager` operator, and to install it in the `openshift-operators` namespace.  Then we define the channel, `stable`, that we want to install from. The install plan approval is `Automatic`, and will come from the `community-operators` catalog. 

We then need to apply the 2 files to the cluster.

```bash
$ oc apply -f cert-mgr-ns.yaml
$ oc apply -f subscription.yaml
```

## Step 2: Setting up cert-manager for Let's Encrypt Staging

To pull a certificate from Let's Encrypt, we have to provide some information. We need to have a Cloudflare API Key with write permissions for the domain you want to issue a certificate. We also need the email address used in Cloudflare, and Let's Encypt. We will use that information to craft the deployment files. 

Create a new file `cloudflare-secret.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: openshift-operators
type: Opaque
data:
  api-key: 
 'YOUR_BASE64_ENCODED_CLOUDFLARE_API_KEY'
```

Replace `YOUR_BASE64_ENCODED_CLOUDFLARE_API_KEY` with your base64 encoded Cloudflare API Key. You can get this value by running:

```bash
$ echo -n 'your-api-key' | base64
```


Since Lets Encrypt will rate limit your calls to the production endpoint, when setting this up and testing, we will use the Staging endpoint.  Once we know its working correctly, we can move to the Production endpoint. Create a new file, clusterissuer-staging.yaml , with the following content:

`clusterissuer-staging.yaml`
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # Use the Let's Encrypt staging environment for testing
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: 'YOUR_EMAIL_ADDRESS'
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - dns01:
        cloudflare:
          email: 'YOUR_CLOUDFLARE_EMAIL_ADDRESS'
          apiKeySecretRef:
            name: cloudflare-api-key
            key: api-key

```


Replace `[YOUR_EMAIL]` and `[YOUR_CLOUDFLARE_EMAIL]` with your respective email addresses. This ClusterIssuer will connect to the staging endpoint at the server listed. It will use your email address to issue the cert from Let's Encrypt. The DNS challenge will use the Cloudflare API key and your Cloudflare Email address to add and remove the needed DNS challenge records to confirm ownership of the domain. 

Apply both yaml files to the cluster:

```bash
$ oc apply -f cloudflare-secret.yaml
$ oc apply -f clusterissuer-staging.yaml  
```

## Step 3: Can we get a cert?

So far, we've setup the cluster to retrieve a certificate from the staging endpoint of Let's Encrypt.  Now, lets see if we can actually get a cert.  We will need to create a yaml file name `cert-staging.yaml` with the following info:

`cert-staging.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: acme-certificate-staging
  namespace: cert-manager
spec:
  secretName: acme-certificate-secret-staging
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
    - 'YOUR_DOMAIN'
  usages:
    - digital signature
    - key encipherment
    - server auth
```

Replace `YOUR_DOMAIN` with the domain you want to issue a certificate for. The kind of resource we are deploying is a certificate.  Its in the cert-manager namespace, and it will be called acme-certificate-staging. The actual certificate will be stored as a secret, with the name `acme-certificate-secret-staging`, issued from the cluster issuer we setup earlier, letsencrypt-staging. The cert will be issued for the listed domains. As always, apply the file:

```bash
$ oc apply -f cert-staging.yaml
```

Wait a few minutes, and when you check the secrets in the namespace cert-manager, you should see the staging cert. I've provided some more commands you can use to review the secret, and confirm the certificate and key were properly created for your domain, from the Let's Encrypt staging server. 

```bash
$ oc get secrets -n cert-manager
NAME                               TYPE                                  DATA   AGE
acme-certificate-secret-staging    kubernetes.io/tls                     2      67d

$ oc edit secret acme-certificate-secret-staging -n cert-manager

kind: Secret
apiVersion: v1
metadata:
  name: acme-certificate-secret-staging
  namespace: cert-manager
  labels:
    controller.cert-manager.io/fao: 'true'
  annotations:
    cert-manager.io/alt-names: >-
      'YOUR_DOMAIN'
    cert-manager.io/certificate-name: acme-certificate-staging
    cert-manager.io/common-name: 'YOUR_DOMAIN'
    cert-manager.io/ip-sans: ''
    cert-manager.io/issuer-group: ''
    cert-manager.io/issuer-kind: ClusterIssuer
    cert-manager.io/issuer-name: letsencrypt-staging
    cert-manager.io/uri-sans: ''
data:
  tls.crt: >- 'BASE64 ENCODED CERT'
  tls.key: >- 'BASE64 ENCODED KEY'
type: kubernetes.io/tls

$ oc get secret -n cert-manager acme-certificate-secret-staging -o json | jq -r '.data."tls.crt"' | base64 -d | openssl x509 -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C = US, O = (STAGING) Lets Encrypt, CN = (STAGING) Artificial Apricot R3
        Validity
            Not Before: Jun 20 16:39:32 2023 GMT
            Not After : Sep 18 16:39:31 2023 GMT
        Subject: CN = 'YOUR_DOMAIN'
```

## Step 4: Automating with GitOps and using Prod endpoint. 

Now that we have our process working, and we've confirmed that certificates and keys are being provisioned, we can setup Openshift GitOps with an application to deploy from our git repo.  The git repo will contain all the files we need to get valid Let's Encrypt certificates from the prod endpoint. 

We will need to add 2 more files, a `clusterissuer-prod.yaml` poiting to the prod endpoint, and a `certificate-prod.yaml`.  Let's get those setup:

`clusterissuer-prod.yaml`


```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: 'YOUR_EMAIL_ADDRESS'
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        cloudflare:
          email: `YOUR_CLOUDFLARE_EMAIL_ADDRESS`
          apiKeySecretRef:
            name: cloudflare-api-key
            key: api-key

```

Please note, you can always add more domains under the dnsNames stanza, if you want a certificate for multiple domains. I currenty pull one that serves 8 domains on one certificate. 

`certificate-prod.yaml`

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: acme-certificate-prod
  namespace: openshift-ingress
spec:
  secretName: acme-certificate-secret-prod
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - 'YOUR_DOMAIN'
  usages:
    - digital signature
    - key encipherment
    - server auth
```

Push these 2 files to your git repo, under the folder cert-manager. We also need to setup an application in Openshift GitOps that will sync with our git repo, and allow us to deploy these certificates automatically. You could login to the web UI and setup the application, but in keeping with our current practice, we will use a yaml file and name this `cert-mgr-app.yaml`. 

`cert-mgr-app.yaml`


```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: 'https://github.com/sigtom/cert-manager'
    path: cert-manager
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: cert-manager
  syncPolicy:
    automated: {}
```

Apply this file to your cluster:

```bash
oc apply -f cert-mgr-app.yaml
```

After a few minutes you can review the secrets in the cert-manager namespace, and make sure you see the new resource, acme-certificate-secret-prod. 

```bash
$ oc get secrets -n cert-manager
NAME                               TYPE                                  DATA   AGE
acme-certificate-secret-prod       kubernetes.io/tls                     2      67d
```

Use the previously provided commands to review the prod certificate, and confirm its been issued from the prod endpoint, and it include the domain(s) you requested. 

