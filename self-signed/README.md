# Install the TLS with sign certs  

## Genera a self sign certs  

openssl req -x509 -newkey rsa:4096 -sha256 -nodes -keyout tls.key -out tls.crt -subj "/CN=example.info" -days 365

## Create a new secret ssl 

kubectl create secret tls example-info-tls --cert=tls.crt --key=tls.key   

## Apply to ingress application  

```
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: hello-world
spec:
  tls:
   - secretName: example-info-tls
     hosts:
      - hello-world.example.info
  rules:
  - host: hello-world.example.info
    http:
      paths:
      - backend:
          serviceName: hello-world-service
          servicePort: 80
```