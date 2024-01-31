# Deployment (CD Pipeline)

## setup minikube cluster

* step 1. install docker

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin \ docker-compose-plugin
```

* step 2. install minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```


* step 3. setup kubectl

```bash
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl

# If the folder `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl
```

* step 4. docker user group add

this will remove the accedd denied issue to the docker deamon

```bash
sudo usermod -aG docker $USER
newgrp docker
```

* step 5. start the minikube cluster

```bash
minikube start --memory=2048
```


## Setup ArgoCD using kubenetes operators

* step 1. install OLM for the cluster , then install argocd operator

```bash
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.26.0/install.sh | bash -s v0.26.0
kubectl create -f https://operatorhub.io/install/argocd-operator.yaml
kubectl get csv -n operators
```

* step 2. check minikube status

```bash
minikube status
```

## Define basic argocd operator config file for the deployment

```yml
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec: {}
```

## Apply the configuration

```bash
kubectl apply -f <argocd-operator-yaml-filepath>
```

## Check cluster pods status , wait till pods up and run

```bash
kubectl get pods

output: 

NAME                                          READY   STATUS    RESTARTS   AGE
example-argocd-application-controller-0       1/1     Running   0          74m
example-argocd-redis-68bb584d8b-dcjsw         1/1     Running   0          74m
example-argocd-repo-server-6f74889cdf-xzhw5   1/1     Running   0          74m
example-argocd-server-58799f4c44-6w6kq        1/1     Running   0          74m
```

If all the pods are up and running, we can update the argocd-server service to the type LoadBalancer with port 8080. After that, we have to set up port forwarding to access the resource from the public internet (EC2 cluster configuration).


```bash
kubectl edit svc example-argocd-server
```

edited yaml file: 
```yaml
ports:
  - name: http
    nodePort: 32617
    port: 8080
    protocol: TCP
    targetPort: 8080
  - name: https
    nodePort: 31810
    port: 443
    protocol: TCP
    targetPort: 8080
  selector:
    app.kubernetes.io/name: example-argocd-server
  sessionAffinity: None
  type: LoadBalancer  #edited here
```

after applying, check the kubectl svc status to verify the changes.

```bash
minikube service list

output -->
|-------------|----------------------------------------------------|--------------|---------------------------|
|  NAMESPACE  |                        NAME                        | TARGET PORT  |            URL            |
|-------------|----------------------------------------------------|--------------|---------------------------|
| default     | example-argocd-metrics                             | No node port |                           |
| default     | example-argocd-redis                               | No node port |                           |
| default     | example-argocd-repo-server                         | No node port |                           |
| default     | example-argocd-server                              | http/8080    | http://192.168.49.2:32617 |
|             |                                                    | https/443    | http://192.168.49.2:31810 |
| default     | example-argocd-server-metrics                      | No node port |                           |
| default     | kubernetes                                         | No node port |                           |
| kube-system | kube-dns                                           | No node port |                           |
| olm         | operatorhubio-catalog                              | No node port |                           |
| olm         | packageserver-service                              | No node port |                           |
| operators   | argocd-operator-controller-manager-metrics-service | No node port |                           |
| operators   | argocd-operator-controller-manager-service         | No node port |                           |
| operators   | argocd-operator-webhook-service                    | No node port |                           |
|-------------|----------------------------------------------------|--------------|---------------------------|

```

* Apply port forwarding to enable public access. First, check the Minikube service list and identify the target port.

```bash
kubectl port-forward --address 0.0.0.0 service/example-argocd-server 32617:8080
```

> now we can access the service using the ```http://<server public ip>:32617```

## Get the argocd access password

```bash
# check kubectl secret stores
kubectl get secrets

#view the secret
kubectl edit secret example-argocd-cluster

# kubernetes secrets are base64 encoded , lets decode it to plan text
encoded_pw: b1JpZnJhdmpFa1BRTlVGR0Q2WlRKWG44MjRLc1ZBMXE=

#convert to plan text model
echo b1JpZnJhdmpFa1BRTlVGR0Q2WlRKWG44MjRLc1ZBMXE= | base64 -d

original_pw: oRifravjEkPQNUFGD6ZTJXn824KsVA1q
```

## Create aws exr secret inside kubernets cluster

```bash

kubectl create secret docker-registry regcred \
  --docker-server=630210676530.dkr.ecr.us-east-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password) \
  --namespace=default
```

## After a successful continuous delivery (CD) workflow, the cluster pod configuration will look like this.

```bash
kubectl get pods

output ->

NAME                                          READY   STATUS    RESTARTS   AGE
example-argocd-application-controller-0       1/1     Running   0          6m44s
example-argocd-redis-68bb584d8b-f2zrp         1/1     Running   0          6m44s
example-argocd-repo-server-6f74889cdf-v2k5v   1/1     Running   0          6m44s
example-argocd-server-58799f4c44-5cbfm        1/1     Running   0          6m44s
todo-app                                      1/1     Running   0          71s
todo-app-5bc5577fbb-6gg8m                     1/1     Running   0          71s
todo-app-5bc5577fbb-d56hz                     1/1     Running   0          71s
todo-app-5bc5577fbb-lhd4g                     1/1     Running   0          71s
```

```bash
kubectl get svc

otuput ->

NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
example-argocd-metrics          ClusterIP      10.108.67.47    <none>        8082/TCP                     21m
example-argocd-redis            ClusterIP      10.106.228.22   <none>        6379/TCP                     21m
example-argocd-repo-server      ClusterIP      10.98.196.80    <none>        8081/TCP,8084/TCP            21m
example-argocd-server           LoadBalancer   10.103.141.5    <pending>     80:30416/TCP,443:32036/TCP   21m
example-argocd-server-metrics   ClusterIP      10.105.43.62    <none>        8083/TCP                     21m
kubernetes                      ClusterIP      10.96.0.1       <none>        443/TCP                      45m
todo-service                    NodePort       10.110.100.91   <none>        8080:31000/TCP               15m
```

## Expose custom python application (todo-service) to outside world from EC2 instance

```bash
kubectl port-forward --address 0.0.0.0 service/todo-service 31000:8080
```
