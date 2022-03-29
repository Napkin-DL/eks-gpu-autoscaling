
# Cluster AutoScaler

https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/autoscaling.html

```bash
export AWS_REGION="ap-northeast-2"

eksctl utils associate-iam-oidc-provider \
    --cluster gpu-v1 \
    --approve
```

```bash
2022-03-28 23:34:19 [ℹ]  eksctl version 0.70.0
2022-03-28 23:34:19 [ℹ]  using region ap-northeast-2
2022-03-28 23:34:20 [ℹ]  will create IAM Open ID Connect provider for cluster "gpu-v1" in "ap-northeast-2"
2022-03-28 23:34:22 [✔]  created IAM Open ID Connect provider for cluster "gpu-v1" in "ap-northeast-2"
```

```bash
aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document file://cluster-autoscaler-policy.json

eksctl create iamserviceaccount \
  --cluster=gpu-v1 \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::681747700094:policy/AmazonEKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve
```

```bash
# cluster-autoscaler-autodiscover.yaml <YOUR CLUSTER NAME> 부분 변경
curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

```bash
# 로그 확인
kubectl logs -n kube-system  -f deployment/cluster-autoscaler
```

```bash
# should be checked
kubectl -n kube-system \
    annotate deployment.apps/cluster-autoscaler \
    cluster-autoscaler.kubernetes.io/safe-to-evict="false"

kubectl version --short | grep 'Server Version:' | sed 's/[^0-9.]*\([0-9.]*\).*/\1/' | cut -d. -f1,2
```

# AWS Load Balancer Controller

```bash
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster gpu-v1  \
    --approve

aws eks describe-cluster --name gpu-v1 --query "cluster.identity.oidc.issuer" --output text

# https://oidc.eks.ap-northeast-2.amazonaws.com/id/E7027A45F2989B456FF1F485FD367E57

aws iam list-open-id-connect-providers | grep 

# "Arn": "arn:aws:iam::681747700094:oidc-provider/oidc.eks.ap-northeast-2.amazonaws.com/id/E7027A45F2989B456FF1F485FD367E57"

curl -o alb_iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.0/docs/install/iam_policy.json
# curl -o alb_iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://alb_iam_policy.json

eksctl create iamserviceaccount \
    --cluster gpu-v1 \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --attach-policy-arn arn:aws:iam::681747700094:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve

helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
curl -Lo v2_4_0_full.yaml https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.0/v2_4_0_full.yaml
# replace cluster name
sed -i.bak -e 's|your-cluster-name|gpu-v1|' ./v2_4_0_full.yaml
kubectl apply -f v2_4_0_full.yaml

kubectl get deployment -n kube-system aws-load-balancer-controller

# log 확인
kubectl logs -f $(kubectl get po -n kube-system | egrep -o 'aws-load-balancer-controller-[A-Za-z0-9-]+') -n kube-system
```

# Uninstall

```bash
kubectl delete -f v2_4_0_full.yaml
```