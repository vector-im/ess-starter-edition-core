<meta name="copyright" value="SPDX-FileCopyrightText:Copyright 2023 New Vector Ltd">
<meta name="license" value="SPDX-License-Identifier: AGPL-3.0-or-later">

### ⚠️ Deprecated ### ⚠️

**Element Starter Edition and Starter Edition Core have been discontinued and deprecated with the [24.10 release](https://ems-docs.element.io/books/element-server-suite-documentation-lts-2410/page/ess-lts-2410-change-logs-and-upgrade-notes). There will be no new releases of this distribution and it is recommended to use a different deployment method.**

For those wanting an example Matrix 2.0 stack for experimentation purposes, take a look at https://github.com/element-hq/element-docker-demo.

---

# Element Starter Core
Copyright 2023 New Vector Ltd

## An Operator for managing Matrix stacks

This operator contains CRDs and a controller for managing numerous components from the Matrix stack.
The list of the managed components can be found in the `watches.yml` file.

The operator supports running in any kubernetes cluster, including Openshift.

### Architecture

The operator is managing the components CRD to deploy the kubernetes workloads, ingresses, etc.

Each component CRD can be configured independantly from the others. Some components CRDs will be waiting for inputs generated by the deployment of other components.

To make it easier to write coherent and integrated components CRDs, it is possible to deploy the updater. The updater watches `ElementDeployment` CRD, and generates element resources CRDs to be ingested by the operator.

### Installing the Operator

#### Introduction

This document will walk you through how to get started with our Element Starter Edition Core. We require that you have a kubernetes environment to deploy into. If you do not have a kuberentes environment in which to deploy this, we have had experience deploying into a single node microk8s environment.

#### Requirements

* Kubernetes
  * `cert-manager` will need to be installed and provide an appropriately configured `ClusterIssuer` named `letsencrypt`
  * Ingress Controller with an `IngressClass` (in this case, we are using an ingress controller with a class name of `public`)
* PostgreSQL database with a UTF-8 encoding and a C Locale.


#### Installing the Helm Chart Repositories

The first step is to start on a machine with helm v3 installed and configured with your kubernetes cluster and pull down the two charts that you will need.


```bash
helm repo add ess-starter-edition-core https://element-hq.github.io/ess-starter-edition-core
```

To install the helm charts and actually deploy the `element-updater` and the `element-operator` with their default configurations, simply run:

```bash
helm install element-updater ess-starter-edition-core/element-updater --namespace element-updater --create-namespace
helm install element-operator ess-starter-edition-core/element-operator --namespace element-operator --create-namespace
```

*N.B. This guide assumes that you are using the `element-updater` and  `element-operator`  namespaces. You can call it whatever you want and if it doesn't exist yet, you can create it with: `kubectl create ns <name>`.*

#### Generating a TLS secret for the webhook

The conversion webhooks need their own self-signed CA and TLS certificate to be integrated into kubernetes.

For example using `easy-rsa` : 
```
easyrsa init-pki
easyrsa --batch "--req-cn=ESS-CA`date +%s`" build-ca nopass
easyrsa --san="DNS:element-operator-conversion-webhook" \
        --san="DNS:element-operator-conversion-webhook.element-operator" \
        --san="DNS:element-operator-conversion-webhook.element-operator.svc" \
        --san="DNS:element-operator-conversion-webhook.element-operator.svc.cluster" \
        --san="DNS:element-operator-conversion-webhook.element-operator.svc.cluster.local" \
        --days=10000 \
  build-server-full element-operator-conversion-webhook nopass
easyrsa --san="DNS:element-updater-conversion-webhook" \
        --san="DNS:element-updater-conversion-webhook.element-updater" \
        --san="DNS:element-updater-conversion-webhook.element-updater.svc" \
        --san="DNS:element-updater-conversion-webhook.element-updater.svc.cluster" \
        --san="DNS:element-updater-conversion-webhook.element-updater.svc.cluster.local" \
        --days=10000 \
  build-server-full element-updater-conversion-webhook nopass
```

Create a secret for each of these two certificates : 

```
kubectl create secret tls element-operator-conversion-webhook --cert=pki/issued/element-operator-conversion-webhook.crt --key=pki/private/element-operator-conversion-webhook.key  --namespace element-operator
kubectl create secret tls element-updater-conversion-webhook --cert=pki/issued/element-updater-conversion-webhook.crt --key=pki/private/element-updater-conversion-webhook.key  --namespace element-updater
```

#### Installing the helm chart for the `element-updater` and the `element-operator`

Create the following values file to deploy the controller managers in their namespace :

`values.element-operator.yml` : 
```
clusterDeployment: true
deployCrds: true  # Deploys the CRDs and the Conversion Webhooks
deployCrdRoles: true  # Deploys roles to give permissions to users to manage specific ESS CRs
deployManager: true  # Deploys the controller managers
crds:
  conversionWebhook:
    caBundle: # Paste here the content of `base64 pki/ca.crt -w 0`
    tlsSecretName: element-operator-conversion-webhook
```

`values.element-updater.yml` : 
```
clusterDeployment: true
deployCrds: true  # Deploys the CRDs and the Conversion Webhooks
deployCrdRoles: true  # Deploys roles to give permissions to users to manage specific ESS CRs
deployManager: true  # Deploys the controller managers
crds:
  conversionWebhook:
    caBundle: # Paste here the content of `base64 pki/ca.crt -w 0`
    tlsSecretName: element-updater-conversion-webhook
```

Run the helm install command : 

```
helm install element-operator element-operator/element-operator --namespace element-operator -f values.yaml 
helm install element-updater element-updater/element-updater --namespace element-updater -f values.yaml
```

Now at this point, you should have the following 4 containers up and running:

```
[user@helm ~]$ kubectl get pods -n element-operator
NAMESPACE            NAME                                                   READY   STATUS    RESTARTS        AGE
element-operator     element-operator-controller-manager-c8fc5c47-nzt2t     2/2     Running   0               6m5s
element-operator     element-operator-conversion-webhook-7477d98c9b-xc89s   1/1     Running   0               6m5s
[user@helm ~]$ kubectl get pods -n element-updater
NAMESPACE            NAME                                                   READY   STATUS    RESTARTS        AGE
element-updater      element-updater-controller-manager-6f8476f6cb-74nx5    2/2     Running   0               106s
element-updater      element-updater-conversion-webhook-65ddcbb569-qzbfs    1/1     Running   0               81s
```

#### Generating the ElementDeployment CRD to Deploy Element Server Suite

1) Create a CRD definition on your own starting from this base template:

   ```yaml
   apiVersion: matrix.element.io/v1alpha1
   kind: ElementDeployment
   metadata:
     name: first-element
     namespace: element-onprem
   spec:
     global:
       k8s:
         ingresses:
           ingressClassName: "public"
       secretName: global
       config:
         genericSharedSecretSecretKey: genericSharedSecret
         domainName: "element.demo"
     components:
       elementWeb:
         secretName: external-elementweb-secrets
         k8s:
           ingress:
             tls:
               certmanager:
                 issuer: letsencrypt
               mode: certmanager
             fqdn: "web.element.demo"
       synapse:
         secretName: external-synapse-secrets
         config:
           additional: |
             enable_registration: True
             enable_registration_without_verification: True
           postgresql:
             host: db.element.demo
             user: postgres
             database: postgres
             passwordSecretKey: pgpassword
             sslMode: disable
         k8s:
           ingress:
             tls:
               certmanager:
                 issuer: letsencrypt
               mode: certmanager
             fqdn: "hs.element.demo"
       wellKnownDelegation:
         secretName: external-wellknowndelegation-secrets
         k8s:
           ingress:
             tls:
               certmanager:
                 issuer: letsencrypt
               mode: certmanager
       slidingSync:
         config:
           postgresql:
             host: web.element.demo
             user: postgres
             database: postgres
             passwordSecretKey: pgpassword
             sslMode: disable
           syncSecretSecretKey: syncSecret
         k8s:
           ingress:
               tls:
                 certmanager:
                   issuer: letsencrypt
                 mode: certmanager
               fqdn: "sync.element.demo"

   ```

 For more information on this option, please see our Element Deployment CRD documentation. Note: At present, this has not been written.

#### Loading secrets into kubernetes in preparation of deployment

*N.B. This guide assumes that you are using the `element-onprem` namespace for deploying Element. You can call it whatever you want and if it doesn't exist yet, you can create it with: `kubectl create ns element-onprem`.*

Now we need to load secrets into kubernetes so that the deployment can access them. If you built your own CRD from scratch, you will need to follow our Element Deployment CRD documentation. 

Here is a basic python script to build the secrets you need to get started:

```py
import os
import base64
import signedjson.key
from datetime import datetime

## Define the secrets file
SECRETS_FILE = 'secrets.yml'

## Function to generate a secret and format it properly
def generate_secret(name):
    value = base64.b64encode(os.urandom(32)).decode('utf-8')
    return f'  {name}: "{value}"'

## Function to format postgres password
def encode_pgpassword(name, pgpassword):
    encoded_value = base64.b64encode(pgpassword.encode('utf-8')).decode('utf-8')
    return f'  {name}: "{encoded_value}"'

## Function to generate unique signing key for Synapse
def generate_signing_key(name):
    signing_key = signedjson.key.generate_signing_key(0)
    value = f'{signing_key.alg} {signing_key.version} {signedjson.key.encode_signing_key_base64(signing_key)}'
    encoded_value = base64.b64encode(value.encode('utf-8')).decode('utf-8')
    return f'  {name}: "{encoded_value}"'

## Check if the secrets file exists
if os.path.isfile(SECRETS_FILE):
    timestamp = datetime.now().strftime("%s")
    backup_file = f"{SECRETS_FILE}.bak.{timestamp}"
    os.rename(SECRETS_FILE, backup_file)
    print(f"Backing up pre-existing {SECRETS_FILE} file to {backup_file}.")

#Prompt user for Postgres Password

pgpassword = input("Enter your Postgres Password: ")


## Populate secrets
print("Populating secrets.")

with open(SECRETS_FILE, 'a') as f:
    f.write('''apiVersion: v1
kind: Secret
metadata:
  name: global
  namespace: element-onprem
data:
''')

    f.write(generate_secret("genericSharedSecret") + '\n')

    f.write('''---
apiVersion: v1
kind: Secret
metadata:
  name: external-synapse-secrets
  namespace: element-onprem
data:
''')

    f.write(generate_secret("macaroon") + '\n')
    f.write(generate_secret("registrationSharedSecret") + '\n')
    f.write(generate_signing_key("signingKey") + '\n')
    f.write(encode_pgpassword("pgpassword", pgpassword) + '\n')

## Tell the user we are done
print(f"Done. Secrets are in {SECRETS_FILE}.")

```

create a file `build_secrets.py` with this content and then run it with `python3 ./build_secrets.py` to creat a `secrets.yml` that can be added to the `element-onprem` namespace with `kubectl apply -f secrets.yml`

#### Deploying the actual CRD

At this point, we are ready to deploy the ElementDeployment CRD into our cluster with the following command:

```bash
kubectl apply -f ./deployment.yml -n element-onprem
```

To check on the progress of the deployment, you will first watch the logs of the updater:

```bash
kubectl logs -f -n element-updater element-updater-controller-manager-<rest of pod name>
```

You will have to tab complete to get the correct hash for the element-updater-controller-manager pod name.

Once the updater is no longer pushing out new logs, you can track progress with the operator or by watching pods come up in the `element-onprem` namespace. 

Operator status:

```bash
kubectl logs -f -n element-operator element-operator element-operator-controller-manager-<rest of pod name>
```

Watching pods come up in the `element-onprem` namespace:

```bash
watch kubectl get pods -n element-onprem
```

#### Updating the helm charts and the underlying deployment

To install the helm charts and actually deploy the `element-updater` and the `element-operator` with their default configurations, simply run:

```bash
helm repo update ess-starter-edition-core
helm upgrade element-updater ess-starter-edition-core/element-updater --namespace element-updater
helm upgrade element-operator ess-starter-edition-core/element-operator --namespace element-operator
```

#### Registering a new user

If you have registration closed, you will need to be able to create new users. To do that with the starter edition core, you can use `kubectl exec` to open a shell in the synapse pod and use the `register_new_matrix_user` command to accomplish this action.

Let's look at how to do this. First, let's find the `synapse-main` pod:

```bash
kubectl get pods -n element-onprem | grep synapse-main
```

In this case, we get output similar to:

```
first-element-synapse-main-0                     1/1     Running   0             27m
```

Now that we know the pod name, we can run the `kubectl exec` command:

```bash
kubectl exec -it -n element-onprem first-element-deployment-synapse-main-0 -- /bin/sh
```

and once in the shell, we can run:

```bash
register_new_matrix_user -c /config/rendered/instance.yaml -u <USER> -p <PASSWORD> -a
```

and this will allow us to register a new matrix user. 

N.B. You will need to register an admin user to perform administrative functions on the server.
