# Requirements:

- Docker
- kubectl
- minikube

# Install minikube

Follow the guide according to your OS and CPU architecture:
https://minikube.sigs.k8s.io/docs/start/


# Start your Kubernetes cluster

Minikube will start a single-node Kubernetes cluster in your local machine
```
minikube start
```

This command will enable the metrics-server that will retrive metrics from your Kubernetes cluster, like cpu and memory usage inside containers and nodes.
```
minikube addons enable metrics-server
```

To check if the metrics-server is running:
```
❯ kubectl get apiservice v1beta1.metrics.k8s.io -o json | jq '.status'

{
  "conditions": [
    {
      "lastTransitionTime": "2022-03-21T17:58:26Z",
      "message": "all checks passed",
      "reason": "Passed",
      "status": "True",
      "type": "Available"
    }
  ]
}
```

```
kubectl create deployment php-apache --image=us.gcr.io/k8s-artifacts-prod/hpa-example
kubectl set resources deploy php-apache --requests=cpu=200m
kubectl expose deploy php-apache --port 80

kubectl get pod -l app=php-apache

NAME                          READY   STATUS    RESTARTS   AGE
php-apache-6f8f979d48-l4h54   1/1     Running   0          30s
```

# Creating a Horizontal Pod Autoscaler

An Horizontal Pod Autoscaler (HPA) is a Kubernetes resource that automatically adjusts the number of pods in a deployment based on the metrics of the pods.
In this case we will create an HPA for the php-apache deployment that scale the number of pods up when it has more than 80% cpu usage.
```
kubectl autoscale deployment php-apache \
    --cpu-percent=80 \
    --min=1  \
    --max=10

❯ kubectl get hpa # It can take few minutes to load the metrics

NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/80%    1         10        1          30s
```

# Load test with HPA

Create a pod 'load-generator' that will generate load to the php-apache deployment.
```
kubectl run -i --tty load-generator --image=busybox /bin/sh

/ # while true; do wget -q -O - http://php-apache; done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!
```
> You should stop the script with Ctrl+C when you see the hpa scaling up

---

After few minutes you should see the following results:
```
kubectl get hpa -w
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/80%    1         10        1          36s       # <--- Starting load-generator
php-apache   Deployment/php-apache   79%/80%   1         10        1          60s
php-apache   Deployment/php-apache   499%/80%   1         10        1          75s      # <--- Stopping load-generator
php-apache   Deployment/php-apache   499%/80%   1         10        4          90s      # <--- Starting scaling up
php-apache   Deployment/php-apache   236%/80%   1         10        7          105s
php-apache   Deployment/php-apache   0%/80%     1         10        7          2m
php-apache   Deployment/php-apache   0%/80%     1         10        7          6m46s    # <--- Starting scaling down
php-apache   Deployment/php-apache   0%/80%     1         10        1          7m1s
```
Once the load-generator is started, the hpa will scale up until its number of replicas can sustain the load. 
When the load-generator is stopped, the hpa will scale down to its minimum number of replicas.


# Troubleshooting
If the metrics-server does not retrive cpu usage, it might be because of the [following issue](https://github.com/kubernetes/minikube/issues/13620)

To solve it try to stop and remove your cluster and start it again with the following parameters:
```
minikube stop
...
minikube delete
...

minikube start --driver docker --kubernetes-version=v1.21.2 --extra-config=kubelet.housekeeping-interval=10s
```