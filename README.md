# AWS - Amazon EKS - Hands on Lab

# Lab1

## 개요

- 실습 내용
  - AWS Load Balancer Controller 추가 기능 설치
  - Game 2048 배포
  - Game 2048 서비스 확인
- 실습 완료 후, 실습 리소스 삭제
  - Game 2048 삭제
  - AWS IAM - AWS Load Balancer Controller 관련 리소스 삭제

## 진행 절차

### 사전 환경 설정

1. EC2 인스턴스 생성 및 SSH 접속

2. AWS CLI v2 설치/업데이트 (https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/getting-started-install.html)

3. eksctl 설치 (https://eksctl.io/introduction/#installation)

- eksctl 설치

```shell
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin
```

- eksctl 설치 버전 확인

```shell
eksctl version
```

4. kubectl 설치
   
- kubectl 설치  
```shell
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.26.2/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```

- kubectl 설치 버전 확인

```shell
kubectl version --short --client
```


```shell
eks_cluster_name="demo-eks"
region_name="us-east-1"
aws eks update-kubeconfig --region $region_name --name $eks_cluster_name
```

5. Helm 설치
- Helm 설치

```shell
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

- Helm 설치 버전 확인

```shell
helm version
```
  


### Amazone VPC, EKS 리소스 생성

1.  aws console에서 생성 했음을 가정하고 진행
2.  aws configure로 접속 하여 리소스 확인


### 클러스터에 대한 IAM OIDC 공급자 생성
 
1. 클러스터에 대한 IAM OIDC 공급자 생성

```shell
eks_cluster_name="demo-eks"
region_name="us-east-1"
oidc_id=$(aws eks describe-cluster --region $region_name --name $eks_cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

- 출력 내용 확인
- **공백일 경우IAM OIDC 공급자 생성**

- IAM OIDC 공급자 생성
```shell
eksctl utils associate-iam-oidc-provider --cluster $eks_cluster_name --approve
```

- IAM OIDC 공급자 확인
```shell
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4
```

### AWS Load Balancer Controller
- https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html

- IAM 역할 생성. AWS Load Balancer Controller의 kube-system 네임스페이스에 aws-load-balancer-controller라는 Kubernetes 서비스 계정을 생성하고 IAM 역할의 이름으로 Kubernetes 서비스 계정에 주석을 답니다.
- my-cluster를 사용자 클러스터 이름으로 바꾸고 111122223333을 계정 ID로 바꾼 다음 명령을 실행합니다.

```shell
eks_cluster_name= 
```

```shell
region_name=
```

```shell
role_name=Custom_EKS_LBC_Role-$eks_cluster_name
```

```shell
account_id=$(aws sts get-caller-identity --query 'Account' --output text)
```
- name, role-name 모두 다르게 해야 error가 안남. **--name=aws-load-balancer-controller-${eks_cluster_name}**  이부분 모두 다르게 나오도록 수정 (2024년 06월 14일)
```shell
eksctl create iamserviceaccount \
  --cluster=${eks_cluster_name} \
  --namespace=kube-system \
  --name=aws-load-balancer-controller-${eks_cluster_name} \
  --role-name ${role_name} \
  --attach-policy-arn=arn:aws:iam::${account_id}:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

- Helm Repository 추가

```shell
helm repo add eks https://aws.github.io/eks-charts
```

- Helm Local Repository 업데이트

```shell
helm repo update
```

- Helm Chart 설치
```shell
cluster_name=
```

```shell
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${eks_cluster_name} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller-${eks_cluster_name} 
```

### Game 2048
- kubectl
- https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/alb-ingress.html

- Game 2048을 샘플 애플리케이션으로 배포하여 AWS Load Balancer Controller가 인그레스 대상의 결과로 AWS ALB를 생성하는지 확인합니다.

```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/examples/2048/2048_full.yaml
```

- 몇 분후, 인그레스 리소스가 다음 명령으로 생성되었는지 확인합니다.

```shell
kubectl get ingress/ingress-2048 -n game-2048
```


## Game 2048 삭제

1. Kubernetes - Game 2048 삭제

```shell
kubectl delete -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.7/docs/examples/2048/2048_full.yaml
```


## LoadbalancerController 삭제

1. Helm을 사용해서 삭제
```shell
helm uninstall aws-load-balancer-controller -n kube-system
```

### AWS IAM - AWS Load Balancer Controller 관련 리소스 삭제

1. AWS IAM - AWS Load Balancer Controller Role - Policy 연결 해제

```shell
account_id=$(aws sts get-caller-identity --query 'Account' --output text)
```

```shell
aws iam detach-role-policy \
  --policy-arn arn:aws:iam::${account_id}:policy/AWSLoadBalancerControllerIAMPolicy \
  --role-name AmazonEKSLoadBalancerControllerRole
```

2. AWS IAM - AWS Load Balancer Controller Policy 삭제

```shell
account_id=$(aws sts get-caller-identity --query 'Account' --output text)
```

```shell
aws iam delete-policy \
  --policy-arn arn:aws:iam::${account_id}:policy/AWSLoadBalancerControllerIAMPolicy
```

3. AWS IAM - AWS Load Balancer Controller Role 삭제

```shell
aws iam delete-role --role-name AmazonEKSLoadBalancerControllerRole
```

### Amazone VPC, EKS 리소스 삭제

1. AWS console로 이동하여 삭제
   
## Troubleshooting

### AWS IAM OIDC Provider 오류 발생 시 해결 방법

1. AWS CLI 인증 정보 디렉토리 생성

```shell
mkdir ~/.aws
```

2. AWS CLI 인증 정보 변경

```shell
vi ~/.aws/credentials
```
AWS CLI Credentials 저장


4. AWS IAM - Policy 생성

```shell
aws iam create-policy \
    --policy-name CustomIAMFullAccess \
    --policy-document file://CustomIAMFullAccess.json
```

5. AWS IAM - LabRole Role - Policy 연결

```shell
account_id=$(aws sts get-caller-identity --query 'Account' --output text)
```

```shell
aws iam attach-role-policy \
  --policy-arn arn:aws:iam::${account_id}:policy/CustomIAMFullAccess \
  --role-name LabRole
```

6. AWS CLI 인증 정보 롤백

```shell
mv ~/.aws ~/.aws.bak
```

7. 추후, IAM Policy 롤백

```shell
account_id=$(aws sts get-caller-identity --query 'Account' --output text)
```

```shell
aws iam detach-role-policy \
  --policy-arn arn:aws:iam::${account_id}:policy/CustomIAMFullAccess \
  --role-name LabRole
```

```shell
account_id=$(aws sts get-caller-identity --query 'Account' --output text)
```

```shell
aws iam delete-policy \
  --policy-arn arn:aws:iam::${account_id}:policy/CustomIAMFullAccess
``` 
