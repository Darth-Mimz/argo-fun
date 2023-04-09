# Configuring LetsEncrypt 

NOTE: This doc follows aws-eks-splunk-exposing-services.md

This document largely follows this URL starting around Step 5: https://cert-manager.io/docs/tutorials/acme/nginx-ingress/ as well as https://cert-manager.io/docs/installation// for installation of cert-manager.

Since the nginx ingress controller along with an ingress resource have already been created, those steps are skipped and the existing ingress resource modified to add TLS after adding the LetsEncyrpt cert-manager issuer. Recommend following the steps in the document for new domains (certmanager-stage to test issue first, then certmanager-prod) to avoid hitting limits on requesting production LetsEncyrpt certs. 

This can be modified to use AWS PCA for internal environments - https://aws.amazon.com/blogs/security/tls-enabled-kubernetes-clusters-with-acm-private-ca-and-amazon-eks-2/


## Deploy

1. Install Cert-Manager

Per cert-manager docs, cert-manager "do[es] the work with Kubernetes to request a certificate and respond to the challenge to validate it."

```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
```

2. Create an issuer 

Modify "user@example.com" to your email address. This will be used by LetsEncyrpt to notify you of certificate expiration/updates. 

NOTE: you will need to deploy the issuer within the same namespace as your application. Add a `namespace` entry under the "metadata" field with a corresponding value. 

issuer.yaml: 
```
apiVersion: cert-manager.io/v1
   kind: Issuer
   metadata:
     name: letsencrypt-prod
     namespace: splunk-operator
   spec:
     acme:
       # The ACME server URL
       server: https://acme-v02.api.letsencrypt.org/directory
       # Email address used for ACME registration
       email: user@example.com
       # Name of a secret used to store the ACME account private key
       privateKeySecretRef:
         name: letsencrypt-prod
       # Enable the HTTP-01 challenge provider
       solvers:
       - http01:
           ingress:
             class: nginx
```

```
kubectl apply -f issuer.yaml
```


3. Modify existing ingress resource to add TLS configs

- Add a `tls` entry under the "spec" section. 
- add additional entry in `annotations` section to point back to the letsencrypt issuer.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-standalone
  namespace: splunk-operator
  annotations:
    # use the shared ingress-nginx
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/default-backend: splunk-single-standalone-service
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
    # pieces for LetsEncrypt integration
    cert-manager.io/issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - splunk.defnotavirus.com
    secretName: splunk.defnotavirus.com-tls
  rules:
  - host: splunk.defnotavirus.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: splunk-single-standalone-service
            port:
              number: 8000
```

and apply: 

```
kubectl apply -f splunk-ingress.yaml
```

## Conclusion 

After a minute or two, if things are configured correctly, a certificate should be issued. `kubectl` commands in the linked articles are useful for debugging if things aren't working as expected.  
