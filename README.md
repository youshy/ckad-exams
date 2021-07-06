# ckad-exams

Example exams and solutions

# Linux Academy

## 1

```
Our pizza restaurant is working on a new web application designed to run in a Kubernetes cluster. The development team has designed a web service which serves a list of pizza toppings. The team has provided a set of specifications which the pizza topping web service needs in order to run.

We have been asked to create a deployment that meets the app's specifications. Then, we need to expose the application using a NodePort service. This setup should meet the following criteria:

* All objects should be in the pizza namespace. This namespace already exists in the cluster.
* The deployment should be named pizza-deployment.
* The deployment should have 3 replicas.
* The deployment's pods should have one container using the linuxacademycontent/pizza-service image with the tag 1.14.6.
* Run the container with the command nginx.
* Run the container with the arguments "-g", "daemon off;".
* The pods should expose port 80 to the cluster.
* The pods should be configured to periodically check the /healthz endpoint on port 8081, and restart automatically if the request fails.
* The pods should not receive traffic from the service until the / endpoint on port 80 responds successfully.
* The service should be named pizza-service.
* The service should forward traffic to port 80 on the pods.
* The service should be exposed externally by listening on port 30080 on each node.
```

## Solution:

### Create deployment

```
kubectl create deployment pizza-deployment --image=linuxacademycontent/pizza-service:1.14.6 -n pizza --dry-run -o yaml > deploy.yaml
```

Add below bits to `deploy.yaml`

```yaml
command: ["nginx"]
args: ["-g", "daemon off;"]
ports:
- containerPort: 80
livenessProbe:
  httpGet:
    path: /healthz
    port: 8081
readinessProbe:
  httpGet:
    path: /
    port: 80
```

```
kubectl apply -f deploy.yaml
```

### Create service

```
kubectl create service nodeport pizza-service --node-port=30080 --tcp=80:30080
```

## 2

```
Our team has a pod that generates some log output. However, they want to consume the data using an external application, which requires the data to be in a specific format. Our task is to create a pod design that utilizes an adapter running fluentd to format the output from the main container.

* There is a fluentd configuration located on the server at /usr/ckad/fluent.conf. Load the data from this file into a ConfigMap called fluentd-config.
* Create the pod descriptor in /usr/ckad/adapter-pod.yml. An empty file has already been created for us.
* The pod should be named counter.
* Add a container to the pod that runs the busybox image, and name it count.
* Run the count container with the following arguments:

- /bin/sh
- -c
- >
i=0;
while true;
do
echo "$i: $(date)" >> /var/log/1.log;
echo "$(date) INFO $i" >> /var/log/2.log;
i=$((i+1));
sleep 1;
done

* Add another container called adapter to the pod, and make it run the k8s.gcr.io/fluentd-gcp:1.30 image.
* Add an environment variable to the adapter container called FLUENTD_ARGS with the value -c /fluentd/etc/fluent.conf.
* Mount the fluentd-config ConfigMap to the adapter container so that the config data is located inside the container in a file at /fluentd/etc/fluent.conf.
* Create a volume for the pod in such a way that the storage will be deleted if the pod is removed from a node. Mount this volume to both containers at /var/log. The count container will output log data to this volume, and the adapter container will read the data from the same volume.
* Create a hostPath volume where the adapter container will output the formatted log data. Store the data at the /usr/ckad/log_output path. Mount the volume to the adapter container at /var/logout.
```

## Solution:

### Create config map

```
kubectl create configmap config --from-file=/usr/ckad/fluent.conf
```

### Create pod

```
kubectl run counter --image=busybox --restart=Never --dry-run -o yaml > adapter-pod.yml
```

Make sure the pod looks like below (add `args` and `volumeMounts`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: adapter
    image: k8s.gcr.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
      value: -c /fluentd/etc/fluent.conf
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /fluentd/etc
    - name: logout
      mountPath: /var/logout
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: config
  - name: logout
    hostPath:
      path: /usr/ckad/log_output
```

Create pod

```
kubectl apply -f adapter-pod.yml
```
