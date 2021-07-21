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
This file has been working just fine for http. Following the example in the cert-manager [Securing Ingress Resources](https://cert-manager.io/docs/usage/ingress/) documentation, I added an annotation and a tls section to the yaml file, as follows"
```
[jkozik@dell2 k8sNw.com]$ cat nwcom-ingress-tls.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nwcom-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - napervilleweather.com
    secretName: napervilleweather-com-tls
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
Cert-manager will see the annotation and use data from the referenced ClusterIssuer (letsencrypt-prod, in my case) to go LetEncrypt to create certificate and store it in the referenced secret. This happens the first time a new URL is referenced.  After that the secret is used. 
```
[jkozik@dell2 k8sNw.com]$ kubectl apply -f nwcom-ingress-tls.yml
ingress.networking.k8s.io/nwcom-ingress configured

[jkozik@dell2 k8sNw.com]$ kubectl get ingress
NAME                      CLASS    HOSTS                     ADDRESS           PORTS     AGE
nwcom-ingress             <none>   napervilleweather.com     192.168.100.174   80, 443   3d

[jkozik@dell2 k8sNw.com]$ kubectl describe ingress nwcom-ingress
Name:             nwcom-ingress
Namespace:        default
Address:          192.168.100.174
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  napervilleweather-com-tls terminates napervilleweather.com
Rules:
  Host                   Path  Backends
  ----                   ----  --------
  napervilleweather.com
                         /   nwcom:80 (10.68.77.157:80)
Annotations:             cert-manager.io/cluster-issuer: letsencrypt-staging
                         nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason             Age                   From                      Message
  ----    ------             ----                  ----                      -------
  Normal  CreateCertificate  2m31s                 cert-manager              Successfully created Certificate "napervilleweather-com-tls"
  Normal  Sync               2m18s (x3 over 2d1h)  nginx-ingress-controller  Scheduled for sync

[jkozik@dell2 k8sNw.com]$ kubectl get  certificate
NAME                        READY   SECRET                      AGE
napervilleweather-com-tls   False   napervilleweather-com-tls   17m
[jkozik@dell2 k8sNw.com]$ kubectl get  certificate
NAME                        READY   SECRET                      AGE
napervilleweather-com-tls   True    napervilleweather-com-tls   17m

[jkozik@dell2 k8sNw.com]$ kubectl get secret napervilleweather-com-tls
NAME                        TYPE                DATA   AGE
napervilleweather-com-tls   kubernetes.io/tls   2      25h

[jkozik@dell2 k8sNw.com]$ kubectl describe   certificate napervilleweather-com-tls
Name:         napervilleweather-com-tls
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2021-07-20T17:41:36Z
  Generation:          2
  Managed Fields:
    API Version:  cert-manager.io/v1
    Fields Type:  FieldsV1
    fieldsV1:
      f:metadata:
        f:ownerReferences:
          .:
          k:{"uid":"f875b374-8790-491a-b3e6-d0f4f85308d2"}:
            .:
            f:apiVersion:
            f:blockOwnerDeletion:
            f:controller:
            f:kind:
            f:name:
            f:uid:
      f:spec:
        .:
        f:dnsNames:
        f:issuerRef:
          .:
          f:group:
          f:kind:
          f:name:
        f:secretName:
        f:usages:
      f:status:
        .:
        f:conditions:
        f:notAfter:
        f:notBefore:
        f:renewalTime:
        f:revision:
    Manager:    controller
    Operation:  Update
    Time:       2021-07-20T17:58:24Z
  Owner References:
    API Version:           networking.k8s.io/v1beta1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  nwcom-ingress
    UID:                   f875b374-8790-491a-b3e6-d0f4f85308d2
  Resource Version:        6043513
  UID:                     61891b3c-5247-4a9d-9ffd-4873e007596d
Spec:
  Dns Names:
    napervilleweather.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-prod
  Secret Name:  napervilleweather-com-tls
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2021-07-20T17:58:58Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   2
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2021-10-18T16:59:12Z
  Not Before:              2021-07-20T16:59:14Z
  Renewal Time:            2021-09-18T16:59:12Z
  Revision:                2
Events:
  Type    Reason     Age                From          Message
  ----    ------     ----               ----          -------
  Normal  Issuing    18m                cert-manager  Issuing certificate as Secret does not exist
  Normal  Generated  18m                cert-manager  Stored new private key in temporary Secret resource "napervilleweather-com-tls-xg6kh"
  Normal  Requested  18m                cert-manager  Created new CertificateRequest resource "napervilleweather-com-tls-mqnp7"
  Normal  Issuing    74s                cert-manager  Issuing certificate as Secret was previously issued by ClusterIssuer.cert-manager.io/letsencrypt-staging
  Normal  Reused     74s                cert-manager  Reusing private key stored in existing Secret resource "napervilleweather-com-tls"
  Normal  Requested  74s                cert-manager  Created new CertificateRequest resource "napervilleweather-com-tls-ms4cm"
  Normal  Issuing    45s (x2 over 17m)  cert-manager  The certificate has been successfully issued
  
[jkozik@dell2 k8sNw.com]$ kubectl describe ingress nwcom-ingress
Name:             nwcom-ingress
Namespace:        default
Address:          192.168.100.174
Default backend:  default-http-backend:80 (<error: endpoints "default-http-backend" not found>)
TLS:
  napervilleweather-com-tls terminates napervilleweather.com
Rules:
  Host                   Path  Backends
  ----                   ----  --------
  napervilleweather.com
                         /   nwcom:80 (10.68.77.157:80)
Annotations:             cert-manager.io/cluster-issuer: letsencrypt-prod
                         nginx.ingress.kubernetes.io/rewrite-target: /
Events:
  Type    Reason             Age                 From                      Message
  ----    ------             ----                ----                      -------
  Normal  CreateCertificate  29m                 cert-manager              Successfully created Certificate "napervilleweather-com-tls"
  Normal  UpdateCertificate  12m                 cert-manager              Successfully updated Certificate "napervilleweather-com-tls"
  Normal  Sync               88s (x6 over 2d2h)  nginx-ingress-controller  Scheduled for sync
  
```
# Verify basic plumbing
As the command below show, the napervilleweather.com ingress's IP address is 192.168.100.174; the ingress controller exposes the cluster to the outside world on ports 80:30140 and 443:30023.  To test basic wiring, curl that IP address with port 30023 to verify things work.  It is difficult to check the certificates using curl, so use the -k option.
```
[jkozik@dell2 k8sNw.com]$ kubectl get ingress  nwcom-ingress
NAME            CLASS    HOSTS                   ADDRESS           PORTS     AGE
nwcom-ingress   <none>   napervilleweather.com   192.168.100.174   80, 443   4d1h

[jkozik@dell2 k8sNw.com]$ kubectl -n ingress-nginx get services
NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.97.70.29     <none>        80:30140/TCP,443:30023/TCP   26d
ingress-nginx-controller-admission   ClusterIP   10.111.250.10   <none>        443/TCP                      26d

[jkozik@dell2 k8sNw.com]$ curl -k -H "Host: napervilleweather.com" https://192.168.100.174:30023 | head
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
   "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
  <head>
    <!-- ##### start AJAX mods ##### -->
    <script type="text/javascript" src="ajaxCUwx.js"></script>
    <!-- AJAX updates by Ken True - http://saratoga-weather.org/wxtemplates/ -->
    <script type="text/javascript" src="ajaxgizmo.js"></script>
    <script type="text/javascript" src="language-en.js"></script>
        <!-- language for AJAX script included -->
100  7802    0  7802    0     0  58268      0 --:--:-- --:--:-- --:--:-- 58661
```


# References
- https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation
- https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes
- https://www.thinktecture.com/en/kubernetes/ssl-certificates-with-cert-manager-in-kubernetes/
- https://www.youtube.com/watch?v=etC5d0vpLZE
