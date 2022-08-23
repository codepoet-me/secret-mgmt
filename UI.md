Expose the Vault UI with port-forwarding:
```shell
kubectl port-forward vault-0 8200:8200
```

Access the URL:
```
http://localhost:8200/ui
```

First, start an interactive shell session on the vault-0 pod.
```shell
kubectl exec -it vault-0 -- /bin/sh
```

Enable the userpass method.

```shell
vault auth enable userpass
```

Create username and password
```shell
vault write auth/userpass/users/admin policies=default password=admin123
```

```shell
vault policy write default - <<EOF
path "secret/*" {
  capabilities = ["read","create","update","delete","list"]
}

path "sys/capabilities-self" {
  capabilities = ["read","create","update","delete","list"]
}

path "sys/policies/acl/*" {
  capabilities = ["read","create","update","delete","list"]
}

path "sys/auth" {
  capabilities = ["read","create","update","delete","list"]
}

path "identity/entity/id" {
  capabilities = ["read","create","update","delete","list"]
}

path "identity/entity-alias/id/*" {
  capabilities = ["read","create","update","delete","list"]
}

path "identity/group/id/*" {
  capabilities = ["read","create","update","delete","list"]
}

path "identity/group-alias/id/*" {
  capabilities = ["read","create","update","delete","list"]
}

path "sys/mounts/*" {
  capabilities = ["read","create","update","delete","list"]
}

path "auth/kubernetes/role/*" {
  capabilities = ["read","create","update","delete","list"]
}

path "auth/kubernetes/config" {
  capabilities = ["read","create","update","delete","list"]
}

path "auth/userpass/users/*" {
  capabilities = ["read","create","update","delete","list"]
}

EOF
```

