Live Migration is the process of moving an instance from one node in a cluster to another without interruption. This process can be manual or automatic. In OpenShift this is controlled by an `evictionStrategy` strategy. If this is set to `LiveMigrate` and the underlying node are placed into maintenance mode, VMs can be automatically moved between nodes with minimal interruption. 

Live migration is an administrative function in OpenShift Virtualization. While the action is visible to all users, only admins can initiate a migration. Migration limits and timeouts are managed via the `kubevirt-config` `configmap`. For more details about limits see the [documentation](https://docs.openshift.com/container-platform/4.10/virt/live_migration/virt-live-migration-limits.html).

In our lab we currently only have one VM running, use the `vmi` utility to view it:

```execute-1
oc get vmi
```

You should see something similar to the output below

~~~bash
NAME             AGE   PHASE     IP            NODENAME                                                 READY
rhel8-aelfassy   68m   Running   10.131.0.53   workshop-n54ln-worker-b-cqvfp.c.almog-elfassy.internal   True
~~~

> **NOTE**: In OpenShift Virtualization, the "Virtual Machine" object can be thought of as the virtual machine "source" that virtual machine instances are created from. A "Virtual Machine Instance" is the actual running instance of the virtual machine. The instance is the object you work with that contains IP, networking, and workloads, etc. That's why we delete a VM, and list VMI's.

As you may recall we deployed this instance with the `LiveMigrate` `evictionStrategy` strategy but you can also review an instance with `oc describe` to ensure it is enabled.


```execute-1
oc describe vmi < rhel8-aelfassy > | egrep '(Eviction|Migration)'
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
  vmiName: < rhel8-aelfassy >
~~~

It's really quite simple, we create a `VirtualMachineInstanceMigration` object and reference the `LiveMigratable ` instance we want to migrate. Let's apply this configuration:

```execute-1
cat << EOF | oc apply -f -
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
metadata:
  name: migration-job
spec:
  vmiName: rhel8-aelfassy
EOF
```

Check that the `VirtualMachineInstanceMigration` object is created

~~~bash
virtualmachineinstancemigration.kubevirt.io/migration-job created
~~~

Now let's watch the migration job in action. First it will show `phase: Scheduling`

```execute-1
watch -n1 oc get virtualmachineinstancemigration/migration-job -o yaml
```

With the output

~~~bash

apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
(...)
spec:
  vmiName: rhel8-aelfassy
status:
  phase: Scheduling    <----------- Here you can see it's scheduling
~~~

And then move to `phase: TargetReady` and onto `phase: Succeeded`

~~~bash

apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstanceMigration
(...)
spec:
  vmiName: rhel8-aelfassy
status:
  phase: Succeeded    <----------- Now it has finished the migration
~~~

Finally view the `vmi` object and you can see the new underlying host (was *ocp4-worker3*, now it's *ocp4-worker1*); your environment may have different source and destination hosts, depending on where `rhel8-server-ocs` was initially scheduled. Don't forget to `ctrl-c` out of the running watch command:

```execute-1
oc get vmi
```

Now check the output:

~~~bash
NAME             AGE   PHASE     IP            NODENAME                                                 READY
rhel8-aelfassy   13m   Running   10.129.2.53   workshop-n54ln-worker-c-x98pv.c.almog-elfassy.internal   True
~~~

As you can see Live Migration in OpenShift Virtualization is quite easy. If you have time, try to do that from the OpenShift console

Click Virtualization → Virtual Machines → < Your VM > → Actions → Migrate Node to Node 

![image](https://user-images.githubusercontent.com/64369864/180619941-04bdd0de-b68a-44bd-b061-da2e7f73d5c3.png)

Here you can see it's scheduling - `Migrating` status

![image](https://user-images.githubusercontent.com/64369864/180620050-8b3b4df9-a49b-49de-882e-3348ce67aea6.png)

Now it has finished the migration - `Running` status and the node is different

![image](https://user-images.githubusercontent.com/64369864/180620103-86fdc4ad-f17b-4d56-8db8-98464d7ba65c.png)

Also, rerun the `oc describe vmi < rhel8-aelfassy >` after running a few migrations. You'll see the object is updated with details of the migrations, including source and target

```execute-1
oc describe vmi < rhel8-aelfassy >
```

## Node Maintenance

Building on-top of live migration, many organisations will need to perform node-maintenance, e.g. for software/hardware updates, or for decommissioning. During the lifecycle of a **pod**, it's almost a given that this will happen without compromising the workloads, but virtual machines can be somewhat more challenging given their nature. To address this OpenShift Virtualization has a node-maintenance feature which manages the process safely by marking nodes unschedulable and migrating workloads automatically.

Let's take a look at the current running virtual machines and the nodes we have available:

```execute-1
oc get nodes
```

This lists the OpenShift nodes:

~~~bash
NAME                                        STATUS                  ROLES    AGE   VERSION
master-0.c.almog-elfassy.internal           Ready                   master   14d   v1.23.5+3afdacb
master-1.c.almog-elfassy.internal           Ready                   master   14d   v1.23.5+3afdacb
master-2.c.almog-elfassy.internal           Ready                   master   14d   v1.23.5+3afdacb
worker-b-4vbz5.c.almog-elfassy.internal     Ready                   worker   14d   v1.23.5+3afdacb
worker-c-9fj7h.c.almog-elfassy.internal     Ready                   worker   14d   v1.23.5+3afdacb
worker-d-hbq8l.c.almog-elfassy.internal     Ready                   worker   14d   v1.23.5+3afdacb
~~~

Now check the VMIs:

```execute-1
oc get vmi
```

You should see the one VM running

~~~bash
NAME        AGE   PHASE     IP       NODENAME          READY
< name >    45h   Running   < IP >  < node nmae >      True
~~~

Let's take down node for maintenance and ensure that our workload (VM) stays up and running:

> **NOTE**: You may need to modify the below command to specify the worker listed in the output from above.  

```copy
cat << EOF | oc apply -f -
apiVersion: nodemaintenance.kubevirt.io/v1beta1
kind: NodeMaintenance
metadata:
  name: worker1-maintenance
spec:
  nodeName: < worker-b-4vbz5.c.almog-elfassy.internal >
  reason: "Worker1 Maintenance"
EOF
```

See that the `NodeMaintenance` object is created:

~~~bash
nodemaintenance.nodemaintenance.kubevirt.io/worker1-maintenance
~~~

let's check the status of our environment

```execute-1
oc project default
```

Ensure you are in the default project

~~~bash
Now using project "default".
~~~

And check the nodes:

```execute-1
oc get nodes
```

Notice that scheduling is disabled for `worker1` (or the worker that you specified maintenance for): 

~~~bash
NAME                                        STATUS                    ROLES    AGE   VERSION
master-0.c.almog-elfassy.internal           Ready                     master   14d   v1.23.5+3afdacb
master-1.c.almog-elfassy.internal           Ready                     master   14d   v1.23.5+3afdacb
master-2.c.almog-elfassy.internal           Ready                     master   14d   v1.23.5+3afdacb
worker-b-4vbz5.c.almog-elfassy.internal     Ready,SchedulingDisabled  worker   14d   v1.23.5+3afdacb
worker-c-9fj7h.c.almog-elfassy.internal     Ready                     worker   14d   v1.23.5+3afdacb
worker-d-hbq8l.c.almog-elfassy.internal     Ready                     worker   14d   v1.23.5+3afdacb

~~~


Now check the VMI

```execute-1
oc get vmi
```

Note that the VM has been **automatically** live migrated back to an available worker and is not on the `SchedulingDisabled` worker, as per the `EvictionStrategy`. 

~~~bash
NAME        AGE   PHASE     IP       NODENAME          READY
< name >    45h   Running   < IP >  < node nmae >      True
~~~

We can remove the maintenance flag by simply deleting the `NodeMaintenance` object - update this to reflect the nodes in your environment

```execute-1
oc get nodemaintenance
```
It should just show the one

~~~bash
NAME                  AGE
worker1-maintenance   5m16s
~~~

Now delete it:

> **NOTE**: You may need to modify the below command to specify your `nodemaintenance` object is the same as in the output from above.

```copy
oc delete nodemaintenance/worker1-maintenance
```

It should return the following output

~~~bash
nodemaintenance.nodemaintenance.kubevirt.io "worker1-maintenance" deleted
~~~

Then check the same node again:

```copy
oc get node/worker-b-4vbz5.c.almog-elfassy.internal
```

Note the removal of the `SchedulingDisabled` annotation on the '**STATUS**' column. Also important is that just because this node has become active again doesn't mean the virtual machine returns to it automatically, i.e. it won't "fail back", it will reside on the new host

~~~bash
NAME                                      STATUS   ROLES    AGE   VERSION
worker-b-4vbz5.c.almog-elfassy.internal   Ready    worker   2d    v1.23.5+3afdacb
~~~

Before proceeding let's remove the `< name >` virtual machine as well as any lingering PVC's we don't need any longer:

```execute-1
oc delete vm/< name >
```

This should be confirmed with

~~~bash
virtualmachine.kubevirt.io "< name >" deleted
~~~

Now delete the PVCs????????????

```execute-1
oc delete pvc rhel8-ocs
```

It should show the removal

~~~bash
persistentvolumeclaim "< name >" deleted
~~~
