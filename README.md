# Overview

This is a setup of PostgreSQL + Prometheus + Grafana.

Implementation is done with Docker, but there is also partial Kubernetes support.

# Docker

Four Docker containers are set up with Ansible:
1. a PostgreSQL instance, listening on port 5432
1. a Prometheus instance, listening on port 9090
1. a PostgreSQL Exporter instance, listening on port 9187
1. a Grafana instance, listening on port 3000

The PostgreSQL Exporter instance gets metrics from the PostgreSQL instance and is connected to the Prometheus instance.<br>
Grafana is connected to the Prometheus and PostgreSQL instances. A Grafana dashboard is made, consisting of data from these sources (takes some time to load).

All PostgreSQL and Grafana data is saved locally in the respective repository directories (postgres/data and grafana/data) through the use of bind mounts.<br>

## Requirements

1. Ansible + Pip
1. Run the following commands as root:

    ```
    ansible-galaxy collection install community.docker #ansible docker module
    pip install docker #docker sdk for python, needed for the ansible docker module to work
    ```

## Usage

***!!! PostgreSQL and Grafana containers run with id 1000:1000. Therefore, all repository data files (directories postgres/data and grafana/data) must belong to this id for the containers to work (chown -R 1000:1000 postgres/data && chown -R 1000:1000 grafana/data). Otherwise, change id in docker/main.yaml !!!***

All commands below are run inside docker dir as root.

To install the containers run the following command:

  ```
  ansible-playbook --connection=local -i localhost, main.yaml
  ```
Go to http://localhost:3000 to view the Grafana dashboard (user:admin,password:password). It displays the state of the database and data from an example table.<br>
Connect to the database with one of the following commands:
  ```
  psql -U postgres -h localhost
  psql -U setup -h localhost -d setup
  ```

### Cleanup

To remove the containers run the following command:
```
bash cleanup
```

## Stucture

| file                             | description               |
|:--------------------------------:|:-------------------------:|
| main.yaml                        | container setup           |
| cleanup                          | container removal         |
| postgres/data                    | postgres data             |
| prometheus/prometheus.yml        | prometheus config         |
| grafana/data                     | grafana data              |
| grafana/data/dashboards          | grafana dashboards        |
| grafana/provisioning/dashboards  | grafana dashboards setup  |
| grafana/provisioning/datasources | grafana datasources setup |

<br>

# Kubernetes

Three Kubernetes services are created:
1. a PostgreSQL service, listening on port 5432.
1. a Prometheus service, listening on port 9090.
1. a Grafana service, listening on port 3000.

All PostgreSQL data is saved locally in /tmp/postgres/data through the use of a hostPath Peristent Volume.

## Usage

If using microk8s, create an alias (run as root):
```
snap alias "microk8s.kubectl" "kubectl"
```

All commands below are run inside kubernetes dir (unless mentioned differently) as root.

### PostgreSQL

To install PostgreSQL run the following commands:
  ```
  kubectl create -f postgres.yaml #create postgres resource
  kubectl port-forward service/postgres 5432:5432 #open postgres port
  ```
Connect to the database with one of the following commands:
  ```
  psql -U postgres -h localhost
  psql -U setup -h localhost
  ```

#### Cleanup

To cleanup run the following command:
```
kubectl delete --ignore-not-found=true -f postgres.yaml
```

### Prometheus

***Prometheus might not work, at least with default microk8s configuration.***

RBAC has to be enabled.

There are two ways to install Prometheus:
* prometheus-operator (uses default namespace):
    ```
    kubectl create -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml #create prometheus operator
    kubectl create -f prometheus.yaml #create prometheus resource
    kubectl port-forward service/prometheus 9090:9090 #open prometheus port
    ```
* kube-prometheus (includes prometheus-operator, creates monitoring namespace):
    ```
    git clone https://github.com/prometheus-operator/kube-prometheus.git && cd kube-prometheus
    kubectl create -f manifests/setup #create monitoring namespace and CRDs
    kubectl create -f manifests/ #create remaining resources
    kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090 #open prometheus port
    ```

#### Cleanup

To cleanup run the following commands:
* prometheus-operator:
    ```
    kubectl delete --ignore-not-found=true -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/master/bundle.yaml
    kubectl delete --ignore-not-found=true -f prometheus.yaml
    ```
* kube-prometheus:
    ```    
    kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup #inside kube-prometheus dir
    ```

### Grafana

Grafana can be installed with the above commands for kube-prometheus. Then, run the following command to open port 3000:
```
kubectl --namespace monitoring port-forward svc/grafana 3000
```
Go to http://localhost:3000 to view Grafana dashboards (user:admin,password:admin). ***Grafana dashboards don't work without working Prometheus.***

#### Cleanup

To cleanup run the following command:
```    
kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup #inside kube-prometheus dir
```
