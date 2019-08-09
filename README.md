
#  Installing nginx ingress-controller
In order for the ingress resource to work, the cluster must have an ingress controller running. Ingress controllers are not automatically installed and started by the cluster.

Nginx Ingress Controller Installation Guide: https://kubernetes.github.io/ingress-nginx/deploy/

In Bare metal cluster, install the nginx controller and nodeport service. Ingress controller and nodeport are installed in a separate namespace “ingress-nginx”.

The installation manifests are located in the ingress-controller folder. In the steps below we assume that you will be running the commands from that folder.

## Install the NGINXC ingress controller
```
kubectl apply -f ingress-controller-mandatory.yaml
```
## Install NodePort Service

```
kubectl apply -f nodeport-service.yaml
```

#  Installing cert-manager
## Create a namespace to run cert-manager in
```
kubectl create namespace cert-manager
```

As part of the installation, cert-manager also deploys a ValidatingWebhookConfiguration resource in order to validate that the Issuer, ClusterIssuer and Certificate resources we will create after installation are valid.

In order to deploy the ValidatingWebhookConfiguration, cert-manager creates a number of ‘internal’ Issuer and Certificate resources in its own namespace.

This creates a chicken-and-egg problem, where cert-manager requires the webhook in order to create the resources, and the webhook requires cert-manager in order to run.

We avoid this problem by disabling resource validation on the namespace that cert-manager runs in:

You can read more about the webhook on (the webhook document)[https://docs.cert-manager.io/en/latest/getting-started/webhook.html]

## Disable resource validation on the cert-manager namespace
```
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true
```

We can now go ahead and install cert-manager. All resources (the CustomResourceDefinitions, cert-manager, and the webhook component) are included in a single YAML manifest file:

## Install the CustomResourceDefinitions and cert-manager itself

```
kubectl apply -f cert-manager.yml
```

If you are running kubectl v1.12 or below, you will need to add the --validate=false flag to your kubectl apply command above else you will receive a validation error relating to the caBundle field of the ValidatingWebhookConfiguration resource. This issue is resolved in Kubernetes 1.13 onwards. More details can be found in (kubernetes/kubernetes#69590.)[https://github.com/kubernetes/kubernetes/issues/69590]


## Verifying the installation
Once you’ve installed cert-manager, you can verify it is deployed correctly by checking the cert-manager namespace for running pods:

```
kubectl get pods --namespace cert-manager

NAME                               READY   STATUS      RESTARTS   AGE
cert-manager-5c6866597-zw7kh       1/1     Running     0          2m
webhook-78fb756679-9bsmf           1/1     Running     0          2m
webhook-ca-sync-1543708620-n82gj   0/1     Completed   0          1m
```

You should see both the cert-manager and webhook component in a Running state, and the ca-sync pod is Completed. If the webhook has not Completed but the cert-manager pod has recently started, wait a few minutes for the ca-sync pod to be retried. If you experience problems, please check the (troubleshooting guide.) [https://docs.cert-manager.io/en/latest/getting-started/troubleshooting.html]

## Create a ClusterIssuer to test the webhook works okay
```
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  commonName: example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```
## Create the test resources
```
kubectl apply -f test-resources.yaml
```
## Check the status of the newly created certificate
## You may need to wait a few seconds before cert-manager processes the
## certificate request
```
kubectl describe certificate -n cert-manager-test
...
Spec:
  Common Name:  example.com
  Issuer Ref:
    Name:       test-selfsigned
  Secret Name:  selfsigned-cert-tls
Status:
  Conditions:
    Last Transition Time:  2019-01-29T17:34:30Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-04-29T17:34:29Z
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  CertIssued  4s    cert-manager  Certificate issued successfully
```
## Clean up the test resources
```
kubectl delete -f test-resources.yaml
```

# URLs

References: 
  https://kubernetes.github.io/ingress-nginx/deploy/  
  https://github.com/jetstack/cert-manager/issues/279    