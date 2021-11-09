# Lab 3: Install the HPE CSI Driver for Kubernetes

To get started with the deployment of the HPE CSI Driver for Kbuernetes, the CSI driver is deployed using industry standard means, either a Helm chart or an Operator. For this tutorial, we will be using Helm to the deploy the HPE CSI driver for Kubernetes.

The official Helm chart for the HPE CSI Driver for Kubernetes is hosted on [Artifact Hub](https://artifacthub.io/packages/helm/hpe-storage/hpe-csi-driver). There, you will find the configuration and installation instructions for the chart.

!!! note
    [Helm](https://helm.sh) is the package manager for Kubernetes. Software is delivered in a format called a "chart". Helm is a [standalone CLI](https://helm.sh/docs/intro/install/) that interacts with the Kubernetes API server using your `KUBECONFIG` file.

## Installing the Helm chart

Open a WSL terminal session, if you don't have one open already.

![](img/wsl_terminal.png)

To install the chart with the name `my-hpe-csi-driver`, add the HPE CSI Driver for Kubernetes Helm repo.

```markdown
helm repo add hpe-storage https://hpe-storage.github.io/co-deployments
helm repo update
```

Install the latest chart.
```markdown
kubectl create ns hpe-storage
helm install my-hpe-csi-driver hpe-storage/hpe-csi-driver -n hpe-storage
```

!!! note
    It is safe to ignore the warnings: <br /> `W1104 14:25:11.178003   18461 warnings.go:70] apiextensions.k8s.io/v1beta1 CustomResourceDefinition is deprecated in v1.16+, unavailable in v1.22+; use apiextensions.k8s.io/v1 CustomResourceDefinition`.
 
Wait a few minutes as the deployment finishes.

Verify that everything is up and running correctly by listing out the `Pods`.

```markdown
kubectl get pods -n hpe-storage
```

The output is **similar** to this:

!!! note
    The `Pod` names will be unique to your deployment.

```markdown
$ kubectl get pods -n hpe-storage
NAME                                      READY   STATUS    RESTARTS   AGE
pod/hpe-csi-controller-6f9b8c6f7b-n7zcr   9/9     Running   0          7m41s
pod/hpe-csi-node-npp59                    2/2     Running   0          7m41s
pod/nimble-csp-5f6cc8c744-rxgfk           1/1     Running   0          7m41s
pod/primera3par-csp-7f78f498d5-4vq9r      1/1     Running   0          7m41s
```

If all of the components show in **Running** state, then the HPE CSI Driver for Kubernetes and the corresponding Container Storage Providers (CSP) for HPE Alletra, Primera and Nimble Storage have been successfully deployed.

!!! Important
    With the HPE CSI Driver deployed, the rest of this guide is designed to demonstrate the usage of the CSI driver with HPE Primera or Nimble Storage. You will need to choose which storage system (HPE Primera or Nimble Storage) to use for the rest of the exercises. While the HPE CSI Driver supports connectivity to multiple backends, configurating multiple backends is outside of the scope of this lab guide.

## Creating a Secret

Once the HPE CSI Driver has been deployed, a `Secret` needs to be created in order for the CSI driver to communicate to the HPE Primera or Nimble Storage. This `Secret`, which contains the storage system IP and credentials, is used by the CSI driver sidecars within the `StorageClass` to authenticate to a specific backend for various CSI operations. For more information, see [adding an HPE storage backend](https://scod.hpedev.io/csi_driver/deployment.html#add_an_hpe_storage_backend) 

Here is an example `Secret`.

```markdown
apiVersion: v1
kind: Secret
metadata:
  name: custom-secret
  namespace: hpe-storage 
stringData:
  serviceName: primera3par-csp-svc 
  servicePort: "8080"
  backend: 10.10.0.2
  username: <user>
  password: <password>
```

Download and modify, using the text editor of your choice, the `Secret` file with the **backend** IP per your environment.

```markdown fct_label="Nimble Storage"
wget https://raw.githubusercontent.com/hpe-storage/scod/master/docs/learn/persistent_storage/yaml/nimble-secret.yaml
```

```markdown fct_label="HPE Primera"
wget https://raw.githubusercontent.com/hpe-storage/scod/master/docs/learn/persistent_storage/yaml/primera-secret.yaml
```

Save the file and create the `Secret` within the cluster.

```markdown fct_label="Nimble Storage"
kubectl create -f nimble-secret.yaml
```

```markdown fct_label="HPE Primera"
kubectl create -f primera-secret.yaml
```

The `Secret` should now be available in the "hpe-storage" `Namespace`:

```markdown 
kubectl -n hpe-storage get secret/custom-secret
NAME                     TYPE          DATA      AGE
custom-secret            Opaque        5         1m
```

If you made a mistake when creating the `Secret`, simply delete the object (`kubectl -n hpe-storage delete secret/custom-secret`) and repeat the steps above.

## Creating a StorageClass

Now we will create a `StorageClass` that will be used in the following exercises. A `StorageClass` (SC) specifies which storage provisioner to use (in our case the HPE CSI Driver) and the volume parameters (such as Protection Templates, Performance Policies, CPG, etc.) for the volumes that we want to create which can be used to differentiate between storage levels and usages. 

This concept is sometimes called “profiles” in other storage systems. A cluster can have multiple `StorageClasses` allowing users to create storage claims tailored for their specific application requirements.

We will start by creating a `StorageClass` called **hpe-standard**. We will use the **custom-secret** created in the previous step and specify the **hpe-storage** `namespace` where the CSI driver was deployed.

Here is an example `StorageClasses` for HPE Primera and Nimble Storage systems and some of the available volume parameters that can be defined. See the respective [CSP](../../container_storage_provider/index.md) for more elaborate examples.

```markdown fct_label="HPE Nimble Storage"
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hpe-standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: csi.hpe.com
parameters:
  csi.storage.k8s.io/fstype: xfs
  csi.storage.k8s.io/provisioner-secret-name: custom-secret
  csi.storage.k8s.io/provisioner-secret-namespace: hpe-storage
  csi.storage.k8s.io/controller-publish-secret-name: custom-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: hpe-storage
  csi.storage.k8s.io/node-stage-secret-name: custom-secret
  csi.storage.k8s.io/node-stage-secret-namespace: hpe-storage
  csi.storage.k8s.io/node-publish-secret-name: custom-secret
  csi.storage.k8s.io/node-publish-secret-namespace: hpe-storage
  csi.storage.k8s.io/controller-expand-secret-name: custom-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: hpe-storage
  performancePolicy: "SQL Server"
  description: "Volume from HPE CSI Driver"
  accessProtocol: iscsi
  limitIops: "76800"
  allowOverrides: description,limitIops,performancePolicy
allowVolumeExpansion: true
```

```markdown fct_label="HPE Primera"
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hpe-standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: csi.hpe.com
parameters:
  csi.storage.k8s.io/fstype: xfs
  csi.storage.k8s.io/provisioner-secret-name: custom-secret
  csi.storage.k8s.io/provisioner-secret-namespace: hpe-storage
  csi.storage.k8s.io/controller-publish-secret-name: custom-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: hpe-storage
  csi.storage.k8s.io/node-stage-secret-name: custom-secret
  csi.storage.k8s.io/node-stage-secret-namespace: hpe-storage
  csi.storage.k8s.io/node-publish-secret-name: custom-secret
  csi.storage.k8s.io/node-publish-secret-namespace: hpe-storage
  csi.storage.k8s.io/controller-expand-secret-name: custom-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: hpe-storage
  cpg: SSD_r6
  provisioningType: tpvv
  accessProtocol: iscsi
  allowOverrides: cpg,provisioningType
allowVolumeExpansion: true
```

Create the `StorageClass` within the cluster
```markdown fct_label="Nimble Storage"
kubectl create -f http://scod.hpedev.io/learn/persistent_storage/yaml/nimble-storageclass.yaml
```

```markdown fct_label="Primera"
kubectl create -f http://scod.hpedev.io/learn/persistent_storage/yaml/primera-storageclass.yaml
```

We can verify the `StorageClass` is now available.

```markdown
kubectl get sc
NAME                     PROVISIONER   AGE
hpe-standard (default)   csi.hpe.com   2m
```

!!! note 
    You can create multiple `StorageClasses` to match the storage requirements of your applications. We set **hpe-standard** `StorageClass` as default using the annotation `storageclass.kubernetes.io/is-default-class: "true"`. There can only be one **default** `StorageClass` per cluster, for any additional `StorageClasses` set this to **false**. To learn more about configuring a default `StorageClass`, see [Default StorageClass](https://kubernetes.io/docs/tasks/administer-cluster/change-default-storage-class/) on kubernetes.io. 