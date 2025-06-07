# Openshift Post Install Notes

## Setup htpasswd identity provider

Details are at:
<https://docs.redhat.com/en/documentation/openshift_container_platform/4.18/html/authentication_and_authorization/configuring-identity-providers>

### Create an htpasswd file

```sh
htpasswd -c -B -b </path/to/htpasswd> <username> <password>
```

- at least one user should be "troy".
- add additional users with the same command sans the `-c`.

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

### Give user "troy" admin priviledges

```zsh
oc adm policy add-cluster-role-to-user cluster-admin troy
```

### Delete the kubeadmin user

```sh
oc delete secret kubeadmin -n kube-system
```

## Replace default ingress certificate with LetsEncrypt certificate

### Install the cert-manager operator

### Get an api token from CloudFlare dns

### Create a secret for Cloudflare dns api token

```zsh
oc create secret generic cloudflare-api-token-secret \
  -n cert-manager \
  --from-literal=api-token=<token>
  ```

### Create a LetsEncrypt ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cluster-issuer
spec:
  acme:
    server: 'https://acme-v02.api.letsencrypt.org/directory'
    privateKeySecretRef:
      name: acme-account-private-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              key: api-token
              name: cloudflare-api-token-secret
```

### Create apps wildcard ingress Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: apps-ocp-ankersen-dev-certificate
  namespace: openshift-ingress
spec:
  commonName: apps.ocp.ankersen.dev
  dnsNames:
    - "apps.ocp.ankersen.dev" 
    - "*.apps.ocp.ankersen.dev"
  secretName: apps-ocp-ankersen-dev-tls
  isCA: false
  privateKey:
    algorithm: ECDSA
    rotationPolicy: Always
    size: 384
  issuerRef:
    group: cert-manager.io
    name: letsencrypt-cluster-issuer
    kind: ClusterIssuer
```

### Download the LetsEncrypt CA certificate

<https://letsencrypt.org/certs/isrgrootx1.pem>

```zsh
curl https://letsencrypt.org/certs/isrgrootx1.pem > ./letsencrypt.pem
```

### Create a ConfigMap for LetsEncrypt ca certificate

```zsh
oc create configmap trusted-ca \
  --from-file=ca-bundle.crt=./letsencrypt.pem \
  -n openshift-config
```

```zsh
oc patch proxy cluster \
  --type=merge \
  --patch='{"spec":{"trustedCA":{"name":"trusted-ca"}}}'
```

This can also be edited directly with:

```zsh
oc edit proxy cluster
```

### Update the default IngressController to use the LetsEncrypt wildcard tls

```zsh
oc patch ingresscontroller.operator default \
  --type=merge \
  --patch '{"spec":{"defaultCertificate": {"name": "apps-ocp-lab-snimmo-com-tls"}}}' \
  -n openshift-ingress-operator
```

This can also be edited directly with:

```zsh
oc edit ingresscontroller.operator default \
  -n openshift-ingress-operator
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
spec:
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

## Set up etcd defragmentation cron job

```zsh
 oc create -k kustomization.yaml
 ```
