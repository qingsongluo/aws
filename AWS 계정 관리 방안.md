- 기업 내에서 AWS 사용 시 업무 에 따른 자원 분리가 불가피 합니다. 따라서 하나의 기업에서는 수많은 aws account 생성해서 사용 하고 있습니다. 개발 환경에서는 Development account 운영 환경에서는 Production account.
이때 신규 입사자가 입사를 했을 때, 업무 환경에 따라 aws 사용자(user)이 필요하고 각각  account 에서 사용자(user) 생성 후 권한 부여 하는 방식으로 운영이 되면 인원이 많고 aws account 늘어지는 시기에 관리는 점점 어려워 집니다.

- AWS account 각각 별도로 사용자를 관리하는 방식 대신 단일 중심에서 모든 사용자를 집중 관리하여 계정 생성, 권한 부여 후 해당 계정이 다른 AWS account에 방문할 수 있도록 하는 것을 추천 합니다.
아래 구성도와 같이 사용자는 authentication account에서 계정 생성 후 Role 사용한 AWS 계정 간 액세스 권한 위임 하는 형태 입니다. 

- 해당 설정을 하기 위해 아래 절차대로 하시면 됩니다.

##### 1,authentication account에서 IAM user 생성
##### 2,Development account 에서 역할 생성 후 두 account간의 신뢰를 구성 및 권한 부여
##### 3,authentication account의 user에 대한 Switch Role 설정

![image](https://user-images.githubusercontent.com/38554040/150494227-a84da653-1608-4ec2-91e7-46034331631b.png)

- authentication account에서 IAM user 생성
![image](https://user-images.githubusercontent.com/38554040/150494343-5d738d28-ab71-4cb9-8a3c-2806fa83197c.png)

- 권한 추가 없이 다음
![image](https://user-images.githubusercontent.com/38554040/150494407-e80bf9ff-5355-4362-99a6-d1a722778fc4.png)

![image](https://user-images.githubusercontent.com/38554040/150494420-89475635-bf57-4e18-8c3c-27a522951205.png)

- Development account 에서 역할 생성 후 두 account간의 신뢰를 구성 및 권한 부여
##### Development account에 로그인 후 역할 생성 시 신뢰할 수 있는 유형의 개체 중 "다른 AWS 계정" 선택, authentication account의 계정 ID 입력
![image](https://user-images.githubusercontent.com/38554040/150494468-fa0d20af-6e84-4424-9086-48dd4eade0df.png)

![image](https://user-images.githubusercontent.com/38554040/150494482-2a8b92f3-43f2-4ea6-9a11-77a0fc49acb0.png)

![image](https://user-images.githubusercontent.com/38554040/150494497-2e100c05-5c9e-4ecc-b9fc-436aff148bcc.png)

##### Cross_admin 생성 후 해당 Role의 2가지 정보를 기억합니다
##### "Role ARN", "콘솔에서 역할을 전환할 수 있는 사용자에게 이 링크 제공"

![image](https://user-images.githubusercontent.com/38554040/150494536-0b0fcbd1-d1bb-40c9-8255-bad10085770e.png)


- authentication account의 user에 대한 Switch Role 설정
##### authentication account에 로그인 후 "dev-admin" 사용자의 "인라인 정책 추가" 설정 합니다.
![image](https://user-images.githubusercontent.com/38554040/150494577-b470343f-fa40-4474-ba1e-07b90bc3658d.png)

![image](https://user-images.githubusercontent.com/38554040/150494593-3492aee8-f09d-4ae2-a606-1308405fc65d.png)

##### 해당 내용으로 편집합니다. 전에 생성한 Cross_admin Role의 RAN을  "Resource"에 입력
```
{
"Version": "2012-10-17",
"Statement": {
"Effect": "Allow",
"Action": "sts:AssumeRole",
"Resource": "{dev account role RAN}"
}
}
```

- authentication account의 dev-admin user이용하여 DEV Account 접속

##### dev-admin(user)로 authentication account 로그인
##### Cross_admin Role에 있는 "콘솔에서 역할을 전환할 수 있는 사용자에게 이 링크 제공" 링크를 브라우저에 복사

![image](https://user-images.githubusercontent.com/38554040/150494727-175ce510-b23b-49d0-b1a7-6f63d7de144a.png)

