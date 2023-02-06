---
title: "Terraform - 기본인프라 구성,설명(vpc, subnet, ec2)"
categories: Infrastructure
tags:
  - Terraform
---

AWS를 이용해서 서버 하나를 구동시키기 위해선 몇가지 필수 자원생성 및 설정이 필요하다. 정말 순수하게 서버를 구동시키기 위한 인프라를 Terraform으로 구현해보자. 먼저 구현할 인프라를 도식화한 이미지를 보자.  

![generic_vpc](https://user-images.githubusercontent.com/52196792/216526331-113ce2e5-4d6a-4bbc-9ff6-591aba1a6b89.png){: .align-center}  
Infrastructure
{: .image-caption style="font-size: 14px;" }  

간단해 보이는 이미지와는 달리 직접 구성하기 위해선 꽤나 많은 리소스를 생성하고 연결시켜줘야한다. 생설해야할 리소스 목록을 정리하자면 아래와 같다.(리소스별 세부 설정까지 더한다면 더 복잡하다)
- vpc
- internet gateway  
- public subnet
- route table
- security group
- eip
- EC2  

왜 이렇게 복잡하게 구성요소들이 엮여있는 것인가? 라고 생각할 수 있지만 각 요소별로 존재의 의미가 있다. 하나씩 차근차근 구현해보면서 알아보자!  

**! Notice :** 루트에서 모든 리소스를 생성할 수 있지만 이글에선 module을 이용하여 자원을 생성/사용합니다.  
{: .notice--primary}  

<br />  

### Terraform 준비
[Terraform 설치 및 기본개념](https://rokwonk.com/infrastructure/Terraform-%EA%B8%B0%EB%B3%B8%EA%B0%9C%EB%85%90/)에 대해서는 이미 포스팅하였다. 시작 전 `provider`로 클라우드 및 계정을 설정해주자. 필자는 `main.tf` 파일을 만들고 아래와 같이 작성하였다.
```hcl
# main.tf
provider "aws" {
  region                   = "ap-northeast-2"
  shared_config_files      = ["~/.aws/config"]
  shared_credentials_files = ["~/.aws/credentials"]
  profile                  = "thepool"
}
```  
로컬 컴퓨터에 저장된 aws 계정 중 `default`를 사용할 경우 profile 및 shared_* 속성들은 작성하지 않아도 문제 없다. 이 후 해당 디렉터리에서 `terraform init` 명령어로 테라폼 프로젝트의 시작을 알리자!  
```bash  
$ terraform init
```  




<br />  

### VPC 구현  
이제 본격적으로 `resource`들을 생성해보자. 그 영광의 첫 번째는 VPC(Virtual Private Cloud)다. **VPC는 클라우드에서의 가상 사설 네트워크**이다. 즉, 클라우드 상에서 개발자의 가상공간을 만드는 것이다. 이 공간에서 부분적으로 네트워크를 나누거나, 개발자가 필요한 자원들을 생성할 수 있다.  
VPC 내부에서는 사설IP를 통해 소통할 수 있다. 인터넷을 통한 소통이 아닌 VPC내부에서 그들만의 리그를 펼칠 수 있는 것이다. 그러기 위해선 **VPC에 사설IP대역을 지정**해줘야한다. AWS는 [CIDR(싸이더, Classless Inter-Domain Routing)](https://ko.wikipedia.org/wiki/CIDR)을 이용하여 대역폭을 지정한다. 그러므로 VPC resource를 생성할때 CIDR은 필수적으로 입력해야할 속성이다.  

`modules/` 디렉터리와 그 내부에 `vpc/` 디렉터리를 만들자. 그리고 아래처럼 코드를 작성하였다. 외부에서 cidr과 name을 입력받을 수 있도록했고 vpc resource를 생성할때 입력받은 값들을 사용하였다. vpc의 `tags`는 해당 자원의 메타데이터 정보입력이라고 생각하면 된다.(필수는 아님)
```hcl
# modules/vpc/main.tf

# 입력받을 값
variable "vpc_cidr" {
  type = string
}

variable "vpc_name" {
  type = string
}

# 생성 리소스
resource "aws_vpc" "vpc" {
  cidr_block = var.vpc_cidr # "10.0.0.0/16"
  tags = merge(
    {
      Name = format("%s-vpc", var.name)
    },
    var.tags
  )
}

```  

**💡 공공IP vs 사설IP**  
공공IP는 인터넷에서 유일한 주소이다. 인터넷 상에서 단 하나의 IP만 존재한다. 이 IP를 이용해서 위치정보를 찾을 수 있다. 사설 IP는 어느 한 네트워크망에서 사용하는 IP로 해당 네트워크망에서 유일한 IP주소를 가진다. 당연히 서로다른 네트워크망에서는 같은 IP가 존재할 수 있다. 사설IP를 이용해 망 내부에서 서로를 인식할 수 있다.  
{: .notice--info}  

<br />  

### Internet Gateway 생성 및 VPC와 연결
VPC가 인터넷을 통해 외부 소통을 하기 위해선 Internet GateWay(이하 IGW)가 필요하다. IGW는 VPC당 하나만 연결가능하다.  



<br />  

### Subnet 생성
public과 private의 창이(연결된 라우트테이블이 igw와 연결되어있는가)


<br />  

### Route Table 생성



<br />  

### Security Group 생성
inbound, outbound 룰설정  
stateful 하다는 것 설명(NACL과의 차이점, NACL이 더 바깥을 감싸고 있음)  

<br />  

### eip 생성
동적인 ip 대신 바뀌지 않는 ip 만들기  
하지만 eip대신 도메인 네임을 등록하고 연결하는게 유연성이 높다는 것 설명  

<br />  

### ec2 생성  
쉽게 설명하자면 단순히 컴퓨팅 자원, 더 상세히 알아보자면 ami 설정, instance type(컴퓨팅 파워설정), 메모리 선택, SSH로 접속을 우한 key pair, 부팅시 실행할 스크립트 작성 등이 필요하다는 것을 알리기만(이후 AWS EC2에 블로깅 하기)  



<br />

### 완성! 서버로 접속해보기  
ec2에 연결된 기본 도메인으로 접속해보기

<br />  

### 추가
실제로 인프라는 이렇게 간단하게 끝나지 않는다. DB도 존재하고 public subnet처럼 외부로 공개되지 않고 private subnet으로 내부에서만 접근가능한 자원들도 있다. 뿐만 아니라 트래픽을 감당하기 위해 Auto Scale, Load Balancer와 같은 AWS 자원들을 사용하며, 구현하고자 하는 기능의 목적별로 무수히 많은 자원들을 사용할 수 있다. 앞으로도 직접 구현해 본 인프라를 블로깅해보도록 하겠다!

[ec2 구현시 필요한 것들](https://kimjingo.tistory.com/197)