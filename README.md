# bootcamp-k8s-intermediate
### Pre-requisite
1- Create Cluster
```shell
aws configure list
terraform init
terraform apply
aws eks update-kubeconfig --name orquestacion
```

2- Deploy App
```shell
cd eks/basics
kubectl apply -f 0-namespace.yaml
kubens bookinfo
kubectl apply -f 11-bookinfo.yaml

# Be careful creating the nginx controller with helm, missing annotations
#  annotations:
#    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
#    service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: 'true'
#    service.beta.kubernetes.io/aws-load-balancer-type: nlb
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.0/deploy/static/provider/aws/deploy.yaml
kubectl get services --all-namespaces
kubectl apply -f 12-Ingress-Rewrite-root.yaml
kubectl apply -f 13-reviews-v3.yaml
# It could take 5 minute to be the DNS available
```

### HPA
1- Stress test
```shell
siege http://a605d5067a47a422e9a75862e920ed58-48d0d1f1a85ab632.elb.eu-west-3.amazonaws.com/productpage
```
Request over >2s

2- Create the HPA
```shell
# kubectl autoscale deployment productpage-v1 --cpu-percent=50 --min=2 --max=10
kubectl apply -f 1-hpa.yaml
```

Describe the HPA
```shell
kubectl get hpa
```
a) It doesn't work because we don't have request and limits
      "HPA was unable to compute the replica count: failed to get cpu utilization:
      unable to get metrics for resource cpu: unable to fetch metrics from resource
      metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)"
HPA as unknown


```shell
kubectl apply -f 2-bookinfo-request.yaml
````
b) It doesn't work either because EKS does not have the metrics servers installed
https://github.com/kubernetes-sigs/metrics-server

3- Install the metrics server with Helm
```shell
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server -n kube-system
```

Now the HPA works
```shell
kubectl get hpa
```

4- Stress the application again and check the scaling
```shell
siege http://a605d5067a47a422e9a75862e920ed58-48d0d1f1a85ab632.elb.eu-west-3.amazonaws.com/productpage
kubectl top pods
```

### ConfigMap
Modification to read the log dir from the configMap
```shell
kubectl apply -f 3-configmap.yaml
```


### Helm
We are going to remote the rating workload and create it with a chart
1- Remove the old exercise and initialize helm
```shell
kubectl delete -f 4-ratings.yaml
helm create ratings-orq
```

2- Fill the values with what we know

3- Missing container port, we need to modify the template
```yaml
          ports:
                - name: http
                  containerPort: {{ .Values.container.port }}
```

4- Install the chart
```shell
helm upgrade --install --wait -n bookinfo --values 5-values.yaml ratings ratings-orq
```
5- Readiness y liveness now are failing, let's put one
```yaml
          livenessProbe:
                httpGet:
                      path: {{ .Values.container.livenessProbe }}
                      port: http
          readinessProbe:
                httpGet:
                      path: {{ .Values.container.readinessProbe }}
                      port: http
```

6- Now the site does not find the original service, because we have name it ratings-ratings-orq (chart-release) 
Override the name in the values.yaml
```yaml
nameOverride: "ratings"
```

http://a605d5067a47a422e9a75862e920ed58-48d0d1f1a85ab632.elb.eu-west-3.amazonaws.com/productpage

7- Enable autoscaler
```yaml
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  # targetMemoryUtilizationPercentage: 80
resources:
      limits:
            cpu: 1000m
            memory: 1024Mi
      requests:
            cpu: 100m
            memory: 128Mi
```
