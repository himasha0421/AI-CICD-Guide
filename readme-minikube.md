### steup minikube cluster

step1. install docker

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

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

step 2. install minikube

```bash

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

```

step 3. setup kubectl

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

step 4. docker user group add

```bash

sudo usermod -aG docker $USER
newgrp docker
```

step 5. start the minikube cluster

```bash

minikube start --memory=2048
```


step 6. setup ArgoCD using kubenetes operators


```bash

curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.26.0/install.sh | bash -s v0.26.0
kubectl create -f https://operatorhub.io/install/argocd-operator.yaml
kubectl get csv -n operators
```

step 7. check minikube status

```bash

minikube status
```

step 8. define basic argocd operator config file

```yml

apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: example-argocd
  labels:
    example: basic
spec: {}
```

apply the configuratoion

```bash

kubectl apply -f <argocd-operator-yaml-filepath>
```

check cluster pods status , wait till pods up and run

```bash
kubectl get pods

output: 

NAME                                          READY   STATUS    RESTARTS   AGE
example-argocd-application-controller-0       1/1     Running   0          74m
example-argocd-redis-68bb584d8b-dcjsw         1/1     Running   0          74m
example-argocd-repo-server-6f74889cdf-xzhw5   1/1     Running   0          74m
example-argocd-server-58799f4c44-6w6kq        1/1     Running   0          74m
```

if all the pods up and run then we can update the argocd-server service to the type loadbalancer with port 8080, then we have to
do port forwarding to access the resource from public internet (EC2 cluster configuration)


```bash

kubectl edit svc example-argocd-server
```

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

after applying check the kubectl svc status to verify the changes

* apply port forwarding to enable the public access , first check the minikube service list and identify the target port

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

```bash
kubectl port-forward --address 0.0.0.0 service/example-argocd-server 32617:8080
```

now we can access the service using the http://<server public ip>:32617

step 8. get the argocd access password

```bash

# check kubectl secret stores
kubectl get secrets

#view the secret
kubectl edit secret example-argocd-cluster

# kubernetes secrets are base64 encoded , lets decode it to plan text
encoded_pw: MDZGeENSOUx3OGpPWlFiVXExc0hTUEVuQll5SWFldmc=

#convert to plan text model
echo MDZGeENSOUx3OGpPWlFiVXExc0hTUEVuQll5SWFldmc= | base64 -d

original_pw: 06FxCR9Lw8jOZQbUq1sHSPEnBYyIaevg
```