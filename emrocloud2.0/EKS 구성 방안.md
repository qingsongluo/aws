## EKS 구성 가이드

### 1 사전 준비

#### 1.1 attach an IAM role to an basrion node
##### role 의대한 최소화 부여 필요 있음
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "iam:*",
                "sts:DecodeAuthorizationMessage",
                "cloudformation:*",
                "ec2:*",
                "autoscaling:*",
                "eks:*"
            ],
            "Resource": "*"
        }
    ]
}
```

#### 1.2 install kubelet - basion node
``` 
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv kubectl /usr/local/bin

https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/install-kubectl.html
```
#### 1.3 install eksctl - basion node
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/eksctl.html
```


#### 1.4 기존에 VPC에 VPN 연결 된 서비스가 많아서 기존 VPC에 배포 해야 됨

##### 1.4.1 eksctl cluster ignition yaml 편집
```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: emrocloud2.0-dev-eks
  region: ap-northeast-2

kubernetesNetworkConfig:
  serviceIPv4CIDR: 172.17.0.0/24
#기존 VPC 정보
vpc:
  id: "vpc-07de72dabe3e92a24"
  cidr: "10.100.0.0/16" 
  subnets:
    private:
      emrocloud-dev-private-1a:
        id: "subnet-0b6ede2a643c6135e"
        cidr: "10.100.2.0/24"
      emrocloud-dev-private-2c:
        id: "subnet-0e92780fbccb23d64"
        cidr: "10.100.3.0/24"
nodeGroups:
  - name: ng-1-workers
    labels: { role: workers }
    tags:
      nodegroup-role: worker
    instanceType: t3.xlarge
    desiredCapacity: 2
    volumeSize: 80
    privateNetworking: true
    ssh: # import public key from file
      publicKeyPath: /home/eksadmin/.ssh/id_rsa.pub
    taints:
    - key: role
      value: worker
      effect: NoSchedule
   - name: ng-2-infra
    labels: { role: infra }
    tags:
      nodegroup-role: infra
    instanceType: t3.xlarge
    desiredCapacity: 2
    volumeSize: 80
    privateNetworking: true
    availabilityZones: ap-northeast-2c
    ssh: # import public key from file
      publicKeyPath: /home/eksadmin/.ssh/id_rsa.pub
cloudWatch:
  clusterLogging:
    enableTypes: ["all"]
    logRetentionInDays: 14
```
##### 1.4.2 EKS SubNets Tag Labeling
##### * 모든 서브넷 적용
|Key|Value|
|--|--|
|`kubernetes.io/cluster/{cluster-name}`|shared |

##### * Kubernetes가 내부 로드 밸런서에 프라이빗 서브넷을 사용하도록 허용하려면 다음 키–값 페어를 사용하여 VPC의 모든 프라이빗 서브넷에 태그를 지정합니다.
|Key|Value|
|--|--|
|`kubernetes.io/role/internal-elb`|1 |

##### * Kubernetes가 외부 로드 밸런서에 태그가 지정된 서브넷만 사용하도록 허용하려면 다음 키–값 페어를 사용하여 VPC의 모든 퍼블릭 서브넷에 태그를 지정합니다.
|Key|Value|
|--|--|
|`kubernetes.io/role/elb`|1 |

##### * Cluster Autoscaler 구성시  해당  Autoscaler 그룹에 태그 지정합니다.
|Key|Value|
|--|--|
|`k8s.io/cluster-autoscaler/<cluster-name>`|owned |
|`k8s.io/cluster-autoscaler/enabled`|TRUE |

#### 1.5 EKS cluster 생성
```
eksctl create cluster --config-file=cluster-demo.yaml 
```
###### --with-oidc 옵션 없으면 수동으로 IAM OIDC provider 생성 해야 합니다.
###### eksctl utils associate-iam-oidc-provider --cluster `cluster_name` --approve 
###### https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/enable-iam-roles-for-service-accounts.html

### 2 EKS 추가 구성
####  Node ROLES 구성
```
worker node role 정의, 각service 별 노드 그룹 생성 필요 없음.
#노드 그룹 생성 후 k8s labeling
kubectl label node ${nodename} node-role.kubernetes.io/worker=worker

kubectl label node ${nodename} node-role.kubernetes.io/infra=infra

```

####  install helm
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh 
chmod 700 get_helm.sh 
./get_helm.sh

helm help

https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/helm.html
```
```
eksctl utils associate-iam-oidc-provider --cluster emrocloud-eks-demo --region=ap-northeast-2 --approve
```

####  IAM OIDC provider 생성
```
eksctl utils associate-iam-oidc-provider --cluster emrocloud-eks-demo --region=ap-northeast-2 --approve
```

####  Adding the Amazon VPC CNI Amazon EKS add-on
##### * Prerequisites
```
삭제 하지 말아요 aws-node {PodSecurityPolicy: no providers available to validate pod request}
# delete eks default PSP
kubectl delete -f privileged-podsecuritypolicy.yaml

https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/pod-security-policy.html#psp-install-or-restore-default 
```
```
# Configuring the Amazon VPC CNI plugin to use IAM roles for service accounts
eksctl create iamserviceaccount \
    --name aws-node \
    --namespace kube-system \
    --cluster {cluster_name} \
    --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy \
    --approve \
    --override-existing-serviceaccounts \
    --region=ap-northeast-2
```
##### * add-on Amazon VPC CNI plugin 
```
# sa role-arn 확인
kubectl get sa -n kube-system aws-node -o yaml |grep role-arn
# output: eks.amazonaws.com/role-arn: arn:aws:iam::375702802892:role/eksctl-emrocloud-eks-demo-addon-iamserviceac-Role1-PUSSROIKWTDD

#add-on
eksctl create addon \
    --name vpc-cni \
    --version latest \
    --cluster my-cluster \
    --region=ap-northeast-2 \
    --service-account-role-arn {arn:aws:iam::111122223333:role/eksctl-my-cluster-addon-iamserviceaccount-kube-sys-Role1-UK9MQSLXK0MW} \
    --force
```
####  Amazon EKS 클러스터에 IAM 사용자 또는 역할 추가
```
kubectl edit configmap -n kube-system aws-auth

apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::375702802892:role/eksctl-emrocloud-eks-demo-nodegro-NodeInstanceRole-1E5OHERZFVU5B
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::375702802892:role/eksctl-emrocloud-eks-demo-nodegro-NodeInstanceRole-1SRVFJ62NHI1
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:masters
      rolearn: arn:aws:iam::375702802892:role/platformteam-qingsong-eksadmin-role
      username: admins
```

####  ingress
```
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.1/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster={my_cluster} \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn={arn:aws:iam::111122223333:policy/AWSLoadBalancerControllerIAMPolicy} \
  --override-existing-serviceaccounts \
  --region=ap-northeast-2 \
  --approve

helm repo add eks https://aws.github.io/eks-charts

helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=cluster-name \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller 
```

#### Enabled  EKS control plane logging
```
aws eks update-cluster-config \ --region `<region-code>` \ --name `<prod>` \ --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/control-plane-logs.html
```

#### Cluster Autoscaler
```
vi cluster-autoscaler-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}

aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document file://cluster-autoscaler-policy.json \
    --profile ecp-qingsong

eksctl create iamserviceaccount \
  --cluster=emrocloud-eks-demo \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws:iam::375702802892:policy/AmazonEKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --profile ecp-qingsong \
  --approve


curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
#edit 1 deploy to infra node 
      nodeSelector:
        role: infra
#edit 2
<YOUR CLUSTER NAME> to {my cluster name}

#edit 3 command options
- --balance-similar-node-groups
- --skip-nodes-with-system-pods=false

kubectl apply -f cluster-autoscaler-autodiscover.yaml

#serviceAccount annotate 확인
#eks.amazonaws.com/role-arn=arn:aws:iam::<ACCOUNT_ID>:role/<AmazonEKSClusterAutoscalerRole> {eksctl create iamserviceaccount output 동일}

kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'



#참고 자료
https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/autoscaling.html
```

####  EFS CSI driver
##### *  Create an IAM policy and role
```
curl -o iam-policy-example.json https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/v1.3.2/docs/iam-policy-example.json

aws iam create-policy \
--policy-name AmazonEKS_EFS_CSI_Driver_Policy \
--policy-document file://iam-policy-example.json \
--profile ecp-qingsong

#eks serviceAccount에 해당 정책 적용
eksctl create iamserviceaccount \
    --name efs-csi-controller-sa \
    --namespace kube-system \
    --cluster emrocloud-eks-demo \
    --attach-policy-arn arn:aws:iam::375702802892:policy/AmazonEKS_EFS_CSI_Driver_Policy \
    --approve \
    --override-existing-serviceaccounts \
    --profile ecp-qingsong
```

##### *  install  EFS driver
```
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm repo update

helm upgrade -i aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
    --namespace kube-system \
    --set image.repository=602401143452.dkr.ecr.ap-northeast-2.amazonaws.com/eks/aws-efs-csi-driver \
    --set controller.serviceAccount.create=false \
    --set controller.serviceAccount.name=efs-csi-controller-sa
```
##### *  create EFS file system
```
vpc_id=$(aws eks describe-cluster \
    --name emrocloud-eks-demo \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --profile ecp-qingsong \
    --output text)
cidr_range=$(aws ec2 describe-vpcs \
    --vpc-ids $vpc_id \
    --query "Vpcs[].CidrBlock" \
    --profile ecp-qingsong \
    --output text)

# NFS 모든 eks에 속한 VPC에 inbound 모두 허용한 SG 생성 
security_group_id=$(aws ec2 create-security-group \
    --group-name eksdemoEFSsecuritygroup \
    --description "EFS security group" \
    --vpc-id $vpc_id \
    --profile ecp-qingsong \
    --output text)
aws ec2 authorize-security-group-ingress \
    --group-id $security_group_id \
    --protocol tcp \
    --port 2049 \
    --cidr $cidr_range \
    --profile ecp-qingsong

# EFS file system 생성
#파일 시스템 생성
file_system_id=$(aws efs create-file-system \
    --profile ecp-qingsong \
    --performance-mode generalPurpose \
    --query 'FileSystemId' \
    --output text)

#subnets 확인
aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$vpc_id" \
    --query 'Subnets[*].{SubnetId: SubnetId,AvailabilityZone: AvailabilityZone,CidrBlock: CidrBlock}' \
    --profile ecp-qingsong \
    --output table

#subnet 별 mount 설정 해야 함
aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id subnet-08f4a7b5f0d900609 \
    --profile ecp-qingsong \
    --security-groups $security_group_id
aws efs create-mount-target \
    --file-system-id $file_system_id \
    --subnet-id subnet-08e1db798feaf7815 \
    --profile ecp-qingsong \
    --security-groups $security_group_id
``` 
##### *  StorageClass 생성
```
curl -o storageclass.yaml https://raw.githubusercontent.com/kubernetes-sigs/aws-efs-csi-driver/master/examples/kubernetes/dynamic_provisioning/specs/storageclass.yaml
kubectl apply -f storageclass.yaml

#test
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-app
  labels:
    app: efs
spec:
  replicas: 2
  selector:
    matchLabels:
      app: efs
  template:
    metadata:
      labels:
        app: efs
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - efs
              topologyKey: kubernetes.io/hostname
      containers:
        - name: app
          image: centos
          command: ["/bin/sh"]
          args: ["-c", "sleep 10000"]
          volumeMounts:
            - name: persistent-storage
              mountPath: /data
      volumes:
        - name: persistent-storage
          persistentVolumeClaim:
            claimName: efs-claim
```
