Openshift Container Platform v3.3
---------------------------------
---------------------------------

[OCP v3.3 Release Notes](https://docs.openshift.com/container-platform/3.3/release_notes/ocp_3_3_release_notes.html)

NOTE:
In order to be able to follow these examples, you need an OCP 3.3 environment with 'redhat/openshift-ovs-multitenant', 
so unfortunately development environments such as minishift, CDK or 'oc cluster up' are not valid.


## Egress Router 

## Egress Firewall 

## [QOS traffic shaping](https://docs.openshift.com/container-platform/3.3/admin_guide/managing_pods.html#admin-guide-manage-pods-limit-bandwidth)

To test this new functionality, inside the folder qos-traffic you can find a bunch of pod definition .yaml files.
Some of them are bandwidth limited wo we can test the feature.

Some of them launch a pod using iperf tool:

* iperf-server-qos-pod-limited.yaml
* iperf-client-qos-pod-limited.yaml
* iperf-client-qos-pod-unlimited.yaml

The others just run a sleep command because the main prupose is to be able to rsh into them:

* qos-pod-limited-debug.yaml
* qos-pod-unlimited-debug.yaml

These last two pods wiil be used to test this feature.

First, we create the pods, keep in mind that qos-pod-limited-debug.yaml has the bandwidth annotations, while qos-pod-unlimited-debug.yaml has no bandwidth limitations:

```
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

