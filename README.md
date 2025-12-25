Sources:
- https://medium.com/@muppedaanvesh/%EF%B8%8F-kubernetes-ingress-securing-the-ingress-using-cert-manager-part-7-366f1f127fd6
- https://medium.com/@chandan.chanddu/installing-nginx-ingress-controller-in-eks-using-helm-41913011ef49

1. helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
2. helm pull --untar ingress-nginx/ingress-nginx
3. 
```helm install ingress-nginx \                  
  ./ingress-nginx \          
  --namespace ingress-nginx \
  --create-namespace \
  --set-string controller.service.annotations."service\.beta\.kubernetes\.io/aws-load-balancer-type"="alb"
NAME: ingress-nginx
LAST DEPLOYED: Wed Dec 24 19:32:09 2025
NAMESPACE: ingress-nginx
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the load balancer IP to be available.
You can watch the status by running 'kubectl get service --namespace ingress-nginx ingress-nginx-controller --output wide --watch'

An example Ingress that makes use of the controller:
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: example
    namespace: foo
  spec:
    ingressClassName: nginx
    rules:
      - host: www.example.com
        http:
          paths:
            - pathType: Prefix
              backend:
                service:
                  name: exampleService
                  port:
                    number: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
      - hosts:
        - www.example.com
        secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
```
4. Create ingress resource like below:
```

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
spec:
  ingressClassName: nginx
  rules:
    - host: www.samsarian.com
      http:
        paths:
          - pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
            path: /
```
5. Install cert-manager:
a. Add the cert-manager repository:
```
helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories
suhaib.saeed@Suhaibs-MacBook-Pro eks-project % helm repo list
NAME            URL                                       
nautobot        https://nautobot.github.io/helm-charts/   
eks             https://aws.github.io/eks-charts          
ingress-nginx   https://kubernetes.github.io/ingress-nginx
jetstack        https://charts.jetstack.io       
```
b. Get latest version of cert-manager: helm repo update
c. Pull the cert-manager chart: helm pull --untar jetstack/cert-manager
d. Install cert-manager:
``` $ helm install cert-manager ./cert-manager --namespace cert-manager --create-namespace --version v1.19.2 --set installCRDs=true
NAME: cert-manager
LAST DEPLOYED: Thu Dec 25 02:01:03 2025
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
⚠️  WARNING: `installCRDs` is deprecated, use `crds.enabled` instead.
⚠️  WARNING: New default private key rotation policy for Certificate resources.
The default private key rotation policy for Certificate resources was
changed to `Always` in cert-manager >= v1.18.0.
Learn more in the [1.18 release notes](https://cert-manager.io/docs/releases/release-notes/release-notes-1.18).

cert-manager v1.19.2 has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://cert-manager.io/docs/configuration/

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://cert-manager.io/docs/usage/ingress/
```
6. Create a cluster issuer:
```
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-nginx-cert
spec:
  # ACME issuer configuration
  # `email` - the email address to be associated with the ACME account (make sure it's a valid one)
  # `server` - the URL used to access the ACME server’s directory endpoint
  # `privateKeySecretRef` - Kubernetes Secret to store the automatically generated ACME account private key
  acme:
    email: muppedaanvesh@gmail.com #Replace with your email
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-nginx-private-key-cert
    solvers:
    - http01:
        ingress:
          class: nginx
```
6a. Apply the cluster issuer: kubectl apply -f cluster-issuer.yaml

7. Update the ingress resource to use the cluster issuer: