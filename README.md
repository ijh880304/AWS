**Contents**

- [Summary](#summary)
- [Architecture](#architecture)
- [AWS IAM User](#aws-iam-user)
- [EKS Cluster](#eks-cluster)
- [ECR Repository](#ecr-repository)
- [Bastion Host](#bastion-host)
- [EKS Cluster Security Group](#eks-cluster-security-group)
- [Kubectl Install](#kubectl-install)
- [Docker Install](#docker-install)
- [BW Container Edition - Base Image](#bw-container-edition---base-image)
- [Sample Project](#sample-project)
- [AWS ELB(ALB)](#aws-elb-alb-)
- [Gitlab Host](#gitlab-host)
- [Jenkins Host](#jenkins-host)



## Summary



> 이 문서는 [EKS][1] 환경에 TIBCO BWCE(Container Edition) 제품을 배포 및 테스트 할 수 있도록 클러스터 환경 및 CI/CD를 구축하는 방법을 정리한 문서입니다.
> EKS 환경은 심플하게 구성 하였으며 일반적인 운영 환경에서는 보안을 위한 네트워크 관리와 보안그룹 관리가 디테일하게 필요하니 별도로 확인 하는 것을 추천 드립니다.





## Architecture



![](images/eks-diagram.png "Diagram")
- REST 서비스 로드밸런싱을 위해 ALB 사용
- EKS 클러스터 워커 노드는 2개(t3.medium)개로 구성
- Docker 컨테이너 이미지를 저장할 공간으로 ECR 사용
- BW 소스의 형상관리를 위해 Gitlab-CE 사용
- CI/CD를 위해 Jenkins 사용
- Build는 Maven을 사용하며 TIBCO BW Maven Plugin을 설치하여 사용
- 라이브러리 관리를 위해 Nexus 사용



## AWS IAM User

>  [AWS](https://aws.amazon.com/)에서 root Account를 생성(가입) 한 후 테스트에 사용 할 IAM User를 하나 생성 합니다.



- 사용자 추가

![](images/create-user.png "Create User")



- 권한은 편의를 위해 Administrator 권한을 부여합니다.

![grant-admin](images\grant-admin.png)



- 로그인

![iam-login2](images\iam-login2.png)



- 액세스 키 생성 - AWS CLI(Command Line Interface)의 권한 부여를 위한 키
- IAM 서비스 > 사용자 > {생성한User} > 보안 자격 증명 > 액세스 키 만들기 > **비밀 액세스 키 보관**

![iam-create-access-key](images\iam-create-access-key.png)



## EKS Cluster

> 실제 운영 환경에서는 VPC, Subnet, IAM Role, Security Group 등을 필요에 맞게 구성하여 사용 하지만 여기서는 eksctl을 사용해 대부분의 설정들을 디폴트로 하여 간단히 클러스터를 생성 합니다.



- AWS CloudShell 접속

![aws-cloud-shell](images\aws-cloud-shell.png)



- Key 생성

```shell
aws ec2 create-key-pair --key-name ESB-KEY-PAIR --query "KeyMaterial" --output text > ESB-KEY-PAIR.pem
```

![aws-cloud-shell-create-key-pair](images\aws-cloud-shell-create-key-pair.png)

생성 한 Key는 계속 사용 해야 하므로 다운로드 받아 보관

![key-pair-download](images\key-pair-download.png)



- eksctl 설치

```shell
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

![aws-cloud-shell-install-eksctl](images\aws-cloud-shell-install-eksctl.png)



- EKS 클러스터 생성 yml 파일 업로드

[create-cluster-nodegroup.yml](config/create-cluster-nodegroup.yml)

```yaml
apiVersion: 'eksctl.io/v1alpha5'
kind: ClusterConfig
metadata:
  name: JEJU-DEV-ESB-CLUSTER
  region: us-east-2
  version: '1.19'
vpc:
  cidr: "172.31.0.0/16" # 클러스터에서 사용할 VPC의 CIDR
  clusterEndpoints:
    publicAccess: true
    privateAccess: true
managedNodeGroups:

  - name: JEJU-DEV-ESB-CLUSTER-NG # 클러스터의 노드 그룹명
    instanceType: t3.medium
    desiredCapacity: 2
    volumeSize: 10
    iam:
      withAddonPolicies:
        ImageBuilder: true # AWS ECR에 대한 권한 추가
        albIngress: true  # alb ingress에 대한 권한 추가
    ssh:
      allow: true # 워커 노드에 SSH 접속 허용
      publicKeyName: ESB-KEY-PAIR # 워커 노드에 SSH 접속을 위해 사용할 pem키 명(aws key pairs에 등록되어 있어야함)
```

![aws-cloud-shell-upload-yml](images\aws-cloud-shell-upload-yml.png)



- EKS 클러스터 생성

```shell
eksctl create cluster -f create-cluster-nodegroup.yml
```

![aws-cloud-shell-create-cluster](images\aws-cloud-shell-create-cluster.png)



## ECR Repository

> Kubernetes에 컨테이너 배포 시 참조 할 Docker Image를 보관 및 관리하기 위한 저장소 생성



- AWS Console의 [ECR 서비스](https://us-east-2.console.aws.amazon.com/ecr/get-started?region=us-east-2)에서 시작

![aws-ecr-create-repository-console](images\aws-ecr-create-repository-console.png)

- 또는 AWS CLI에서 생성(e.g. BW 베이스 이미지를 저장할 저장소)

```shell
aws ecr create-repository --repository-name tibco/bwce-base
```



## Bastion Host
> VPC 내에서 Docker Image 관리나 클러스터 관리 및 배포를 위한 Host를 생성합니다.(Optional)



- EC2 서비스 > 인스턴스 > 인스턴스 시작 (t2.micro 타입으로도 충분)

![ec2-bastion-create-2](images\ec2-bastion-create-2.png)



- 클러스터 생성 중 자동으로 생성 된 VPC가 선택 되며, 서브넷은 3개의 Public 서브넷 중 임의 선택

![ec2-bastion-create-3](images\ec2-bastion-create-3.png)



- 기존에 생성했던 키페어 선택 (해당 호스트에 ssh 접속 시 키페어 파일이 필요함)

![ec2-bastion-create-5](images\ec2-bastion-create-5.png)



- 인스턴스 기동 완료 후 SSH 접속 - 여러 SSH 툴 중 MobaXterm 사용함

![MobaXterm-edit-session](images\MobaXterm-edit-session.PNG)



- [eksctl 설치 참고](#EKS-Cluster-Create)

- AWS CLI Key 설정

```shell
aws configure
```

- [AWS 사용자 생성](#AWS-IAM-User-Create) 중 생성한 Key를 입력하고, 클러스터를 생성한 리전(AWS Console 우측 상단)을 입력

```yaml
AWS Access Key ID [None]: AKIATJWC3KXXLUUIC5EY
AWS Secret Access Key [None]: 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
Default region name [None]: us-east-2
Default output format [None]: json
```



## EKS Cluster Security Group

> 다른 Host나 Loadbalancer 등에서 해당 클러스터에 접근하기 위해 클러스터의 보안그룹에 인바운드 규칙을 추가해 주어야 합니다.



- 클러스터 > 네트워크 > 클러스터 보안 그룹

![eks-cluster-security-allow-bastion-1](images\eks-cluster-security-allow-bastion-1.png)

- 클러스터 API Server Endpoint에 접근하기 위해 HTTPS(443 포트)에 대해 Bastion Host 보안그룹을 추가

![eks-cluster-security-allow-bastion-2](images\eks-cluster-security-allow-bastion-2.png)



## Kubectl Install



- 설치 [참고](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html)

```shell
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version --short --client
```



- AWS CLI 설정이 완료 되어 있다면 아래 명령어로 .kube/config 파일이 자동으로 업데이트 됨

```shell
aws eks update-kubeconfig --name JEJU-DEV-ESB-CLUSTER
```



## Docker Install



- 설치 [참고](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html)

```shell
sudo yum update -y
sudo amazon-linux-extras install docker -y
sudo yum install docker -y
sudo service docker start
sudo usermod -a -G docker ec2-user
```



- 그룹에 사용자 추가는 세션을 다시 열었을 때 적용 되므로 세션을 다시 열고 아래 명령어로 설치 확인

```shell
docker info
```




## BW Container Edition - Base Image
> BWCE Application을 Container 환경에 배포, 기동하기 위해 사용 할 Base Docker 이미지가 필요합니다.
>
> Base 이미지는 linux OS에 BW Appnode(JVM)를 기동할 수 있도록 제품들이 설치 된 상태의 이미지입니다.



- Bastion Host에 적당한 디렉토리를 생성

```shell
mkdir applications/bwceBase
```



- bwce-runtime-2.6.1.zip 업로드 및 압축 해제 및 준비 사항

```shell
unzip bwce-runtime-2.6.1.zip
cd /home/ec2-user/applications/bwceBase/bwce-runtime-2.6.1/tibco.home/bwce/2.6/docker
cp ~/applications/bwceBase/bwce-runtime-2.6.1.zip resources/bwce-runtime/
//Base 이미지에 추가하고자 하는 라이브러리가 있을 경우
cp ~/applications/bwceBase/*.jar resources/addons/jars/
```



- [Dockerfile](config/Dockerfile_base) 수정

```shell
vi Dockerfile
```

```dockerfile
FROM centos:7
LABEL maintainer="TIBCO Software Inc."
ADD . /
RUN chmod 755 /scripts/*.sh && yum -y update && yum -y install unzip ssh net-tools
RUN localedef -f UTF-8 -i ko_KR ko_KR.utf8 && ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime
ENV LANG ko_KR.UTF-8
ENV LC_ALL ko_KR.UTF-8
ENTRYPOINT ["/scripts/start.sh"]
```



- Docker Image Build (이미지 테그의 ECR Domain 명은 [ECR 생성](#ECR-Repository-Create) 시 확인 가능)

```shell
docker build -t 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/bwce-base .
```



## Sample Project



- BW Sample Project 생성

New - BusinessWorks Application Module

Deployment Target : Container 선택

![bw-sample-project-1](images\bw-sample-project-1.png)

Simple REST Service 생성(port: 8080)

![bw-sample-project-2](images\bw-sample-project-2.png)

EAR 배포파일 생성

![bw-sample-project-3](images\bw-sample-project-3.png)



- Bastion Host에서 Docker Build 및 EKS 배포

Bastion Host에 디렉토리 생성 후 ear 파일 업로드

```shell
mkdir sampleProject
```

[Dockerfile](config/Dockerfile_sampleProject) 생성

```dockerfile
FROM 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/bwce-base:latest
MAINTAINER Tibco
ADD SampleProject.application_1.0.0.ear /
EXPOSE 8080
```

Docker build

```
docker build -t 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/sample-project .
```

Docker image 테스트

```shell
docker run -p 8080:8080 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/sample-project
curl http://localhost:8080/sample -X POST -d '{"key1":"value1"}' -H "Content-Type: application/json"
"hello-sample-1"
```

Docker Login & Docker Push (ECR 업로드)

```shell
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 226968163822.dkr.ecr.us-east-2.amazonaws.com
aws ecr create-repository --repository-name tibco/sample-project
docker push 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/sample-project
```

Kubernetes Deploy, [deployment-service.yml](config/deployment-service_sampleProject.yml)

```shell
vi deployment-service.yml
kubectl apply -f deployment-service.yml
kubectl get pods
kubectl logs bwce-sample-project-6df9b89965-4pkv9
```

Kubernetes Service(NodePort) Test

```shell
curl http://ip-172-31-78-104.us-east-2.compute.internal:30080/sample -X POST -d '{"key1":"value1"}' -H "Content-Type: application/json"
```



- CI/CD 구성 중 Maven Build를 지원하기 위해 개발 환경에 Maven Plugin 설치

Studio 종료 후 TIB_BW_Maven_Plugin_2.8.0.zip 압축해제

TIB_BW_Maven_Plugin_2.8.0 > install.bat 실행



- Studio 재기동 후 프로젝트 POM 생성

![sample-project-generate-POM](images\sample-project-generate-POM.png)

![sample-project-generate-POM-2](images\sample-project-generate-POM-2.png)

![sample-project-generate-POM-3](images\sample-project-generate-POM-3.png)

![sample-project-generate-POM-4](images\sample-project-generate-POM-4.png)

read-project-properties Goal은 무시하도록 quick fix(Permanently mark goal read-project-properties in pom.xml as ignored in Eclipse build)

![sample-project-generate-POM-5](images\sample-project-generate-POM-5.png)



- Maven build 수행 (**application.parent 프로젝트에서 수행**)

![sample-project-maven-build](images\sample-project-maven-build.png)

![sample-project-maven-build-2](images\sample-project-maven-build-2.png)

![sample-project-maven-build-3](images\sample-project-maven-build-3.png)

```shell
mvn clean package initialize com.tibco.plugins:bw6-maven-plugin:bwfabric8json
```

> properties 파일 기준으로 template yml 파일 및 관련 배포 yml 모두 재 생성

```shell
mvn clean package initialize fabric8:resource
```

> template yml 기준으로 배포 yml만 생성



- application 프로젝트의 pom.xml Property 수정

<docker.env.property.file>docker-host-env-dev.properties</docker.env.property.file>

~~C:\Users\IJH_DESKTOP\workspace_git_repo\SampleProject.application\docker-host-env-dev.properties~~

> 절대경로에서 상대경로로 수정







## AWS ELB(ALB)



- EC2 > LoadBalancer > Load Balancer 생성

Application Load Balancer 선택

![create-alb-1](images\create-alb-1.png)

![create-alb-2](images\create-alb-2.png)

![create-alb-3](images\create-alb-3.png)

대상그룹 포트는 30080으로 설정

![create-alb-4](images\create-alb-4.png)

![create-alb-5](images\create-alb-5.png)

EKS 2개 Node 선택하여 추가

![create-alb-6](images\create-alb-6.png)

[EKS Cluster 보안그룹](#EKS-Cluster-Security-Group)에 30080포트에 대해 ALB 보안그룹을 추가

![create-alb-7](images\create-alb-7.png)

SampleProject 내 HealthCheck REST 서비스 추가

![bw-sample-project-healthcheck](images\bw-sample-project-healthcheck.png)

![bw-sample-project-healthcheck-2](images\bw-sample-project-healthcheck-2.png)

![bw-sample-project-healthcheck-3](images\bw-sample-project-healthcheck-3.png)

[Sample Project](#Sample-Project) 컨테이너 재 배포 후 ALB 대상그룹 Health Check 상태 확인

![create-alb-healthcheck-1](images\create-alb-healthcheck-1.png)

ALB를 통한 REST Test

```sh
curl http://ALB-JEJU-DEV-ESB-CLUSTER-58579496.us-east-2.elb.amazonaws.com/sample -X POST -d '{"key1":"value1"}' -H "Content-Type: application/json"
```



## Gitlab Host
> EC2 Instance 생성 시 t2.medium 이상 권장하며 Gitlab 접속 포트(e.g. 8099) 오픈하여 생성

- 설치 [참고](https://about.gitlab.com/install/#centos-7), EXTERNAL_URL은 해당 Host의 "http://도메인명:포트"

```shell
sudo yum install -y curl policycoreutils-python openssh-server openssh-clients perl
sudo systemctl enable sshd
sudo systemctl start sshd
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo EXTERNAL_URL="http://ec2-18-223-206-234.us-east-2.compute.amazonaws.com:8099" yum install -y gitlab-ce
```



- 설치 및 기동 완료 후 최초 로그인 시 root pw 초기화

![gitlab-root-pw](images\gitlab-root-pw.png)



- User 생성

![gitlab-add-user](images\gitlab-add-user.png)



- Gitlab Project 생성 및 BW Project 연결

![gitlab-create-project-1](images\gitlab-create-project-1.png)

Gitlab Project 생성 완료 된 후 HTTP URL 확인

![gitlab-create-project-2](images\gitlab-create-project-2.png)

Gitlab 형상관리 대상 프로젝트를 모두 선택 후 우클릭 > Team > Share Project.. > Git > Local Repository를 생성하여 선택

![gitlab-create-project-3](images\gitlab-create-project-3.png)

프로젝트 우클릭 > Team > Remote > Fetch From

![gitlab-create-project-4](images\gitlab-create-project-4.png)

URI에 Gitlab Project URI(http://{gitlab-host}:{gitlab-port}/{user}/{project-name.git})와 인증정보 입력

![gitlab-create-project-5](images\gitlab-create-project-5.png)

Git Staging View에서 Unstaged Changes의 모든 항목을 Staged Changes로 옮기고 Commit

![gitlab-create-project-6](images\gitlab-create-project-6.png)

Gitlab UI에서 정상 Commit 확인

![gitlab-create-project-7](images\gitlab-create-project-7.png)



## Jenkins Host
>  EC2 Instance 생성 시 t2.medium 이상 권장하며 Jenkins 접속 포트(e.g. 8080) 오픈하여 생성

- Jenkins 설치 [참고](https://pkg.jenkins.io/redhat-stable/)

```shell
sudo yum install java-1.8.0-openjdk-devel -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade -y
sudo yum install jenkins -y
sudo systemctl daemon-reload
sudo systemctl start jenkins
sudo systemctl status jenkins
```



- 최초 Admin PW 확인

```sh
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

![jenkins-cat-admin-pw](images\jenkins-cat-admin-pw.png)



- Jenkins UI 접속, 최초 Admin PW 입력

http://ec2-18-223-210-59.us-east-2.compute.amazonaws.com:8080

![jenkins-init-admin-pw](images\jenkins-init-admin-pw.png)

![jenkins-install-plugin-suggested](images\jenkins-install-plugin-suggested.png)



- PipeLine 구성을 위한 User 생성

![jenkins-admin-user](images\jenkins-admin-user.png)

![jenkins-welcome](images\jenkins-welcome.png)



- Maven 추가

![jenkins-add-maven](images\jenkins-add-maven.png)



- Git 설치

```sh
sudo yum install git -y
```



- [Docker 설치](#Docker-Install) 후 jenkins User도 docker 그룹에 추가

```sh
sudo usermod -a -G docker jenkins
```



- Kubectl 설치

```shell
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo chown jenkins:jenkins ./kubectl
sudo mv ./kubectl /usr/bin/
sudo su -s /bin/bash jenkins
kubectl version --short --client
```

```shell
aws configure
```

```yaml
AWS Access Key ID [None]: AKIATJWC3KXXLUUIC5EY
AWS Secret Access Key [None]: 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
Default region name [None]: us-east-2
Default output format [None]: json
```

```shell
aws eks update-kubeconfig --name JEJU-DEV-ESB-CLUSTER
```



- Plugin 설치 - Jenkins 관리 > 플러그인 관리 > 설치 가능 탭 > 설치할 플러그인 검색 및 설치
  - [GitLab Plugin](https://plugins.jenkins.io/gitlab-plugin)
  - [Git plugin](https://plugins.jenkins.io/git)
  - [Amazon ECR plugin](https://plugins.jenkins.io/amazon-ecr)
  - [AWS Global Configuration Plugin](https://plugins.jenkins.io/aws-global-configuration)
  - [Docker Pipeline](https://plugins.jenkins.io/docker-workflow)
  - [Docker plugin](https://plugins.jenkins.io/docker-plugin)
  - [docker-build-step](https://plugins.jenkins.io/docker-build-step)



- BWCE Maven Build를 위해 TIBCO 제품 설치 - 설치 파일 업로드 및 압축 해제

[TIB_bwce_2.6.1_linux26gl23_x86_64.zip](files/TIB_bwce_2.6.1_linux26gl23_x86_64.zip)

[TIB_BW_Maven_Plugin_2.8.0.zip](files/TIB_BW_Maven_Plugin_2.8.0.zip)

```shell
unzip TIB_bwce_2.6.1_linux26gl23_x86_64.zip -d TIB_bwce_2.6.1_linux26gl23_x86_64
unzip TIB_BW_Maven_Plugin_2.8.0.zip -d TIB_BW_Maven_Plugin_2.8.0
```

- BWCE 설치 (silent 설치는 Option)

```shell
cd TIB_bwce_2.6.1_linux26gl23_x86_64
vi TIBCOUniversalInstaller_bwce_2.6.1.silent
```

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
<comment>---Universal Installer Silent Installation Properties---</comment>


<!--accept the license agreement-->
<entry key="acceptLicense">true</entry>

<!--the root installation directory-->
<!--if the product does not support multiple instances and TIBCO_HOME has already been set then-->
<!--this value is ignored and the existing TIBCO_HOME is used as the installation root-->
<entry key="installationRoot">/home/ec2-user/tibco/bwce</entry>

<!--If using an existing environment then the installationRoot AND environmentName MUST match a pre-existing environment-->
<!--If creating a new environment then the installationRoot AND environmentName MUST BE UNIQUE and not match a pre-existing environment-->
<entry key="environmentName">bwce</entry>

<!-- The following configuration parameters control the download of all LGPL components used by BW, e.g. Hibernate -->
<!-- LGPLAssemblyLicenseAccepted - true or false to accept the license -->
<!-- LGPLAssemblyDownload - true or false to download the assembly from the public tibco server -->
<!-- LGPLAssemblyPath - if LGPLAssemblyDownload is false, this is the path where the LGPL assembly is located on the system (note that all LGPL assemblies must be in the same directory -->
<entry key="LGPLAssemblyLicenseAccepted">true</entry>
<entry key="LGPLAssemblyDownload">true</entry>
<entry key="LGPLAssemblyPath">/home/ec2-user/tibco/thirdpartyDownload</entry>

<!--Profiles Selection-->
<!-- The 'selectedProfiles' value is a comma separated list of profiles to install. If a profile listed below does not exist, then it is ignored and the default profile will be used. At least one profile must be listed if useInstallProfile is set to true -->
<entry key="useInstallProfile">true</entry>
<entry key="selectedProfiles">Typical</entry>

</properties>
```

```sh
./TIBCOUniversalInstaller-lnx-x86-64.bin -silent
```

- Maven Plugin 설치

```sh
cd TIB_BW_Maven_Plugin_2.8.0
./install.sh /home/ec2-user/tibco/bwce
```