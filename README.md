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
- Change argocd server's service (patch) from ClusterIP to NodePort(providing one of the worker nodes public ip)
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
