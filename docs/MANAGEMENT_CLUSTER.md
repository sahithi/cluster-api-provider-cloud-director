# Management Cluster Setup

## Create a Management cluster

All the below steps are expected to be performed by an organization administrator.

### Create a bootstrap Kubernetes cluster

It is a common practice to create a temporary bootstrap cluster which is then used to provision a
management cluster on the Cloud Director (infrastructure provider).

Choose one of the options below to set up a management cluster on VMware Cloud Director:

1. [CSE](https://github.com/vmware/container-service-extension) provisioned TKG cluster as a bootstrap cluster to
   further create a Management cluster in VCD tenant organization.
2. [Kind as a bootstrap cluster](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-andor-configure-a-kubernetes-cluster)
   to create Management cluster in VCD tenant organization

We recommend CSE provisioned TKG cluster as bootstrap cluster.

<a name="management_cluster_init"></a>
### Initialize the bootstrap cluster with Cluster API
1. Set up the [clusterctl](CLUSTERCTL.md#clusterctl_set_up) on the cluster
2. [Initialize the Cluster API and CAPVCD](CLUSTERCTL.md#init_management_cluster) on the cluster
3. [Apply CRS definitions](CRS.md#apply_crs) to ensure CPI, CNI, and CSI are installed on the (children) workload clusters
4. Wait until `kubectl get pods -A` shows below pods in Running state
    1. ```shell
       kubectl get pods -A
       NAMESPACE                           NAME                                                            READY   STATUS
       capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-7dc44947-v5nlv        1/1     Running
       capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-cb9d954f5-ct5cp   1/1     Running
       capi-system                         capi-controller-manager-7594c7bc57-smjtg                        1/1     Running
       capvcd-system                       capvcd-controller-manager-769d64d4bf-54bf4                      1/1     Running
       ```  

### Create a multi-controlplane management cluster
Now that bootstrap cluster is ready, you can use Cluster API to create multi control-plane workload cluster

1. Generate the [cluster manifest](CLUSTERCTL.md#generate_cluster_manifest) and apply it on the bootstrap cluster.
2. [Apply CRS labels](CRS.md#apply_crs_labels) on the workload cluster.
3. The last step is to [enable add-ons (CPI, CSI)](CRS.md#enable_add_ons) on the workload cluster to access VCD resources
4. Transform this workload cluster into management cluster by [initializing it with CAPVCD](#management_cluster_init).
5. This cluster is now a fully functional multi-control plane management cluster.

<a name="tenant_user_management"></a>
## Prepare the Management cluster to enable VCD tenant users' access
Below steps enable tenant users to deploy the workload clusters in their own private namespaces of a given management
cluster, while adhering to their own user quota in VCD.

The organization administrator creates a new and unique Kubernetes namespace for
each tenant user and creates a respective Kubernetes configuration with access to only the
required CRDs. This is a one-time operation per VCD tenant user.

Run below commands for each tenant user. The USERNAME and KUBE_APISERVER_ADDRESS parameter should be
changed as per your requirements.

```sh
USERNAME="user1"

NAMESPACE=${USERNAME}-ns
kubectl create ns ${NAMESPACE}

cat > user-rbac.yaml << END
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ${USERNAME}
  namespace: ${NAMESPACE}
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: ${NAMESPACE}
  name: ${USERNAME}-full-access
rules:
- apiGroups: ["", "extensions", "apps", "cluster.x-k8s.io", "infrastructure.cluster.x-k8s.io", "bootstrap.cluster.x-k8s.io", "controlplane.cluster.x-k8s.io", "apiextensions.k8s.io"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ${USERNAME}-view-${NAMESPACE}
  namespace: ${NAMESPACE}
subjects:
- kind: ServiceAccount
  name: ${USERNAME}
  namespace: ${NAMESPACE}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ${USERNAME}-full-access
---
END

kubectl create -f user-rbac.yaml

SECRETNAME=$(kubectl -n ${NAMESPACE} describe sa ${USERNAME} | grep "Tokens" | cut -f2 -d: | tr -d " ")
USERTOKEN=$(kubectl -n ${NAMESPACE} get secret ${SECRETNAME} -o "jsonpath={.data.token}" | base64 -d)
CERT=$(kubectl -n ${NAMESPACE} get secret ${SECRETNAME} -o "jsonpath={.data['ca\.crt']}")
KUBE_APISERVER_ADDRESS=https://127.0.0.1:64265

cat > user1-management-kubeconfig.conf <<END
apiVersion: v1
kind: Config
users:
- name: ${USERNAME}
  user:
    token: ${USERTOKEN}
clusters:
- cluster:
    certificate-authority-data: ${CERT}
    server: ${KUBE_APISERVER_ADDRESS}
  name: my-cluster
contexts:
- context:
    cluster: my-cluster
    user: ${USERNAME}
  name: ${USERNAME}-context
current-context: ${USERNAME}-context
END
```
The "user1-management-kubeconfig.conf" generated at the end ensures that the user, user1, can only access CRDs of the
workload cluster in his/her own created namespace ${NAMESPACE} (user1-ns) of the management cluster.

Notes:
* Organization administrator relays the "user1-management-kubeconfig.conf" to the tenant user "user1"
* Once the above operation is complete, there is no need of further interaction between organization administrator and the tenant user.
* The mechanism used above to generate a Kubernetes Config has a default lifetime of one year.
* We recommend strongly that the USERNAME match that of VCD tenant username.

## Resize, Upgrade and Delete Management cluster
These workflows need to be run from the bootstrap cluster (the parent of the management cluster).

In the `kubectl` commands specified in the below workflows, update the `namespace` parameter to the value `default`
and `kubeconfig` to the value of bootstrap cluster's admin Kubeconfig
* [Resize workflow](WORKLOAD_CLUSTER.md#resize_workload_cluster)
* [Upgrade workflow](WORKLOAD_CLUSTER.md#upgrade_workload_cluster)
* [Delete workflow](WORKLOAD_CLUSTER.md#delete_workload_cluster)
