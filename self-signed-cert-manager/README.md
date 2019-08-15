# Install the TLS with sign certs Certs Manager 

## (Optional) Generate a signing key pair
The CA Issuer does not automatically create and manage a signing key pair for you. As a result, you will need to either supply your own or generate a self signed CA using a tool such as openssl or cfssl.

This guide will explain how to generate a new signing key pair, however you can substitute it for your own so long as it has the CA flag set.

## Generate a CA private key
$ openssl genrsa -out ca.key 2048

## Create a self signed Certificate, valid for 10yrs with the 'signing' option set
openssl req -x509 -new -sha256 -nodes -key ca.key -subj "/CN=sampleissuer.local" -days 1024 -out ca.crt -extensions v3_ca -config openssl-with-ca.cnf

The output of these commands will be two files, ca.key and ca.crt, the key and certificate for your signing key pair. If you already have your own key pair, you should name the private key and certificate ca.key and ca.crt respectively.

# Save the signing key pair as a Secret

```
kubectl create secret tls ca-key-pair \
   --cert=ca.crt \
   --key=ca.key \
   --namespace=default

```

# Creating an Issuer referencing the Secret

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: ca-issuer
  namespace: default
spec:
  ca:
    secretName: ca-key-pair

```
# Obtain a signed Certificate

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: ca-issuer
    # We can reference ClusterIssuers by changing the kind here.
    # The default value is Issuer (i.e. a locally namespaced Issuer)
    kind: Issuer
  commonName: example.com
  organization:
  - Example CA
  dnsNames:
  - example.com
  - www.example.com
```

In order to use the Issuer to obtain a Certificate, we must create a Certificate resource in the same namespace as the Issuer, as an Issuer is a namespaced resource. We could alternatively create a ClusterIssuer if we wanted to reuse the signing key pair across multiple namespaces.

Once we have created the Certificate resource, cert-manager will attempt to use the Issuer ca-issuer to obtain a certificate. If successful, the certificate will be stored in a Secret resource named example-com-tls in the same namespace as the Certificate resource (default).

The example above explicitly sets the commonName field to example.com. cert-manager automatically adds the commonName field as a DNS SAN if it is not already contained in the dnsNames field.

If we had not specified the commonName field, then the first DNS SAN that is specified (under dnsNames) would be used as the certificate’s common name.

After creating the above Certificate, we can check whether it has been obtained successfully like so:

```
$ kubectl describe certificate example-com
Events:
  Type     Reason                 Age              From                     Message
  ----     ------                 ----             ----                     -------
  Warning  ErrorCheckCertificate  26s              cert-manager-controller  Error checking existing TLS certificate: secret "example-com-tls" not found
  Normal   PrepareCertificate     26s              cert-manager-controller  Preparing certificate with issuer
  Normal   IssueCertificate       26s              cert-manager-controller  Issuing certificate...
  Normal   CertificateIssued      25s              cert-manager-controller  Certificate issued successfully
```

You can also check whether issuance was successful with kubectl get secret example-com-tls -o yaml. You should see a base64 encoded signed TLS key pair.

# URLs

References    
  https://docs.cert-manager.io/en/latest/tasks/issuers/setup-ca.html#optional-generate-a-signing-key-pair  
  https://github.com/jetstack/cert-manager/issues/279  