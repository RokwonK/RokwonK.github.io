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

**💡 VPC ++**  
AWS는 국가수준인 Region이 존재하며 Region은 가용역역인 AZ으로 나누어진다. 그리고 각 AZ는 여러개의 데이터센터를 가지고 있다. VPC는 Region 레벨에서 생성되고 내부 리소스들은 AZ를 지정하여 해당 AZ에 생성할 수 있다.
{: .notice--info}  

<br />  

### Internet Gateway 생성 및 VPC와 연결
VPC가 인터넷을 통해 외부 소통을 하기 위해선 Internet GateWay(이하 IGW)가 필요하다. IGW는 VPC당 하나생성 가능하다. 생성이후 VPC 내부의 리소스들은 IGW와 연결하여 인터넷과 소통할 수 있다.  

IGW를 생성할때 연결할 VPC를 선택한다. 이때 앞서 생선한 VPC를 참조하여 연결한다. 리소스 참조는 `${리소스명}.${지정한이름}`으로 참조한다.  
```hcl  
# modules/vpc/main.tf

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id
  tags = merge(
    {
      Name = format("%s-igw", var.name)
    },
    var.tags
  )
}
```

<br />  

### Subnet 생성
Subnet(서브넷)이란 **VPC의 IP 네트워크 주소대역을 나누어 만든 네트워크**다. 서브넷은 앞선 말한 것처럼 VPC 내부에 존재하며 하나의 네트워크망이기 때문에 네트워크 대역을 가지고 있으다. 또한 VPC Region 레벨에 생성된다면 Subnet은 AZ(가용영역) 레벨에 생성된다. 그렇기 때문에 Subnet 생성시 특정 AZ를 지정해 주어야한다. 이렇게 만들어진 서브넷 안에 RDS, EC2와 같은 리소스들을 위치시킬 수 있다. 각 Subnet은 아래 설명할 Route Table과 연결되며 **연결된 Route Table이 IGW로 통하는 길이 있다면 Public Subnet, 없다면 Private Subnet으로 나뉘어 진다.**  

아래는 Public Subnet을 만드는 코드다. 아직 IGW로 통하는 Route Table과 연결이 되어 있지는 않지만 Public으로 만들 예정이다. Subnet 같은 경우 보통 하나만 만들어지지 않는다. 다수의 Subnet을 만들기 위해 `for_each` 속성을 이용하였다. 생성할 Subnet의 정보는 모듈을 사용하는 곳에서 입력변수로 넘겨준다.  
```hcl
# modules/vpc/main.tf

variable "public_subnets" {
}

resource "aws_subnet" "public" {
  for_each          = var.public_subnets
  vpc_id            = aws_vpc.vpc.id
  cidr_block        = each.value["cidr"]
  availability_zone = each.value["zone"]

  tags = merge(
    {
      Name = format(
        "%s-public-subnet-%s",
        var.name,
        each.key
      )
    },
    var.tags
  )
}
```  

**🤔 서브넷을 나누는 이유**  
서브넷으로 나누는 이유는 더 많은 네트워크망을 만들기 위해서다. 망 별로 리소스가 존재하고 망끼리 소통을 할 수 있다. 더 많은 네트워크망을 만듬으로써 리소스끼리 그룹화 시킬 수 있다.
{: .notice--info}  

<br />  

### Route Table 생성
요청 발생 시 라우터로 향한다. **라우터에는 Route Table이 존재하는데 이것은 이정표 역할**을 한다. Route Table을 만들고 어느 경로에 대한 목적지를 지정한다면 해당 Route Table과 연결된 서브넷은 지정된 룰을 따른다.  

route table 리소스를 생성하고 경로에 대한 목적지 `route`를 설정한다. 이 후 만든 public subnet과 route table을 연결하여 subnet에서 나오는 요청들을 igw 내보낸다. `0.0.0.0/0`은 모든요청을 의미한다. 아래 작성된 `route`는 모든 요청을 igw로 보낸다는 뜻이다. route table에는 기본적으로 vpc 내부와 소통할 수 있는 테이블이 존재하기 때문에 사설IP를 이용한다면 요청이 igw로 통하지 않는다.  
```hcl  
# modules/vpc/main.tf

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = merge(
    {
      Name = format(
        "%s-public-route-table",
        var.name,
      )
    },
    var.tags,
  )
}

resource "aws_route_table_association" "public" {
  for_each       = var.public_subnets
  subnet_id      = aws_subnet.public[each.key].id
  route_table_id = aws_route_table.public.id
}
```  

마지막으로 생성된 VPC를 다른 모듈에서 사용할 수 있도록 output을 이용하여 값을 내보내자. 뿐만 아니라 다른 모듈에서 생성된 Subnet 내부에 리소스를 만들 수 있도록 Subnet 또한 output으로 값을 내보내자.  
```hcl  
# modules/vpc/main.tf

output "vpc_id" {
  value = aws_vpc.vpc.id
}

output "public_subnet_ids" {
  value = values(aws_subnet.public)[*].id
}
```

<br />  

### Security Group 생성  
Security Group(이하 보안그룹)은 방화벽과 같은 역할을 한다. 연결된 리소스로의 인바운드(들어오는), 아웃바운드(나가는)트래픽에 CIDR, Protocol, Port를 지정하여 보안정책을 설정할 수 있다.  

보안그룹을 생성해준다. vpc와는 다른 파일인 것을 인지하자. 즉, 보안그룹은 다른 모듈로써 만들고 있는 중이다. 생성된 VPC의 정보를 입력변수를 통해 받고 VPC에 종속되는 보안그룹을 생성한다.  
```hcl
# modules/security_group/main.tf

variable "vpc_id" {
  description = "vpc id"
  type        = string
}

# sg 구축
resource "aws_security_group" "ec2_sg" {
  vpc_id      = var.vpc_id
  name        = "ec2 security group"
  description = "ec2 security group"
  tags = {
    Name = "ec2 security group"
  }
}
```  

보안그룹 생성 후에는 보안그룹에 룰을 적용해주어야한다. 인바운드룰로써 http, https, ssh와 관련된 요청을 허락하고 아웃바운드로는 모든 트래픽을 허용해준다.  
```hcl  
# modules/security_group/main.tf

locals {
  http_port  = 80
  https_port = 443
  ssh_port   = 22
  any_port   = 0

  tcp_protocol = "tcp"
  any_protocol = "-1"

  all_ips = ["0.0.0.0/0"]
}

resource "aws_security_group_rule" "allow_http_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.ec2_sg.id

  from_port   = local.http_port
  to_port     = local.http_port
  protocol    = local.tcp_protocol
  cidr_blocks = local.all_ips
}

resource "aws_security_group_rule" "allow_https_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.ec2_sg.id

  from_port   = local.https_port
  to_port     = local.https_port
  protocol    = local.tcp_protocol
  cidr_blocks = local.all_ips
}

resource "aws_security_group_rule" "allow_ssh_inbound" {
  type              = "ingress"
  security_group_id = aws_security_group.ec2_sg.id

  from_port   = local.ssh_port
  to_port     = local.ssh_port
  protocol    = local.tcp_protocol
  cidr_blocks = local.all_ips
}

resource "aws_security_group_rule" "allow_all_outbound" {
  type              = "egress"
  security_group_id = aws_security_group.ec2_sg.id

  from_port   = local.any_port
  to_port     = local.any_port
  protocol    = local.any_protocol
  cidr_blocks = local.all_ips
}
```  

마지막으로 보안그룹을 참조할 수 있도록 생성된 값을 내보내준다.(ec2 생성시 해당 보안그룹을 사용할 예정)  

```hcl  
# modules/security_group/main.tf

output "id" {
  value = aws_security_group.ec2_sg.id
}
```


**🤔 Stateful한 Security Group**  
보안그룹은 Stateful하다. 여기서 Stateful하다는 것은 들어거나/내보낼 요청에 대해서 그 정보를 가지고 있다는 뜻이다. 예를 들어 외부로부터 들어오는 요청은 Inbound 룰을 따른다. 하지만 해당 요청을 응답할땐? Outbound 룰을 따를까? 아니다. 처음 Inbound로 들어왔을때 그 정보를 가지고 있기때문에 응답 시 Outbound 룰을 확인하지 않는다. 보안그룹과 마찬가지로 보안정책 설정도구인 NACL은 Stateless하여 요청 및 응답에 모두 Inbound, Outbound 룰을 적용하여 검사한다.  
{: .notice--info}  

<br />  

### eip 생성
ec2를 생성하면 동적 공공IP가 적용된다. 동적이기에 인스턴스를 잠시 멈춘다면 IP는 바뀌게 된다. 이런 문제를 방지하기 위해 AWS에서는 정적 IP인 EIP를 지원해준다. EIP region당 5개로 제한되어 있다. 지금은 EC2로만 연결할 예정이니 문제없다. 

빠르게 생성해본다. EC2 파일을 만들고 eip를 생성한다. ec2를 생성한 이후 eip를 할당해 줄 예정이다.   
```hcl
# modules/ec2/main.tf

resource "aws_eip" "eip" {
  vpc = true
  lifecycle {
    create_before_destroy = true
  }
}
```  

<br />  

### EC2 생성  
마지막으로 서버 역할을 할 EC2를 만들어준다. EC2는 하나의 컴퓨터로써 역할을 한다. 자체로 컴퓨팅 자원이기 때문에 많은 설정들이 필요하다. 설치한 소프트웨어 템플릿인 AMI 설정, 컴퓨팅 파워를 설정하는 Instance Type, 메모리 선택, 서버접속을 위한 SSH key pair 뿐만아니라 첫 부팅시 실행할 스크립트도 작성할 수 있다. 여기서는 AWS 프리티어 스펙을 사용하여 EC2를 생성하는 로직까지만 포함하였다.  

입력으로 이전에 생성한 보안그룹과 Subnet의 정보를 받는다. EC2가 위치할 Subnet과 적용할 보안그룹을 할당하기 위해서이다. EC2에 OS를 비롯하여 기본 소프트웨어가 설치된 Amazon Linux2 ami를 설치한다. instance type은 프리터가 적용되는 t2.micro를 이용하고 입력받은 보안그룹을 적용, 위치할 Subnet을 설정한다.  
```hcl  
# modules/ec2/main.tf

variable "name" {
  type = string
}

variable "tags" {

}

variable "public_subnet_ids" {
  type = list(string)
}

variable "security_group_id" {
  type = string
}

# ec2 생성
resource "aws_instance" "free_tier_ec2" {
  # Amazon Linux2 ami
  ami           = "ami-0e4a9ad2eb120e054"
  instance_type = "t2.micro"

  subnet_id = var.public_subnet_ids[0]
  vpc_security_group_ids = [
    var.security_group_id
  ]

  tags = merge(
    {
      Name = format(
        "%s-public-ec2",
        var.name
      )
    },
    var.tags,
  )
}
```

작성했던 EIP를 EC2에 적용시켜준다.  
```hcl  
# modules/ec2/main.tf  

resource "aws_eip_association" "eip_association" {
  allocation_id = aws_eip.eip.id
  instance_id   = aws_instance.free_tier_ec2.id
}
```  

마무리로 출력할 값들을 설정해준다.  
```hcl  
output "public_ip" {
  value = aws_eip.eip.public_ip
}

output "domain" {
  value = aws_instance.free_tier_ec2.public_dns
}

output "ec2_id" {
  value = aws_instance.free_tier_ec2.id
}
```  

<br />  

### 루트 모듈 구현하기
필요한 리소스를 생성하는 로직을 모두 작성하였다. 이제 이 로직을 불러오고 실제로 생성할 수 있도록하는 루트 모듈을 만들기만 하면 된다.

<br />  

### 완성! 서버로 접속해보기  
ec2에 연결된 기본 도메인으로 접속해보기

<br />  

### 추가
실제로 인프라는 이렇게 간단하게 끝나지 않는다. DB도 존재하고 public subnet처럼 외부로 공개되지 않고 private subnet으로 내부에서만 접근 가능하도록 만들기도 한다. 뿐만 아니라 트래픽을 감당하기 위해 Auto Scale, Load Balancer와 같은 AWS 자원들을 사용하며, 구현하고자 하는 기능의 목적별로 무수히 많은 AWS 자원들을 사용할 수 있다. 앞으로도 직접 구현해 본 인프라를 블로깅해보도록 하겠다!  

[ec2 구현시 필요한 것들](https://kimjingo.tistory.com/197)