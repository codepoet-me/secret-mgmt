# Hashicorp Vault Secrets in Kubernetes with CSI Driver

Source:
1. https://pavan1999-kumar.medium.com/hashicvault-secrets-in-kubernetes-with-csi-driver-ec917d4a2672
2. https://learn.hashicorp.com/tutorials/vault/kubernetes-secret-store-driver?in=vault/kubernetes
3. https://learn.hashicorp.com/tutorials/vault/kubernetes-azure-aks?in=vault/kubernetes

## 1. Install the Vault Helm chart

Add the HashiCorp Helm repository.

```shell
helm repo add hashicorp https://helm.releases.hashicorp.com
```

Update all the repositories to ensure helm is aware of the latest versions.

```shell
helm repo update
```

Install the Vault Helm chart running in development mode with the injector service disabled and CSI enabled.

```shell
helm install vault hashicorp/vault \
--set "server.dev.enabled=true" \
--set "injector.enabled=false" \
--set "csi.enabled=true" \
--version 0.19.0
```

Example output:
```shell
NAME: vault
LAST DEPLOYED: Mon Apr 25 17:06:20 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/

Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault
```

The Vault server runs in development mode on a single pod server.dev.enabled=true. The Vault Agent Injector pod is disabled injector.enabled=false and the Vault CSI Provider pod csi.enabled=true is enabled.

Display all the pods within the default namespace.

```shell
kubectl get pods
```

Example output:
```shell
NAME                       READY   STATUS    RESTARTS   AGE
vault-0                    1/1     Running   0          58s
vault-csi-provider-t874l   1/1     Running   0          58s
```



Wait until the vault-0 pod is running and ready (1/1).

## 2. Set a secret in Vault

The volume mounted to the pod in the Create a pod with secret mounted section expects a secret stored at the path secret/data/db-pass. When Vault is run in development a KV secret engine is enabled at the path /secret.

First, start an interactive shell session on the vault-0 pod.

```shell
kubectl exec -it vault-0 -- /bin/sh
```

Your system prompt is replaced with a new prompt / $. Commands issued at this prompt are executed on the vault-0 container.

Create a secret at the path secret/db-secrets with a password.

```shell
vault kv put secret/db-secrets password="pass123"
vault kv put secret/db-secrets user="dbuser"
```

Verify that the secret is readable at the path secret/db-secrets.

```shell
vault kv get secret/db-secrets
```


## 3. Configure Kubernetes authentication

Vault provides a Kubernetes authentication method that enables clients to authenticate with a Kubernetes Service Account Token. The Kubernetes resources that access the secret and create the volume authenticate through this method through a role.

Enable the Kubernetes authentication method.

```shell
vault auth enable kubernetes
```

Configure the Kubernetes authentication method with the Kubernetes API address. It will automatically use the Vault pod's own service account token.

```shell
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

The environment variable KUBERNETES_PORT_443_TCP_ADDR references the internal network address of the Kubernetes host.

Create a policy named internal-app-plcy. This will be used to give the webapp-sa service account permission to read the kv secret created earlier.

```shell
vault policy write myapp-plcy - <<EOF
path "secret/data/db-secrets" {
  capabilities = ["read"]
}
EOF
```

The data of kv-v2 requires that an additional path element of data is included after its mount path (in this case, secret/).

Finally, create a Kubernetes authentication role named database that binds this policy with a Kubernetes service account named webapp-sa.


```shell
vault write auth/kubernetes/role/secrets-ro \
    bound_service_account_names=webapp-sa \
    bound_service_account_namespaces=default \
    policies=myapp-plcy \
    ttl=20m
```

The role connects the Kubernetes service account, webapp-sa, in the namespace, default, with the Vault policy, internal-app-plcy. 
The tokens returned after authentication are valid for 20 minutes. This Kubernetes service account name, webapp-sa, will be created below.

Lastly, exit the vault-0 pod.

```shell
exit
```

## 4. Install the secrets store CSI driver

The Secrets Store CSI driver secrets-store.csi.k8s.io allows Kubernetes to mount multiple secrets, keys, and certs stored in enterprise-grade external secrets stores into their pods as a volume. Once the Volume is attached, the data in it is mounted into the container's file system.

Add the Secrets Store CSI driver Helm repository.

```shell
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
```


Install the latest version of the Kubernetes Secrets Store CSI Driver.

```shell
helm install csi secrets-store-csi-driver/secrets-store-csi-driver \
    --set syncSecret.enabled=true
```

Example output:

```shell
NAME: csi
LAST DEPLOYED: Mon Apr 25 17:12:21 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Secrets Store CSI Driver is getting deployed to your cluster.

To verify that Secrets Store CSI Driver has started, run:

  kubectl --namespace=default get pods -l "app=secrets-store-csi-driver"

Now you can follow these steps https://secrets-store-csi-driver.sigs.k8s.io/getting-started/usage.html
to create a SecretProviderClass resource, and a deployment using the SecretProviderClass.
```

## 5. Verify the Vault CSI provider is running

The Secrets Store CSI driver enables extension through providers. A provider is launched as a Kubernetes DaemonSet alongside of Secrets Store CSI driver DaemonSet.

The Vault CSI provider was installed above alongside Vault by the Vault Helm chart.

This DaemonSet launches its own provider pod and runs a gRPC server which the Secrets Store CSI Driver connects to to make volume mount requests.

Get all the pods within the default namespace to check that the Vault CSI provider is running.

```shell
kubectl get pods
```

Example output:
```shell
NAME                                 READY   STATUS    RESTARTS   AGE
csi-secrets-store-csi-driver-vkppq   3/3     Running   0          20s
vault-0                              1/1     Running   0          3m10s
vault-csi-provider-t874l             1/1     Running   0          3m10s
```

## 6. Define a SecretProviderClass resource

```shell
kubectl apply -f spc.yaml
```

The vault-database SecretProviderClass describes one secret object:

- objectName is a symbolic name for that secret, and the file name to write to.
- secretPath is the path to the secret defined in Vault.
- secretKey is a key name within that secret.

Verify that the SecretProviderClass, named vault-database-provider has been defined in the default namespace.

```shell
kubectl describe SecretProviderClass vault-database-provider
```

Example output:
```shell
Name:         vault-database-provider
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"secrets-store.csi.x-k8s.io/v1","kind":"SecretProviderClass","metadata":{"annotations":{},"name":"vault-database","namespace...
API Version:  secrets-store.csi.x-k8s.io/v1
Kind:         SecretProviderClass
## ...
```

## 7. Create a pod with secret mounted

With the secret stored in Vault, the authentication configured and role created, the provider-vault extension installed and the SecretProviderClass defined it is finally time to create a pod that mounts the desired secret.

Create a service account named webapp-sa.

```shell
kubectl apply -f serviceaccount.yaml
```
Create the webapp pod.

The webapp pod defines and mounts a read-only volume to /mnt/secrets-store. The objects defined in the vault-database-provider SecretProviderClass are written as files within that path.

```shell
kubectl apply -f deploy.yaml
```

Get all the pods within the default namespace.

```shell
kubectl get pods
```

Example output:
```shell
NAME                                     READY   STATUS    RESTARTS   AGE
csi-secrets-store-csi-driver-6rf2k       3/3     Running   0          13m
csi-secrets-store-provider-vault-qm44g   1/1     Running   0          8m
webapp-788c95b4fc-9wffn                  1/1     Running   0          5m
vault-0                                  1/1     Running   0          27m
```

Wait until the webapp pod is running and ready (1/1).

Display the password secret written to the file system at /mnt/secrets-store/db-password on the webapp pod.

```shell
kubectl exec webapp-679745f86b-nhb2t -- cat /mnt/secrets-store/db-password
```

```shell
kubectl exec webapp-679745f86b-nhb2t -- cat /mnt/secrets-store/db-user
```


## 8. Sync to a Kubernetes Secret

The Secrets Store CSI Driver also supports syncing to Kubernetes secret objects.
Kubernetes secrets are populated with the contents of files from your CSI volume, and their lifetime is closely tied to the lifetime of the pod they are created for.

To add secret syncing for your webapp pod, update the SecretProviderClass to add a secretObjects entry:

```shell
kubectl delete SecretProviderClass vault-secret-provider && kubectl apply -f spc-sync.yaml
```

When a pod references this SecretProviderClass, the CSI driver will create a Kubernetes secret called "dbpass" with the "password" field set to the contents of the "db-password" object from the parameters. The pod will wait for the secret to be created before starting, and the secret will be deleted when the pod stops.

Next, update the pod to reference the new secret:

Notice there is now an env entry, referencing a secret. Delete and redeploy the pod:

Verify that the SecretProviderClass, named vault-database-provider has been updated in the default namespace.

```shell
kubectl describe SecretProviderClass vault-secret-provider
```

You can now verify the Kubernetes secret has been created:

```shell
kubectl get secret db-secrets
```

```shell
kubectl edit secret db-secrets
```


```shell
kubectl delete deploy webapp && kubectl apply -f deploy-sync.yaml
```

Deploy the updated configs and wait until the webapp pod has come up again.

```shell
kubectl get pods
```

Example output:
```shell
NAME                                 READY   STATUS    RESTARTS   AGE
csi-secrets-store-csi-driver-w2xxv   3/3     Running   0          4m28s
vault-0                              1/1     Running   0          5m57s
vault-csi-provider-qxz8d             1/1     Running   0          5m57s
webapp                               1/1     Running   0          36s
```


And you can also verify the secret is available in the pod's environment:

```shell
kubectl exec webapp-747887d8b9-r9wzx -- env | grep DB_PASSWORD
```

```shell
kubectl exec webapp-747887d8b9-r9wzx -- env | grep DB_USER
```

## Clean up

### Undeploy webapp
```shell
kubectl delete deploy webapp
```

### Uninstall helm charts
```shell
helm uninstall csi vault
```
