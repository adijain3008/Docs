# Configuration for Kubernetes to expose Metrics

## etcd
ETCD generally exposes its metrics on `http://127.0.0.1:2381`. But to configure it to expose the metrics at `http://node_ip:2381`, we have to add it in the etcd manifest file. Manifest files are generally stored in the location `/etc/kubernetes/manifests/`.

- Change the dir to the above location.
```
cd /etc/kubernetes/manifests/
```
- Open the `etcd.yaml` file in a text editor.
```
sudo vi etcd.yaml
```
- Find the tag `--listen-metrics-urls` in the `etcd` command in the `spec` block. Add the `http://node_ip:2381` to the tag as shown below. Save and exit.
```
 --listen-metrics-urls=http://127.0.0.1:2381,http://172.31.27.23:2381
 ```
Changing the manifest file should automatically trigger the old `etcd` pod to stop and a new `etcd` pod to spin up with the new configs.

## kube-proxy
`kube-proxy` pod configurations are stored in a ConfigMap with the name `kube-proxy` in the `kube-system` namespace. We need to edit this ConfigMap to expose `kube-proxy` metrics at `http://node_ip:port`.

- Get the name of the ConfigMap for `kube-proxy`.
```
kubectl get cm -n kube-system | grep proxy
```
- Edit the ConfigMap.
```
kubectl edit cm -n kube-system kube-proxy
```
- Find the tag `metricsBindAddress` in the `config.conf` block in the `data` block. Update its value as shown below. Save and exit.
```
metricsBindAddress: 0.0.0.0:10249
```
- Changing the ConfigMap doesn't automatically trigger a new pod to be created. So we have to manually delete the old pod which should spin up a new pod.
  - Get the name of the `kube-proxy` pod.
  ```
  kubectl get po -n kube-system | grep proxy
  ```
  - Delete the `kube-proxy` pod. Replace the `pod_name` with the name of the pod you got in previous step.
  ```
  kubectl delete pod pod_name
  ```
This should spin up a new pod for `kube-proxy` with the updated config settings.

## kube-scheduler
`kube-scheduler` exposes its metrics on `https://127.0.0.1:10259`. So we have to edit the `kube-scheduler` manifest file to expose the metrics at `https://node_ip:10259`. Manifest files are generally stored in the location `/etc/kubernetes/manifests/`.

- Change the dir to the above location.
```
cd /etc/kubernetes/manifests/
```
- Open the `kube-scheduler.yaml` file in a text editor.
```
sudo vi kube-scheduler.yaml
```
- Find the tag `--bind-address` in the `kube-scheduler` command in the `spec` block. Update its value as shown below. Save and exit.
```
 --bind-address=0.0.0.0
 ```
Changing the manifest file should automatically trigger the old `kube-scheduler` pod to stop and a new `kube-scheduler` pod to spin up with the new configs. 

Now we need to configure the endpoints for `kube-scheduler` service monitor to listen to the proper port. Also we need to update the port name in the `kube-scheduler` service.

- Find the name of the `kube-scheduler` service monitor. It will be in the `default` namespace.
```
kubectl get servicemonitors | grep scheduler
```
- Edit the service monitor.
```
kubectl edit servicemonitors prometheus-operator-kube-scheduler
```
- Update the `endpoints` tag in the `spec` block as shown below. Save and exit.
```
endpoints:
- bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
  port: https-metrics
  scheme: https
  tlsConfig:
    caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecureSkipVerify: true
```
- Now we need to update the `kube-scheduler` service. Find the name of the `kube-scheduler` service. It will be in the `kube-system` namespace.
```
kubectl -n kube-system get svc | grep scheduler
```
- Edit the service.
```
kubectl -n kube-system edit svc prometheus-operator-kube-scheduler
```
- Update the `ports` tag in the `spec` block as shown below. Save and exit.
```
ports:
- name: https-metrics
  port: 10259
  protocol: TCP
  targetPort: 10259
```

This should allow `kube-scheduler` to expose its metrics on `https://node_ip:10259` and also configure Prometheus to scrape these metrics from that endpoint.

## kube-controller-manager
`kube-controller-manager` exposes its metrics on `https://127.0.0.1:10257`. So we have to edit the `kube-controller-manager` manifest file to expose the metrics at `https://node_ip:10257`. Manifest files are generally stored in the location `/etc/kubernetes/manifests/`.

- Change the dir to the above location.
```
cd /etc/kubernetes/manifests/
```
- Open the `kube-controller-manager.yaml` file in a text editor.
```
sudo vi kube-controller-manager.yaml
```
- Find the tag `--bind-address` in the `kube-controller-manager` command in the `spec` block. Update its value as shown below. Save and exit.
```
 --bind-address=0.0.0.0
 ```
Changing the manifest file should automatically trigger the old `kube-controller-manager` pod to stop and a new `kube-controller-manager` pod to spin up with the new configs. 

Now we need to configure the endpoints for `kube-controller-manager` service monitor to listen to the proper port. Also we need to update the port name in the `kube-controller-manager` service.

- Find the name of the `kube-controller-manager` service monitor. It will be in the `default` namespace.
```
kubectl get servicemonitors | grep controller
```
- Edit the service monitor.
```
kubectl edit servicemonitors prometheus-operator-kube-controller-manager
```
- Update the `endpoints` tag in the `spec` block as shown below. Save and exit.
```
endpoints:
- bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
  port: https-metrics
  scheme: https
  tlsConfig:
    caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecureSkipVerify: true
```
- Now we need to update the `kube-controller-manager` service. Find the name of the `kube-controller-manager` service. It will be in the `kube-system` namespace.
```
kubectl -n kube-system get svc | grep controller
```
- Edit the service.
```
kubectl -n kube-system edit svc prometheus-operator-kube-controller-manager
```
- Update the `ports` tag in the `spec` block as shown below. Save and exit.
```
ports:
- name: https-metrics
  port: 10257
  protocol: TCP
  targetPort: 10257
```

This should allow `kube-controller-manager` to expose its metrics on `https://node_ip:10257` and also configure Prometheus to scrape these metrics from that endpoint.
