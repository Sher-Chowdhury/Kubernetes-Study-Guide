There are three types of healthchecks:

- readiness probe
- liveness probe
- startup probe 

https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

# Pods Healthchecks

There can be situation where the primary application inside a pod is malfunctioning, but is still running. So to help Kubernetes to identify and rebuild these pods, you need to provide some additional details to Kubernetes on how perform periodic healthecks on your pod. That's done by adding the pod.spec.containers.livenessProbe setting to your pod yaml definition:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-httpd
  labels:
    component: httpd
spec:
  containers:
    - name: cntr-httpd
      image: httpd
      ports:
        - containerPort: 80
      livenessProbe:            # this section defines the healthcheck.
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 10
        failureThreshold: 2
        successThreshold: 1
        periodSeconds: 10
        timeoutSeconds: 10
      command: ["/bin/bash", "-c"]
      args:
        - |
          /bin/sleep 600                        # change this to simulate malfunctions
          /usr/local/bin/httpd-foreground  
```

If you set the sleep seconds too high, the the following events will start occuring:

```bash
LAST SEEN   TYPE      REASON      KIND   MESSAGE
28s         Normal    Pulling     Pod    pulling image "httpd"
26s         Normal    Pulled      Pod    Successfully pulled image "httpd"
26s         Normal    Created     Pod    Created container
26s         Normal    Started     Pod    Started container
8s          Warning   Unhealthy   Pod    Liveness probe failed: Get http://172.17.0.7:80/: dial tcp 172.17.0.7:80: connect: connection refused
28s         Normal    Killing     Pod    Killing container with id docker://cntr-httpd:Container failed liveness probe.. Container will be killed and recreated.
```

You also start seeing lots of restarts:

```bash
$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
pod-httpd   1/1     Running   3          3m31s
```

The httpGet test is essentially to running a curl test. However you can also do other liveness tests such as running a command (exec) test, or doing a simple telnet/nc (tcpSocket) test:

```bash
$ kubectl explain pod.spec.containers.livenessProbe --recursive
...
FIELDS:
   exec <Object>
      command   <[]string>
   failureThreshold     <integer>
   httpGet      <Object>
      host      <string>
      httpHeaders       <[]Object>
         name   <string>
         value  <string>
      path      <string>
      port      <string>
      scheme    <string>
   initialDelaySeconds  <integer>
   periodSeconds        <integer>
   successThreshold     <integer>
   tcpSocket    <Object>
      host      <string>
      port      <string>
   timeoutSeconds       <integer>
```

## The readinessProbe

Sometimes a pod only malfunctions because it's under a lot of load and becomes temporarily unresponsive. In that situation you might want to give the pod some time to recover, and you can help it to recover by temporarily stop sending it traffic. That's possible to setup by setting the pod.spec.containers.readinessProbe setting:

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-httpd
  labels:
    component: httpd
spec:
  containers:
    - name: cntr-httpd
      image: httpd
      ports:
        - containerPort: 80
      livenessProbe:            # this section defines the healthcheck.
        httpGet:     # there's also an 'exec' setting available.
          path: /
          port: 80
        initialDelaySeconds: 10
        failureThreshold: 3
        successThreshold: 1
        periodSeconds: 30
        timeoutSeconds: 10
      readinessProbe:            # this section defines the healthcheck.
        httpGet:     # there's also an 'exec' setting available.
          path: /
          port: 80
        initialDelaySeconds: 10
        failureThreshold: 1
        successThreshold: 1
        periodSeconds: 5
        timeoutSeconds: 10
      command: ["/bin/bash", "-c"]
      args:
        - |
          /bin/sleep 60                        # change this to simulate malfunctions
          /usr/local/bin/httpd-foreground  

```

Here we're using both healthchecks together. When the readinessProbe healthcheck fails the kubernetes take the pod out of service to give it a chance to heal before reaching the livenessProbe. But if the pod doesn't heal by the time the livenessProbe threshold is reached then kubernetes will kill and rebuild the pod.

Like livenessProbe, readinessProbe also has exec and tcpSocket tests.

```bash
$ kubectl explain pod.spec.containers.readinessProbe --recursive
...

FIELDS:
   exec <Object>
      command   <[]string>
   failureThreshold     <integer>
   httpGet      <Object>
      host      <string>
      httpHeaders       <[]Object>
         name   <string>
         value  <string>
      path      <string>
      port      <string>
      scheme    <string>
   initialDelaySeconds  <integer>
   periodSeconds        <integer>
   successThreshold     <integer>
   tcpSocket    <Object>
      host      <string>
      port      <string>
   timeoutSeconds       <integer>
```

## Reference

[https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)
