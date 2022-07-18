# Exercise 1 - OpenShift Virtualization Installation

Navigate to the top-level `Operators` menu entry, and select `OperatorHub`. This lists all of the available operators that you can install from the operator catalogue. Start typing 'virtualization' in the search box and you should see an entry called "OpenShift Virtualization".

<img width="774" alt="Screen Shot 2022-07-17 at 16 36 39" src="https://user-images.githubusercontent.com/64369864/179401011-3770f09a-c234-4ae0-b02e-49b8e9c3cd04.png">

Next you'll want to select the 'Install' button. Leave the defaults here as they'll automatically select the latest version of OpenShift Virtualization, and will be placed into a new "openshift-cnv" project.

<img width="781" alt="Screen Shot 2022-07-17 at 16 39 03" src="https://user-images.githubusercontent.com/64369864/179401096-47cd96b0-4008-49fd-b046-c7664360fd5b.png">

Make sure that the namespace it will be installed to is "openshift-cnv". Press the 'Install' button.

<img width="737" alt="Screen Shot 2022-07-17 at 16 40 55" src="https://user-images.githubusercontent.com/64369864/179401155-d9a3f865-8897-43a5-a9df-be9d28b108a1.png">

Next we need to deploy the **HyperConverged** resource, which, in addition to the OpenShift Virtualization operator, creates and maintains an OpenShift Virtualization Deployment for the cluster.

> This may raise some questions about a similar term: "hyperconverged infrastructures" and the relationship of workloads and storage associated with that term. For OpenShift Virtualization this relates to the fact that we're converging virtual machines and containers and is how an "instance" of OpenShift Virtualization is instantiated - it does not impact the relation between compute and storage as we will see later in the labs. 

Click on "**Create HyperConverged**", as a required operand, in the same screen to proceed.

This will open a new screen. We can again accept all the defaults for this lab - for real world use, there are many additional flags, parameters, and attributes that can be applied at this stage, such as enabling tech-preview features, specifying host devices, implementing CPU masks, and so on.

Continue the installation by clicking on "**Create**" at the bottom.

<img width="996" alt="Screen Shot 2022-07-17 at 16 43 38" src="https://user-images.githubusercontent.com/64369864/179401246-ba142016-d678-4755-b944-10aa48fc3bb5.png">

You can move to the 'Workloads' --> 'Pods' menu entry and watch it start all of its resources

<img width="1027" alt="Screen Shot 2022-07-17 at 16 44 40" src="https://user-images.githubusercontent.com/64369864/179401295-b5a066ee-0d1d-459c-8058-3d4b577f161f.png">

You can also briefly return to the 'Terminal' tab in your hosted lab guide and watch via the CLI

```
watch -n1 'oc get pods -n openshift-cnv'
```

You will know the process is complete when you see that the operator installation has been successful by running the following command

```
oc get csv -n openshift-cnv
```

You should see the output below

```
NAME                                      DISPLAY                    VERSION   REPLACES                                  PHASE
kubevirt-hyperconverged-operator.v4.9.3   OpenShift Virtualization   4.9.3     kubevirt-hyperconverged-operator.v4.9.2   Succeeded
```
> Once the `PHASE` changes to `Succeeded` you can validate that the required resources and the additional components have been deployed across the nodes, And all the pods shown from this command should be in the `Running` state.

# Viewing the OpenShift Virtualization Dashboard
When OpenShift Virtualization is deployed it adds additional components to OpenShift's web console so you can interact with objects and custom resources defined by OpenShift Virtualization, including VirtualMachine types. You can now navigate to "Workloads" --> "Virtualization" on the left-hand side panel, and you should see the new snap-in component for OpenShift Virtualization.

<img width="777" alt="Screen Shot 2022-07-17 at 17 31 54" src="https://user-images.githubusercontent.com/64369864/179403200-8410dc54-2afb-4920-8b84-41cbb11cbbf4.png">

>  Please don't try and create any virtual machines just yet, we'll get to that shortly!

