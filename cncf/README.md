# CNCF Demo: Insights Advisor for Red Hat OpenShift

Find the slides at #TODO: publish the slides.

##  1. Create a cluster

Follow [Installing a cluster quickly on AWS
](https://docs.openshift.com/container-platform/4.10/installing/installing_aws/installing-aws-default.html). Basically I just had to run

```
./openshift-install create cluster --dir <installation_directory> --log-level=info 
```

But it's recommended to follow the instructions to check that all prerequisites are configured.


This command will deploy a cluster in AWS and will last 72 hours. You may have to wait for ~30 minutes for the cluster to be up ad running. Once deployed you will have access to the cluster's console. The user, pass and console URL are shown in the output.

You can then press on your profile (top right side), then on "Copy login command" and copy the `oc login ...` command in your terminal. Now you'll be able to launch `oc` commands against the cluster.

## 2. Explore the OpenShift console

You can then check all the available options in the left side of the console: creating projects, managing workloads, networking, users, storage, etc.

In the home page, if you pressed on "Insights" you may find some firing alerts. Press on "View all in OpenShift Cluster Manager" to see the explanation and fix.

## 3. Break the cluster

1. Start using a dedicated namespace for this demo:
```
❯ oc new-project cncf-demo
Now using project "cncf-demo" on server "https://api.cncf-demo.{CLUSTER_URL}".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-25-centos7~https://github.com/sclorg/ruby-ex.git

to build a new example application in Ruby.
```
2. Check that the project is empty with `oc get all`.
3. Break the project by adding the same egress IP as other project:
   a. `oc new-project cncf-demo-copy`
   b. `oc patch netnamespace cncf-demo --type=merge -p '{"egressIPs": ["192.168.1.99"]}'`
   c. `oc patch netnamespace cncf-demo-copy --type=merge -p '{"egressIPs": ["192.168.1.99"]}'`
4. Check the configuration was applied:
```
❯ oc get netnamespace.network.openshift.io | grep cncf-demo
cncf-demo                                          8976363    ["192.168.1.99"]
cncf-demo-copy                                     81310      ["192.168.1.99"]
```
5. Restart the IO pod to force the data upload:
    a. `oc project openshift-insights`
    b. `io_pod=$(oc get pods | tail -n1 | awk '{print $1;}')`
    c. `oc delete pod $io_pod`
6. Visit the console. If no new issues are found, wait for some minutes.
7. The we will be able to see "The OpenShift cluster drops traffic when two NetNamespaces contain the same egress IP".

## 4. Fix the cluster

### First issue

We just need to change the egress IP of one of these two projects:

```
oc patch netnamespace cncf-demo-copy --type=merge -p '{"egressIPs": ["192.168.1.100"]}'
```

Once patched we can reload the advisor the same way we did in the previous section:
```
oc project openshift-insights
io_pod=$(oc get pods | tail -n1 | awk '{print $1;}')
oc delete pod $io_pod
```

Wait for the recommendations to be updated...

### Second issue

There should be another recommendation called "The cluster drops outgoing traffic from a project when its egress IP address is not assigned to any node". We can fix it by assigning these egress to different workers:

1. Pick two workers:
```
❯ oc get nodes
NAME                                         STATUS    ROLES     AGE       VERSION
ip-10-0-128-134.us-east-2.compute.internal   Ready     master    109m      v1.22.0-rc.0+894a78b
ip-10-0-147-78.us-east-2.compute.internal    Ready     worker    102m      v1.22.0-rc.0+894a78b
ip-10-0-166-55.us-east-2.compute.internal    Ready     master    109m      v1.22.0-rc.0+894a78b
ip-10-0-190-250.us-east-2.compute.internal   Ready     worker    102m      v1.22.0-rc.0+894a78b
ip-10-0-215-106.us-east-2.compute.internal   Ready     master    109m      v1.22.0-rc.0+894a78b
ip-10-0-216-180.us-east-2.compute.internal   Ready     worker    103m      v1.22.0-rc.0+894a78b
```
for example `ip-10-0-216-180.us-east-2.compute.internal` and `ip-10-0-190-250.us-east-2.compute.internal`.
2. Assign them two different egress IPs:
```
oc patch hostsubnet <node1> --type=merge -p '{"egressIPs": ["192.168.1.99"]}'
oc patch hostsubnet <node2> --type=merge -p '{"egressIPs": ["192.168.1.100"]}'
```
3. Refresh the advisor:
```
oc project openshift-insights
io_pod=$(oc get pods | tail -n1 | awk '{print $1;}')
oc delete pod $io_pod
```

## 5. Clean up the cluster

### 5.1 Destroy the projects (in case you don't want to destroy the whole cluster)

Delete the projects:
```
oc delete project/project
oc delete project/project-copy
```

Get the host subnets to the original state (use the nodes from the previous step):
```
oc get hostsubnet
oc patch hostsubnet <node1> --type=json -p '[{"op": "remove", "path": "/egressIPs"}]'
oc patch hostsubnet <node2> --type=json -p '[{"op": "remove", "path": "/egressIPs"}]'
```

### 5.2 Destroy the cluster

```
./openshift-install destroy cluster --dir=<installation_directory> --log-level=info
```
