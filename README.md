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

### ⚠️ Service Account

Service account name is hardcoded as `mongodb-kubernetes-appdb` in community operator. If you intend to deploy multiple replicasets in the same namespace, make sure to disable service account creation for the second deployment. Otherwise, Helm will report an ownership conflict.

```yaml
serviceAccount:
  create: false
```

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
