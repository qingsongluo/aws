docker desktop install
https://docs.docker.com/desktop/windows/install/

wsl 설치 참고 링크
https://www.lainyzine.com/ko/article/how-to-install-wsl2-and-use-linux-on-windows-10/


환경 세팅
WSL2로 설치된 Linux 환경에서 작업. 
####aws cli 설치
'''
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
'''

## credentials파일 와 config 정보는 aws console에서 확인 할 수 있음
`mkdir ~/.aws`

cat credentials
[AIyundg]
aws_access_key_id = AKIAXVxxxxxxxxx5IKE5
aws_secret_access_key = mRjJxxxxxxxxxxxxxxxxxxCMPdN+BJ


cat config
[profile aieks]
role_arn = arn:aws:iam::533812731935:role/AI-EKS-Developer-admin-role
region = ap-northeast-2
source_profile=AIyundg
```
https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html


####kubectl 설치
```
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl 
chmod +x ./kubectl
sudo mv kubectl /usr/local/bin
```
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html

####kubeconfig 자동 생성
```
export profilename=aieks
export clustername=$(aws eks list-clusters --profile ${profilename} --query clusters --output text)
aws eks update-kubeconfig --region ap-northeast-2 --name ${clustername} --profile ${profilename}
```
확인 
kubectl get nodes
qsluo@DESKTOP-PHHH89P:~/tools$ kubectl get nodes
NAME                                                STATUS   ROLES    AGE   VERSION
ip-172-29-135-252.ap-northeast-2.compute.internal   Ready    <none>   18h   v1.22.6-eks-7d68063

####helm 설치
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh 
chmod 700 get_helm.sh 
./get_helm.sh

helm help

####helm S3 plugin 설치
helm plugin install https://github.com/hypnoglow/helm-s3.git







배포
#### ecr repository 확인
aws ecr describe-repositories --profile aieks --query 'repositories[].{repositoryName: repositoryName, repositoryUri: repositoryUri}'
```
[
    {
        "repositoryName": "emroai/webadmin",
        "repositoryUri": "533812731935.dkr.ecr.ap-northeast-2.amazonaws.com/emroai/webadmin"
    },
    {
        "repositoryName": "demo",
        "repositoryUri": "533812731935.dkr.ecr.ap-northeast-2.amazonaws.com/demo"
    }
]
```


ecr 생성 
aws ecr create-repository --repository-name emroai/webadmin --profile aieks

ecr registry certs 자동 생성
aws ecr get-login-password --region ap-northeast-2 --profile aieks | docker login --username AWS --password-stdin 533812731935.dkr.ecr.ap-northeast-2.amazonaws.com

docker buildx 세팅
```
1, docker desktop 실행(docker desktop에서는 buildx 내장 되여 있음 linux에서는 해당 plugin 설치 해야 함 Jenkins dind 이미지에서 해당 buildx 설치 해야 함)
2, docker pull docker/binfmt:820fdd95a9972a5308930a2bdfb8573dd4447ad3 (QEMU 이미지 다운 로드)
# 3. Create a new builder called mbuilder
docker buildx create --name mbuilder

# 4 Switch to the new builder drive
docker buildx use mbuilder

# 5. Start up the builder container
docker buildx inspect --bootstrap

```


##push 후 해당 이미지는 docker hub에서 OS/ARCH 항목에 보면 현재 amd64, arm64지원 되는 것을 보입니다.
##그냥 docker build 하면 작업 pc 환경의 cpu ARCH에 따라 싱클 ARCH만 지원 합니다. 그래서 buildx 사용해야 합니다.
##docker buildx build --push --platform linux/amd64,linux/arm64 -t luoqs1989/buildxtest:tomcatbulidx . (docker hub에 push하면 OS/ARCH항목에서 지원 되는 CUP가 보입니다.)

docker buildx build --push --platform linux/amd64,linux/arm64 -t 533812731935.dkr.ecr.ap-northeast-2.amazonaws.com/emroai/webadmin:v0.1 .

helm 으로 배포





