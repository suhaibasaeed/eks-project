# EKS Project
## Sources
- https://medium.com/@muppedaanvesh/%EF%B8%8F-kubernetes-ingress-securing-the-ingress-using-cert-manager-part-7-366f1f127fd6
- https://medium.com/@chandan.chanddu/installing-nginx-ingress-controller-in-eks-using-helm-41913011ef49
- https://medium.com/@KushanJanith/host-your-web-apps-on-eks-with-nginx-ingress-and-external-dns-3721622e271f
## Steps
### Install ingress-nginx via helm
1. helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
2. helm pull --untar ingress-nginx/ingress-nginx
3. Install ingress-nginx:
```
helm install ingress-nginx \                  
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
4. Create ingress resource like below to verify the ingress-nginx installation:
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
### Install cert-manager via helm
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
b. Get latest version of cert-manager: `helm repo update`  
c. Pull the cert-manager chart: `helm pull --untar jetstack/cert-manager`  
d. Install cert-manager:
``` 
$ helm install cert-manager ./cert-manager --namespace cert-manager --create-namespace --version v1.19.2 --set installCRDs=true
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

### Secure the ingress with cert-manager
7. Update the ingress resource to use the cluster issuer:

```
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  annotations: # NEW
    # NEW:Specify the cluster issuer created previously to use for the ingress
    cert-manager.io/cluster-issuer: letsencrypt-nginx-cert
spec:
  tls: # NEW TLS section to secure the ingress
    - hosts:
      # NEW: Specify the hosts to use for the ingress - same as the host in the ingress resource
      - www.samsarian.com
      # NEW: Specify the secret name  to use for the ingress - this will be dynamically created by cert-manager
      secretName: letsencrypt-nginx-cert-samsarian
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
8. Apply the ingress resource: `kubectl apply -f test-secure-ingress.yaml`
9. Verify certificate has been created `kubectl describe certificate
```
kubectl describe certificate
Name:         letsencrypt-nginx-cert-samsarian
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  cert-manager.io/v1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2025-12-25T02:17:04Z
  Generation:          1
  Owner References:
    API Version:           networking.k8s.io/v1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  example
    UID:                   1e3668bc-f991-43e5-9cfd-e1418504d007
  Resource Version:        254268
  UID:                     7a6b2e2d-df35-46c8-ba26-6ecfc6a5d4e3
Spec:
  Dns Names:
    www.samsarian.com
  Issuer Ref:
    Group:      cert-manager.io
    Kind:       ClusterIssuer
    Name:       letsencrypt-nginx-cert
  Secret Name:  letsencrypt-nginx-cert-samsarian
  Usages:
    digital signature
    key encipherment
Status:
  Conditions:
    Last Transition Time:  2025-12-25T02:17:30Z
    Message:               Certificate is up to date and has not expired
    Observed Generation:   1
    Reason:                Ready <<<
    Status:                True <<<
    Type:                  Ready
  Not After:               2026-03-25T01:18:56Z
  Not Before:              2025-12-25T01:18:57Z
  Renewal Time:            2026-02-23T01:18:56Z <<<
  Revision:                1
Events:                    <none>
```
### Verify ingress works with HTTPS
```
curl https://www.samsarian.com/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
➜  prodse
```

### External DNS Installation via helm
1. Add the external-dns repository: ```helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/
2. Pull the external-dns chart: `helm pull --untar external-dns/external-dns`
3. Create AWS IAM role for external DNS with OIDC as seen here: https://medium.com/@KushanJanith/host-your-web-apps-on-eks-with-nginx-ingress-and-external-dns-3721622e271f
3. Update the keys in the external-dns values.yaml file to include the following:
```
provider: aws

domainFilters:
  - your_domain_1.example.com
  - your_domain_2.example.com
serviceAccount:
  create: true
  name: external-dns
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::XXXXXXXXXXX:role/<iam_role_name>
txtOwnerId: <name_of_your_cluster>

```
3. Install external-dns:
```
helm install external-dns ./external-dns --version 1.19.0 -f ./external-dns/values.yaml
```

4. Verify external-dns is installed:
```
kubectl get pods -n external-dns
```
5. Verify external-dns is working: `kubectl logs -n external-dns <pod_name>`

6. Re-create ingress resource to verify external-dns is working:
7. Check AWS console or external-dns logs to verify the DNS records have been created.
```
$ kubectl logs -n external-dns <pod_name>
time="2025-12-26T01:38:06Z" level=info msg="All records are already up to date"
time="2025-12-26T01:39:07Z" level=info msg="Applying provider record filter for domains: [samsarian.com. .samsarian.com.]"
time="2025-12-26T01:39:08Z" level=info msg="Desired change: CREATE aaaa-www.samsarian.com TXT" profile=default zoneID=/hostedzone/Z00877263M8QB2Y7Z5R2A zoneName=samsarian.com.
time="2025-12-26T01:39:08Z" level=info msg="Desired change: CREATE cname-www.samsarian.com TXT" profile=default zoneID=/hostedzone/Z00877263M8QB2Y7Z5R2A zoneName=samsarian.com.
time="2025-12-26T01:39:08Z" level=info msg="Desired change: CREATE www.samsarian.com A" profile=default zoneID=/hostedzone/Z00877263M8QB2Y7Z5R2A zoneName=samsarian.com.
time="2025-12-26T01:39:08Z" level=info msg="Desired change: CREATE www.samsarian.com AAAA" profile=default zoneID=/hostedzone/Z00877263M8QB2Y7Z5R2A zoneName=samsarian.com.
time="2025-12-26T01:39:08Z" level=info msg="4 record(s) were successfully updated" profile=default zoneID=/hostedzone/Z00877263M8QB2Y7Z5R2A zoneName=samsarian.com.
```


### Install ArgoCD
1. Add argoCD repo: `helm repo add argo https://argoproj.github.io/argo-helm`
2. Download helm chart into local dir: `helm pull --untar argo/argo-cd`
3. Install chart from local dir:
```
helm install argocd ./argo-cd --namespace argocd --create-namespace
NAME: argocd
LAST DEPLOYED: Fri Dec 26 19:05:42 2025
NAMESPACE: argocd
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
In order to access the server UI you have the following options:

1. kubectl port-forward service/argocd-server -n argocd 8080:443

    and then open the browser on http://localhost:8080 and accept the certificate

2. enable ingress in the values file `server.ingress.enabled` and either
      - Add the annotation for ssl passthrough: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-1-ssl-passthrough
      - Set the `configs.params."server.insecure"` in the values file and terminate SSL at your ingress: https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#option-2-multiple-ingress-objects-and-hosts


After reaching the UI the first time you can login with username: admin and the random password generated during the installation. You can find the password by running:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
3. Setup ingress resource to expose argocd-server service to expose dashboard:
```

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    # Specify the cluster issuer created previously to use for the ingress
    cert-manager.io/cluster-issuer: letsencrypt-nginx-cert
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true" # Make pod do SSL termination
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS" # Force Ingress to pod communication to be HTTPS
spec:
  tls:
    - hosts:
      # Specify the hosts to use for the ingress - same as the host in the ingress resource
      - argocd.samsarian.com
      # Specify the secret name to use for the ingress - this will be dynamically created by cert-manager
      secretName: argocd-server-tls
  ingressClassName: nginx
  rules:
    - host: argocd.samsarian.com
      http:
        paths:
          - pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 443
            path: /

```
4. Apply the ingress resource: `kubectl apply -f argocd-ingress.yaml`
5. Verify ingress is working: `curl https://argocd.samsarian.com/`
6. Login to the argocd dashboard with the username: admin and the password generated during the installation: `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`
7. Create a new application following the instructions here: https://medium.com/@muppedaanvesh/a-hands-on-guide-to-argocd-on-kubernetes-part-1-%EF%B8%8F-7a80c1b0ac98
8. Once done, test argo is functioning correctly by creating a manifest and deploy it to the cluster. E.g. `k run new-pod --image=nginx --dry-run=client -o yaml > new-pod.yaml && k apply -f new-pod.yaml`
9. Check dashboard to verify the pod has been deployed.

### Install Prometheus & Grafana for Cluster Monitoring
1. Add repo for prometheus: `helm repo add prometheus https://prometheus-community.github.io/helm-charts`
2. Pull latest version of repo: `helm repo update`
3. Pull Prometheus chart into local dir: `helm pull --untar prometheus/prometheus`
4. Install Prometheus chart from local dir: `helm install prometheus ./prometheus --namespace monitoring --create-namespace`
5. Verify the installation: `kubectl get pods -n monitoring`


