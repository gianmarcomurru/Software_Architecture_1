# Install minikube

https://minikube.sigs.k8s.io/docs/start/

e.g.
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64
sudo install minikube-darwin-arm64 /usr/local/bin/minikube
```

# Start your Kubernetes clusterr :)

`minikube start`

`minikube addons enable metrics-server`

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
kubectl set resources deploy php-apache --requests=cpu=1m
kubectl expose deploy php-apache --port 80

kubectl get pod -l app=php-apache

NAME                          READY   STATUS    RESTARTS   AGE
php-apache-6f8f979d48-l4h54   1/1     Running   0          30s
```

```
kubectl autoscale deployment php-apache \
    --me-percent=10 \
    --min=1  \
    --max=10

kubectl get hpa # It can take a while
```

```
kubectl --generator=run-pod/v1 run -i --tty load-generator --image=busybox /bin/sh
```