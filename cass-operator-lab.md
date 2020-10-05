
# Loading the operator

Installing the Cass Operator itself is straightforward. We have provided manifests for each Kubernetes version from 1.13 through 1.17. Apply the relevant manifest to your cluster as follows:

> __Note__: Change to your version of Kubernetes

```bash
K8S_VER=v1.16
kubectl apply -f https://raw.githubusercontent.com/datastax/cass-operator/v1.4.1/docs/user/cass-operator-manifests-$K8S_VER.yaml
```

Note that since the manifest will install a Custom Resource Definition, the user running the above command will need cluster-admin privileges.

This will deploy the operator, along with any requisite resources such as Role, RoleBinding, etc., to the cass-operator namespace. You can check to see if the operator is ready as follows:

`kubectl -n cass-operator get pods --selector name=cass-operator`

```bash
NAME                             READY   STATUS    RESTARTS   AGE
cass-operator-555577b9f8-zgx6j   1/1     Running   0          25h
```

> Task: Create / Configure local storage to be used by cass-operator

## Creating a CassandraDatacenter

You can pass the Storage name created above as a param the apply the following resource to create a simple cssanra cluster

```yaml
apiVersion: cassandra.datastax.com/v1beta1
kind: CassandraDatacenter
metadata:
  name: dc1
spec:
  clusterName: cluster1
  serverType: cassandra
  serverVersion: 3.11.7
  managementApiAuth:
    insecure: {}
  size: 3
  storageConfig:
    cassandraDataVolumeClaimSpec:
      storageClassName: $STORAGE_NAME
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
  config:
    cassandra-yaml:
      authenticator: org.apache.cassandra.auth.PasswordAuthenticator
      authorizer: org.apache.cassandra.auth.CassandraAuthorizer
      role_manager: org.apache.cassandra.auth.CassandraRoleManager
    jvm-options:
      initial_heap_size: 800M
      max_heap_size: 800M
```

You can check the status of pods in the Cassandra cluster as follows:

```bash
kubectl -n cass-operator get pods --selector cassandra.datastax.com/cluster=cluster1
NAME                         READY   STATUS    RESTARTS   AGE
cluster1-dc1-default-sts-0   2/2     Running   0          2m
cluster1-dc1-default-sts-1   2/2     Running   0          2m
cluster1-dc1-default-sts-2   2/2     Running   0          2m
```

You can check to see the current progress of bringing the Cassandra datacenter online by checking the `cassandraOperatorProgress` field of the __CassandraDatacenter's__ status sub-resource as follows:

```bash
kubectl -n cass-operator get cassdc/dc1 -o "jsonpath={.status.cassandraOperatorProgress}"
Ready
```

(`cassdc` and `cassdcs` are supported short forms of `CassandraDatacenter`.)

A value of "Ready", as above, means the operator has finished setting up the Cassandra datacenter.

## Installing DSE

To use Datastax DSE distribution instead apply the following resource:

> Note: Create appropriate storage resource 

```yaml
# Sized to work on 3 k8s workers nodes with 1 core / 4 GB RAM
# See neighboring example-cassdc-full.yaml for docs for each parameter
apiVersion: cassandra.datastax.com/v1beta1
kind: CassandraDatacenter
metadata:
  name: dc1
spec:
  clusterName: cluster2
  serverType: dse
  serverVersion: "6.8.4"
  managementApiAuth:
    insecure: {}
  size: 3
  storageConfig:
    cassandraDataVolumeClaimSpec:
      storageClassName: server-storage
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
  config:
    jvm-server-options:
      initial_heap_size: "800M"
      max_heap_size: "800M"
      additional-jvm-opts:
        # As the database comes up for the first time, set system keyspaces to RF=3
        - "-Ddse.system_distributed_replication_dc_names=dc1"
        - "-Ddse.system_distributed_replication_per_dc=3"
```

> Exercise: Install Datastax Studio on the cluster.