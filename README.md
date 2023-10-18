Requirements:
1. eksctl
2. awscli
3. kubectl
4. helm
5. aws-load-balancer-controller

How to create:

```bash
# install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

#create cluster 
eksctl create cluster --name=kaiyo-cluster --nodes=2 --region=ap-south-1  --node-type=t2.micro --managed

#create IAM policy 
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json
#added - elasticloadbalancing:AddTags to actions for the ALB IAM policy
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

#create a service account
#replace <account-id> with your account id
eksctl create iamserviceaccount \
    --cluster=kaiyo-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
    
#install helm
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh

helm repo add eks https://aws.github.io/eks-charts
helm repo update

#install aws load balancer controller
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
    --set clusterName=kaiyo-cluster \
    --set serviceAccount.create=false \
    --set serviceAccount.name=aws-load-balancer-controller \
    
#create namespace
kubectl create namespace kaiyo_1
kubectl create namespace kaiyo_2

#create deployment, service,ingress - use the yaml file in AWS_ALB_EKS_demo folder
#use the elastic IP of the ALB to access the application

```
How to delete:

```bash
#delete cluster
eksctl delete cluster --name=kaiyo-cluster --region=ap-south-1

#delete IAM policy
aws iam delete-policy --policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy

#delete service account
eksctl delete iamserviceaccount \
    --cluster=kaiyo-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller
```


