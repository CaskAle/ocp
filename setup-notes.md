# Openshift Post Install Notes

## Setup htpasswd identity provider

Details are at:
https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/configuring-identity-providers

### Create an htpasswd file

```sh
htpasswd -c -B -b </path/to/htpasswd> <username> <password>
```

add additional users with the same command sans the `-c`.

### Create the htpasswd secret

This will create a secret in the openshift-config namespace based upon the htpasswd file created in the step above.

```sh
oc create secret generic htpasswd-secret \
  --from-file=htpasswd=htpasswd \
  -n openshift-config
```

### Create OAuth htpasswd provider

This will create the htpasswd identity provider.  It will use the htpasswd-secret secret created in the step above

```yaml
# htpasswd-oauth-provider.yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
name: cluster
spec:
identityProviders:
- name: htpasswd-oauth-provider 
   mappingMethod: claim 
   type: HTPasswd
   htpasswd:
      fileData:
      name: htpasswd-secret
```

```zsh
oc apply -f htpasswd-oauth-provider.yaml
```

1. Give myself admin priviledges

   ```sh
   oc adm policy add-cluster-role-to-user cluster-admin troy
   ```

1. Delete the kubeadmin user

   ```sh
   oc delete secret kubeadmin -n kube-system
   ```

## Replace default ingress certificate

1. Make ConfigMap for CA certificate in openshift-config namespace.

   ```sh
   oc create configmap ankersen-ca \
     --from-file=ca-bundle.crt=/home/troy/data/keys/ankersen-ca/ankersen-ca-ec384_crt.pem \
     -n openshift-config
   ```

1. Update the cluster-wide proxy configuration with the newly created ConfigMap.

   ```sh
   oc patch proxy/cluster \
     --type=merge \
     --patch='{"spec":{"trustedCA":{"name":"ankersen-ca"}}}'
   ```

   This can also be done by direct edit of the proxy config directly and replace the empty quotes from the defaultCertificate key with the name of the ConfigMap.

   ```sh
   oc edit proxy/cluster
   ```

1. Create a tls secret in the openshift-ingress namespace.

   ```sh
   oc create secret tls ankersen-ingress-cert \
     --cert=/home/troy/data/keys/ankersen-ca/apps.ocp.ankersen.dev-wildcard-ec384_bundle.pem \
     --key=/home/troy/data/keys/ankersen-ca/apps.ocp.ankersen.dev-wildcard-ec384_prv.pem \
     -n openshift-ingress
   ```

   > Note: The pem formatted cert file should contain, in order, the certificate, any intermediate certificates, and the CA certificate.

1. Update the Ingress Controller configuration with the secret.

   ```sh
   oc patch ingresscontroller.operator default \
     --type=merge \
     -p '{"spec":{"defaultCertificate": {"name": "ankersen-ingress-cert"}}}' \
     -n openshift-ingress-operator
   ```

   This can also be edited directly with:

   ```sh
   oc edit ingresscontroller.operator -n openshift-ingress-operator
   ```

## Set up etcd defragmentation cron job

```sh
 oc create -k kustomization.yaml
 ```

## Set up persistent storage for cluster monitoring

- Use the ConfigMap already created
- Assumes that the LVM Storage Operator is installed.
- StorageClass `odf-lvm-vg1`

```sh
oc apply -f cluster-monitoring-config.yaml
```

## Set up image registry storage

### Create the PVC in advance for SNO

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### Modify the image registry configuration

```sh
oc edit configs.imageregistry.operator.openshift.io
```

Changes:

```yaml
managementState: Managed
rolloutStrategy: Recreate
storage:
  pvc:
    claim: image-registry-storage
```

## AlertManager Receivers

gmail smtp: smtp.gmail.com:587

gmail userid: CaskAle13c

gmail password: use app password

## Grow the root filesystem in CoreOS

```zsh
sudo su
growpart /dev/sda 4
sudo su -
unshare --mount
mount -o remount,rw /sysroot
xfs_growfs /sysroot
```
