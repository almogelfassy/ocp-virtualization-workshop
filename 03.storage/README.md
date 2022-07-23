Now that we've got OpenShift Virtualization deployed, let's take a look at storage. What you'll find is that OpenShift Data Foundation (ODF, although
formerly known as OpenShift Container Storage/OCS - a software storage platform based on [Ceph](https://www.redhat.com/en/technologies/storage/ceph)) has been deployed for you:

```execute-1
oc get sc
```

This command should show the list of Storage Classes that are installed:

~~~bash
NAME                                    PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
ocs-storagecluster-ceph-rbd (default)   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   4d13h
ocs-storagecluster-cephfs               openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   4d13h
openshift-storage.noobaa.io             openshift-storage.noobaa.io/obc         Delete          Immediate              false                  4d13h
standard                                kubernetes.io/gce-pd                    Delete          WaitForFirstConsumer   true                   4d15h
standard-csi                            pd.csi.storage.gke.io                   Delete          WaitForFirstConsumer   true                   4d15h
~~~

Then check which version of the odf operator is installed by executing the following:

```execute-1
oc get csv -n openshift-storage
```

Result should be similar to:

~~~bash
NAME                              DISPLAY                       VERSION   REPLACES                          PHASE
mcg-operator.v4.10.5              NooBaa Operator               4.10.5    mcg-operator.v4.10.4              Succeeded
ocs-operator.v4.10.5              OpenShift Container Storage   4.10.5    ocs-operator.v4.10.4              Succeeded
odf-csi-addons-operator.v4.10.5   CSI Addons                    4.10.5    odf-csi-addons-operator.v4.10.4   Succeeded
odf-operator.v4.10.5              OpenShift Data Foundation     4.10.5    odf-operator.v4.10.4              Succeeded
~~~

We're going to use the already-deployed ODF based shared storage for the majority of the tasks.

In the next few steps we're going to be creating a storage volume for a virtual machine that we'll create in a later lab section, and for this we'll pull in the contents of an existing disk image, so there's an operating system for our virtual machine to boot up. 

???????First, make sure you're in the default project:

```execute-1
oc project default
```

You should see:

~~~bash
Now using project "default" on server "https://api.workshop.aelfassy.com:6443".
~~~
?????????

>**NOTE**: If you don't use the default project for the next few lab steps, it's likely that you'll run into some errors - some resources are scoped, i.e. aligned to a namespace, and others are not. Ensuring you're in the default namespace now will ensure that all of the coming lab steps should flow together.

Now let's create a new ODF-based Peristent Volume Claim (PVC) - a request for a dedicated storage volume that can be used for persistent storage with VM's and containerised applications. For this volume claim we will use a special annotation `cdi.kubevirt.io/storage.import.endpoint` which utilises the Kubernetes Containerized Data Importer (CDI).

> **NOTE**: CDI is a utility to import, upload, and clone virtual machine images for OpenShift Virtualization. The CDI controller watches for this annotation on the PVC and if found it starts a process to import, upload, or clone. When the annotation is detected the `CDI` controller starts a pod which imports the image from that URL. Cloning and uploading follow a similar process. Read more about the Containerized Data Importer [here](https://github.com/kubevirt/containerized-data-importer).

Basically we are asking OpenShift to create this Persistent Volume Claim and use the image in the endpoint to fill it. In this case we use `"https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.5/x86_64/product-software"`.

In addition to triggering the CDI utility we also specify the storage class that OCS/ODF uses (`ocs-storagecluster-ceph-rbd`) which will dynamically create the PV in the backend Ceph storage platform. Finally note the `requests` section - we are asking for a 30GB volume size.

OK, let's create the PVC with all this included:

Download the `Red Hat Enterprise Linux 8.5 Update KVM Guest Image` QEMU image from [here](https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.5/x86_64/product-software)

![image](https://user-images.githubusercontent.com/64369864/180614252-f634ef3f-78de-48fd-8178-2aa7e894785a.png)


Once the image downloaded, move to the 'Storage' --> 'PersistentVolumeClaims' --> 'Create PersistentVolumeClaim' --> 'With Data un oad torm' 

![image](https://user-images.githubusercontent.com/64369864/180614282-f5a1ddd0-c2c0-4e9c-b1f9-5fb20d837b8e.png)

Upload the `Red Hat Enterprise Linux 8.5 Update KVM Guest Image` QEMU image and filed the info

![image](https://user-images.githubusercontent.com/64369864/180614366-cf234cdf-621d-4c6e-93e2-bc82a17af1de.png)

Once the uploading is completed the Persistent Volume Claim will become available and ready to use

![image](https://user-images.githubusercontent.com/64369864/180614421-aff72504-ac53-4e64-9e44-f8b66894f2ea.png)

![image](https://user-images.githubusercontent.com/64369864/180614454-9bc9ea76-1b09-4856-a5c5-0058d3a83af3.png)

You can check PVC readiness by executing the command below:

```execute-1 
oc get pvc
```

~~~bash
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS                  AGE
aelfassy-pvc   Bound    pvc-81f75bd9-aea2-4cf4-a303-bede96037c7d   30Gi       RWX            ocs-storagecluster-ceph-rbd   19m
~~~

This same configuration should be reflected when asking OpenShift for a list of *all* persistent volumes (`PV`):

```execute-1 
oc get pv
```

Noting that there will be some additional PV's that are used with OpenShift Data Foundation as well as the image registry

~~~bash
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                            STORAGECLASS                  REASON   AGE
pvc-055ee63c-8940-4486-9b00-13aa76b473e5   30Gi       RWO            Delete           Bound    openshift-virtualization-os-images/centos7-680e9b4e0fba          standard                               4d19h
pvc-0c2bcd66-6f76-49cc-8a72-db09cc630813   30Gi       RWO            Delete           Bound    openshift-virtualization-os-images/rhel9-e04dfadb2d71            standard                               4d19h
pvc-11687443-710e-4a5f-90a9-5e07660f1f37   50Gi       RWO            Delete           Bound    openshift-storage/db-noobaa-db-pg-0                              ocs-storagecluster-ceph-rbd            4d18h
pvc-1d7e7f23-f5e6-4160-8892-e5163869c078   2Ti        RWO            Delete           Bound    openshift-storage/ocs-deviceset-standard-1-data-0d5wx6           standard                               4d18h
pvc-1ed86975-20bf-4c76-83cb-5beecd993629   30Gi       RWX            Delete           Bound    openshift-virtualization-os-images/rhel8                         ocs-storagecluster-ceph-rbd            4d17h
pvc-2241b1fa-febd-487c-89a5-8383a0c090a1   30Gi       RWO            Delete           Bound    openshift-virtualization-os-images/rhel8-d8b84352ee28            standard                               4d19h
pvc-2388d225-0317-4397-834b-f2d01184584e   2Ti        RWO            Delete           Bound    openshift-storage/ocs-deviceset-standard-2-data-0lng2s           standard                               4d18h
pvc-246a3f03-dcab-47a1-8857-739f5c5cf75e   50Gi       RWO            Delete           Bound    openshift-storage/rook-ceph-mon-c                                standard                               4d18h
pvc-56d75bf1-f409-44bf-92d7-8fede2a3a8de   50Gi       RWO            Delete           Bound    openshift-storage/rook-ceph-mon-a                                standard                               4d18h
pvc-5ae20aba-516d-45cb-a581-e51835e7160c   30Gi       RWX            Delete           Bound    openshift-virtualization-os-images/centos-stream9-d4a96992f850   ocs-storagecluster-ceph-rbd            4d6h
pvc-5c9e27cd-74c7-45e1-af50-35ca322eb948   30Gi       RWO            Delete           Bound    openshift-virtualization-os-images/fedora-3b3fc310abea           standard                               4d19h
pvc-81f75bd9-aea2-4cf4-a303-bede96037c7d   30Gi       RWX            Delete           Bound    workshop/aelfassy-pvc                                            ocs-storagecluster-ceph-rbd            22m
pvc-82cb1d22-1ebf-4312-864a-6f1fad308814   50Gi       RWO            Delete           Bound    openshift-storage/rook-ceph-mon-b                                standard                               4d18h
pvc-a8a2f21e-d273-492a-8a80-50fef61d9f5d   30Gi       RWO            Delete           Bound    openshift-virtualization-os-images/centos-stream9-8f10faac35fa   standard                               4d19h
pvc-bfba65c9-8b1d-4c20-b5cf-86cf16e9de82   2Ti        RWO            Delete           Bound    openshift-storage/ocs-deviceset-standard-0-data-0dm6v5           standard                               4d18h
pvc-fff108e1-7c3a-432f-bf61-61ea3e797b63   30Gi       RWO            Delete           Bound    openshift-virtualization-os-images/centos-stream8-178796a2fbe3   standard                               4d19h
~~~

Let's take a look on the Ceph-backed storage system for more information about the image. We can do this by matching the PVC to the underlying RBD image. First describe the persistent volume to get the UUID of the image name by matching the ID of the PV for `aelfassy-pvc` in the output above. You'll have to manually enter these values for your environment. In the example above that value is `pvc-81f75bd9-aea2-4cf4-a303-bede96037c7d` 


```
oc describe pv/pvc-81f75bd9-aea2-4cf4-a303-bede96037c7d | grep imageName
                 imageName=csi-vol-c785dd3f-0aa3-11ed-b298-0a580a83000a
```

This gives us the imageName we need. Now we need to look at the image on the OpenShift cluster itself. We do this by first attaching to a special pod containing Ceph CLI tools, and then asking for details about the image in question

```execute-1 
oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch '[{ "op": "replace", "path":"/spec/enableCephTools","value":true}]'

```
This should create our new tools pod

```
ocsinitialization.ocs.openshift.io/ocsinit patched
```
Once created, move to the pod's terminal

```execute-1 
oc exec -it -n openshift-storage \
    $(oc get pods -n openshift-storage | awk '/tools/ {print $1;}') bash
```

Now, in the pod's terminal we can use the `rbd` command to inspect the image, noting we must specify the pool name "*ocs-storagecluster-cephblockpool*" as this is a "block" type of PV

```
rbd info csi-vol-c785dd3f-0aa3-11ed-b298-0a580a83000a \
          --pool=ocs-storagecluster-cephblockpool
```

This will display information about the image

~~~yaml
rbd image 'csi-vol-c785dd3f-0aa3-11ed-b298-0a580a83000a':
	size 30 GiB in 7680 objects
	order 22 (4 MiB objects)
	snapshot_count: 0
	id: 2373c16693ddb
	block_name_prefix: rbd_data.2373c16693ddb
	format: 2
	features: layering
	op_features:
	flags:
	create_timestamp: Sat Jul 23 16:23:26 2022
	access_timestamp: Sat Jul 23 16:23:26 2022
	modify_timestamp: Sat Jul 23 16:23:26 2022
~~~

Then execute an `rbd disk-usage` request against the same image to see the disk usage; don't forget to specifiy the correct pool

```
rbd disk-usage csi-vol-c785dd3f-0aa3-11ed-b298-0a580a83000a \
          --pool=ocs-storagecluster-cephblockpool
```

This will display the usage

~~~bash
NAME                                          PROVISIONED  USED
csi-vol-c785dd3f-0aa3-11ed-b298-0a580a83000a       30 GiB  1.7 GiB
~~~

That's it! Don't forget to exit from the pod when done.

```execute-1 
exit
```

You'll see that this image matches the correct size, corresponds to the PV that we requested be created for us, reflecting that we've cloned a copy of RHEL8.5 into our persistent volume via the data importer tool. We'll use this persistent volume to spawn a virtual machine in a later step.
