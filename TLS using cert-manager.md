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
# References
-https://cert-manager.io/docs/installation/kubernetes/#verifying-the-installation
-https://www.digitalocean.com/community/tutorials/how-to-set-up-an-nginx-ingress-with-cert-manager-on-digitalocean-kubernetes
-https://www.thinktecture.com/en/kubernetes/ssl-certificates-with-cert-manager-in-kubernetes/

