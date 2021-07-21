# SSL Certificates. Get them from Let's Encrypt using Cert-Manager
My cluster has websites that need to be accessed through https. Kubernetes has an add-on called Cert-Manager that sets up TLS certificates using the automated process defined by [Let's Encrypt](https://letsencrypt.org/). 

After my cluster was setup and I got some applications working using the nginx ingress controller for http (port 80), I added cert-manager and then updated my ingress files to  add tls support.

From my kubectl login for my cluster, I installed cert-manager and verified that its pods were created:
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.yaml

[jkozik@dell2 k8sNw.com]$ kubectl get pods --namespace cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5d7f97b46d-gszqm              1/1     Running   4          24h
cert-manager-cainjector-69d885bf55-l6b7j   1/1     Running   7          24h
cert-manager-webhook-54754dcdfd-ncdw6      1/1     Running   0          24h
```
Note: on my cluster, this took a long time.  I thought the apply was being ignored.  But eventually it worked.  Maybe I need more resources for my cluster?!

# Verify setup, create temporary certificate
As recommended in the [cert-mangager setup guide](https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation), I created a temporary self-signed certificate, to verify that my setup was good.
```
[jkozik@dell2 k8sNw.com]$ cat <<EOF > test-resources.yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: cert-manager-test
> ---
> apiVersion: cert-manager.io/v1
> kind: Issuer
> metadata:
>   name: test-selfsigned
>   namespace: cert-manager-test
> spec:
>   selfSigned: {}
> ---
> apiVersion: cert-manager.io/v1
> kind: Certificate
> metadata:
>   name: selfsigned-cert
>   namespace: cert-manager-test
> spec:
>   dnsNames:
>     - example.com
>   secretName: selfsigned-cert-tls
>   issuerRef:
>     name: test-selfsigned
> EOF

[jkozik@dell2 k8sNw.com]$ kubectl apply -f test-resources.yaml
namespace/cert-manager-test created
issuer.cert-manager.io/test-selfsigned created
certificate.cert-manager.io/selfsigned-cert created

```
Check the results and then remove the test.
```
[jkozik@dell2 k8sNw.com]$ kubectl describe certificate -n cert-
Name:         selfsigned-cert
Namespace:    cert-manager-test
Labels:       <none>
Annotations:  API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2021-07-20T16:59:55Z
  Generation:          1
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:annotations:
          .:
          f:kubectl.kubernetes.io/last-applied-configuration:
      f:spec:
        .:
        f:dnsNames:
        f:issuerRef:
          .:
          f:name:
        f:secretName:
    Manager:      kubectl
    Operation:    Update
    Time:         2021-07-20T16:59:55Z
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:status:
        .:
        f:conditions:
        f:notAfter:
        f:notBefore:
        f:renewalTime:
        f:revision:
    Manager:         controller
    Operation:       Update
    Time:            2021-07-20T16:59:56Z
  Resource Version:  6033944
  UID:               90355574-fada-4ea5-90cc-6df10fc33af1
Spec:
  Dns Names:
    example.com
  Issuer Ref:
    Name:       test-selfsigned
  Secret Name:  selfsigned-cert-tls
Status:
  Conditions:
    Last Transition Time:  2021-07-20T17:00:01Z
    Message:               Certificate is up to date and has no
    Observed Generation:   1
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2021-10-18T17:00:01Z
  Not Before:              2021-07-20T17:00:01Z
  Renewal Time:            2021-09-18T17:00:01Z
  Revision:                1
Events:
  Type    Reason     Age   From          Message
  ----    ------     ----  ----          -------
  Normal  Issuing    38s   cert-manager  Issuing certificate as
  Normal  Generated  37s   cert-manager  Stored new private key
  Normal  Requested  37s   cert-manager  Created new Certificat
  Normal  Issuing    37s   cert-manager  The certificate has be
  
[jkozik@dell2 k8sNw.com]$  kubectl delete -f test-resources.yam
namespace "cert-manager-test" deleted
issuer.cert-manager.io "test-selfsigned" deleted
certificate.cert-manager.io "selfsigned-cert" deleted
```
# Issuer
When working with Let's Encrypt, they follow an Automated Certificate Management Environment  Certificate Authority.  The cert-manager add-on creates a resource type called ClusterIssuer that does challenge validates with Let's Encrypt to create a certificate.  For Let's Encrypt, one must create two Issuers, one for staging and the other for production.  From the [cert-manager setup guide for ACME](https://cert-manager.io/docs/configuration/acme/), the issuer file needs to follow this template:
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # You must replace this email address with your own.
    # Let's Encrypt will use this to contact you about expiring
    # certificates, and issues related to your account.
    email: user@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret resource that will be used to store the account's private key.
      name: example-issuer-account-key
    # Add a single challenge solver, HTTP01 using nginx
    solvers:
    - http01:
        ingress:
          class: nginx
```
Create two Issuers (note, I chose to user type ClusterIssuer, not Issuer)

```
[jkozik@dell2 k8sNw.com]$ cat staging_issuer.yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
 name: letsencrypt-staging
 namespace: cert-manager
spec:
 acme:
   # The ACME server URL
   server: https://acme-staging-v02.api.letsencrypt.org/directory
   # Email address used for ACME registration
   email: jackkozik@email.com
   # Name of a secret used to store the ACME account private key
   privateKeySecretRef:
     name: letsencrypt-staging
   # Enable the HTTP-01 challenge provider
   solvers:
   - http01:
       ingress:
         class:  nginx
         
[jkozik@dell2 k8sNw.com]$ kubectl create -f staging_issuer.yaml
clusterissuer.cert-manager.io/letsencrypt-staging created         

[jkozik@dell2 k8sNw.com]$ cat prod_issuer.yaml
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: cert-manager
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: jackkozik@email.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    solvers:
    - http01:
        ingress:
          class: nginx

[jkozik@dell2 k8sNw.com]$ kubectl create -f prod_issuer.yaml
clusterissuer.cert-manager.io/letsencrypt-prod created

[jkozik@dell2 ~]$ kubectl get clusterissuer
NAME                  READY   AGE
letsencrypt-prod      True    24h
letsencrypt-staging   True    24h
```
As far as I know, these issuers don't do anything.  No certificates are created until the ingress for the specific domain name is created.

# Create a certificate
My first certificate was created for https://napervilleweather.com.  I started with the ingress manifest:
```
[jkozik@dell2 k8sNw.com]$ cat nwcom-ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nwcom-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: napervilleweather.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nwcom
            port:
              number: 80
```
This file has been working just fine for http. Following the example in the cert-manager [Securing Ingress Resources}(https://cert-manager.io/docs/usage/ingress/) documentation, 

# References
- https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation
- https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes
- https://www.thinktecture.com/en/kubernetes/ssl-certificates-with-cert-manager-in-kubernetes/
- https://www.youtube.com/watch?v=etC5d0vpLZE
