
목차
[1.개요](#개요)
[2.AWS 사용자 계정 준비](#AWS-사용자-계정-준비)


## 개요
> 이 문서는 [EKS][1] 환경에 TIBCO BWCE(Container Edition) 제품을 배포 및 테스트 할 수 있도록 클러스터(+ALB) 환경 및 CI/CD를 구축하는 방법을 정리한 문서입니다.
1. 






   [1]: https://www.google.com/search?q=eks "Elastic Kubernetes Service"

## AWS 사용자 계정 준비




AWS
root Login - IAM User 생성 - IAM User Login
AWSCloudShell
eksctl create cluster -f create-cluster-nodegroup.yml

bastion host 생성
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

Key 생성
aws ec2 create-key-pair --key-name ESB-KEY-PAIR --query "KeyMaterial" --output text > ESB-KEY-PAIR.pem


aws configure
AWS Access Key ID [None]: AKIATJWC3KXXLUUIC5EY
AWS Secret Access Key [None]: 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
Default region name [None]: us-east-2
Default output format [None]: json



eksctl create nodegroup --config-file create-cluster-nodegroup.yml

https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/install-kubectl.html
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
kubectl version --short --client

aws eks update-kubeconfig --name JEJU-DEV-ESB-CLUSTER

https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html
sudo yum update -y
sudo amazon-linux-extras install docker -y
sudo yum install docker -y
sudo service docker start
sudo usermod -a -G docker ec2-user
docker info

bwce base
mkdir applications/bwceBase
upload bwce-runtime-2.6.1.zip
unzip bwce-runtime-2.6.1.zip
cd /home/ec2-user/applications/bwceBase/bwce-runtime-2.6.1/tibco.home/bwce/2.6/docker
cp ~/applications/bwceBase/bwce-runtime-2.6.1.zip resources/bwce-runtime/
vi Dockerfile
----
FROM centos:7
LABEL maintainer="TIBCO Software Inc."
ADD . /
RUN chmod 755 /scripts/*.sh && yum -y update && yum -y install unzip ssh net-tools
RUN localedef -f UTF-8 -i ko_KR ko_KR.utf8 && ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime
ENV LANG ko_KR.UTF-8
ENV LC_ALL ko_KR.UTF-8
ENTRYPOINT ["/scripts/start.sh"]
	
docker build -t 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/bwce-base .


mkdir /home/ec2-user/applications/sampleProject
upload SampleProject.application_1.0.0.ear
----
FROM 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/bwce-base:latest
MAINTAINER Tibco
ADD SampleProject.application_1.0.0.ear /
EXPOSE 8080

docker build -t 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/sample-project .
docker run -p 8080:8080 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/sample-project
curl http://ip-172-31-21-8.us-east-2.compute.internal:8080/sample -X POST -d '{"key1":"value1"}' -H "Content-Type: application/json"

aws ecr create-repository --repository-name tibco/sample-project
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 226968163822.dkr.ecr.us-east-2.amazonaws.com

docker push 226968163822.dkr.ecr.us-east-2.amazonaws.com/tibco/sample-project

vi deployment-service.yml
kubectl apply -f deployment-service.yml
deployment.apps/bwce-sample-project created
service/bwce-sample-project-service created

kubectl  get pods
kubectl  logs bwce-sample-project-6df9b89965-4pkv9

curl http://ip-172-31-78-104.us-east-2.compute.internal:30080/sample -X POST -d '{"key1":"value1"}' -H "Content-Type: application/json"

alb

curl http://ALB-JEJU-DEV-ESB-CLUSTER-58579496.us-east-2.elb.amazonaws.com/sample -X POST -d '{"key1":"value1"}' -H "Content-Type: application/json"



gitlab
https://about.gitlab.com/install/#centos-7
sudo yum install -y curl policycoreutils-python openssh-server openssh-clients perl
sudo systemctl enable sshd
sudo systemctl start sshd
sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo EXTERNAL_URL="https://ec2-3-128-171-196.us-east-2.compute.amazonaws.com" yum install -y gitlab-ce

ec2-3-128-171-196.us-east-2.compute.amazonaws.com

bpCY9s-obvGL6FyyDW5m

jenkins
https://pkg.jenkins.io/redhat-stable/
sudo yum install java-1.8.0-openjdk-devel -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo yum upgrade -y
sudo yum install jenkins -y
sudo systemctl daemon-reload
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
http://ec2-18-223-210-59.us-east-2.compute.amazonaws.com:8080









[googlelink]: https://google.com "Go google"

***

**double asterisks**

1. 순서가 필요한 목록
1. 순서가 필요한 목록
   - 순서가 필요하지 않은 목록(서브) 
   - 순서가 필요하지 않은 목록(서브) 
1. 순서가 필요한 목록
   1. 순서가 필요한 목록(서브)
   1. 순서가 필요한 목록(서브)

값 | 의미 | 기본값
---|:---:|---:
`static` | 유형(기준) 없음 / 배치 불가능 | `static`
`relative` | 요소 **자신**을 기준으로 배치 |
`absolute` | 위치 상 **_부모_(조상)요소**를 기준으로 배치 |
`fixed` | **브라우저 창**을 기준으로 배치 |

BREAK!

> 인용문을 작성하세요!
>> 중첩된 인용문(nested blockquote)을 만들 수 있습니다.
>>> 중중첩된 인용문 1


## AWS
![alt text](images/aws.png "Title")


```
public class BootSpringBootApplication {
  public static void main(String[] args) {
    System.out.println("Hello, Honeymon");
  }s
}
```
