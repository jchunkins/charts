# LogDNA Kubernetes Agent

[LogDNA](https://logdna.com) - Easy, beautiful logging in the cloud.

## TL;DR;

```bash
$ helm install --set logdna.ingestionKeySecret=LOGDNA_INGESTION_SECRET stable/logdna-agent
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
- apiGroups: ["extensions"]
  resourceNames: ["ibm-anyuid-hostpath-psp"]
  resources: ["podsecuritypolicies"]
  verbs: ["use"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "create", "delete", "patch"]
```

Run:
```
kubectl create -f clusterrole.yaml
```

- Create RoleBinding.

Create a RoleBinding which binds ClusterRole created in previous step to default service account of the namespace.

Run:

_(NOTE: replace `<namespace>` with your namespace )_

```
kubectl create rolebinding logdna-rolebinding --clusterrole=logdna-clusterrole --serviceaccount=<namespace>:default --namespace=<namespace>
```

or using RoleBinding definition:

Save the following defintion into a file (e.g. rolebinding.yaml)

```yaml
- apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
    name: logdna-rolebinding
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

### Creating a Secret

The logdna-agent chart requires that you store your ingestion key inside a kubernetes secret using the key name logdna-agent-key. We suggest that you name the secret with the name of your release followed by logdna-agent, separated with a dash. For example if you plan to create a deployment with release name my-release then use this command (replaceing the X's with your ingestion key and substituting namespace_name with the namespace you want to use):

```
kubectl create secret generic my-release-logdna-agent \
  --from-literal='logdna-agent-key=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX' \
  --namespace namespace_name
When installing the chart you must refer to the name of the secret you have created. In our previous example that name is my-release-logdna-agent.
```

## Installing the Chart

### Obtain LogDNA Ingestion Key from IBM Cloud LogDNA server instance
View your [logging instance](https://cloud.ibm.com/observe/logging) and select `View LogDNA`.
Select the help icon in lower left (i.e. install instructions) to view your LogDNA Ingestion Key

### Obtain LogDNA Ingestion Key from LogDNA server instance
Please follow directions from https://app.logdna.com/pages/add-source to obtain your LogDNA Ingestion Key.

### Install
To install the chart with the release name `my-release`:

```bash
$ helm install --name my-release \
    --set logdna.ingestionKeySecret=LOGDNA_INGESTION_SECRET,logdna.autoupdate=1 stable/logdna-agent
```

You should see logs in https://app.logdna.com in a few seconds.

### Tags support:
```bash
$ helm install --name my-release \
    --set logdna.ingestionKeySecret=LOGDNA_INGESTION_SECRET,logdna.tags=production,logdna.autoupdate=1 stable/logdna-agent
```

## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```bash
$ helm delete my-release
```

The command removes the Kubernetes daemonset associated with the chart and deletes the release. You must 
manually remove the secret. To remove the secret for release `my-release`:

```bash
kubectl delete secret my-release-logdna-agent
```

## Configuration

The following tables lists the configurable parameters of the LogDNA Agent chart and their default values.

Parameter | Description | Default
--- | --- | ---
`logdna.ingestionKeySecret` | LogDNA Ingestion Key (Required) | None
`logdna.tags` | Optional tags such as `production` | None
`logdna.autoupdate` | Optionally turn on autoupdate by setting to 1 (auto sets image.pullPolicy to always) | `0`
`image.pullPolicy` | Image pull policy | `IfNotPresent`
`image.tag` | Image tag | `latest`
`resources.limits.memory` | Memory resource limits | 500Mi                                      |
`tolerations` | List of node taints to tolerate | `[]`

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example,

```bash
$ helm install --name my-release \
    --set logdna.ingestionKeySecret=LOGDNA_INGESTION_SECRET,logdna.tags=production,logdna.autoupdate=1 stable/logdna-agent
```

Alternatively, a YAML file that specifies the values for the above parameters can be provided while installing the chart. For example,

```bash
$ helm install --name my-release -f values.yaml stable/logdna-agent
```

> **Tip**: You can use the default [values.yaml](values.yaml)



## Support

### Connecting to IBM Cloud LogDNA server instance
If you configure your agent to connect to a IBM Cloud LogDNA instance, you may contact 
[IBM cloud support](https://cloud.ibm.com/unifiedsupport/supportcenter) if you experience issues connecting to the instance.

### Connecting to LogDNA server instance
If you configure your agent to connect to LogDNA directly, you may contact LogDNA support through
the [LogDNA web console](https://app.logdna.com/) if you experience issues connecting to LogDNA directly.