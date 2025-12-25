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