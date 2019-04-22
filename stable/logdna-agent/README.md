# LogDNA Kubernetes Agent

[LogDNA](https://logdna.com) - Easy, beautiful logging in the cloud.

## TL;DR;

```bash
$ helm install --set logdna.key=LOGDNA_INGESTION_KEY stable/logdna-agent
```

## Introduction

This chart deploys LogDNA collector agents to all nodes in your cluster. Logs will ship from all containers. We extract pertinent Kubernetes metadata: pod name, container name, container id, namespace, and labels. View your logs at https://app.logdna.com or live tail using our [CLI](https://github.com/logdna/logdna-cli).

## Prerequisites

- Kubernetes 1.2+
- A LogDNA Ingestion Key

### Pod Security Policies

LogDNA-agent containers needs to mount host volumes to read the logs for ingestion, and needs to run as root. This chart requires a PodSecurityPolicy (with host volume and root access) to be bound to the target namespace prior to installation. If the `default` PodSecurityPolicy on your namespace is not restrictive then this step is not needed.

If the default is restrictive, In ICP you can create a new namespace with either a predefined PodSecurityPolicy.

- Predefined PodSecurityPolicy name: [ibm-anyuid-hostpath-psp](https://github.com/IBM/cloud-pak/blob/master/spec/security/psp/README.md)

Alternatively, you can have your cluster administrator setup a custom PodSecurityPolicy for you using the below definition:

- Custom PodSecurityPolicy definition:

```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
annotations:
    kubernetes.io/description: 'This policy allows pods to run with any UID and GID
    and any volume, including the host path. WARNING:  This policy allows hostPath
    volumes. Use with caution.'
name: custom-anyuid-hostpath-psp
spec:
allowPrivilegeEscalation: true
allowedCapabilities:
- SETPCAP
- AUDIT_WRITE
- CHOWN
- NET_RAW
- DAC_OVERRIDE
- FOWNER
- FSETID
- KILL
- SETUID
- SETGID
- NET_BIND_SERVICE
- SYS_CHROOT
- SETFCAP
fsGroup:
    rule: RunAsAny
requiredDropCapabilities:
- MKNOD
runAsUser:
    rule: RunAsAny
seLinux:
    rule: RunAsAny
supplementalGroups:
    rule: RunAsAny
volumes:
- '*'
```

- Create a [ClusterRole](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings)

Save the following defintion into a file (e.g. clusterrole.yaml)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
name: logdna-clusterrole
rules:
- apiGroups:
    - ''
    resources:
    - pods
    - secrets
    - jobs
    verbs:
    - get
    - create
    - delete
```

Run:
```
kubectl create -f clusterrole.yaml
```

- Create ClusterRoleBinding.

Create a ClusterRoleBinding which binds ClusterRole created in previous step to default service account of the namespace.

Run:

_(NOTE: replace `<namespace>` with your namespace )_

```
kubectl create rolebinding logdna-clusterrolebinding --clusterrole=logdna-clusterrole --serviceaccount=<namespace>:default --namespace=<namespace>
```

or using ClusterRoleBinding definition:

Save the following defintion into a file (e.g. clusterrolebinding.yaml)

```yaml
- apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
    name: logdna-clusterrolebinding
roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: ClusterRole
    name: logdna-clusterrole
subjects:
    - kind: ServiceAccount
    name: default
    namespace: <namespace>
```

Run:
```
kubectl create -f clusterrolebinding.yaml
```

Alternatively, a cluster admin can create a [service account](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/) for LogDNA itself and bind the ClusterRole to it.

Save the following defintion into a file (e.g. serviceaccount.yaml):

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
    name: logdna-serviceaccount
    namespace: <namespace>
```

Run:
```
kubectl create -f serviceaccount.yaml
```


### Image Security Policies

If the cluster has image security policies enforced, logdna's docker image should be added to it 

For example on [IBM Private Cloud (ICP)](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.2/manage_images/image_security.html) :

```
apiVersion: securityenforcement.admission.cloud.ibm.com/v1beta1
kind: ClusterImagePolicy
metadata:
name: ibmcloud-default-cluster-image-policy
spec:
 repositories:
   - name: docker.io/logdna/logdna-agent:*
```

## Installing the Chart

To install the chart with the release name `my-release`, please follow directions from https://app.logdna.com/pages/add-source to obtain your LogDNA Ingestion Key:

```bash
$ helm install --name my-release \
    --set logdna.key=LOGDNA_INGESTION_KEY,logdna.autoupdate=1 stable/logdna-agent
```

You should see logs in https://app.logdna.com in a few seconds.

### Tags support:
```bash
$ helm install --name my-release \
    --set logdna.key=LOGDNA_INGESTION_KEY,logdna.tags=production,logdna.autoupdate=1 stable/logdna-agent
```

## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```bash
$ helm delete my-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration

The following tables lists the configurable parameters of the LogDNA Agent chart and their default values.

Parameter | Description | Default
--- | --- | ---
`logdna.key` | LogDNA Ingestion Key (Required) | None
`logdna.tags` | Optional tags such as `production` | None
`logdna.autoupdate` | Optionally turn on autoupdate by setting to 1 (auto sets image.pullPolicy to always) | `0`
`image.pullPolicy` | Image pull policy | `IfNotPresent`
`image.tag` | Image tag | `latest`
`resources.limits.memory` | Memory resource limits | 500Mi                                      |
`tolerations` | List of node taints to tolerate | `[]`

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```bash
$ helm install --name my-release \
    --set logdna.key=LOGDNA_INGESTION_KEY,logdna.tags=production,logdna.autoupdate=1 stable/logdna-agent
```

Alternatively, a YAML file that specifies the values for the above parameters can be provided while installing the chart. For example,

```bash
$ helm install --name my-release -f values.yaml stable/logdna-agent
```

> **Tip**: You can use the default [values.yaml](values.yaml)

## Support

When using IBM Cloud LogDNA instance, reach out to [IBM cloud support](https://cloud.ibm.com/unifiedsupport/supportcenter)

When using LogDNA SAAS, reach out to support from [LogDNA web console](https://app.logdna.com/)

[ICP Support](https://ibm.biz/icpsupport)
