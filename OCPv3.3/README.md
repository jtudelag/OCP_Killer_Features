Openshift Container Platform v3.3
---------------------------------
---------------------------------

[OCP v3.3 Release Notes](https://docs.openshift.com/container-platform/3.3/release_notes/ocp_3_3_release_notes.html)

NOTE:
In order to be able to follow these examples, you need an OCP 3.3 environment with 'redhat/openshift-ovs-multitenant', 
so unfortunately development environments such as minishift, CDK or 'oc cluster up' are not valid.


## [Egress Router](https://docs.openshift.com/container-platform/3.3/admin_guide/managing_pods.html#admin-guide-limit-pod-access-egress-router)

## [Egress Firewall](https://docs.openshift.com/container-platform/3.3/admin_guide/managing_pods.html#admin-guide-limit-pod-access-egress)
Cluster admins can now limit the addresses that some or all pods can access from within the cluster.

OCP uses EgressNetworkPolicies to achieve this. NetworkPolicy objects are project scoped, so all the pods of a project are affected by them.
Multiple EgressNetworkPolicies can be created by project.In case of rule collision, rules are checked in order, and the first one that matches is enforced.
Only IPs can be used to define the rules (URLs are not valid).

To test this new functionality, inside the folder qos-traffic you can find an EgressNetworkPolicy and a POD .yaml files.

First of all, go to one of the OCP instance, a master for example,(reachabale from inside POD) and run a simple web server inside a folder with files.
Also, note the IP of the instance

```
python -mSimpleHTTPServer 8080
ip a
```

Now creates the POD, and rsh into it to test the connectivity to the web server:
```
oc new-project egress-firewall
oc create -f debug-egress-firewall.yaml 
oc rsh debug-egress-firewall
curl http://<web server ip>:8080
```

Now edit the file egress-network-policy.json and change the "Deny cidrSelector" with the IP 
of the instance where the web server is running. 

Then create the EgressNetworkPolic and rsh into the pod again to test if the connectivity to web server is still possible:


```
oc create -f egress-network-policy.json
oc rsh debug-egress-firewall
curl http://<web server ip>:8080
```

The same can be tested the opposite way, just create an EgressNetworkPolicy that Deny to 0.0.0.0 and Allow to an specific ip.

## [QOS traffic shaping](https://docs.openshift.com/container-platform/3.3/admin_guide/managing_pods.html#admin-guide-manage-pods-limit-bandwidth)
We can now limit the bandwith of a POD, both the ingress and the egress.

To test this new functionality, inside the folder qos-traffic you can find a bunch of pod definition .yaml files.
Some of them are bandwidth limited so we can test the feature.

Some of them launch a pod using iperf tool:

* iperf-server-qos-pod-limited.yaml
* iperf-client-qos-pod-limited.yaml
* iperf-client-qos-pod-unlimited.yaml

The others just run a sleep command because the main prupose is to be able to rsh into them:

* qos-pod-limited-debug.yaml
* qos-pod-unlimited-debug.yaml

These last two pods will be used to test this feature.

First, we create the pods, keep in mind that qos-pod-limited-debug.yaml has the bandwidth annotations, 
wile qos-pod-unlimited-debug.yaml has no bandwidth limitations:

```
oc new-project qos-traffic
oc create -f qos-pod-limited-debug.yaml
oc create -f qos-pod-unlimited-debug.yaml

oc get pod
NAME                        READY     STATUS    RESTARTS   AGE
iperf-qos-limited-debug     1/1       Running   0          1h
iperf-qos-unlimited-debug   1/1       Running   0          2h

```

Once the two pods are created, rsh into the unlimited one:

```
oc rsh iperf-qos-unlimited-debug
```

Download a file, in this case, Terraform binaries, for example.
We use curl to download the file, It also shows the download speed (ingress bandwidth) :).

```
curl -o /tmp/t.zip 'https://releases.hashicorp.com/terraform/0.7.8/terraform_0.7.8_linux_amd64.zip'

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.4M  100 16.4M    0     0  10.1M      0  0:00:01  0:00:01 --:--:-- 10.1M
```

Now, we want to test the upload speed (egress bandwidth), so we are going to serve the terrafom binary 
we just download through a simple HTTP server. 

```
cd /tmp
python -mSimpleHTTPServer 8080
```

Then, on any OCP server,(because we need to reach the POD Ip), we are going to download
the file, and see the download speed.
 
From one of the masters for example:
```
ip=$(oc describe pod iperf-qos-unlimited-debug | grep IP | awk '{print $2}')

curl "http://$ip:8080/t.zip -o /tmp/t.zip

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.4M  100 16.4M    0     0  18.6M      0 --:--:-- --:--:-- --:--:-- 18.6M
```

Now, we have both ingress and egress bandwidth data for and unlimited POD.
Do the same steps with the iperf-qos-limited-debug POD, and compare the results.
You should see that both metrics are lower.

Here you can see some sample results:

```
Unlimited Download BW:
--------------------
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.4M  100 16.4M    0     0  10.1M      0  0:00:01  0:00:01 --:--:-- 10.1M


Limited to 1M BW:
-----------------
url -o /tmp/t.zip 'https://releases.hashicorp.com/terraform/0.7.8/terraform_0.7.8_linux_amd64.zip'
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.4M  100 16.4M    0     0   115k      0  0:02:25  0:02:25 --:--:--  117k


Unlimited upload (Dowload file inside a POD, from master):
---------------------------------------------------------
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.4M  100 16.4M    0     0  18.6M      0 --:--:-- --:--:-- --:--:-- 18.6M


Limited upload to 1M (Dowload file inside a POD, from master):
-------------------------------------------------------------
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 16.4M  100 16.4M    0     0   125k      0  0:02:14  0:02:14 --:--:--  120k
```

## [Configuring route timeouts](https://docs.openshift.com/container-platform/3.3/install_config/configuring_routing.html#install-config-configuring-route-timeouts)

We can now set specific timeouts per route, instead of having one unique timeout per router :).
This allow OCP users to controll in a very fine graned way the timeout of their apps.

The mechanism is very simple, we have to annotate our route with the desired timeout, and thats it.
This can be done when creating the route, by seting the annottation, or overriding an already 
created route.

[Route-specific timeouts](https://docs.openshift.com/container-platform/3.3/architecture/core_concepts/routes.html#route-specific-timeouts)

In order to test this feature, we are going to use a simple python app, that allow us to change the
number of seconds the app will wait before answering, so we can play with this value and seehow the
route behaves.

Execute the following commands from a Master node:

```
oc new-project route-timeouts
oc new-app https://github.com/jtudelag/python-variable-timeout
oc expose svc python-variable-timeout
```
Wait some seconds till the POD is running:

```
route=$(oc describe route python-variable-timeout | grep Requested | awk '{print $3 }')
curl "http://$route/hello"
```

As you can see, It takes 5second before it responds. No we are going to annotate our route to 4s timeout. 
And see what happens.

``` 
oc annotate route python-variable-timeout --overwrite haproxy.router.opensfhit.io/timeout=4s

curl "http://$route/hello"
<html><body><h1>504 Gateway Time-out</h1>
The server didn't respond in time.
</body></html>
```

You can play with the python app changing the reponse time just by calling it like this:
``` 
curl "http://$route/timeout/7"
<b>Timeout set to: 7</b>!
``` 

## [Routes with multiple backends](https://docs.openshift.com/container-platform/3.3/dev_guide/routes.html#routes-load-balancing-for-AB-testing)
Routes can now have more than one service backend, making it easier to perform A/B testing. 
Each route can now have multiple services assigned to it, and those services can come from different applications or pods.

This is done by using the field alternateBackends. 
Besides that, every service backend can have a weigh that sets the % of traffic that will be sent to that service.

To try this feature, we just need to run two different applications that listen on the same port,
and create a route with this two apps as backend, and give them a weigh.


``` 
oc new-project route-multiple-backend
oc new-app openshift/hello-openshift
oc new-app https://github.com/jtudelag/python-variable-timeout
```

Inside the route-multiple-backends you can find a .yaml file with a route definition, have a look a it.
Once created, from a master, curl several times to the route, and see what happens.

```
cd route-multiple-backends/
oc create -f route-multiple-backends.yaml
curl http://weight-route-things.router.default.svc.cluster.local
curl http://weight-route-things.router.default.svc.cluster.local
curl http://weight-route-things.router.default.svc.cluster.local
...

```

## [Seccomp](https://docs.openshift.com/container-platform/3.3/admin_guide/seccomp.html)

[Docker seccomp docs](https://docs.docker.com/engine/security/seccomp/)
We can now use Docker seccomp profiles in Openshif to filter out the syscalls containers can make.

* Openshift allows to set a default custom seccomp profile(different from the Docker default profile), by editing the restricted SCC.
* Openshift allows to set individual seccomp profiles per pod via  annotations.

To test this feature, we asume you have already followed the documentation and have a environment with seccomp profiles configured.

In the folder seccomp-profiles you can find the profile mkdir.json,that blocks the syscall mkdir. 
You should place in the seccomp root folder of your cluster (In every node),
Also two .yaml files with POD definitions, one annotated to use the mkdir.json profile.

```
oc new-project seccomp-profiles
oc create -f pod-no-seccomp-profile.yaml
oc create -f pod-mkdir-seccomp-profile.yaml
```

Now, log into both PODs, try to execute the command mkdir and see what happens.
```
oc rsh mkdir-seccomp-profile
sh-4.2# mkdir temp-dir
mkdir: cannot create directory 'temp-dir': Operation not permitted

oc rsh no-seccomp-profiles
sh-4.2# mkdir tmp-dir
```


## [Idling PODs](https://docs.openshift.com/container-platform/3.3/release_notes/ocp_3_3_release_notes.html#ocp-33-idling-unidling)
