# Exercise 1 - OpenShift Virtualization Installation

Navigate to the top-level `Operators` menu entry, and select `OperatorHub`. This lists all of the available operators that you can install from the operator catalogue. Start typing 'virtualization' in the search box and you should see an entry called "OpenShift Virtualization".

<img width="1395" alt="Screen Shot 2022-07-19 at 2 30 03" src="https://user-images.githubusercontent.com/64369864/179633707-ab6d6ee5-afec-4044-befd-9837d2103582.png">

Next you'll want to select the 'Install' button. Leave the defaults here as they'll automatically select the latest version of OpenShift Virtualization, and will be placed into a new "openshift-cnv" project.

<img width="993" alt="Screen Shot 2022-07-19 at 2 31 31" src="https://user-images.githubusercontent.com/64369864/179633859-628660f1-75f8-4dc9-af66-15f4bc6bdd87.png">

Next we need to deploy the **HyperConverged** resource, which, in addition to the OpenShift Virtualization operator, creates and maintains an OpenShift Virtualization Deployment for the cluster.

<img width="501" alt="Screen Shot 2022-07-19 at 2 34 07" src="https://user-images.githubusercontent.com/64369864/179634092-59547c13-f5b6-4db7-875c-4cac939c9dd9.png">

> This may raise some questions about a similar term: "hyperconverged infrastructures" and the relationship of workloads and storage associated with that term. For OpenShift Virtualization this relates to the fact that we're converging virtual machines and containers and is how an "instance" of OpenShift Virtualization is instantiated - it does not impact the relation between compute and storage as we will see later in the labs. 

Click on "**Create HyperConverged**", as a required operand, in the same screen to proceed.

This will open a new screen. We can again accept all the defaults for this lab - for real world use, there are many additional flags, parameters, and attributes that can be applied at this stage, such as enabling tech-preview features, specifying host devices, implementing CPU masks, and so on.

Continue the installation by clicking on "**Create**" at the bottom.

<img width="996" alt="Screen Shot 2022-07-17 at 16 43 38" src="https://user-images.githubusercontent.com/64369864/179401246-ba142016-d678-4755-b944-10aa48fc3bb5.png">

You can move to the `Workloads` --> `Pods` menu entry and watch it start all of its resources

<img width="870" alt="Screen Shot 2022-07-19 at 2 37 34" src="https://user-images.githubusercontent.com/64369864/179634433-0845bd20-cdd1-47e8-9c9d-1c11b2b955e8.png">

Watch via the CLI

```
watch -n1 'oc get pods -n openshift-cnv'
```

You will know the process is complete when you see that the operator installation has been successful by running the following command

```
oc get csv -n openshift-cnv
```

You should see the output below

```
NAME                                       DISPLAY                    VERSION   REPLACES                                   PHASE
kubevirt-hyperconverged-operator.v4.10.2   OpenShift Virtualization   4.10.2    kubevirt-hyperconverged-operator.v4.10.1   Succeeded
```
> Once the `PHASE` changes to `Succeeded` you can validate that the required resources and the additional components have been deployed across the nodes, And all the pods shown from this command should be in the `Running` state.

# Viewing the OpenShift Virtualization Dashboard
When OpenShift Virtualization is deployed it adds additional components to OpenShift's web console so you can interact with objects and custom resources defined by OpenShift Virtualization, including VirtualMachine types. You can now navigate to "Workloads" --> "Virtualization" on the left-hand side panel, and you should see the new snap-in component for OpenShift Virtualization.

<img width="1095" alt="Screen Shot 2022-07-19 at 2 39 15" src="https://user-images.githubusercontent.com/64369864/179634572-a1b6f411-62c3-4767-b571-03a5fba616c3.png">

>  Please don't try and create any virtual machines just yet, we'll get to that shortly!

