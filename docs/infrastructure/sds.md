# Using Software Defined Storage

There are many types of Software Defined Storage (SDS) implementations in the market, and they may apply with different types of compute technology.  The purpose of this section is to document how to install and configure [Portworx Enterprise](https://cloud.ibm.com/catalog/services/portworx-enterprise) with [OpenShift on IBM Cloud](https://cloud.ibm.com/kubernetes/catalog/create?platformType=openshift).  Portworx acts as a storage layer in kubernetes that sits on top of some underlying physical storage attached to the worker nodes in the cluster.  It also provides additional storage classes for creating persistent volumes in Portworx.   Additional information on using Portworx with OpenShift on IBM Cloud can be found in the [documentation](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx).

_**Note:** These instructions were tested with OpenShift 4.5.13 running in [Virtual Private Cloud](https://cloud.ibm.com/vpc-ext/overview) and assume that you already have an OpenShift cluster provisioned._

## Preparing for Portworx

Review the documentation for [Planning your Portworx setup](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#portworx_planning) to verify that your existing cluster meets the requirements.  If not, create a new cluster.

_**Note:** It would appear that the provisioning page for Portworx in the IBM Cloud Catalog does some sort of requirements checking after you supply an API key.  It did not recognize my cluster when it had `bx2.8x32` nodes, but id did when I added `bx2.16x64` nodes._

There are two common topologies for adding SDS to your cluster.  The first one is called `hyper-converged`, where every worker node in the cluster has attached storage for Portworx.  This is the topology with the best performance, but it requires you to have storage attached to every worker node.

The other topology is called `storage-heavy`; in this topology only some of the worker nodes have attached storage.  Portworx will still make the storage available to pods running on any worker node in the cluster, but for pods running on non-SDS nodes the storage access requests will be routed on the private network to one of the SDS nodes.  



_**Note:** As far as I can tell the installation and configuration of Portworx is the same either way, but that may not be true.  For purposes of this exercise I have created an OpenShift cluster with 3 worker nodes in `US South`, with one worker node in each zone._




### Creating block storage

Your worker nodes will need to have block storage volumes attached to them in order for Portworx to work.  See the [documentation](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#portworx_planning) for more details.  Since these instuctions are for a cluster in VPC, follow these steps to create and attach the block storage volumes to your worker nodes.  Documentation can also be found [here](https://cloud.ibm.com/docs/openshift?topic=openshift-utilities#vpc_api_attach).

You can create the block storage volumes using the [IBM Cloud UI](https://cloud.ibm.com/vpc-ext/storage/storageVolumes).  Make sure to create the block storage volume in the same Availability zone as the worker node to which you want to attach it.  For visibility purposes, you should consider a naming convention that will help you determine where the storage volumes are being used.  In this example the name of the storage volume includes the `worker ID` of the worker node to which it will be attached.

If you don't want to look up the worker nodes in the UI you can run this command to get a list of your worker nodes:

`ic oc worker ls -c <cluster-name>`

The output should look like this:

```
OK
ID                                                    Primary IP    Flavor     State    Status   Zone         Version   
kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000162   10.1.128.12   bx2.8x32   normal   Ready    us-south-3   4.5.13_1515_openshift   
kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000209   10.1.120.6    bx2.8x32   normal   Ready    us-south-2   4.5.13_1515_openshift   
kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000386   10.1.112.17   bx2.8x32   normal   Ready    us-south-1   4.5.13_1515_openshift
```

_**Note:** As you create each storage volume, take note of it's `volume ID`.  You will need it in the next step._


### Attaching block storage to worker nodes

In order for Portworx to work it needs the storage volumes created above to be attached to the worker nodes.  The [documentation](https://cloud.ibm.com/docs/openshift?topic=openshift-utilities#vpc_api_attach) was used to derive these instructions.


Make sure you are logged into the IBM Cloud CLI.

Run this command to fetch your oauth token and store it in an environment variable:

`export iamtoken=$(ibmcloud iam oauth-tokens)`  <--- This does not work right!!  Need to figure out the right command.

In this example the cluster was created in `US South`, with worker nodes in `us-south-`, `us-south-2` and `us-south-3`.  Block storage volumes of equal size were also created in those three zones.

Since worker nodes are not visible in the IBM Cloud portal we cannot attach the storage using normal methods in the UI or CLI.  In order to attach the volumes to the worker nodes we have to use the IKS API.  The format of the API call looks like this:

```
 curl -X POST -H "Authorization: $iamtoken" "https://<region>.containers.cloud.ibm.com/v2/storage/vpc/createAttachment?cluster=<cluster_ID>&worker=<worker_ID>&volumeID=<volume_ID>"
```

Use this command to get your worker IDs:

```
ic oc worker ls -c <cluster-name>
```

The output should look like this:

```
OK
ID                                                    Primary IP    Flavor     State    Status   Zone         Version   
kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000162   10.1.128.12   bx2.8x32   normal   Ready    us-south-3   4.5.13_1515_openshift   
kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000209   10.1.120.6    bx2.8x32   normal   Ready    us-south-2   4.5.13_1515_openshift   
kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000386   10.1.112.17   bx2.8x32   normal   Ready    us-south-1   4.5.13_1515_openshift
```

The volume IDs should have been collected above when you created the volumes.  If not, you can get them from the UI.

_**Note:** Make sure to update this section with a CLI command to get the volumes!_

Run the cURL command above for each worker node and volume.  For example:

```
curl -X POST -H "Authorization: $iamtoken" "https://us-south.containers.cloud.ibm.com/v2/storage/vpc/createAttachment?cluster=bu6rbsid0j80c74a6u2g&worker=kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000386&volumeID=r006-0be0c31c-3d1f-4931-bc4e-c554b4d3fa58"
```

The output should look like this:

```json
{"id":"0717-85669abb-a268-4fff-95ca-5c8ae953af0e","volume":{"name":"px-kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000386","id":"r006-0be0c31c-3d1f-4931-bc4e-c554b4d3fa58"},"device":{"id":""},"name":"volume-attachment","status":"attaching","type":"data"}
```


To review the volume attachments for a worker node you can run this command:

```
curl -X GET -H "Authorization: <IAM_token>" -H "Content-Type: application/json" -H "X-Auth-Resource-Group-ID: <resource_group_ID>" "https://<region>.containers.cloud.ibm.com/v2/storage/clusters/<cluster_ID>/workers/<worker_ID>/volume_attachments"
```

For example:

```
curl -X GET -H "Authorization: $iamtoken" -H "Content-Type: application/json" -H "X-Auth-Resource-Group-ID: 5e6a2492dd9249659469ea51da471197" "https://us-south.containers.cloud.ibm.com/v2/storage/vpc/getAttachmentsList?cluster=bu6rbsid0j80c74a6u2g&worker=kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000386"
```

The output should look like this:

```json
{"volume_attachments":[{"id":"0717-85669abb-a268-4fff-95ca-5c8ae953af0e","volume":{"name":"px-kube-bu6rbsid0j80c74a6u2g-sbxocp11-default-00000386","id":"r006-0be0c31c-3d1f-4931-bc4e-c554b4d3fa58"},"device":{"id":"0717-85669abb-a268-4fff-95ca-5c8ae953af0e-9nh8x"},"name":"volume-attachment","status":"attached","type":"data"},{"id":"0717-6acdae87-58cd-4a05-acfa-b178c3544c31","volume":{"name":"hypnotist-freezing-willow-display","id":"r006-a354ecea-4cc3-4445-9431-12cd9ce569a8"},"device":{"id":"0717-6acdae87-58cd-4a05-acfa-b178c3544c31-fzd6s"},"name":"volume-attachment","status":"attached","type":"boot"}]}
```

Provision an instance of [Databases for Etcd](https://cloud.ibm.com/catalog/services/databases-for-etcd) in the same resource group as the OpenShift cluster.  For visibility purposes, consider a naming convention that includes the cluster name, such as `px-sbx-ocp-11`, and add a tag with the name of the cluster as well.  Review the [documentation](https://cloud.ibm.com/docs/openshift?topic=openshift-portworx#databases-for-etcd) for more infomration on configuration options.

Follow the steps in the documentation.

When it comes to encoding the username and password, I found that the instructions did not work, as they added the `-n` from the command into the output.  I used this command:

```
echo "<the username or password>" | base64
```

At this point the Databases for Etcd instance is created, the username and password have been encoded and the secret has been created.

Now we need to [copy the image pull secrets](https://cloud.ibm.com/docs/openshift?topic=openshift-registry#copy_imagePullSecret) and [add them to the service account](https://cloud.ibm.com/docs/openshift?topic=openshift-registry#store_imagePullSecret).

### Copy the image pull secrets

To list the secrets in the `default` namespace:

```
oc get secrets -n default | grep icr-io
```

The output should look like this:

```
all-icr-io                 kubernetes.io/dockerconfigjson        1         115m
```

To copy the `all-icr-io` secret to the `kube-system` namespace:

```
oc get secret all-icr-io -n default -o yaml | sed 's/default/kube-system/g' | oc create -n kube-system -f -
```

The output should look like this:

```
secret/all-icr-io created
```

### Store the pull secret in the service account

Check if the image pull secret already exists for the default service account:

```
oc describe serviceaccount default -n kube-system
```

The output should look like this:

```
Name:                default
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  default-dockercfg-bkxjw
Mountable secrets:   default-token-fbhwh
                     default-dockercfg-bkxjw
Tokens:              default-token-fbhwh
                     default-token-wsnrg
Events:              <none>
```

Notice that the `Image pull secrets` sction only lists `default-dockercfg-bkxjw`.  This means that the image pull secrets are already defined, but do not yet contain `all-icr-io`.  

Run this command to add it:

```
oc patch -n kube-system serviceaccount/default --type='json' -p='[{"op":"add","path":"/imagePullSecrets/-","value":{"name":"all-icr-io"}}]'
```

The output should look like this:

```
serviceaccount/default patched
```

You can verify that it worked using this command:

```
oc describe serviceaccount default -n kube-system
```

The output should look like this:

```
Name:                default
Namespace:           kube-system
Labels:              <none>
Annotations:         <none>
Image pull secrets:  default-dockercfg-bkxjw
                     all-icr-io
Mountable secrets:   default-token-fbhwh
                     default-dockercfg-bkxjw
Tokens:              default-token-fbhwh
                     default-token-wsnrg
Events:              <none>
```

Notice that now `all-icr-io` shows up in the `Image pull secrets` section.












