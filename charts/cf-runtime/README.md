## Codefresh Runner

![Version: 6.1.3](https://img.shields.io/badge/Version-6.1.3-informational?style=flat-square)

Helm chart for deploying [Codefresh Runner](https://codefresh.io/docs/docs/installation/codefresh-runner/) to Kubernetes.

## Table of Content

- [Prerequisites](#prerequisites)
- [Get Repo Info](#get-repo-info)
- [Install Chart](#install-chart)
- [Chart Configuration](#chart-configuration)
- [Upgrade Chart](#upgrade-chart)
  - [To 2.x](#to-2-x)
  - [To 3.x](#to-3-x)
  - [To 4.x](#to-4-x)
  - [To 5.x](#to-5-x)
  - [To 6.x](#to-6-x)
- [Architecture](#architecture)
- [Configuration](#configuration)
  - [EBS backend volume configuration](#ebs-backend-volume-configuration)
  - [Custom volume mounts](#custom-volume-mounts)
  - [Custom global environment variables](#custom-global-environment-variables)
  - [Volume reuse policy](#volume-reuse-policy)
  - [Volume cleaners](#volume-cleaners)
  - [Rootless DinD](#rootless-dind)
  - [ARM](#arm)
  - [Openshift](#openshift)
  - [On-premise](#on-premise)

## Prerequisites

- Kubernetes **1.19+**
- Helm **3.8.0+**

## Get Repo Info

```console
helm repo add cf-runtime http://chartmuseum.codefresh.io/cf-runtime
helm repo update
```

## Install Chart

**Important:** only helm3 is supported

- Specify the following mandatory values

```yaml
# -- Global parameters
# @default -- See below
global:
  # -- User token in plain text (required if `global.codefreshTokenSecretKeyRef` is omitted!)
  # Ref: https://g.codefresh.io/user/settings (see API Keys)
  codefreshToken: ""
  # -- User token that references an existing secret containing API key (required if `global.codefreshToken` is omitted!)
  codefreshTokenSecretKeyRef: {}
  # E.g.
  # codefreshTokenSecretKeyRef:
  #   name: my-codefresh-api-token
  #   key: codefresh-api-token

  # -- Account ID (required!)
  # Can be obtained here https://g.codefresh.io/2.0/account-settings/account-information
  accountId: ""

  # -- K8s context name (required!)
  context: ""
  # E.g.
  # context: prod-ue1-runtime-1

  # -- Agent Name (optional!)
  # If omitted, the following format will be used '{{ .Values.global.context }}_{{ .Release.Namespace }}'
  agentName: ""
  # E.g.
  # agentName: prod-ue1-runtime-1

  # -- Runtime name (optional!)
  # If omitted, the following format will be used '{{ .Values.global.context }}/{{ .Release.Namespace }}'
  runtimeName: ""
  # E.g.
  # runtimeName: prod-ue1-runtime-1/namespace
```

- Install chart

```console
helm upgrade --install cf-runtime cf-runtime/cf-runtime --create-namespace --namespace codefresh
```

*Install from OCI-based registry*
```console
helm upgrade --install cf-runtime oci://quay.io/codefresh/cf-runtime --create-namespace --namespace codefresh
```

## Chart Configuration

See [Customizing the Chart Before Installing](https://helm.sh/docs/intro/using_helm/#customizing-the-chart-before-installing). To see all configurable options with detailed comments, visit the chart's [values.yaml](./values.yaml), or run these configuration commands:

```console
helm show values cf-runtime/cf-runtime
```

## Upgrade Chart

### To 2.x

This major release renames and deprecated several values in the chart. Most of the workload templates have been refactored.

Affected values:
- `dockerRegistry` is deprecated. Replaced with `global.imageRegistry`
- `re` is renamed to `runtime`
- `storage.localVolumeMonitor` is replaced with `volumeProvisioner.dind-lv-monitor`
- `volumeProvisioner.volume-cleanup` is replaced with `volumeProvisioner.dind-volume-cleanup`
- `image` values structure has been updated. Split to `image.registry` `image.repository` `image.tag`
- pod's `annotations` is renamed to `podAnnotations`

### To 3.x

⚠️⚠️⚠️
### READ this before the upgrade!

This major release adds [runtime-environment](https://codefresh.io/docs/docs/installation/codefresh-runner/#runtime-environment-specification) spec into chart templates.
That means it is possible to set parametes for `dind` and `engine` pods via [values.yaml](./values.yaml).

**If you had any overrides (i.e. tolerations/nodeSelector/environment variables/etc) added in runtime spec via [codefresh CLI](https://codefresh-io.github.io/cli/) (for example, you did use [get](https://codefresh-io.github.io/cli/runtime-environments/get-runtime-environments/) and [patch](https://codefresh-io.github.io/cli/runtime-environments/apply-runtime-environments/) commands to modify the runtime-environment), you MUST add these into chart's [values.yaml](./values.yaml) for `.Values.runtime.dind` or(and) .`Values.runtime.engine`**

**For backward compatibility, you can disable updating runtime-environment spec via** `.Values.runtime.patch.enabled=false`

Affected values:
- added **mandatory** `global.codefreshToken`/`global.codefreshTokenSecretKeyRef` **You must specify it before the upgrade!**
- `runtime.engine` is added
- `runtime.dind` is added
- `global.existingAgentToken` is replaced with `global.agentTokenSecretKeyRef`
- `global.existingDindCertsSecret` is replaced with `global.dindCertsSecretRef`

### To 4.x

This major release adds **agentless inCluster** runtime mode (relevant only for [Codefresh On-Premises](#on-premise) users)

Affected values:
- `runtime.agent` / `runtime.inCluster` / `runtime.accounts` / `runtime.description` are added

### To 5.x

This major release converts `.runtime.dind.pvcs` from **list** to **dict**

> 4.x chart's values example:
```yaml
runtime:
  dind:
    pvcs:
      - name: dind
        storageClassName: my-storage-class-name
        volumeSize: 32Gi
        reuseVolumeSelector: 'codefresh-app,io.codefresh.accountName'
        reuseVolumeSortOrder: pipeline_id
```

> 5.x chart's values example:
```yaml
runtime:
  dind:
    pvcs:
      dind:
        name: dind
        storageClassName: my-storage-class-name
        volumeSize: 32Gi
        reuseVolumeSelector: 'codefresh-app,io.codefresh.accountName'
        reuseVolumeSortOrder: pipeline_id
```

Affected values:
- `.runtime.dind.pvcs` converted from **list** to **dict**

### To 6.x

⚠️⚠️⚠️
### READ this before the upgrade!

This major release deprecates previously required `codefresh runner init --generate-helm-values-file`.

Affected values:
- **Replaced** `.monitor.clusterId` with `.global.context` as **mandatory** value!
- **Deprecated** `.global.agentToken` / `.global.agentTokenSecretKeyRef`
- **Removed** `.global.agentId`
- **Removed** `.global.keys` / `.global.dindCertsSecretRef`
- **Removed** `.global.existingAgentToken` / `existingDindCertsSecret`
- **Removed** `.monitor.clusterId` / `.monitor.token` / `.monitor.existingMonitorToken`

#### Migrate the Helm chart from version 5.x to 6.x

Given this is the legacy `generated_values.yaml` values:

> legacy `generated_values.yaml`
```yaml
{
    "appProxy": {
        "enabled": false,
    },
    "monitor": {
        "enabled": false,
        "clusterId": "my-cluster-name",
        "token": "1234567890"
    },
    "global": {
        "namespace": "namespace",
        "codefreshHost": "https://g.codefresh.io",
        "agentToken": "0987654321",
        "agentId": "agent-id-here",
        "agentName": "my-cluster-name_my-namespace",
        "accountId": "my-account-id",
        "runtimeName": "my-cluster-name/my-namespace",
        "codefreshToken": "1234567890",
        "keys": {
            "key": "-----BEGIN RSA PRIVATE KEY-----...",
            "csr": "-----BEGIN CERTIFICATE REQUEST-----...",
            "ca": "-----BEGIN CERTIFICATE-----...",
            "serverCert": "-----BEGIN CERTIFICATE-----..."
        }
    }
}
```

Update `values.yaml` for new chart version:

> For existing installation for backward compatibility `.Values.global.agentToken/agentTokenSecretKeyRef` **must be provided!** For installation from scratch this value is no longer required.

> updated `values.yaml`
```yaml
global:
  codefreshToken: "1234567890"
  accountId: "my-account-id"
  context: "my-cluster-name"
  agentToken: "0987654321"  # MANDATORY when migrating from < 6.x chart version !
  agentName: "my-cluster-name_my-namespace" # optional
  runtimeName: "my-cluster-name/my-namespace" # optional
```

> **Note!** Though it's still possible to update runtime-environment via [get](https://codefresh-io.github.io/cli/runtime-environments/get-runtime-environments/) and [patch](https://codefresh-io.github.io/cli/runtime-environments/apply-runtime-environments/) commands, it's recommended to enable sidecar container to pull runtime spec from Codefresh API to detect any drift in configuration.

```yaml
runner:
  # -- Sidecar container
  # Reconciles runtime spec from Codefresh API for drift detection
  sidecar:
    enabled: true
```

## Architecture

[Codefresh Runner architecture](https://codefresh.io/docs/docs/installation/codefresh-runner/#codefresh-runner-architecture)

## Configuration

See [Customizing the Chart Before Installing](https://helm.sh/docs/intro/using_helm/#customizing-the-chart-before-installing). To see all configurable options with detailed comments, visit the chart's [values.yaml](./values.yaml), or run these configuration commands:

```console
helm show values cf-runtime/cf-runtime
```

### EBS backend volume configuration

`dind-volume-provisioner` should have permissions to create/attach/detach/delete/get EBS volumes

Minimal IAM policy for `dind-volume-provisioner`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AttachVolume",
        "ec2:CreateSnapshot",
        "ec2:CreateTags",
        "ec2:CreateVolume",
        "ec2:DeleteSnapshot",
        "ec2:DeleteTags",
        "ec2:DeleteVolume",
        "ec2:DescribeInstances",
        "ec2:DescribeSnapshots",
        "ec2:DescribeTags",
        "ec2:DescribeVolumes",
        "ec2:DetachVolume"
      ],
      "Resource": "*"
    }
  ]
}
```

There are three options:

1. Run `dind-volume-provisioner` pod on the node/node-group with IAM role

```yaml
storage:
  # -- Set backend volume type (`local`/`ebs`/`ebs-csi`/`gcedisk`/`azuredisk`)
  backend: ebs-csi

  ebs:
    availabilityZone: "us-east-1a"

volumeProvisioner:
  # -- Set node selector
  nodeSelector: {}
  # -- Set tolerations
  tolerations: []
```

2. Pass static credentials in `.Values.storage.ebs.accessKeyId/accessKeyIdSecretKeyRef` and `.Values.storage.ebs.secretAccessKey/secretAccessKeySecretKeyRef`

```yaml
storage:
  # -- Set backend volume type (`local`/`ebs`/`ebs-csi`/`gcedisk`/`azuredisk`)
  backend: ebs-csi

  ebs:
    availabilityZone: "us-east-1a"

    # -- Set AWS_ACCESS_KEY_ID for volume-provisioner (optional)
    accessKeyId: ""
    # -- Existing secret containing AWS_ACCESS_KEY_ID.
    accessKeyIdSecretKeyRef: {}
    # E.g.
    # accessKeyIdSecretKeyRef:
    #   name:
    #   key:

    # -- Set AWS_SECRET_ACCESS_KEY for volume-provisioner (optional)
    secretAccessKey: ""
    # -- Existing secret containing AWS_SECRET_ACCESS_KEY
    secretAccessKeySecretKeyRef: {}
    # E.g.
    # secretAccessKeySecretKeyRef:
    #   name:
    #   key:
```

3. Assign IAM role to `dind-volume-provisioner` service account

```yaml
storage:
  # -- Set backend volume type (`local`/`ebs`/`ebs-csi`/`gcedisk`/`azuredisk`)
  backend: ebs-csi

  ebs:
    availabilityZone: "us-east-1a"

volumeProvisioner:
  # -- Service Account parameters
  serviceAccount:
    # -- Create service account
    create: true
    # -- Additional service account annotations
    annotations:
      eks.amazonaws.com/role-arn: "arn:aws:iam::<ACCOUNT_ID>:role/<IAM_ROLE_NAME>"
```

### Custom volume mounts

You can add your own volumes and volume mounts in the runtime environment, so that all pipeline steps will have access to the same set of external files.

```yaml
runtime:
  dind:
    userVolumes:
      regctl-docker-registry:
        name: regctl-docker-registry
        secret:
          items:
            - key: .dockerconfigjson
              path: config.json
          secretName: regctl-docker-registry
          optional: true
    userVolumeMounts:
      regctl-docker-registry:
        name: regctl-docker-registry
        mountPath: /home/appuser/.docker/
        readOnly: true

```

### Custom global environment variables

You can add your own environment variables to the runtime environment. All pipeline steps have access to the global variables.

```yaml
runtime:
  engine:
    userEnvVars:
    - name: GITHUB_TOKEN
      valueFrom:
        secretKeyRef:
          name: github-token
          key: token
```

### Volume reuse policy

Volume reuse behavior depends on the configuration for `reuseVolumeSelector` in the runtime environment spec.

```yaml
runtime:
  dind:
    pvcs:
      - name: dind
        ...
        reuseVolumeSelector: 'codefresh-app,io.codefresh.accountName'
        reuseVolumeSortOrder: pipeline_id
```

The following options are available:
- `reuseVolumeSelector: 'codefresh-app,io.codefresh.accountName'` - PV can be used by ANY pipeline in the specified account (default).
Benefit: Fewer PVs, resulting in lower costs. Since any PV can be used by any pipeline, the cluster needs to maintain/reserve fewer PVs in its PV pool for Codefresh.
Downside: Since the PV can be used by any pipeline, the PVs could have assets and info from different pipelines, reducing the probability of cache.

- `reuseVolumeSelector: 'codefresh-app,io.codefresh.accountName,project_id'` - PV can be used by ALL pipelines in your account, assigned to the same project.

- `reuseVolumeSelector: 'codefresh-app,io.codefresh.accountName,pipeline_id'` - PV can be used only by a single pipeline.
Benefit: More probability of cache without “spam” from other pipelines.
Downside: More PVs to maintain and therefore higher costs.

- `reuseVolumeSelector: 'codefresh-app,io.codefresh.accountName,pipeline_id,io.codefresh.branch_name'` - PV can be used only by single pipeline AND single branch.

- `reuseVolumeSelector: 'codefresh-app,io.codefresh.accountName,pipeline_id,trigger'` - PV can be used only by single pipeline AND single trigger.

### Volume cleaners

Codefresh pipelines require disk space for:
  * [Pipeline Shared Volume](https://codefresh.io/docs/docs/pipelines/introduction-to-codefresh-pipelines/#sharing-the-workspace-between-build-steps) (`/codefresh/volume`, implemented as [docker volume](https://docs.docker.com/storage/volumes/))
  * Docker containers, both running and stopped
  * Docker images and cached layers

Codefresh offers two options to manage disk space and prevent out-of-space errors:
* Use runtime cleaners on Docker images and volumes
* [Set the minimum disk space per pipeline build volume](https://codefresh.io/docs/docs/pipelines/pipelines/#set-minimum-disk-space-for-a-pipeline-build)

To improve performance by using Docker cache, Codefresh `volume-provisioner` can provision previously used disks with Docker images and pipeline volumes from previously run builds.

### Types of runtime volume cleaners

Docker images and volumes must be cleaned on a regular basis.

* [IN-DIND cleaner](https://github.com/codefresh-io/dind/tree/master/cleaner): Deletes extra Docker containers, volumes, and images in **DIND pod**.
* [External volume cleaner](https://github.com/codefresh-io/dind-volume-cleanup): Deletes unused **external** PVs (EBS, GCE/Azure disks).
* [Local volume cleaner](https://github.com/codefresh-io/dind-volume-utils/blob/master/local-volumes/lv-cleaner.sh): Deletes **local** volumes if node disk space is close to the threshold.

### IN-DIND cleaner

**Purpose:** Removes unneeded *docker containers, images, volumes* inside Kubernetes volume mounted on the DIND pod

**How it runs:** Inside each DIND pod as script

**Triggered by:** SIGTERM and also during the run when disk usage > 90% (configurable)

**Configured by:**  Environment Variables which can be set in Runtime Environment spec

**Configuration/Logic:** [README.md](https://github.com/codefresh-io/dind/tree/master/cleaner#readme)

Override `.Values.runtime.dind.env` if necessary (the following are **defaults**):

```yaml
runtime:
  dind:
    env:
      CLEAN_PERIOD_SECONDS: '21600' # launch clean if last clean was more than CLEAN_PERIOD_SECONDS seconds ago
      CLEAN_PERIOD_BUILDS: '5' # launch clean if last clean was more CLEAN_PERIOD_BUILDS builds since last build
      IMAGE_RETAIN_PERIOD: '14400' # do not delete docker images if they have events since current_timestamp - IMAGE_RETAIN_PERIOD
      VOLUMES_RETAIN_PERIOD: '14400' # do not delete docker volumes if they have events since current_timestamp - VOLUMES_RETAIN_PERIOD
      DISK_USAGE_THRESHOLD: '0.8' # launch clean based on current disk usage DISK_USAGE_THRESHOLD
      INODES_USAGE_THRESHOLD: '0.8' # launch clean based on current inodes usage INODES_USAGE_THRESHOLD
```

### External volumes cleaner

**Purpose:** Removes unused *kubernetes volumes and related backend volumes*

**How it runs:** Runs as `dind-volume-cleanup` CronJob. Installed in case the Runner uses non-local volumes `.Values.storage.backend != local`

**Triggered by:** CronJob every 10min (configurable)

**Configuration:**

Set `codefresh.io/volume-retention` for dinds' PVCs:

```yaml
runtime:
  dind:
    pvcs:
      - name: dind
        ...
        annotations:
          codefresh.io/volume-retention: 7d
```

Or override environment variables for `dind-volume-cleanup` cronjob:

```yaml
volumeProvisioner:
  dind-volume-cleanup:
    env:
      RETENTION_DAYS: 7   # clean volumes that were last used more than `RETENTION_DAYS` (default is 4) ago
```

### Local volumes cleaner

**Purpose:** Deletes local volumes when node disk space is close to the threshold

**How it runs:** Runs as `dind-lv-monitor` DaemonSet. Installed in case the Runner uses local volumes `.Values.storage.backend == local`

**Triggered by:** Disk space usage or inode usage that exceeds thresholds (configurable)

**Configuration:**

Override environment variables for `dind-lv-monitor` daemonset:

```yaml
volumeProvisioner:
  dind-lv-monitor:
    env:
      KB_USAGE_THRESHOLD: 60  # default 80 (percentage)
      INODE_USAGE_THRESHOLD: 60  # default 80
```

### Rootless DinD

DinD pod runs a `priviliged` container with **rootfull** docker.
To run the docker daemon as non-root user (**rootless** mode), change dind image tag:

`values.yaml`
```yaml
runtime:
  dind:
    image:
      tag: rootless
```

### ARM

With the Codefresh Runner, you can run native ARM64v8 builds.

> **Note!**
> You cannot run both amd64 and arm64 images within the same pipeline. As one pipeline can map only to one runtime, you can run either amd64 or arm64 within the same pipeline.

Provide `nodeSelector` and(or) `tolerations` for dind pods:

`values.yaml`
```yaml
runtime:
  dind:
    nodeSelector:
      arch: arm64
    tolerations:
    - key: arch
      operator: Equal
      value: arm64
      effect: NoSchedule
```

### Openshift

To install Codefresh Runner on OpenShift use the following `values.yaml` example

```yaml
runner:
  podSecurityContext:
    enabled: false

volumeProvisioner:
  podSecurityContext:
    enabled: false
  env:
    PRIVILEGED_CONTAINER: true
  dind-lv-monitor:
    containerSecurityContext:
      enabled: true
      privileged: true
    volumePermissions:
      enabled: true
      securityContext:
        privileged: true
        runAsUser: auto
```

Grant `privileged` SCC to `cf-runtime-runner` and `cf-runtime-volume-provisioner` service accounts.

```console
oc adm policy add-scc-to-user privileged system:serviceaccount:codefresh:cf-runtime-runner

oc adm policy add-scc-to-user privileged system:serviceaccount:codefresh:cf-runtime-volume-provisioner
```

### On-premise

If you have [Codefresh On-Premises](https://artifacthub.io/packages/helm/codefresh-onprem/codefresh) deployed, you can install Codefresh Runner in **agentless** mode.

**What is agentless mode?**

Agent (aka venona) is Runner component which responsible for calling Codefresh API to run builds and create dind/engine pods and pvc objects. Agent can only be assigned to a single account, thus you can't share one runtime across multiple accounts. However, with **agentless** mode it's possible to register the runtime as **system**-type runtime so it's registered on the platform level and can be assigned/shared across multiple accounts.

**What are the prerequisites?**
- You have a running [Codefresh On-Premises](https://artifacthub.io/packages/helm/codefresh-onprem/codefresh) control-plane environment
- You have a Codefresh API token with platform **Admin** permissions scope

### How to deploy agentless runtime when it's on the SAME k8s cluster as On-Premises control-plane environment?

- Enable cluster-level permissions for cf-api (On-Premises control-plane component)

> `values.yaml` for [Codefresh On-Premises](https://artifacthub.io/packages/helm/codefresh-onprem/codefresh) Helm chart
```yaml
cfapi:
  ...
  # -- Enable ClusterRole/ClusterRoleBinding
  rbac:
    namespaced: false
```

- Set the following values for Runner Helm chart

`.Values.global.codefreshHost=...` \
`.Values.global.codefreshToken=...` \
`.Values.global.runtimeName=system/...` \
`.Values.runtime.agent=false` \
`.Values.runtime.inCluster=true`

> `values.yaml` for [Codefresh Runner](https://artifacthub.io/packages/helm/codefresh-runner/cf-runtime) helm chart
```yaml
global:
  # -- URL of Codefresh On-Premises Platform
  codefreshHost: "https://myonprem.somedomain.com"
  # -- User token in plain text with Admin permission scope
  codefreshToken: ""
  # -- User token that references an existing secret containing API key.
  codefreshTokenSecretKeyRef: {}
  # E.g.
  # codefreshTokenSecretKeyRef:
  #   name: my-codefresh-api-token
  #   key: codefresh-api-token

  # -- Distinguished runtime name
  # (for On-Premise only; mandatory!) Must be prefixed with "system/..."
  runtimeName: "system/prod-ue1-some-cluster-name"

# -- Set runtime parameters
runtime:
  # -- (for On-Premise only; mandatory!) Disable agent
  agent: false
  # -- (for On-Premise only; optional) Set inCluster runtime (default: `true`)
  # `inCluster=true` flag is set when Runtime and On-Premises control-plane are run on the same cluster
  # `inCluster=false` flag is set when Runtime and On-Premises control-plane are on different clusters
  inCluster: true
  # -- (for On-Premise only; optional) Assign accounts to runtime (list of account ids; default is empty)
  # Accounts can be assigned to the runtime in Codefresh UI later so you can kepp it empty.
  accounts: []
  # -- Set parent runtime to inherit.
  runtimeExtends: []
```

- Install the chart

```console
helm upgrade --install cf-runtime oci://quay.io/codefresh/cf-runtime -f values.yaml --create-namespace --namespace cf-runtime
```

- Verify the runtime and run test pipeline

Go to [https://<YOUR_ONPREM_DOMAIN_HERE>/admin/runtime-environments/system](https://<YOUR_ONPREM_DOMAIN_HERE>/admin/runtime-environments/system) to check the runtime. Assign it to the required account(s). Run test pipeline on it.

### How to deploy agentless runtime when it's on the DIFFERENT k8s cluster than On-Premises control-plane environment?

In this case, it's required to mount runtime cluster's `KUBECONFIG` into On-Premises `cf-api` deployment

- Create the neccessary RBAC resources

> `values.yaml` for [Codefresh Runner](https://artifacthub.io/packages/helm/codefresh-runner/cf-runtime) helm chart
```yaml
extraResources:
- apiVersion: rbac.authorization.k8s.io/v1
  kind: Role
  metadata:
    name: codefresh-role
    namespace: '{{ .Release.Namespace }}'
  rules:
    - apiGroups: [""]
      resources: ["pods", "persistentvolumeclaims", "persistentvolumes"]
      verbs: ["list", "watch", "get", "create", "patch", "delete"]
    - apiGroups: ["snapshot.storage.k8s.io"]
      resources: ["volumesnapshots"]
      verbs: ["list", "watch", "get", "create", "patch", "delete"]
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: codefresh-runtime-user
    namespace: '{{ .Release.Namespace }}'
- apiVersion: rbac.authorization.k8s.io/v1
  kind: RoleBinding
  metadata:
    name: codefresh-runtime-user
    namespace: '{{ .Release.Namespace }}'
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: codefresh-role
  subjects:
  - kind: ServiceAccount
    name: codefresh-runtime-user
    namespace: '{{ .Release.Namespace }}'
- apiVersion: v1
  kind: Secret
  metadata:
    name: codefresh-runtime-user-token
    namespace: '{{ .Release.Namespace }}'
    annotations:
      kubernetes.io/service-account.name: codefresh-runtime-user
  type: kubernetes.io/service-account-token
```

- Set up the following environment variables to create a `KUBECONFIG` file

```shell
NAMESPACE=cf-runtime
CLUSTER_NAME=prod-ue1-some-cluster-name
CURRENT_CONTEXT=$(kubectl config current-context)

USER_TOKEN_VALUE=$(kubectl -n cf-runtime get secret/codefresh-runtime-user-token -o=go-template='{{.data.token}}' | base64 --decode)
CURRENT_CLUSTER=$(kubectl config view --raw -o=go-template='{{range .contexts}}{{if eq .name "'''${CURRENT_CONTEXT}'''"}}{{ index .context "cluster" }}{{end}}{{end}}')
CLUSTER_CA=$(kubectl config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}"{{with index .cluster "certificate-authority-data" }}{{.}}{{end}}"{{ end }}{{ end }}')
CLUSTER_SERVER=$(kubectl config view --raw -o=go-template='{{range .clusters}}{{if eq .name "'''${CURRENT_CLUSTER}'''"}}{{ .cluster.server }}{{end}}{{ end }}')

export -p USER_TOKEN_VALUE CURRENT_CONTEXT CURRENT_CLUSTER CLUSTER_CA CLUSTER_SERVER CLUSTER_NAME
```

- Create a kubeconfig file

```console
cat << EOF > $CLUSTER_NAME-kubeconfig
apiVersion: v1
kind: Config
current-context: ${CLUSTER_NAME}
contexts:
- name: ${CLUSTER_NAME}
  context:
    cluster: ${CLUSTER_NAME}
    user: codefresh-runtime-user
    namespace: ${NAMESPACE}
clusters:
- name: ${CLUSTER_NAME}
  cluster:
    certificate-authority-data: ${CLUSTER_CA}
    server: ${CLUSTER_SERVER}
users:
- name: ${CLUSTER_NAME}
  user:
    token: ${USER_TOKEN_VALUE}
EOF
```

- **Switch context to On-Premises control-plane cluster**. Create k8s secret (via any tool like [ESO](https://external-secrets.io/v0.4.4/), `kubectl`, etc ) containing runtime cluster's `KUBECONFG` created in previous step.

```shell
NAMESPACE=codefresh
kubectl create secret generic dind-runtime-clusters --from-file=$CLUSTER_NAME=$CLUSTER_NAME-kubeconfig -n $NAMESPACE
```

- Mount secret containing runtime cluster's `KUBECONFG` into cf-api in On-Premises control-plane cluster

> `values.yaml` for [Codefresh On-Premises](https://artifacthub.io/packages/helm/codefresh-onprem/codefresh) helm chart
```yaml
cf-api:
  ...
  volumes:
    dind-clusters:
      enabled: true
      type: secret
      nameOverride: dind-runtime-clusters
      optional: true
```
> volumeMount `/etc/kubeconfig` is already configured in cf-api Helm chart template. No need to specify it.

- Set the following values for Runner helm chart

> `values.yaml` for [Codefresh Runner](https://artifacthub.io/packages/helm/codefresh-runner/cf-runtime) helm chart

`.Values.global.codefreshHost=...` \
`.Values.global.codefreshToken=...` \
`.Values.global.runtimeName=system/...` \
`.Values.runtime.agent=false` \
`.Values.runtime.inCluster=false`

**Important!**
`.Values.global.name` ("system/" prefix is ignored!) should match the cluster name (key in `dind-runtime-clusters` secret created previously)
```yaml
global:
  # -- URL of Codefresh On-Premises Platform
  codefreshHost: "https://myonprem.somedomain.com"
  # -- User token in plain text with Admin permission scope
  codefreshToken: ""
  # -- User token that references an existing secret containing API key.
  codefreshTokenSecretKeyRef: {}
  # E.g.
  # codefreshTokenSecretKeyRef:
  #   name: my-codefresh-api-token
  #   key: codefresh-api-token

  # -- Distinguished runtime name
  # (for On-Premise only; mandatory!) Must be prefixed with "system/..."
  name: "system/prod-ue1-some-cluster-name"

# -- Set runtime parameters
runtime:
  # -- (for On-Premise only; mandatory!) Disable agent
  agent: false
  # -- (for On-Premise only; optional) Set inCluster runtime (default: `true`)
  # `inCluster=true` flag is set when Runtime and On-Premises control-plane are run on the same cluster
  # `inCluster=false` flag is set when Runtime and On-Premises control-plane are on different clusters
  inCluster: false
  # -- (for On-Premise only; optional) Assign accounts to runtime (list of account ids; default is empty)
  # Accounts can be assigned to the runtime in Codefresh UI later so you can kepp it empty.
  accounts: []
  # -- (optional) Set parent runtime to inherit.
  runtimeExtends: []
```

- Install the chart

```console
helm upgrade --install cf-runtime oci://quay.io/codefresh/cf-runtime -f values.yaml --create-namespace --namespace cf-runtime
```

- Verify the runtime and run test pipeline

Go to [https://<YOUR_ONPREM_DOMAIN_HERE>/admin/runtime-environments/system](https://<YOUR_ONPREM_DOMAIN_HERE>/admin/runtime-environments/system) to see the runtime. Assign it to the required account(s).

## Requirements

| Repository | Name | Version |
|------------|------|---------|
| https://chartmuseum.codefresh.io/cf-common | cf-common | 0.13.0 |

## Values

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| appProxy.affinity | object | `{}` | Set affinity |
| appProxy.enabled | bool | `false` | Enable app-proxy |
| appProxy.env | object | `{}` | Add additional env vars |
| appProxy.image | object | `{"registry":"quay.io","repository":"codefresh/cf-app-proxy","tag":"latest"}` | Set image |
| appProxy.ingress.annotations | object | `{}` | Set extra annotations for ingress object |
| appProxy.ingress.class | string | `""` | Set ingress class |
| appProxy.ingress.host | string | `""` | Set DNS hostname the ingress will use |
| appProxy.ingress.pathPrefix | string | `"/"` | Set path prefix for ingress |
| appProxy.ingress.tlsSecret | string | `""` | Set k8s tls secret for the ingress object |
| appProxy.nodeSelector | object | `{}` | Set node selector |
| appProxy.podAnnotations | object | `{}` | Set pod annotations |
| appProxy.podSecurityContext | object | `{}` | Set security context for the pod |
| appProxy.rbac | object | `{"create":true,"namespaced":true,"rules":[]}` | RBAC parameters |
| appProxy.rbac.create | bool | `true` | Create RBAC resources |
| appProxy.rbac.namespaced | bool | `true` | Use Role(true)/ClusterRole(true) |
| appProxy.rbac.rules | list | `[]` | Add custom rule to the role |
| appProxy.readinessProbe | object | See below | Readiness probe configuration |
| appProxy.replicasCount | int | `1` | Set number of pods |
| appProxy.resources | object | `{}` | Set requests and limits |
| appProxy.serviceAccount | object | `{"annotations":{},"create":true,"name":"","namespaced":true}` | Service Account parameters |
| appProxy.serviceAccount.annotations | object | `{}` | Additional service account annotations |
| appProxy.serviceAccount.create | bool | `true` | Create service account |
| appProxy.serviceAccount.name | string | `""` | Override service account name |
| appProxy.serviceAccount.namespaced | bool | `true` | Use Role(true)/ClusterRole(true) |
| appProxy.tolerations | list | `[]` | Set tolerations |
| appProxy.updateStrategy | object | `{"type":"RollingUpdate"}` | Upgrade strategy |
| dockerRegistry | string | `""` |  |
| event-exporter | object | See below | Event exporter parameters |
| event-exporter.affinity | object | `{}` | Set affinity |
| event-exporter.enabled | bool | `false` | Enable event-exporter |
| event-exporter.env | object | `{}` | Add additional env vars |
| event-exporter.image | object | `{"registry":"docker.io","repository":"codefresh/k8s-event-exporter","tag":"latest"}` | Set image |
| event-exporter.nodeSelector | object | `{}` | Set node selector |
| event-exporter.podAnnotations | object | `{}` | Set pod annotations |
| event-exporter.podSecurityContext | object | See below | Set security context for the pod |
| event-exporter.rbac | object | `{"create":true,"rules":[]}` | RBAC parameters |
| event-exporter.rbac.create | bool | `true` | Create RBAC resources |
| event-exporter.rbac.rules | list | `[]` | Add custom rule to the role |
| event-exporter.replicasCount | int | `1` | Set number of pods |
| event-exporter.resources | object | `{}` | Set resources |
| event-exporter.serviceAccount | object | `{"annotations":{},"create":true,"name":""}` | Service Account parameters |
| event-exporter.serviceAccount.annotations | object | `{}` | Additional service account annotations |
| event-exporter.serviceAccount.create | bool | `true` | Create service account |
| event-exporter.serviceAccount.name | string | `""` | Override service account name |
| event-exporter.tolerations | list | `[]` | Set tolerations |
| event-exporter.updateStrategy | object | `{"type":"Recreate"}` | Upgrade strategy |
| extraResources | list | `[]` | Array of extra objects to deploy with the release |
| fullNameOverride | string | `""` | String to fully override cf-runtime.fullname template |
| global | object | See below | Global parameters |
| global.accountId | string | `""` | Account ID (required!) Can be obtained here https://g.codefresh.io/2.0/account-settings/account-information |
| global.agentName | string | `""` | Agent Name (optional!) If omitted, the following format will be used `{{ .Values.global.context }}_{{ .Release.Namespace }}` |
| global.agentToken | string | `""` | DEPRECATED Agent token in plain text. !!! MUST BE provided if migrating from < 6.x chart version |
| global.agentTokenSecretKeyRef | object | `{}` | DEPRECATED Agent token that references an existing secret containing API key. !!! MUST BE provided if migrating from < 6.x chart version |
| global.codefreshHost | string | `"https://g.codefresh.io"` | URL of Codefresh Platform (required!) |
| global.codefreshToken | string | `""` | User token in plain text (required if `global.codefreshTokenSecretKeyRef` is omitted!) Ref: https://g.codefresh.io/user/settings (see API Keys) |
| global.codefreshTokenSecretKeyRef | object | `{}` | User token that references an existing secret containing API key (required if `global.codefreshToken` is omitted!) |
| global.context | string | `""` | K8s context name (required!) |
| global.imagePullSecrets | list | `[]` | Global Docker registry secret names as array |
| global.imageRegistry | string | `""` | Global Docker image registry |
| global.runtimeName | string | `""` | Runtime name (optional!) If omitted, the following format will be used `{{ .Values.global.context }}/{{ .Release.Namespace }}` |
| monitor.affinity | object | `{}` | Set affinity |
| monitor.enabled | bool | `false` | Enable monitor Ref: https://codefresh.io/docs/docs/installation/codefresh-runner/#install-monitoring-component |
| monitor.env | object | `{}` | Add additional env vars |
| monitor.image | object | `{"registry":"quay.io","repository":"codefresh/agent","tag":"stable"}` | Set image |
| monitor.nodeSelector | object | `{}` | Set node selector |
| monitor.podAnnotations | object | `{}` | Set pod annotations |
| monitor.podSecurityContext | object | `{}` |  |
| monitor.rbac | object | `{"create":true,"namespaced":true,"rules":[]}` | RBAC parameters |
| monitor.rbac.create | bool | `true` | Create RBAC resources |
| monitor.rbac.namespaced | bool | `true` | Use Role(true)/ClusterRole(true) |
| monitor.rbac.rules | list | `[]` | Add custom rule to the role |
| monitor.readinessProbe | object | See below | Readiness probe configuration |
| monitor.replicasCount | int | `1` | Set number of pods |
| monitor.resources | object | `{}` | Set resources |
| monitor.serviceAccount | object | `{"annotations":{},"create":true,"name":""}` | Service Account parameters |
| monitor.serviceAccount.annotations | object | `{}` | Additional service account annotations |
| monitor.serviceAccount.create | bool | `true` | Create service account |
| monitor.serviceAccount.name | string | `""` | Override service account name |
| monitor.tolerations | list | `[]` | Set tolerations |
| monitor.updateStrategy | object | `{"type":"RollingUpdate"}` | Upgrade strategy |
| nameOverride | string | `""` | String to partially override cf-runtime.fullname template (will maintain the release name) |
| podMonitor | object | See below | Add podMonitor (for engine pods) |
| podMonitor.main.enabled | bool | `false` | Enable pod monitor for engine pods |
| podMonitor.runner.enabled | bool | `false` | Enable pod monitor for runner pod |
| re | object | `{}` |  |
| runner | object | See below | Runner parameters |
| runner.affinity | object | `{}` | Set affinity |
| runner.enabled | bool | `true` | Enable the runner |
| runner.env | object | `{}` | Add additional env vars |
| runner.image | object | `{"registry":"quay.io","repository":"codefresh/venona","tag":"1.10.1"}` | Set image |
| runner.init | object | `{"image":{"registry":"quay.io","repository":"codefresh/cli","tag":"0.85.0-rootless"},"resources":{"limits":{"cpu":"1","memory":"512Mi"},"requests":{"cpu":"0.2","memory":"256Mi"}}}` | Init container |
| runner.nodeSelector | object | `{}` | Set node selector |
| runner.podAnnotations | object | `{}` | Set pod annotations |
| runner.podSecurityContext | object | See below | Set security context for the pod |
| runner.rbac | object | `{"create":true,"rules":[]}` | RBAC parameters |
| runner.rbac.create | bool | `true` | Create RBAC resources |
| runner.rbac.rules | list | `[]` | Add custom rule to the role |
| runner.readinessProbe | object | See below | Readiness probe configuration |
| runner.replicasCount | int | `1` | Set number of pods |
| runner.resources | object | `{}` | Set requests and limits |
| runner.serviceAccount | object | `{"annotations":{},"create":true,"name":""}` | Service Account parameters |
| runner.serviceAccount.annotations | object | `{}` | Additional service account annotations |
| runner.serviceAccount.create | bool | `true` | Create service account |
| runner.serviceAccount.name | string | `""` | Override service account name |
| runner.sidecar | object | `{"enabled":false,"env":{"RECONCILE_INTERVAL":300},"image":{"registry":"quay.io","repository":"codefresh/codefresh-shell","tag":"0.0.2"},"resources":{}}` | Sidecar container Reconciles runtime spec from Codefresh API for drift detection |
| runner.tolerations | list | `[]` | Set tolerations |
| runner.updateStrategy | object | `{"type":"RollingUpdate"}` | Upgrade strategy |
| runtime | object | See below | Set runtime parameters |
| runtime.accounts | list | `[]` | (for On-Premise only) Assign accounts to runtime (list of account ids) |
| runtime.agent | bool | `true` | (for On-Premise only) Enable agent |
| runtime.description | string | `""` | Runtime description |
| runtime.dind | object | `{"affinity":{},"env":{},"image":{"registry":"quay.io","repository":"codefresh/dind","tag":"20.10.24-1.27.0"},"nodeSelector":{},"podAnnotations":{},"pvcs":{"dind":{"name":"dind","reuseVolumeSelector":"codefresh-app,io.codefresh.accountName","reuseVolumeSortOrder":"pipeline_id","storageClassName":"{{ include \"dind-volume-provisioner.storageClassName\" . }}","volumeSize":"16Gi"}},"resources":{"limits":{"cpu":"400m","memory":"800Mi"},"requests":null},"schedulerName":"","serviceAccount":"codefresh-engine","tolerations":[],"userAccess":true,"userVolumeMounts":{},"userVolumes":{}}` | Parameters for DinD (docker-in-docker) pod (aka "runtime" pod). |
| runtime.dind.affinity | object | `{}` | Set affinity |
| runtime.dind.env | object | `{}` | Set additional env vars. |
| runtime.dind.image | object | `{"registry":"quay.io","repository":"codefresh/dind","tag":"20.10.24-1.27.0"}` | Set dind image. |
| runtime.dind.nodeSelector | object | `{}` | Set node selector. |
| runtime.dind.podAnnotations | object | `{}` | Set pod annotations. |
| runtime.dind.pvcs | object | `{"dind":{"name":"dind","reuseVolumeSelector":"codefresh-app,io.codefresh.accountName","reuseVolumeSortOrder":"pipeline_id","storageClassName":"{{ include \"dind-volume-provisioner.storageClassName\" . }}","volumeSize":"16Gi"}}` | PV claim spec parametes. |
| runtime.dind.pvcs.dind | object | `{"name":"dind","reuseVolumeSelector":"codefresh-app,io.codefresh.accountName","reuseVolumeSortOrder":"pipeline_id","storageClassName":"{{ include \"dind-volume-provisioner.storageClassName\" . }}","volumeSize":"16Gi"}` | Default dind PVC parameters |
| runtime.dind.pvcs.dind.name | string | `"dind"` | PVC name prefix. Keep `dind` as default! Don't change! |
| runtime.dind.pvcs.dind.reuseVolumeSelector | string | `"codefresh-app,io.codefresh.accountName"` | PV reuse selector. Ref: https://codefresh.io/docs/docs/installation/codefresh-runner/#volume-reuse-policy |
| runtime.dind.pvcs.dind.storageClassName | string | `"{{ include \"dind-volume-provisioner.storageClassName\" . }}"` | PVC storage class name. Change ONLY if you need to use storage class NOT from Codefresh volume-provisioner |
| runtime.dind.pvcs.dind.volumeSize | string | `"16Gi"` | PVC size. |
| runtime.dind.resources | object | `{"limits":{"cpu":"400m","memory":"800Mi"},"requests":null}` | Set dind resources. |
| runtime.dind.schedulerName | string | `""` | Set scheduler name. |
| runtime.dind.serviceAccount | string | `"codefresh-engine"` | Set service account for pod. |
| runtime.dind.tolerations | list | `[]` | Set tolerations. |
| runtime.dind.userAccess | bool | `true` | Keep `true` as default! |
| runtime.dind.userVolumeMounts | object | `{}` | Add extra volume mounts |
| runtime.dind.userVolumes | object | `{}` | Add extra volumes |
| runtime.dindDaemon | object | See below | DinD pod daemon config |
| runtime.engine | object | `{"affinity":{},"command":["npm","run","start"],"env":{},"image":{"registry":"quay.io","repository":"codefresh/engine","tag":"1.165.2"},"nodeSelector":{},"podAnnotations":{},"resources":{"limits":{"cpu":"1000m","memory":"2048Mi"},"requests":{"cpu":"100m","memory":"128Mi"}},"runtimeImages":{"COMPOSE_IMAGE":"quay.io/codefresh/compose:v2.20.3-1.4.0","CONTAINER_LOGGER_IMAGE":"quay.io/codefresh/cf-container-logger:1.10.3","DOCKER_BUILDER_IMAGE":"quay.io/codefresh/cf-docker-builder:1.3.6","DOCKER_PULLER_IMAGE":"quay.io/codefresh/cf-docker-puller:8.0.14","DOCKER_PUSHER_IMAGE":"quay.io/codefresh/cf-docker-pusher:6.0.13","DOCKER_TAG_PUSHER_IMAGE":"quay.io/codefresh/cf-docker-tag-pusher:1.3.11","FS_OPS_IMAGE":"quay.io/codefresh/fs-ops:1.2.3","GIT_CLONE_IMAGE":"quay.io/codefresh/cf-git-cloner:10.1.21","KUBE_DEPLOY":"quay.io/codefresh/cf-deploy-kubernetes:16.1.11","PIPELINE_DEBUGGER_IMAGE":"quay.io/codefresh/cf-debugger:1.3.0","TEMPLATE_ENGINE":"quay.io/codefresh/pikolo:0.13.8"},"schedulerName":"","serviceAccount":"codefresh-engine","tolerations":[],"userEnvVars":[]}` | Parameters for Engine pod (aka "pipeline" orchestrator). |
| runtime.engine.affinity | object | `{}` | Set affinity |
| runtime.engine.command | list | `["npm","run","start"]` | Set container command. |
| runtime.engine.env | object | `{}` | Set additional env vars. |
| runtime.engine.image | object | `{"registry":"quay.io","repository":"codefresh/engine","tag":"1.165.2"}` | Set image. |
| runtime.engine.nodeSelector | object | `{}` | Set node selector. |
| runtime.engine.podAnnotations | object | `{}` | Set pod annotations. |
| runtime.engine.resources | object | `{"limits":{"cpu":"1000m","memory":"2048Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}` | Set resources. |
| runtime.engine.runtimeImages | object | See below. | Set system(base) runtime images. |
| runtime.engine.schedulerName | string | `""` | Set scheduler name. |
| runtime.engine.serviceAccount | string | `"codefresh-engine"` | Set service account for pod. |
| runtime.engine.tolerations | list | `[]` | Set tolerations. |
| runtime.engine.userEnvVars | list | `[]` | Set extra env vars |
| runtime.gencerts | object | See below | Parameters for `gencerts-dind` post-upgrade/install hook |
| runtime.inCluster | bool | `true` | (for On-Premise only) Set inCluster runtime |
| runtime.patch | object | See below | Parameters for `runtime-patch` post-upgrade/install hook |
| runtime.rbac | object | `{"create":true,"rules":[]}` | RBAC parameters |
| runtime.rbac.create | bool | `true` | Create RBAC resources |
| runtime.rbac.rules | list | `[]` | Add custom rule to the engine role |
| runtime.runtimeExtends | list | `["system/default/hybrid/k8s_low_limits"]` | Set parent runtime to inherit. Should not be changes. Parent runtime is controlled from Codefresh side. |
| runtime.serviceAccount | object | `{"annotations":{},"create":true}` | Set annotation on engine Service Account Ref: https://codefresh.io/docs/docs/administration/codefresh-runner/#injecting-aws-arn-roles-into-the-cluster |
| serviceMonitor | object | See below | Add serviceMonitor |
| serviceMonitor.main.enabled | bool | `false` | Enable service monitor for dind pods |
| storage.azuredisk.cachingMode | string | `"None"` |  |
| storage.azuredisk.skuName | string | `"Premium_LRS"` | Set storage type (`Premium_LRS`) |
| storage.backend | string | `"local"` | Set backend volume type (`local`/`ebs`/`ebs-csi`/`gcedisk`/`azuredisk`) |
| storage.ebs.accessKeyId | string | `""` | Set AWS_ACCESS_KEY_ID for volume-provisioner (optional) Ref: https://codefresh.io/docs/docs/installation/codefresh-runner/#dind-volume-provisioner-permissions |
| storage.ebs.accessKeyIdSecretKeyRef | object | `{}` | Existing secret containing AWS_ACCESS_KEY_ID. |
| storage.ebs.availabilityZone | string | `"us-east-1a"` | Set EBS volumes availability zone (required) |
| storage.ebs.encrypted | string | `"false"` | Enable encryption (optional) |
| storage.ebs.kmsKeyId | string | `""` | Set KMS encryption key ID (optional) |
| storage.ebs.secretAccessKey | string | `""` | Set AWS_SECRET_ACCESS_KEY for volume-provisioner (optional) Ref: https://codefresh.io/docs/docs/installation/codefresh-runner/#dind-volume-provisioner-permissions |
| storage.ebs.secretAccessKeySecretKeyRef | object | `{}` | Existing secret containing AWS_SECRET_ACCESS_KEY |
| storage.ebs.volumeType | string | `"gp2"` | Set EBS volume type (`gp2`/`gp3`/`io1`) (required) |
| storage.fsType | string | `"ext4"` | Set filesystem type (`ext4`/`xfs`) |
| storage.gcedisk.availabilityZone | string | `"us-west1-a"` | Set GCP volume availability zone |
| storage.gcedisk.serviceAccountJson | string | `""` | Set Google SA JSON key for volume-provisioner (optional) |
| storage.gcedisk.serviceAccountJsonSecretKeyRef | object | `{}` | Existing secret containing containing Google SA JSON key for volume-provisioner (optional) |
| storage.gcedisk.volumeType | string | `"pd-ssd"` | Set GCP volume backend type (`pd-ssd`/`pd-standard`) |
| storage.local.volumeParentDir | string | `"/var/lib/codefresh/dind-volumes"` | Set volume path on the host filesystem |
| storage.mountAzureJson | bool | `false` |  |
| volumeProvisioner | object | See below | Volume Provisioner parameters |
| volumeProvisioner.affinity | object | `{}` | Set affinity |
| volumeProvisioner.dind-lv-monitor | object | See below | `dind-lv-monitor` DaemonSet parameters (local volumes cleaner) |
| volumeProvisioner.enabled | bool | `true` | Enable volume-provisioner |
| volumeProvisioner.env | object | `{}` | Add additional env vars |
| volumeProvisioner.image | object | `{"registry":"quay.io","repository":"codefresh/dind-volume-provisioner","tag":"1.33.3"}` | Set image |
| volumeProvisioner.nodeSelector | object | `{}` | Set node selector |
| volumeProvisioner.podAnnotations | object | `{}` | Set pod annotations |
| volumeProvisioner.podSecurityContext | object | See below | Set security context for the pod |
| volumeProvisioner.rbac | object | `{"create":true,"rules":[]}` | RBAC parameters |
| volumeProvisioner.rbac.create | bool | `true` | Create RBAC resources |
| volumeProvisioner.rbac.rules | list | `[]` | Add custom rule to the role |
| volumeProvisioner.replicasCount | int | `1` | Set number of pods |
| volumeProvisioner.resources | object | `{}` | Set resources |
| volumeProvisioner.serviceAccount | object | `{"annotations":{},"create":true,"name":""}` | Service Account parameters |
| volumeProvisioner.serviceAccount.annotations | object | `{}` | Additional service account annotations |
| volumeProvisioner.serviceAccount.create | bool | `true` | Create service account |
| volumeProvisioner.serviceAccount.name | string | `""` | Override service account name |
| volumeProvisioner.tolerations | list | `[]` | Set tolerations |
| volumeProvisioner.updateStrategy | object | `{"type":"Recreate"}` | Upgrade strategy |
