# MongoDB Instance Helm Chart

This repository contains a Helm chart for deploying MongoDB ReplicaSet on Kubernetes using the MongoDB Kubernetes Operator.

## 📌 General Information

- **MongoDB Kubernetes Operator**: [MongoDB For Kubernetes](https://github.com/mongodb/mongodb-kubernetes)
- **Git Repository**: [mongodb-for-kubernetes-community](https://github.com/plopoyop/mongodb-for-kubernetes-community)
- **Helm Repository**: [https://plopoyop.github.io/charts/](https://plopoyop.github.io/charts/)
- **Maintainer**: [plopoyop](https://github.com/plopoyop)
- **License**: MPL2

## 📦 Installation

Follow [official documentation](https://www.mongodb.com/docs/kubernetes/current/) to install the MongoDb Kubernetes Operator.

Add the Helm repository:

```sh
helm repo add plopoyop https://plopoyop.github.io/charts/
helm repo update
```

Install the MongoDB Instance chart:

```sh
helm install my-mongodb plopoyop/mongodb-instance
```

## ⚙️ Configuration

You can customize the deployment by modifying the values in the `values.yaml` file.

To see all available values:

```sh
helm show values plopoyop/mongodb-instance
```

Example of installation with custom values:

```sh
helm install my-mongodb plopoyop/mongodb-instance -f my-values.yaml
```

### Main Configuration Options

Below are some key values from `values.yaml`:

```yaml
name: example-mongodb
members: 3
version: "8.0.18"
persistent: true
adminPassword: change-me

user:
  name: admin
  db: admin
  passwordSecretRef: mongo-admin-password
  roles:
    - name: clusterAdmin
      db: admin
    - name: userAdminAnyDatabase
      db: admin
    - name: dbAdminAnyDatabase
      db: admin
    - name: readWriteAnyDatabase
      db: admin

scramCredentialsSecretName: ""
connectionStringSecretName: ""

dataVolumeStorage: "10Gi"
logVolumeStorage: "2Gi"
```

### 🔐 Admin Password

The `adminPassword` value **must** be changed from its default placeholder `change-me`.
If it is left as `change-me` (or empty), the chart refuses to render and `helm install`,
`helm upgrade` and `helm template` fail with an actionable error message. Set a strong
value before installing:

```sh
helm install my-mongodb plopoyop/mongodb-instance --set adminPassword=<strong-password>
```

### additionalUsers

By default the chart creates a single primary user from the `user` block. To create
additional MongoDB users, add entries to the `additionalUsers` list. For each entry a
dedicated Opaque Secret holding the user's password is created automatically.

The secret name is computed by the `mongodb.userSecretName` helper: it uses
`<release-fullname>-<passwordSecretRef>` when `passwordSecretRef` is set, otherwise it
falls back to `<release-fullname>-<name>-password`. `passwordSecretRef` is therefore
optional.

```yaml
# default
additionalUsers: []

# example
additionalUsers:
  - name: "app-user"
    db: "myapp"
    password: "a-strong-password"
    # passwordSecretRef is optional; defaults to "<name>-password"
    roles:
      - name: "readWrite"
        db: "myapp"
    scramCredentialsSecretName: ""
    connectionStringSecretName: ""
```

### ⚠️ Service Account

Service account name is hardcoded as `mongodb-kubernetes-appdb` in community operator. If you intend to deploy multiple replicasets in the same namespace, make sure to disable service account creation for the second deployment. Otherwise, Helm will report an ownership conflict.

```yaml
serviceAccount:
  create: false
```

### TLS / Encryption

You can enable TLS encryption for connections to the MongoDB cluster. This requires [cert-manager](https://cert-manager.io/) (or a pre-created certificate `Secret` and CA `ConfigMap`) to provide the certificates out-of-band. The operator references them and sets the connection `ssl` option to `true` when TLS is enabled.

Provide a `Secret` holding `tls.crt` + `tls.key` via `certificateKeySecretRef`, and the CA certificate via either `caConfigMapRef` (a `ConfigMap`) or `caCertificateSecretRef` (a `Secret` holding `ca.crt`).

```yaml
# default (disabled)
tls:
  enabled: false
  optional: false
  certificateKeySecretRef:
    name: ""
  caConfigMapRef:
    name: ""
  caCertificateSecretRef:
    name: ""

# example value:
tls:
  enabled: true
  # optional: true allows both TLS and non-TLS connections
  optional: false
  certificateKeySecretRef:
    name: "tls-secret-name"
  caConfigMapRef:
    name: "tls-ca-configmap-name"
  # Alternative to caConfigMapRef, a Secret holding ca.crt:
  # caCertificateSecretRef:
  #   name: "tls-ca-secret-name"
```

#### Automatic certificate generation with cert-manager

cert-manager does **not** create a certificate on its own — it only issues one when a
`Certificate` resource pointing to an `Issuer`/`ClusterIssuer` exists. Enable
`tls.certManager.enabled` to let the chart create that `Certificate` for you. The chart
then generates a `Secret` named `<release-fullname>-tls` and automatically wires both
`certificateKeySecretRef` and `caCertificateSecretRef` to it, so `helm install` is enough
— no manual resource required. You only need an existing `Issuer`/`ClusterIssuer`.

```yaml
tls:
  enabled: true
  certManager:
    enabled: true
    issuerRef:
      name: "my-ca-issuer"   # required: an existing Issuer/ClusterIssuer
      kind: "ClusterIssuer"  # or "Issuer"
    duration: "8760h"        # 365 days
    renewBefore: "720h"      # 30 days
```

The generated certificate covers `*.<release-fullname>-svc.<namespace>.svc.cluster.local`.
Setting `certificateKeySecretRef.name` overrides the default secret name.

[Official documentation](https://github.com/mongodb/mongodb-kubernetes/blob/master/docs/mongodbcommunity/secure.md) on securing MongoDBCommunity resources.

### additionalMongodConfig

Additional configuration that can be passed to each data-bearing mongod at runtime

[Example](https://github.com/mongodb/mongodb-kubernetes/blob/master/docs/mongodbcommunity/deploy-configure.md) in official repository

```yaml
# default
additionalMongodConfig: {}

# example value:
additionalMongodConfig:
  operationProfiling:
    mode: slowOp
    slowOpThresholdMs: 100
```

### additionalConnectionStringConfig

Additional options to be appended to the connection string

[Example](https://github.com/mongodb/mongodb-kubernetes/blob/master/docs/mongodbcommunity/users.md) in official repository

```yaml
# default
additionalConnectionStringConfig: {}

# example value:
additionalConnectionStringConfig:
  authenticationMechanism: "scram-sha256"
```

### containersAdditionalConfig

Override default pods configuration

```yaml
# default
containersAdditionalConfig: {}

# example
containersAdditionalConfig:
  - name: mongod
    resources:
      limits:
        cpu: "1"
        memory: 900M
      requests:
        cpu: "500m"
        memory: 400M
  - name: mongodb-agent
    readinessProbe:
      failureThreshold: 40
      initialDelaySeconds: 5
      timeout: 60
```

### Prometheus

Expose metrics for prometheus

[Example](https://github.com/mongodb/mongodb-kubernetes/blob/master/docs/mongodbcommunity/prometheus/README.md) in official repository

```yaml
metrics:
  enabled: true
  username: "prometheus"
  password: "change-me"
  passwordSecretRef: "metrics-password"
```

### Backups

You can add a backup CronJob. Below are env vars exported to the job:

```yaml
USERNAME : mongodb username from connection string secret
PASSWORD : mongodb password from connection string secret
HOST : mongodb service host
REPLICATSET : replicatset name
PORT : mongodb default port
STORAGE_PATH : value of backup.cronjob.storage.mountPath
TZ : timezone from CronJob
```

You can set some env vars to the job:

```yaml
container_env:
  - name: "MY_ENV_VAR"
    value: "my-env-value"
```

You can set some sensitive env vars to job. Secret will be created for you.

```yaml
container_secret_env:
  - name: "MY_SECRET_ENV_VAR"
    secret_key: "key_in_k8s_secret"
    value: "my-sensitive-value"
```

Full example:

```yaml
backup:
  enabled: true
  cronjob:
    schedule: "*/15 * * * *"
    timeZone: "Europe/Paris"
    command:
      - "bash"
      - "-c"
      - "/usr/local/bin/backup_mongodb.sh;"
    container_secret_env:
      - name: "S3_ACCESS_KEY"
        secret_key: "s3_access_key"
        value: "ak"
      - name: "S3_SECRET_KEY"
        secret_key: "s3_secret_key"
        value: "as"
    container_env:
      - name: S3_ENDPOINT
        value: "s3.us-east-2.amazonaws.com"
      - name: S3_STORAGE_PATH
        value: "my-bucket/mongodb-backups"
    storage:
      mountPath: "/backup"
    imagePullSecrets:
      - name: "my-secret"
        registry: "docker.io"
        username: "username"
        password: "password"
        email: "user@example.com"
    image:
      repository: mongo-backup
      tag: latest
      pullPolicy: Always
    concurrencyPolicy: Allow
    failedJobsHistoryLimit: 3
    successfulJobsHistoryLimit: 2
```

## Chart Information

Details from `Chart.yaml`:

```yaml
apiVersion: v2
name: mongodb-instance
description: Deploy and manage a MongoDB ReplicaSet on Kubernetes using the MongoDB Kubernetes Operator
version: "0.1.0"
appVersion: "8.0.18"
```

## 🚀 Upgrade

To upgrade an existing installation:

```sh
helm upgrade my-mongodb plopoyop/mongodb-instance
```

## 🗑 Uninstallation

To uninstall MongoDB:

```sh
helm uninstall my-mongodb
```

## 📜 License

This project is licensed under the Mozilla Public License 2.0 - see the [LICENSE](https://github.com/plopoyop/mongodb-for-kubernetes-community/blob/main/LICENSE) file for details.
