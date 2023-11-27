---
layout: default
title: Terraform 기본개념
last_modified_date: 2023-01-27 12:00:00.
parent: Archive
---

Terraform 기본개념
{: .head}

{: .no_toc }

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

IaC 도구 중 하나인 `Terraform`은 선언형 언어(HCL)을 사용하고 불변성, 멀티 클라우드 관리가 가능하다는 장점을 가지고 있다. 이러한 장점 덕분에 많은 개발자/팀들이 인프라 프로비저닝 도구로써 사용하고 있다. 이 글에서는 Terraform 설치부터 테라폼을 사용하기 위한 기본적인 개념을 정리해보았다.  
(IaC에 대한 설명은 [블로그 글](https://rokwonk.com/infrastructure/IaC%EB%9E%80/)을 참고)  

**Notice :** 이 글의 모든 예시는 Terraform으로 관리가능한 클라우드 중 AWS를 사용하고 있습니다.
{: .notice}  

![Terraform](https://developer.hashicorp.com/_next/image?url=https%3A%2F%2Fcontent.hashicorp.com%2Fapi%2Fassets%3Fproduct%3Dterraform%26version%3Drefs%252Fheads%252Fv1.3%26asset%3Dwebsite%252Fimg%252Fdocs%252Fintro-terraform-apis.png%26width%3D2048%26height%3D644&w=2048&q=75){: .align-center style="width: 100%;"}  
출처: [Terraform](https://developer.hashicorp.com/terraform/intro)
{: .image-caption style="font-size: 14px;" }  

<br />  

### Terraform 설치하기  
macOS의 경우 brew를 통하여 설치가 가능하고 `@버전명`으로 특정 버전을 설치할 수 있다.(팀과 함께 협업한다면 버전을 맞추시길!) 다른 OS의 경우 [테라폼 공식문서 다운로드 페이지](https://developer.hashicorp.com/terraform/downloads)에서 확인할 수 있다! 

```bash
$ brew install terraform@1.2
$ terraform --version
Terraform v1.2.9
```
<br />  

### Terraform 사용 전 준비사항
사용하고 있는 컴퓨터에 [**AWS CLI**](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/getting-started-install.html)가 설치되어 있어야하며 Terraform에서 사용할 [**AWS IAM User가 생성**](https://gurumee92.tistory.com/230?category=933068)되어 있어야한다. AWS IAM User 생성시에는 꼭 '프로그래밍 방식 액세스'에 체크해야 Access Key와 Secret Access Key를 발급 받을 수 있다!(걸어둔 링크에서 설치 및 생성법에 대해 자세히 설명되어 있다)  
마지막으로 생성한 **AWS IAM User를 로컬 컴퓨터에 셋팅**해야한다. 아래 명령어를 입력해서 설정가능하다!  

```bash
$ aws configure 
AWS Access Key ID [None]: ${IAM User 생성시 지급받은 Access Key}
AWS Secret Access Key [None]: ${IAM User 생성시 지급받은 Secret Access Key}
Default region name [None]: 
Default output format [None]: json
```  

{% capture notice-2 %}
**💡 다수의 AWS 계정을 사용하고 있을 경우**  
'set --profile ${이름}' 를 이용하여 지정된 이름을 저장할 수 있다.  

```bash
$ aws configure set --profile ${지정하고 싶은 이름}
AWS Access Key ID [None]: ${IAM User 생성시 지급받은 Access Key}
AWS Secret Access Key [None]: ${IAM User 생성시 지급받은 Secret Access Key}
Default region name [None]: 
Default output format [None]: json
```  
{% endcapture %}  

<div class="notice--info">{{ notice-2 | markdownify }}</div>

AWS Configure 셋팅을 확인하고 싶다면 아래 명령어를 통해 확인가능하다. 
```bash
$ cat ~/.aws/config # 계정별 region 등 정보 출력
$ cat ~/.aws/credentials # 계정별 인증정보 출력
```  

<br />  

### Terraform 시작, 핵심 - provider, resource  
Terraform 파일의 확장자는 `.tf`이다. `main.tf` 파일을 하나 만들고 해당 파일에서 HCL(HashiCorp Language) 언어를 사용해서 인프라 구성을 위한 코드를 작성할 수 있다.  

**💡 VSCode Tip!**  
VSCode를 사용한다면 Terraform 코드의 Syntax Highlighting 및 자동완성을 지원하는 익스텐션 설치를 권장한다! HarshiCorp에서 공식적으로 지원하는 익스텐션이다. [링크](https://marketplace.visualstudio.com/items?itemName=HashiCorp.terraform)
{: .notice--info}

Terraform은 멀티 클라우드를 지원한다. 따라서 **어떤 클라우드를 사용할 것인지를 명시해 주어야하는데 이때 사용하는 키워드가 바로 `provider`**이다. 같은 클라우드 내에서 다른 region을 사용할때에도 `provider`를 따로 명시해 줄 수 있다.  

```hcl  
provider "aws" {
  region = "us-east-1"
}
```  
```hcl  
provider "aws" {
  region                   = "ap-northeast-2"
  shared_config_files      = ["~/.aws/config"]
  shared_credentials_files = ["~/.aws/credentials"]
  profile                  = "${aws configure에 저장한 계정 이름}"
}
```
provider 키워드 옆에 사용할 클라우드의 이름을 작성한다. 내부는 key, value 형식으로 작성할 수 있고 key는 지정되어 있다. 첫번째 provider는 region만 지정해 주었다. 이때는 로컬에 저장된 default aws 계정이 사용된다. 두번째 provider는 default가 아닌 다른 계정을 사용하고 싶을때 계정을 지정해 주는 방법이다.  

**`resource` 키워드는 terraform으로 관리할 인프라 요소를 표현**한다. vpc, subnet, ec2 등 모든 자원들이 각 하나의 resource로써 표현된다.  
```hcl
resource "aws_instance" "rokwon_ec2" {
  # Amazon Linux2 ami
  ami           = "ami-0e4a9ad2eb120e054"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.key_pair.key_name

  subnet_id = var.public_subnet_ids[0]
  vpc_security_group_ids = [
    var.security_group_id
  ]
}
```
resource 키워드 뒤에는 2개의 문자열이 들어간다. 첫번째 문자열은 사용할 자원이름이다. 이 명칭은 Terraform에서 지정해 두었다. 두번째 문자열은 개발자가 내부에서 사용할 이름을 지정한다. 이후 해당 자원을 사용할때 `aws_instance.rokwon_ec2`와 같이 `${자원이름}.${지정한이름}`으로 참조할 수 있다. 내부에 key들은 각 자원들의 설정들이다. 위는 EC2를 만드는 예시로 ami, instance_type, 접속을 위한 key, 위치할 subnet, 적용할 security group을 작성하였다.

<br />  

### Terraform 설정 - terraform
사용하는 Terraform의 환경설정을 지정해줄 수 있다. 이대 `terraform` 키워드를 사용하고 terraform 버전이나 사용하는 provider의 버전, 팀과 협업을 한다면 terraform backend도 설정할 수도 있다.  

아래는 terraform 버전과 provider 버전을 명시하는 예시이다.  
```hcl
terraform {
  required_version = "1.2.9"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "4.33.0"
    }
  }
}
```

보다 자세한 설명은 [테라폼 공식문서](https://developer.hashicorp.com/terraform/language/settings)에서 확인 가능하다.  

<br />  

### Terraform 명령어
Terraform 프로젝트를 초기화하고, 바뀔 인프라 정보확인, 적용 등에 몇 가지 명령어가 쓰인다.  

```bash
$ terraform init
```  
**Terraform 프로젝트를 초기화하는데 사용**된다. 첫 프로젝트 시작이나 새로운 module을 추가할때 init 명령어를 사용해야한다. `.terraform` 디렉터리가 생성되는데 이 디렉터리에는 의존되는 파일들이 전부 다운로드된다.(like node_modules)  

```bash
$ terraform plan
```  
**기존 인프라와 비교하여 어떤 인프라가 만들어지게 되는지 예측 결과를 확인할 수 있다.** 이 명령어는 확인만 할 뿐 기존 인프라의 형상에 변화를 주진 않는다.  

```bash
$ terraform apply
```  
**코드를 기반으로 실제로 인프라를 적용시킨다.** 변경된 내용은 `terraform.tfstate` 파일에 저장되어진다.  

```bash
$ terraform destroy
```  
**현재 만들어져 있는 인프라를 전부 삭제한다.**  

**🤔 Terraform의 동작원리**  
필자는 Terraform 코드가 AWS CloudFormation Stack으로 변환되어 사용되는 줄 알았다. 하지만 이것은 진실이 아니다! Terraform 코드는 AWS API로 변환되어 실제 인프라로 구성하게 된다.  
{: .notice--info}  

<br />  

### Terraform 기본 파일/디렉터리  
명령어들을 이용하면 여러 파일/디렉터리 들이 생성되고 변형된다. 각 파일들이 어떤 역할을 하는지 알아보자.  

**.terraform 디렉터리**  
- init 명령어 사용시 생성된다.  
- 모든 의존성 파일들이 다운로드된다. apply 시 해당 디렉터리에서 의존파일을 읽는다.
- node.js의 node_modules와 같은 역할

**.terraform.lock.hcl**
- 의존성 잠금 파일이다.
- 어떤 파일에 의존되어 있는지, 버전등이 기록되어 있다.
- 이 파일이 있다면 init 명령시 이 파일을 토대로 의존파일을 다운받는다
- node.js의 package-lock.json과 같은 역할

**terraform.tfstate**
- init 명령어 사용시 생성된다.  
- apply 명령어 사용시 구축된 인프라 정보를 이 파일에 저장한다.
- 이후 plan 명령에서 이 파일의 정보와 비교하여 바뀔 정보를 표시한다.
- 즉, 현재 테라폼으로 만든 인프라 정보가 들어가 있다.
- 단, 실제 인프라와 다를 수 있다!(테라폼으로 사용하다가 console로 인프라를 변경하는 경우)  

<br />  

### 변수, 데이터 사용하기 - locals, variables, output, data  
Terraform에서 일반 프로그래밍 언어처럼 지역변수, 입력변수 등을 사용할 수 있다. 각 변수마다 활용방안이 다르다. 

지역변수 선언 키워드인 `locals`이다. 현재 파일에서만 사용이 가능하다. **`locals` 블록내 key(키), value(값)로 선언**하며 값들은 연산(merge, concat, max)하여 작성할 수 있다.  
```hcl  
locals {
  common_tags = {
    Service = local.service_name
    Owner   = local.owner
  }
}

resource "aws_instance" "example" {
  # ...
  tags = local.common_tags
}
```  
다른 블럭에서 `local.${key 이름}`을 이용하여 사용할 수 있다.  

**`variable`은 입력변수이다. 뿐만아니라 같은 레이어의 여러 파일에서 공유하여 사용가능**하다. default 값을 지정할 수 있으며 명령어로 입력, module이라면 외부 module에서 값을 주입하는데 사용할 수도 있다.  
```hcl  
variable NAME {
  type = TYPE
  defalut = VALUE
}
```  
```hcl  
variable "vpc_name" {
  type    = string
  default = "test-vpc"
}

resource "aws_vpc" "the_pool_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = var.vpc_name
  }
}
```  
variable 키워드 옆에 문자열은 이름을 지정하고 타입, default 속성을 지정할 수 있다. 해당 변수는 `var.{변수명}`으로 사용가능하다.  

**output은 자원생성 후 필요한 정보를 출력**한다. 뿐만아니라 출력된 값을 **다른 자원의 variable로 입력하여 사용**할 수 있다. 
```hcl
resource "aws_ecr_repository" "ecr" {
  name                 = "test_ecr"
  image_tag_mutability = "MUTABLE"
  image_scanning_configuration {
    scan_on_push = true
  }
}

output "ecr_registry_url" {
  value = aws_ecr_repository.ecr.repository_url
}
```
output 변수명을 작성하고 어떤 값을 받을지를 지정할 수 있다.  

**`data`는 Terraform 외부에 정의된 정보를 가져와서 사용할 수 있게 해준다.** Terraform에서는 provider들의 `data`와 관련되 서비스들을 지원한다. 예를 들어 AWS의 Policy와 같은 것들이다.  
```hcl  
data "aws_iam_policy_document" "s3_policy" {
  statement {
    principals {
      type        = "*"
      identifiers = ["*"]
    }

    actions = [
      "s3:*",
    ]

    effect = "Allow"

    resources = [
      "${aws_s3_bucket.s3.arn}",
    ]
  }
}
```  
`data`의 구조또한 Terraform에서 지원해주기에 편리하게 작성할 수 있다.  

<br />  

### 리소스 재사용 - module  
Terraform은 모듈화로 인프라를 재사용할 수 있다. 이때 `module` 키워드를 사용하여 리소스를 불러올 수 있다. module로 만들어진 리소스에 대해 `output`을 통해 정보를 받을 수 있고 `variable`을 통해 입력을 받을 수 있다. 즉, **module은 프로그래밍에서의 함수**라고 볼 수 있다.  
`module`의 소스로써 하위 디렉터리 파일, Git, Terraform Registry 등에서 가져올 수 있다. 
```hcl
module NAME {
  source = "PATH"
  # ... 필요한 인풋값 작성
}
```  

module을 사용합으로써 **캡슐화(필요한 값만 노출), 재사용성, 일관성(리소스별 사용 방법을 통일)**의 이점을 얻어갈 수 있다.  

<br />  

### 기존 인프라 가져오기 - import  
Terraform로 작성 전 혹은 작성 중 console이나 api를 통해 인프라 구성요소를 생성/변경할 수 있다. 이때 `.tfstate` 파일과 동기화가 진행되지 않아 문제가 발생한다. 이러한 불균형의 문제를 해결하기 위해 Terraform에서는 **현재 동작중인 인프라 자원을 Terraform에 반영할 수 있는 방법**을 지원한다.  

아래 명령어를 통해 만들어져있는 자원을 Terraform에 정의한 자원으로 가져올 수 있다. 하지만 실제로 코드가 만들어지는 것은 아니므로 plan 명령어를 통해 바뀌는 인프라를 확인 후 직접 코드로 작성해야 한다.  
```bash  
$ terraform import ‘{resoure 이름}.{정의한이름}' {console에서 확인한 이름}
```

<br />  

### etc - pluraltih  
Terraform으로 작성한 인프라를 시각화해서 보고 싶다면 이 프로그램을 사용해서 확인가능하다! 아래 다운로드 링크와 잘 소개되어 있는 링크를 걸어 놓겠다.  
[pluraltih 공식문서](https://docs.pluralith.com/docs/category/get-started/)  
[블로그 - 테라폼 시각화와 인프라 문서화를 위한 도구](https://www.saltedcoke.com/?p=95)  





참고  
- [terraform 기본 명령어](https://medium.com/humanscape-tech/iac-infrastructure-as-code-with-terraform-1fb0c6486fbc)
- [terraform 미세 팁 - backend, module, loop](https://swalloow.github.io/tf-tips/)
- [terraform 미세 팁 - backend, module, loop](https://swalloow.github.io/tf-tips/)