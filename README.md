# 1: Install and configure Jenkins on ec2 master
  Install Pluggins:
 - OWASP Depecency check
 - SonarQube Scanner
 - Docker
 - Pipeline: Stage View

# 2: Create EKS Cluster on AWS (Master machine)
connect master node with aws
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

- Install kubectl (Master machine)(Setup kubectl )
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

- Install eksctl (Master machine) (Setup eksctl)
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```
- Create EKS Cluster (just create control plane not whole node group)
```
eksctl create cluster --name=wanderlust \
                    --region=us-east-2 \
                    --version=1.30 \
                    --without-nodegroup
```

- Associate IAM OIDC Provider (Kubernetes Pod having access to other aws services)
```
eksctl utils associate-iam-oidc-provider \
  --region us-east-2 \
  --cluster wanderlust \
  --approve
```
- Create Nodegroup (creates worker instances and conects with control node)
```
eksctl create nodegroup --cluster=wanderlust \
                     --region=us-east-2 \
                     --name=wanderlust \
                     --node-type=t2.large \
                     --nodes=2 \
                     --nodes-min=2 \
                     --nodes-max=2 \
                     --node-volume-size=29 \
                     --ssh-access \
                     --ssh-public-key=eks-nodegroup-key
```
# 5: Install docker (image build)
- Provide permission to docker socket so that docker build and push command do not fail 
```
chmod 777 /var/run/docker.sock
```
# 6: Install and configure SonarQube (static code analysis)
# 7: Install Trivy(vulnerabilities and file system detection)
# 8: Setup Gmail
turn on 2 step verification -> create app password 
jenkins Manage Jenkins --> Credentials to add username and password for email notification
save it on jenkins crdentials ->
go to system -> settings of Extnded Email Notification
<img width="726" height="361" alt="image" src="https://github.com/user-attachments/assets/6da0c6f4-08c3-47a5-a448-9ac7a20e1ac4" />

Owasp vs trivy
OWASP Dependency-Check: focuses on aapke Application Code ki Libraries 
Trivy:  libraries, Docker Image (OS, Linux packages, system configuration) and Infrastructure 

# 9: Owasp installation
After OWASP plugin is installed, Now move to Manage jenkins --> Tools -> download from github with a specified version

# 10: Integrate Sonarqube with jenkins
- Login to SonarQube server --> Administration --> Security --> Users --> Token
- Now, go to Manage Jenkins --> credentials and add Sonarqube credentials
- Go to Manage Jenkins --> Tools and search for SonarQube Scanner installations
- Go to Manage Jenkins --> System and search for SonarQube installations
  
# 11: Integrate Github with jenkins
- Create tokens on Github
- Manage Jenkins --> credentials and add Github credentials
- Manage Jenkins --> System and search for Global Trusted Pipeline Libraries

# 12: Install and Configure ArgoCD (Master Machine)
- Create argocd namespace
```
kubectl create namespace argocd
```
- Apply argocd manifest
```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
- Make sure all pods are running in argocd namespace
```
watch kubectl get pods -n argocd
```
- Install argocd CLI
```
sudo curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
```
- Provide executable permission
```
sudo chmod +x /usr/local/bin/argocd
```
- Check argocd services
```
kubectl get svc -n argocd
```
- Change argocd server's service (patch) from ClusterIP to NodePort(kubernetes provides a port between a specific range on which our app is can run on our worker nodes)
```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```
- Check the port where ArgoCD server is running and add it as inbound rule for the all worker nodes inside that cluster
```
<public-ip-worker>:<port>
```
- Fetch the initial password of argocd server
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
* Note:the argocd is installed through master node in the cluster and runs on worker nodes on a specific port provided after patching it to 'NodePort'

# 13: Go to Master Machine and add our own eks cluster to argocd for application deployment using cli
- Login to argoCD from CLI
 ```
argocd login 52.53.156.187:32738 --username admin
```
- Check how many clusters are available in argocd
```
argocd cluster list
```
- Get your cluster name
```
kubectl config get-contexts
```
- Add your cluster to argocd
```
argocd cluster add Wanderlust@wanderlust.us-west-1.eksctl.io --name wanderlust-eks-cluster
```
- Once your cluster is added to argocd, go to argocd console Settings --> Clusters and verify it
- Go to Settings --> Repositories and click on Connect repo and add your git repo url
- Now, go to Applications and click on New App and create your app

# 14: monitoring EKS cluster, kubernetes components and workloads using prometheus and grafana via HELM (On Master machine)
- Install Helm Chart
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
- Add Helm Stable Charts for Your Local Client
```
helm repo add stable https://charts.helm.sh/stable
```
- Add Prometheus(Grafana included) Helm Repository
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```
- Create Prometheus Namespace
```
kubectl create namespace prometheus
kubectl get ns
```
- Install Prometheus using Helm
```
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
```
- Verify prometheus installation
```
kubectl get pods -n prometheus
```
- Check the services file (svc) of the Prometheus
```
kubectl get svc -n prometheus
```
- Expose Prometheus and Grafana to the external world through Node Port
```
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
kubectl get svc -n prometheus
```
- change the SVC file of the Grafana and expose it to the outer world
```
kubectl edit svc stable-grafana -n prometheus
```
- Check grafana service
```
kubectl get svc -n prometheus
```
- Get a password for grafana and login to the account on browser
```
kubectl get secret --namespace prometheus stable-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```
# 15: Clean Up
- Delete eks cluster
```
eksctl delete cluster --name=wanderlust --region=us-west-1
``
