# Let's create workloads

The virtual machine we're going to create will have the following properties-
- We are going to create a machine called rhel8-server-ocs.
- Use RHEL8 templet
- We are going to create a clusterIP service and Loadbalancer service 
- Lastly, we will do Live Migrate to the instance move to another node (we will explore this in more depth in a later lab).

<img width="724" alt="Screen Shot 2022-07-17 at 23 17 02" src="https://user-images.githubusercontent.com/64369864/179423383-16f3bc10-1084-4e28-b48e-deca02eed4f7.png">

Once the OpenShift virtualization operator is installed under the workloads tab found the virtualization option, Now virtual machines can be created. Has to be repeated for each VMs in the project.
- Click Workloads → Virtualization → Create Virtual Machine.
- Choose a template, In this case, Red Hat Enterprise Linux 8.0+ was chosen.

<img width="652" alt="Screen Shot 2022-07-17 at 23 21 17" src="https://user-images.githubusercontent.com/64369864/179423527-41e2432a-56ce-45bc-a4da-cb8032a6a297.png">

<img width="652" alt="Screen Shot 2022-07-17 at 23 21 49" src="https://user-images.githubusercontent.com/64369864/179423545-0f953cbe-2419-40a0-9576-f0dc1367392b.png">

- Provide a custom boot source and PVC size. In this case, Red Hat Enterprise Linux qcow2 image from the [RHEL download page](https://access.cdn.redhat.com/content/origin/files/sha256/b2/b2bfd0abc871143b97e6648183e69987b6bc2aded58972a866f967fd355d0e9a/rhel-8.5-update-2-x86_64-kvm.qcow2?user=413f1dbc5f8e9da85b56b5a47d6187e5&_auth_=1658103754_63115abb1c7dde03003e604914959fbf) and 20 GiB PVC size was chosen. Starting from OpenShift Virtualization 4.10, the manual creation of the is not required anymore, because the boot source for RHEL will be provided out of the box.

<img width="652" alt="Screen Shot 2022-07-17 at 23 24 35" src="https://user-images.githubusercontent.com/64369864/179423614-f7467ea9-0a48-4ad1-98b1-770c21b59aef.png">

<img width="662" alt="Screen Shot 2022-07-17 at 23 25 05" src="https://user-images.githubusercontent.com/64369864/179423628-6fb6cf01-ded5-4b36-ae2b-68a82be00864.png">

Make sure the workload is created and in a `running` state by following commands

```
oc get vm
```

> This command will list the VirtualMachine objects

```
NAME               AGE   STATUS     READY
< name >            4s    Starting   False
```

Now execute following command to list the instance of that virtual machine object

```
oc get vmi
```

> This command will list the VirtualMachineInstance objects

```
NAME               AGE   PHASE     IP    NODENAME                       READY
< name >           15s   Running         < node name >                   True
```

> NOTE: A vm object is the definition of the virtual machine, whereas a vmi is an instance of that virtual machine definition. In addition, you need to have the qemu-guest-agent installed in the guest for the IP address to show in this list, and it may take a minute or two to appear.

What you'll find is that OpenShift spawns a pod that manages the provisioning of the virtual machine in our environment, known as the virt-launcher.

```
oc get pods
```

> Check whether virt-launcher is running

```
NAME                                   READY   STATUS    RESTARTS   AGE
< name >                               1/1     Running   0          3m5s
```

Then execute following to describe the details

```
oc describe pod < virt launcher name >
```

When you look into this launcher pod, you'll see that it has the same libvirt functionality we find in existing Red Hat virtualisation products like RHV and OpenStack. First get a shell on the pod that's operating our virtual machine, recalling that each VM has a virt-launcher pod associated to it

```
oc get pods
```

> See the pod is running

Execute following to open a bash shell in the pod

```
oc exec -it  < virt launcher name > bash
```

And then you can run the usual virsh commands

```
virsh list --all
```

> Verify the process is running

```
 Id   Name                       State
------------------------------------------
 1   < name >                   running
 ```
 
 We can also verify the storage attachment, which should be an RBD volume
 
``` 
virsh domblklist < name >
```

> This command will list the block devices

 Target   Source
--------------------------
 sda      /dev/< Source >
 
And execute the following to check block device

`lsblk /dev/< Source >`

This command will list information about specified block device:

```
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
rbd1     251:16   0   40G  0 disk 
└─rbd1p1 251:17   0  7.8G  0 part
```

And for networking

`virsh domiflist < name >`

Exit the shell before proceeding

`exit`

Then ensure you are in the default project

`oc project default`

That's it for deploying basic workloads as we've successfully deployed a VM!
