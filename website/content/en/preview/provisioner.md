---
title: "Provisioner API"
linkTitle: "Provisioner API"
weight: 70
date: 2017-01-05
description: >
  Provisioner API reference page
---

## Example Provisioner Resource

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  # Enables consolidation which attempts to reduce cluster cost by both removing un-needed nodes and down-sizing those
  # that can't be removed.  Mutually exclusive with the ttlSecondsAfterEmpty parameter.
  consolidation:
    enabled: true
    
  # If omitted, the feature is disabled and nodes will never expire.  If set to less time than it requires for a node
  # to become ready, the node may expire before any pods successfully start.
  ttlSecondsUntilExpired: 2592000 # 30 Days = 60 * 60 * 24 * 30 Seconds;

  # If omitted, the feature is disabled, nodes will never scale down due to low utilization
  ttlSecondsAfterEmpty: 30
  
  # Priority given to the provisioner when the scheduler considers which provisioner
  # to select. Higher weights indicate higher priority when comparing provisioners.
  # Specifying no weight is equivalent to specifying a weight of 0.
  weight: 10

  # Provisioned nodes will have these taints
  # Taints may prevent pods from scheduling if they are not tolerated by the pod.
  taints:
    - key: example.com/special-taint
      effect: NoSchedule


  # Provisioned nodes will have these taints, but pods do not need to tolerate these taints to be provisioned by this
  # provisioner. These taints are expected to be temporary and some other entity (e.g. a DaemonSet) is responsible for
  # removing the taint after it has finished initializing the node.
  startupTaints:
    - key: example.com/another-taint
      effect: NoSchedule

  # Labels are arbitrary key-values that are applied to all nodes
  labels:
    billing-team: my-team

  # Requirements that constrain the parameters of provisioned nodes.
  # These requirements are combined with pod.spec.affinity.nodeAffinity rules.
  # Operators { In, NotIn } are supported to enable including or excluding values
  requirements:
    - key: "node.kubernetes.io/instance-type"
      operator: In
      values: ["m5.large", "m5.2xlarge"]
    - key: "topology.kubernetes.io/zone"
      operator: In
      values: ["us-west-2a", "us-west-2b"]
    - key: "kubernetes.io/arch"
      operator: In
      values: ["arm64", "amd64"]
    - key: "karpenter.sh/capacity-type" # If not included, the webhook for the AWS cloud provider will default to on-demand
      operator: In
      values: ["spot", "on-demand"]

  # Karpenter provides the ability to specify a few additional Kubelet args.
  # These are all optional and provide support for additional customization and use cases.
  kubeletConfiguration:
    clusterDNS: ["10.0.1.100"]
    containerRuntime: containerd
    systemReserved:
      cpu: 1
      memory: 5Gi
      ephemeral-storage: 2Gi
    maxPods: 20

  # Resource limits constrain the total size of the cluster.
  # Limits prevent Karpenter from creating new instances once the limit is exceeded.
  limits:
    resources:
      cpu: "1000"
      memory: 1000Gi

  # These fields vary per cloud provider, see your cloud provider specific documentation
  provider: {}
```

## Node deprovisioning

If neither of these values are set, Karpenter will *not* delete instances. It is recommended to set the `ttlSecondsAfterEmpty` value, to enable scale down of the cluster.

### spec.ttlSecondsAfterEmpty

Setting a value here enables Karpenter to delete empty/unnecessary instances. DaemonSets are excluded from considering a node "empty". This value is in seconds.

### spec.ttlSecondsUntilExpired

Setting a value here enables node expiry. After nodes reach the defined age in seconds, they will be deleted, even if in use. This enables nodes to effectively be periodically "upgraded" by replacing them with newly provisioned instances.

Note that Karpenter does not automatically add jitter to this value. If multiple instances are created in a small amount of time, they will expire at very similar times. Consider defining a [pod disruption budget](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) to prevent excessive workload disruption.



## spec.requirements

Kubernetes defines the following [Well-Known Labels](https://kubernetes.io/docs/reference/labels-annotations-taints/), and cloud providers (e.g., AWS) implement them. They are defined at the "spec.requirements" section of the Provisioner API.

These well known labels may be specified at the provisioner level, or in a workload definition (e.g., nodeSelector on a pod.spec). Nodes are chosen using both the provisioner's and pod's requirements. If there is no overlap, nodes will not be launched. In other words, a pod's requirements must be within the provisioner's requirements. If a requirement is not defined for a well known label, any value available to the cloud provider may be chosen.

For example, an instance type may be specified using a nodeSelector in a pod spec. If the instance type requested is not included in the provisioner list and the provisioner has instance type requirements, Karpenter will not create a node or schedule the pod.

📝 None of these values are required.

### Instance Types

- key: `node.kubernetes.io/instance-type`

Generally, instance types should be a list and not a single value. Leaving this field undefined is recommended, as it maximizes choices for efficiently placing pods.

☁️ **AWS**

Review [AWS instance types](https://aws.amazon.com/ec2/instance-types/).

The default value includes most instance types with the exclusion of [non-HVM](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/virtualization_types.html).  The full list of supported instance types can be seen [here](../aws/instance-types/)

**Example**

*Set Default with provisioner.yaml*

```yaml
spec:
  requirements:
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["m5.large", "m5.2xlarge"]
```

*Override with workload manifest (e.g., pod)*

```yaml
spec:
  template:
    spec:
      nodeSelector:
        node.kubernetes.io/instance-type: m5.large
```

### Availability Zones

- key: `topology.kubernetes.io/zone`
- value example: `us-east-1c`

☁️ **AWS**

- value list: `aws ec2 describe-availability-zones --region <region-name>`

Karpenter can be configured to create nodes in a particular zone. Note that the Availability Zone `us-east-1a` for your AWS account might not have the same location as `us-east-1a` for another AWS account.

[Learn more about Availability Zone
IDs.](https://docs.aws.amazon.com/ram/latest/userguide/working-with-az-ids.html)

### Architecture

- key: `kubernetes.io/arch`
- values
  - `amd64` (default)
  - `arm64`

Karpenter supports `amd64` nodes, and `arm64` nodes.


### Capacity Type

- key: `karpenter.sh/capacity-type`

☁️ **AWS**

- values
  - `spot`
  - `on-demand` (default)

Karpenter supports specifying capacity type, which is analogous to [EC2 purchase options](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-purchasing-options.html).

Karpenter prioritizes Spot offerings if the provisioner allows Spot and on-demand instances. If the provider API (e.g. EC2 Fleet's API) indicates Spot capacity is unavailable, Karpenter caches that result across all attempts to provision EC2 capacity for that instance type and zone for the next 45 seconds. If there are no other possible offerings available for Spot, Karpenter will attempt to provision on-demand instances, generally within milliseconds.

Karpenter also allows `karpenter.sh/capacity-type` to be used as a topology key for enforcing topology-spread.

## spec.weight

Karpenter allows you to describe provisioner preferences through a `weight` mechanism similar to how weight is described with [pod and node affinities](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#affinity-and-anti-affinity). 

For more information on weighting provisioners, see the [Scheduling Details](../tasks/scheduling/).

## spec.kubeletConfiguration

Karpenter provides the ability to specify a few additional Kubelet args. These are all optional and provide support for
additional customization and use cases. Adjust these only if you know you need to do so.

```yaml
spec:
  kubeletConfiguration:
    clusterDNS: ["10.0.1.100"]
    containerRuntime: containerd
    systemReserved:
      cpu: 1
      memory: 5Gi
      ephemeral-storage: 2Gi
    maxPods: 20
```

☁️ **AWS**

You can specify the container runtime to be either `dockerd` or `containerd`.

* `dockerd` will be chosen by default for [Inferentia instanceTypes](https://aws.amazon.com/ec2/instance-types/inf1/). For all other instances `containerd` is the default.
* You can only use `containerd` with the Bottlerocket AMI Family.

### System Reserved Resources

Karpenter will automatically configure the system reserved resource requests on the fly on your behalf. These requests are used to configure your node and to make scheduling decisions for your pods. If you have specific requirements or know that you will have additional capacity requirements, you can optionally override the `--system-reserved` configuration defaults with the `.spec.kubeletConfiguration.systemReserved` value.

These values will be accounted for in scheduling and be passed through when your node is bootstrapped to the kubelet.

For more information on the deafult `--system-reserved` configuration refer to the [Kubelet Docs](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#system-reserved)

### Max Pods

By default, AWS will configure the maximum density of pods on a node [based on the node instance type](https://github.com/awslabs/amazon-eks-ami/blob/master/files/eni-max-pods.txt). For small instances that require an increased pod density or large instances that require a reduced pod density, you can override this default value with `.spec.kubeletConfiguration.maxPods`. This value will be used during Karpenter pod scheduling and passed through to `--max-pods` on kubelet startup.

{{% alert title="Note" color="primary" %}}
When using small instance types, it may be necessary to enable [prefix assignment mode](https://aws.amazon.com/blogs/containers/amazon-vpc-cni-increases-pods-per-node-limits/) in the AWS VPC CNI plugin to support a higher pod density per node.  Prefix assignment mode was introduced in AWS VPC CNI v1.9 and allows ENIs to manage a broader set of IP addresses.  Much higher pod densities are supported as a result.
{{% /alert %}}

For a more detailed description of pod density considerations, see [Control Pod Density](../tasks/pod-density).

## spec.limits.resources

The provisioner spec includes a limits section (`spec.limits.resources`), which constrains the maximum amount of resources that the provisioner will manage.

Karpenter supports limits of any resource type that is reported by your cloud provider.

CPU limits are described with a `DecimalSI` value. Note that the Kubernetes API will coerce this into a string, so we recommend against using integers to avoid GitOps skew.

Memory limits are described with a [`BinarySI` value, such as 1000Gi.](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-memory)

Karpenter limits instance types when scheduling to those that will not exceed the specified limits.  If a limit has been exceeded, nodes provisioning is prevented until some nodes have been terminated.

Review the [resource limit task](../tasks/set-resource-limits) for more information.

## spec.provider

This section is cloud provider specific. Reference the appropriate documentation:

- [AWS](../aws/provisioning/)
