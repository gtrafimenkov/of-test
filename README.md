# Secure access to workload outside Kubernetes

- [Task](#task)
- [Possible options](#possible-options)
- [More details on a separate group of k8s nodes option](#more-details-on-a-separate-group-of-k8s-nodes-option)
	- [Drawbacks](#drawbacks)

## Task

As part of the Kubernetes migration project, the client will migrate all of it’s
applications to Kubernetes, however, the database services such as MongoDB or
PostgreSQL will continue to run on existing EC2 instances (in the same VPC as the Kubernetes cluster).

One of the requirements posed by the regulation is to restrict access to the MongoDB
database to the services that need it. This means that we won’t be able to allow all
the pods running on our Kubernetes clusters to freely access the MongoDB instance.

Create a short README that explains briefly a few possible approaches for meeting
the restrictions mentioned above. Take into account that since the client doesn’t
run on Kubernetes at the moment, you can control almost any aspect of the solution,
including the Kubernetes infrastructure and bootstrapping methods (the cloud provider
should remain AWS).

## Possible options

Most likely the only way of restricting access to MongoDB, which will satisfy regulators
and auditors, is usage of AWS Security Groups.  Unfortunately at the moment it's not possible
to assign individual security groups to particular Kubernetes pods.

Although there are a couple of options:

- create a separate group of k8s nodes that will be used to run pods, which require access
  to the MongoDB database.  Assign a special security group to this group of nodes, allow
  access from this security group to the MongoDB.  Use node affinity and tolerations
  to allow only particular pods run on these worker nodes;

- consider using [Calico Enterprise](https://www.tigera.io/tigera-products/calico-enterprise/)
  product which they claim can assign security groups to individual pods;

Also the new version of AWS VPC CNI Plugin should support assigning security groups to
individual pods: https://github.com/aws/containers-roadmap/issues/398.  Unfortunately
it's not known when this new version will be available.

It is also worth monitoring issue https://github.com/aws/containers-roadmap/issues/177 for
new development and possible ideas.

## More details on a separate group of k8s nodes option

- a special AWS security group will be assigned to the worker nodes in this node group;
- it is also will be necessary to assign:
    - a particular taint to the nodes of this group, for example, named "OnlyForDBAccess";
    - a particular label to the nodes of this group.  This label will be used to require
      node affinity of the pods;
- Kubernetes pods, which require access to the MongoDB instance, will have to include
  toleration for "OnlyForDBAccess" taint.  Also they will have to include required node
  affinity to these worker nodes.  By doing so:
    - only pods with this toleration will be allowed to run on the worker nodes designated
      to DB access;
    - due to required affinity these pods will not be scheduled to other worker nodes;
- MongoDB instance security group should be configured to allow connections from the security
  group mentioned above.

### Drawbacks

Since there will be at least two groups of worker nodes, their utilization might not be
as effective as if every pod were allowed to run on any worker node.
