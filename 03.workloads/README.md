# Let's create workloads

The virtual machine we're going to create will have the following properties
- We are going to create a machine.
- We are going to use RHEL8 templet
- Lastly, we will do Live Migrate to the instance move to another node (we will explore this in more depth in a later lab).

<img width="687" alt="Screen Shot 2022-07-17 at 23 59 44" src="https://user-images.githubusercontent.com/64369864/179424557-6288110b-2a5a-407f-87bb-ae64da849b0f.png">

Once the OpenShift virtualization operator is installed under the workloads tab found the virtualization option, Now virtual machines can be created. Has to be repeated for each VMs in the project.
- Click Workloads → Virtualization → < Your project > → Create Virtual Machine.
- Choose a template, In this case, Red Hat Enterprise Linux 8.0+ was chosen.

<img width="1102" alt="Screen Shot 2022-07-19 at 2 41 26" src="https://user-images.githubusercontent.com/64369864/179634812-f1644214-cc37-4d00-9366-8dd43530decb.png">

<img width="1271" alt="Screen Shot 2022-07-19 at 2 42 20" src="https://user-images.githubusercontent.com/64369864/179634894-9fba2911-122f-456e-a38a-de8ac95dec97.png">

- Provide a custom boot source and PVC size. In this case, we use our PVC from the preview step. Starting from OpenShift Virtualization 4.10, the manual creation of the is not required anymore, because the boot source for RHEL will be provided out of the box

> Another option is get the `Red Hat Enterprise Linux` qcow2 image from the [RHEL download page](https://access.redhat.com/downloads/content/479/ver=/rhel---8/8.5/x86_64/product-software). (In this case, we won't use this option, but we wanted to let you know it exists)

![image](https://user-images.githubusercontent.com/64369864/180616946-35fdfe93-0c5e-4159-8f0d-2ee66127cdf2.png)

![image](https://user-images.githubusercontent.com/64369864/180616987-b37daf39-a1e4-459b-bfda-35cbebe04c19.png)

<img width="662" alt="Screen Shot 2022-07-17 at 23 25 05" src="https://user-images.githubusercontent.com/64369864/179423628-6fb6cf01-ded5-4b36-ae2b-68a82be00864.png">

<img width="1423" alt="Screen Shot 2022-07-23 at 20 55 00" src="https://user-images.githubusercontent.com/64369864/180617181-10179749-6acf-41c9-930a-2b17e6bbeda9.png">

Click Virtualization → Virtual Machines → < Your VM > → Console

<img width="1006" alt="Screen Shot 2022-07-23 at 21 32 55" src="https://user-images.githubusercontent.com/64369864/180618481-74a55c42-5fd2-4bd9-9654-f13c621c8f01.png">

Let's check your VM from the CLI. You need to use your project by following commands

```
oc project < workshop >
```

Make sure the workload is created and in a `running` state by following commands

```
oc get vm
```

> This command will list the VirtualMachine objects

```
NAME             AGE   STATUS    READY
rhel8-aelfassy   21m   Running   True
```

Now execute following command to list the instance of that virtual machine object

```
oc get vmi
```

> This command will list the VirtualMachineInstance objects

```
NAME             AGE   PHASE     IP            NODENAME                                                 READY
rhel8-aelfassy   22m   Running   10.131.0.53   workshop-n54ln-worker-b-cqvfp.c.almog-elfassy.internal   True
```

> NOTE: A vm object is the definition of the virtual machine, whereas a vmi is an instance of that virtual machine definition. In addition, you need to have the qemu-guest-agent installed in the guest for the IP address to show in this list, and it may take a minute or two to appear.

What you'll find is that OpenShift spawns a pod that manages the provisioning of the virtual machine in our environment, known as the virt-launcher.

```
oc get pods
```

> Check whether virt-launcher is running

```
NAME                                 READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-aelfassy-kw2sf   1/1     Running   0          23m
```

Then execute following to describe the details

```copy
oc describe pod < virt-launcher-rhel8-aelfassy-kw2sf >
```

This command will list pod details in yaml format

~~~yaml
Name:         virt-launcher-rhel8-aelfassy-kw2sf
Namespace:    workshop
Priority:     0
Node:         workshop-n54ln-worker-b-cqvfp.c.almog-elfassy.internal/10.0.128.2
Start Time:   Sat, 23 Jul 2022 17:34:50 +0000
Labels:       flavor.template.kubevirt.io/small=true
              kubevirt.io=virt-launcher
              kubevirt.io/created-by=b68b2615-5b8b-4c92-9d68-d561e97939ae
              kubevirt.io/domain=rhel8-aelfassy
              kubevirt.io/size=small
              os.template.kubevirt.io/rhel7.9=true
              vm.kubevirt.io/name=rhel8-aelfassy
              workload.template.kubevirt.io/server=true
Annotations:  k8s.v1.cni.cncf.io/network-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.131.0.53"
                    ],
                    "default": true,
                    "dns": {}
                }]
              k8s.v1.cni.cncf.io/networks-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.131.0.53"
                    ],
                    "default": true,
                    "dns": {}
                }]
(...)
~~~

When you look into this launcher pod, you'll see that it has the same libvirt functionality we find in existing Red Hat virtualisation products like RHV and OpenStack. First get a shell on the pod that's operating our virtual machine, recalling that each VM has a virt-launcher pod associated to it

```
oc get pods
```

> See the pod is running

~~~bash
NAME                                 READY   STATUS    RESTARTS   AGE
virt-launcher-rhel8-aelfassy-kw2sf   1/1     Running   0          28m
~~~

Execute following to open a bash shell in the pod

```
oc exec -it < virt-launcher-rhel8-aelfassy-kw2sf > bash
```

And then you can run the usual virsh commands

```
virsh list --all
```

> Verify the process is running

```
 Id   Name                      State
-----------------------------------------
 1    workshop_rhel8-aelfassy   running
 ```
 
 We can also verify the storage attachment, which should be an RBD volume
 
``` 
virsh domblklist < workshop_rhel8-aelfassy  >
```

> This command will list the block devices

 Target   Source
-------------------------------------------------------------------------------------------------
 vda      /dev/rhel8-aelfassy
 vdb      /var/run/kubevirt-ephemeral-disks/cloud-init-data/workshop/rhel8-aelfassy/noCloud.iso
 
And execute the following to check block device

```

lsblk /dev/< rhel8-aelfassy >
```

This command will list information about specified block device:

~~~bash
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
rbd0     252:0    0   30G  0 disk
├─rbd0p1 252:1    0    1M  0 part
├─rbd0p2 252:2    0  100M  0 part
└─rbd0p3 252:3    0  9.9G  0 part
~~~

And for networking 

```

virsh domiflist <  workshop_rhel8-aelfassy >
```
This command will list information about network interfaces

~~~bash
 Interface   Type       Source   Model                     MAC
------------------------------------------------------------------------------
 tap0        ethernet   -        virtio-non-transitional   02:b8:f7:00:00:00
~~~

But let's go a little deeper with the following command

```execute-1
ip link | grep -A2 tap0
```

If we look at tap0 we'll see that it's part of a bridge called "*k6t-net0*":

~~~bash
5: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1410 qdisc fq_codel master k6t-eth0 state UP mode DEFAULT group default qlen 1000
    link/ether 5a:dd:38:3d:70:eb brd ff:ff:ff:ff:ff:ff
~~~

Let's look more deeply at that bridge

```execute-1
ip link | grep -A2 k6t-eth0
```

 You should see the details

~~~bash
4: k6t-eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1410 qdisc noqueue state UP mode DEFAULT group default
    link/ether 02:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
5: tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1410 qdisc fq_codel master k6t-eth0 state UP mode DEFAULT group default qlen 1000
    link/ether 5a:dd:38:3d:70:eb brd ff:ff:ff:ff:ff:ff
~~~

```execute-1

virsh dumpxml workshop_rhel8-aelfassy | grep -A8 "interface type"
```

This will show interface information in XML format

~~~xml
    <interface type='ethernet'>
      <mac address='02:b8:f7:00:00:00'/>
      <target dev='tap0' managed='no'/>
      <model type='virtio-non-transitional'/>
      <driver name='vhost'/>
      <mtu size='1410'/>
      <alias name='ua-default'/>
      <rom enabled='no'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
~~~

Exit the shell before proceeding

`exit`

Then ensure you are in the default project

`oc project default`

That's it for deploying basic workloads as we've successfully deployed a VM!

