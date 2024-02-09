


![cover](https://github.com/najmi9/jenkins-symfony-kubernetes/assets/61061620/3184e8cd-2b7d-4570-9008-b48ed1413812)

# Workflow
- Setup version control system: Bitbucket, Gitlab, AWS CodeCommit, Azure DevOps...
- Setup container registry service: AWS ECR, DockerHub, Google Container Registry...
- Create a Jenkins server on EC2.

1. create ec2 ubuntu 22
2. install jenkins
4. add ecr-cc2 role
5. generate key for bitbucket
6. create credentails: private key, aws credentials
7. create webhook
8. create pipeline
9. create eks
10. push changes to bitbucket

# Create EKS cluster
- Create eks cluster:
```bash
eksctl create cluster \
  --name cluster-jenkins-test \
  --region eu-west-3 \
  --node-type t2.small \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

# Install docker and docker-compose on Jenkins server:
```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

sudo usermod -aG docker ${USER}
```

# Create a ssh key
```bash
ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa
```


# Install Jenkins:
```bash
sudo apt install openjdk-11-jdk -y ## Install JAVA
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key |sudo gpg --dearmor -o /usr/share/keyrings/jenkins.gpg
sudo sh -c 'echo deb [signed-by=/usr/share/keyrings/jenkins.gpg] http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt install jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```


# Install EKSCTL command line
```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
```

# Solve Jenkins permission issue:
https://stackoverflow.com/questions/15174194/jenkins-host-key-verification-failed
```bash
sudo su - jenkins
git ls-remote -h git@bitbucket.org:person/projectmarket.git HEAD
```

## Permission issue related to docker inside Jenkins server:
```bash
sudo usermod -aG docker jenkins
groups jenkins
sudo systemctl restart jenkins
```

## Configure aws
```bash
sudo apt  install awscli
aws configure
cat ~/.aws/credentials
```

# Cluster commands
```
aws eks --region eu-west-3 update-kubeconfig --name cluster-jenkins-test

kubectl apply -f kubernetes/develop/configmap.yaml
kubectl apply -f kubernetes/develop/deploy.yaml
kubectl apply -f kubernetes/develop/service.yaml

kubectl get services

kubectl exec -it nginx-deployment-cbdccf466-twvdd -- ls /usr/share/nginx/html
50x.html  index.html
```


# Delete cluster
eksctl delete cluster --region=eu-west-3 --name=cluster-jenkins-test


# Usefull commands:
kubectl exec -it nginx-deployment-df796f676-8kq8n --container php bash
kubectl get pods -o wide
kubectl get services -o wide
kubectl get deployments -o wide
kubectl delete service nginx-service
kubectl delete pod nginx-deployment-df796f676-8kq8n
kubectl delete deployment nginx-deployment
