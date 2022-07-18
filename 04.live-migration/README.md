Live Migration is the process of moving an instance from one node in a cluster to another without interruption. This process can be manual or automatic. In OpenShift this is controlled by an `evictionStrategy` strategy. If this is set to `LiveMigrate` and the underlying node are placed into maintenance mode, VMs can be automatically moved between nodes with minimal interruption. 

Live migration is an administrative function in OpenShift Virtualization. While the action is visible to all users, only admins can initiate a migration. Migration limits and timeouts are managed via the `kubevirt-config` `configmap`. For more details about limits see the [documentation](https://docs.openshift.com/container-platform/4.10/virt/live_migration/virt-live-migration-limits.html).

In our lab we currently only have one VM running, use the `vmi` utility to view it:

```execute-1
oc get vmi
```

You should see something similar to the output below

~~~bash
NAME        AGE   PHASE     IP       NODENAME          READY
< name >    45h   Running   < IP >  < node nmae >.     True
~~~

> **NOTE**: In OpenShift Virtualization, the "Virtual Machine" object can be thought of as the virtual machine "source" that virtual machine instances are created from. A "Virtual Machine Instance" is the actual running instance of the virtual machine. The instance is the object you work with that contains IP, networking, and workloads, etc. That's why we delete a VM, and list VMI's.

As you may recall we deployed this instance with the `LiveMigrate` `evictionStrategy` strategy but you can also review an instance with `oc describe` to ensure it is enabled.


```execute-1
oc describe vmi < name > | egrep '(Eviction|Migration)'
```

This command should have a similar output as below, although trimmed:

~~~yaml
  Eviction Strategy:  LiveMigrate
  Migration Method:  LiveMigration
~~~

The easiest way to initiate a migration is to create an `VirtualMachineInstanceMigration` object in the cluster directly against the `vmi` we want to migrate (this can also be conducted via the OpenShift console if we like, and we'll take a look at it in a later lab section). But wait! Once we create this object it will immediatly trigger the migration, so first, let's see what the code looks like:

~~~yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: < name >
~~~

It's really quite simple, we create a `VirtualMachineInstanceMigration` object and reference the `LiveMigratable ` instance we want to migrate. Let's apply this configuration:

```execute-1
cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: < name >
EOF
```

Check that the `VirtualMachineInstanceMigration` object is created:

~~~bash
virtualmachineinstancemigration.kubevirt.io/migration-job created
~~~

Now let's watch the migration job in action. First it will show `phase: Scheduling`

```execute-1
watch -n1 oc get virtualmachineinstancemigration/migration-job -o yaml
```

With the output:

~~~bash

apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
(...)
spec:
  vmiName: < name >
status:
  phase: Scheduling    <----------- Here you can see it's scheduling
~~~

And then move to `phase: TargetReady` and onto `phase: Succeeded`

~~~bash

apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
(...)
spec:
  vmiName: < name >
status:
  phase: Succeeded    <----------- Now it has finished the migration
~~~

Finally view the `vmi` object and you can see the new underlying host (was *ocp4-worker3*, now it's *ocp4-worker1*); your environment may have different source and destination hosts, depending on where `rhel8-server-ocs` was initially scheduled. Don't forget to `ctrl-c` out of the running watch command:

```execute-1
oc get vmi
```

Now check the output:

~~~bash
NAME        AGE   PHASE     IP       NODENAME          READY
< name >    45h   Running   < IP >  < node nmae >.     True
~~~

As you can see Live Migration in OpenShift Virtualization is quite easy. If you have time, try some other examples. Perhaps start a ping and migrate the machine back. Do you see anything in the ping to indicate the process?

> **NOTE**: If you try and run the same migration job it will report `unchanged`. To run a new job, run the same example as above, but change the job name in the metadata section to something like `name: migration-job2`

Also, rerun the `oc describe vmi < name >` after running a few migrations. You'll see the object is updated with details of the migrations, including source and target:

```execute-1
oc describe vmi < name >
```

## Node Maintenance

Building on-top of live migration, many organisations will need to perform node-maintenance, e.g. for software/hardware updates, or for decommissioning. During the lifecycle of a **pod**, it's almost a given that this will happen without compromising the workloads, but virtual machines can be somewhat more challenging given their nature. To address this OpenShift Virtualization has a node-maintenance feature which manages the process safely by marking nodes unschedulable and migrating workloads automatically.

Let's take a look at the current running virtual machines and the nodes we have available:

```execute-1
oc get nodes
```

This lists the OpenShift nodes:

~~~bash
NAME                           STATUS   ROLES    AGE   VERSION
ocp4-master1.aio.example.com   Ready    master   2d    v1.22.0-rc.0+a44d0f0
ocp4-master2.aio.example.com   Ready    master   2d    v1.22.0-rc.0+a44d0f0
ocp4-master3.aio.example.com   Ready    master   2d    v1.22.0-rc.0+a44d0f0
ocp4-worker1.aio.example.com   Ready    worker   2d    v1.22.0-rc.0+a44d0f0
ocp4-worker2.aio.example.com   Ready    worker   2d    v1.22.0-rc.0+a44d0f0
ocp4-worker3.aio.example.com   Ready    worker   2d    v1.22.0-rc.0+a44d0f0
~~~

Now check the VMIs:

```execute-1
oc get vmi
```

You should see the one VM running:

~~~bash
NAME               AGE   PHASE     IP               NODENAME                       READY
rhel8-server-ocs   45h   Running   192.168.123.64   ocp4-worker1.aio.example.com   True
~~~

In this environment, we have one virtual machine running on *ocp4-worker1* (yours may vary). Let's take down the node for maintenance and ensure that our workload (VM) stays up and running:

> **NOTE**: You may need to modify the below command to specify the worker listed in the output from above.  

```copy
cat << EOF | oc apply -f -
apiVersion: nodemaintenance.kubevirt.io/v1beta1
kind: NodeMaintenance
metadata:
  name: worker1-maintenance
spec:
  nodeName: ocp4-worker1.aio.example.com
  reason: "Worker1 Maintenance"
EOF
```

See that the `NodeMaintenance` object is created:

~~~bash
nodemaintenance.nodemaintenance.kubevirt.io/worker1-maintenance
~~~

> **NOTE**: You **may** lose your browser based web terminal like this:
>
> ![live-mirate-terminal-closed](img/live-mirate-terminal-closed.png)
>
> If this happens you'll need to wait a few seconds for it to become accessible again. Try reloading the terminal from the drop down menu in the upper right of the browser. This is because the OpenShift router and/or workbook pods may be running on the worker you put into maintenance.

Assuming you're connected back in, let's check the status of our environment:

```execute-1
oc project default
```

Ensure you are in the default project:

~~~bash
Now using project "default" on server "https://172.30.0.1:443".
~~~

And check the nodes:

```execute-1
oc get nodes
```

Notice that scheduling is disabled for `worker1` (or the worker that you specified maintenance for): 

~~~bash
NAME                           STATUS                     ROLES    AGE   VERSION
ocp4-master1.aio.example.com   Ready                      master   2d    v1.22.0-rc.0+a44d0f0
ocp4-master2.aio.example.com   Ready                      master   2d    v1.22.0-rc.0+a44d0f0
ocp4-master3.aio.example.com   Ready                      master   2d    v1.22.0-rc.0+a44d0f0
ocp4-worker1.aio.example.com   Ready,SchedulingDisabled   worker   2d    v1.22.0-rc.0+a44d0f0
ocp4-worker2.aio.example.com   Ready                      worker   2d    v1.22.0-rc.0+a44d0f0
ocp4-worker3.aio.example.com   Ready                      worker   2d    v1.22.0-rc.0+a44d0f0
~~~


Now check the VMI:

```execute-1
oc get vmi
```

Note that the VM has been **automatically** live migrated back to an available worker and is not on the `SchedulingDisabled` worker, as per the `EvictionStrategy`, in this case `ocp4-worker3.aio.example.com`. 

~~~bash
NAME               AGE   PHASE     IP               NODENAME                       READY
rhel8-server-ocs   46h   Running   192.168.123.64   ocp4-worker3.aio.example.com   True
~~~


We can remove the maintenance flag by simply deleting the `NodeMaintenance` object - update this to reflect the nodes in your environment:


```execute-1
oc get nodemaintenance
```
It should just show the one:

~~~bash
NAME                  AGE
worker1-maintenance   5m16s
~~~

Now delete it:

> **NOTE**: You may need to modify the below command to specify your `nodemaintenance` object is the same as in the output from above.

```copy
oc delete nodemaintenance/worker1-maintenance
```

It should return the following output:

~~~bash
nodemaintenance.nodemaintenance.kubevirt.io "worker1-maintenance" deleted
~~~

Then check the same node again:

```copy
oc get node/ocp4-worker1.aio.example.com
```

Note the removal of the `SchedulingDisabled` annotation on the '**STATUS**' column. Also important is that just because this node has become active again doesn't mean the virtual machine returns to it automatically, i.e. it won't "fail back", it will reside on the new host: 

~~~bash
NAME                           STATUS   ROLES    AGE   VERSION
ocp4-worker1.aio.example.com   Ready    worker   2d    v1.22.0-rc.0+a44d0f0
~~~

Before proceeding let's remove the `rhel8-server-ocs` virtual machine as well as any lingering PVC's we don't need any longer:

```execute-1
oc delete vm/rhel8-server-ocs
```

This should be confirmed with:

~~~bash
virtualmachine.kubevirt.io "rhel8-server-ocs" deleted
~~~

Now delete the PVCs:

```execute-1
oc delete pvc rhel8-ocs
```

It should show the removal:

~~~bash
persistentvolumeclaim "rhel8-ocs" deleted
~~~

Choose "**Clone a Virtual Machine**" to continue with the lab.
