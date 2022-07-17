# Before we begin, it's important to make sanity tests

Check your user

```execute
oc whoami
```
This should return "**system:serviceaccount:workbook:cnv**".

Check your cluster nodes

```execute
oc get nodes
```

You should be able to see the list of nodes as below:

~~~bash
NAME                           STATUS   ROLES    AGE   VERSION
ocp4-master1.aio.example.com   Ready    master   18h   v1.22.3+b93fd35
ocp4-master2.aio.example.com   Ready    master   18h   v1.22.3+b93fd35
ocp4-master3.aio.example.com   Ready    master   18h   v1.22.3+b93fd35
ocp4-worker1.aio.example.com   Ready    worker   18h   v1.22.3+b93fd35
ocp4-worker2.aio.example.com   Ready    worker   18h   v1.22.3+b93fd35
ocp4-worker3.aio.example.com   Ready    worker   18h   v1.22.3+b93fd35
~~~


Check the cluster version 

```execute
oc get clusterversion
```

Then you should see the following (your minor version may be different, but should be no less than below):

~~~bash
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.9.23     True        False         27m     Cluster version is 4.9.23
~~~

Check the cluster opertators

```execute
oc get co
```
This command will list the all cluster operators, the main components of OpenShift itself, and their availability as shown below:

~~~bash
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.9.23    True        False         False      6h19m
baremetal                                  4.9.23    True        False         False      6h40m
cloud-controller-manager                   4.9.23    True        False         False      6h42m
cloud-credential                           4.9.23    True        False         False      6h54m
cluster-autoscaler                         4.9.23    True        False         False      6h40m
config-operator                            4.9.23    True        False         False      6h42m
console                                    4.9.23    True        False         False      6h23m
csi-snapshot-controller                    4.9.23    True        False         False      6h41m
dns                                        4.9.23    True        False         False      6h40m
etcd                                       4.9.23    True        False         False      6h40m
image-registry                             4.9.23    True        False         False      6h17m
(...)
~~~

----

# Making sure OpenShift is fully functional

Create a new project 

```execute
oc new-project < your-name >
```

Now execute following command to deploy example application

Now execute following command to deploy example application:

```execute
oc new-app \
	nodejs~https://github.com/vrutkovs/DuckHunt-JS
```

This will create all Kubernetes resources to deploy and run the example application as below:

~~~bash
--> Creating resources ...
    imagestream.image.openshift.io "duckhunt-js" created
    buildconfig.build.openshift.io "duckhunt-js" created
    deployment.apps "duckhunt-js" created
    service "duckhunt-js" created
--> Success
    Build scheduled, use 'oc logs -f buildconfig/duckhunt-js' to track its progress.
    Application is not exposed. You can expose services to the outside world by executing one or more of the commands below:
     'oc expose service/duckhunt-js'
    Run 'oc status' to view your app.
~~~


Our application will now build from source, you can watch it happen by tailing the build log file. When it's finished it will push the image into the OpenShift image registry:

```execute
oc logs duckhunt-js-1-build -f
```

This process will likely take approximately 3-4 minutes for the following output to confirm the new image is pushed to registry:

~~~bash
Successfully pushed image-registry.openshift-image-registry.svc:5000/test/duckhunt-js@sha256:c4e64bc633ae09ce0f2f2f6de2ca9eaca8e11dc5b335301a2be78216df4b6929
Push successful
~~~

> **NOTE**: You may get an error saying "Error from server (BadRequest): container "sti-build" in pod "duckhunt-js-1-build" is waiting to start: PodInitializing"; you were just too quick to ask for the log output of the pods, simply re-run the command.

Once finished, check the status of the pods by executing the command below:

```execute
oc get pods 
```

You'll see that a couple of pods have been created, one that just completed our build, and then the application itself, which should be in a `Running` state, if it's still showing as `ContainerCreating` just give it a few more seconds:


~~~bash
NAME                           READY   STATUS      RESTARTS   AGE
duckhunt-js-1-build            0/1     Completed   0          4m7s
duckhunt-js-5b75fd5ccf-j7lqj   1/1     Running     0          105s   <-- this is our app!
~~~

Now expose the application (via the service) so we can route to it from the outside...


```execute
oc expose svc/duckhunt-js
```

As a result, a route is created:

~~~bash
route.route.openshift.io/duckhunt-js exposed
~~~

To check the route execute following command:

```execute
oc get route duckhunt-js
```

Now you should be able to see the route endpoint as below:

~~~bash
NAME          HOST/PORT                                  PATH   SERVICES      PORT       TERMINATION   WILDCARD
duckhunt-js   duckhunt-js-test.%cluster_subdomain%          duckhunt-js   8080-tcp                 None
~~~

You should be able to open up the application in the same browser that you're reading this guide from, either copy and paste the address, or click this link: [http://duckhunt-js-test.%cluster_subdomain%](http://duckhunt-js-test.%cluster_subdomain%). If your OpenShift cluster is working as expected and the application build was successful, you should now be able to have a quick play with this... good luck ;-)
> **NOTE**: If the link above doesn't work just run `oc get route duckhunt-js` to find the exposed route for the app. 

<img width="1000" alt="Screen Shot 2022-07-17 at 15 50 28" src="https://user-images.githubusercontent.com/64369864/179399155-f31e6051-46ca-490c-b07e-6e5e7138c41b.png">


Before we start looking at OpenShift Virtualization, let's just clean up the test project and have OpenShift remove the resources...

```execute
oc delete project < your-name >
```
Then wait for project deletion

~~~bash
project.project.openshift.io "< your-name >" deleted
~~~



Now we can move onto deploying OpenShift Virtualization...



