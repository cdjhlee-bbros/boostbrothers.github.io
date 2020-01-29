---
layout: post
title:  "[AWS SAM] 1. CloudFormation"
date:   2019-10-12 00:46:00
author: 김광호
categories: Technology
tags:   [technology, redis]
cover:  "/assets/instacode.png"
---
# [AWS SAM] 1. CloudFormation

비브로스 제품개발팀 김광호입니다.

똑닥에서는 AWS Lambda를 적극 활용하고 있습니다. 현재 똑닥에서 사용 중인 람다 함수만 197개 정도 됩니다. 그렇다고 사용자에게 직접적으로 제공하는 서비스에서 사용하는 것은 아닙니다. 상시적으로 서버를 유지할 필요가 없는 간단한 배치 처리, CSV 파일 생성, 푸시 알림 등에서 주로 사용하고 있습니다. (처리 시간이 오래 걸리는 배치 작업은 AWS Batch, Elastic Map Reduce, Atlas DataLake를 활용하고 있습니다.)

AWS SAM(Serverless Application Model)은 CloudFormation을 확장한 버전으로 서버리스 애플리케션을 보다 편하게 작성할 수 있게 해주는 템플릿입니다.

먼저 SAM을 이해하려면 CloudFormation에 대한 간단한 배경지식이 필요한데요. 이번 포스팅에서는 CloudFormation에 대해 간단하게 다루고 다음 포스팅에서 SAM에 대해 다룰 예정입니다.

## CloudFormation

CloudFormation은 크게 다음의 요소로 구성됩니다.

- Parameters
- Mappings
- Outputs
- Resources

### Parameters

```yml
Parameters:
  Environment:
    Description: Environment Name
    Type: String
    AllowedValues: [development, production]
    ConstraintDescription: must be development or production
```

Parameter는 함수의 파라메터와 같다고 보시면 됩니다. 한 번 CloudFormation 템플릿을 만들면 같은 상황에서 계속 재활용할 수 있습니다. EC2 인스턴스를 생성하는 템플릿은 인스턴스명만 바꾸거나 인스턴스 타입만 바꿔서 계속 재활용할 수 있습니다. 이렇게 Parameters를 이용하면 재사용성이 향상됩니다.

Paramters는 반드시 정의해야하는 속성은 아닙니다. 필요할 때만 사용하시면 됩니다.

예제는 환경을 선택하는 Parameters를 정의하는 템플릿입니다. 파라메터의 이름은 "Environment"이고 AllowedValues로 입력할 수 있는 값을 제한했습니다. 이 속성이 없으면 자유롭게 텍스트를 입력할 수 있습니다.

이렇게 정의하면 AWS CloudFormation에서는 다음처럼 선택 옵션이 뜹니다. 

![image-20191011235205857](/assets/2019-10-12/image-20191011235205857.png)

### Mappings

```yml
Mappings:
  EnvironmentToInstanceType:
    development:
      instanceType: t3.micro
    production:
      instanceType: t3.small
```

Mappings는 키-값 형태로 구성하며 고정값을 정의하는 속성입니다. Mappings 자체만 본다면 활용도가 떨어질 수 있으나 Parameters와 함께 사용하면 사용자 입력에 따른 구성을 편리하게 할 수 있습니다. 

Mappings는 3단으로 구성됩니다. 1단계: EnvironmentToInstanceType, 2단계: development, 3단계: instanceType로 3단계에서 실제 사용할 값을 정의됩니다.

Mappings에 정의한 값은 다음처럼 CloudFormation이 제공하는 함수를 통해 사용할 수 있습니다.

```yml
InstanceType: !FindInMap [EnvironmentToInstanceType, !Ref Environment, instanceType]
```

- !FindInMap: Mappings에서 원하는 값을 찾는 함수입니다.
- !Ref: 속성의 값을 참조하는 함수입니다.

예제에서는 Parameters에서 사용자가 선택한 환경에 따라 t3.micro를 사용할지 t3.small로 사용할지 결정합니다.

Mappings는 반드시 정의해야하는 속성은 아닙니다. 필요할 때 선택적으로 사용하면 됩니다.

### Resources

```yml
Resources:
	EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: EC2 Basic Security Group. Created By CloudFormation.
      GroupName: cf-sg-ec2-basic
      SecurityGroupIngress:
        - CidrIp: 0.0.0.0/0
          FromPort: 80
          ToPort: 80
          IpProtocol: tcp
        - CidrIp: 0.0.0.0/0
          FromPort: 22
          ToPort: 22
          IpProtocol: tcp
          
  EC2Server:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d097db2fb6e0f05e
      InstanceType: !FindInMap [EnvironmentToInstanceType, !Ref Environment, instanceType]
      KeyName: icelancer # EC2 접근에 필요한 키 페어
      SecurityGroupIds:
      - !Ref EC2SecurityGroup
      Tags:
      - Key: Name
        Value: !Ref AWS::StackName
```

Resources는 AWS에서 제공하는 각각의 자원을 정의하는 속성입니다. 어떤 값을 사용할 수 있는지는 AWS CloudFormation 페이지에 가면 상세히 볼 수 있습니다. 그 중 필요한 프로퍼티을 찾아서 정의하면 됩니다.

https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html#cfn-ec2-instance-imageid

저는 주로 구글에 "CloudFormation EC2", "CloudFormation S3" 등을 검색해서 해당 페이지로 바로 접근합니다.

- Type: 사용할 AWS에서 제공하는 자원

- Properties: 자원 정의에 필요한 속성 입력

  - ImageId: 사용할 AMI의 ID

  - InstanceType: 사용할 인스턴스 타입

  - KeyName: EC2 키 페어로 EC2에 SSH로 접근할 필요가 없다면 정의하지 않아도 된다.

  - SecurityGroupIds: 시큐리티 그룹 아이디 목록

  - Tags: EC2에 정의할 태그. Name 태그를 지정하면 인스턴스 명으로 입력된다.

    

### Outputs

CloudFormation 템플릿에서 정의한 값을 다른 템플릿에서도 참조할 필요가 있을 때 Outputs에 정의하면 다른 곳에서도 사용할 수 있습니다.

```yml
Outputs:
  BoostEC2PublicIP: 
    Description: "Boost EC2 Public IP Address"
    Value: !GetAtt EC2Server.PublicIp
```

예제와 같이 입력하면 정의한 EC2의 퍼블릭 아이피를 다른 템플릿에서 사용할 수 있습니다. 참조할 때는 다음처럼 사용하면 됩니다. !GetAtt 함수는 정의된 속성을 반환합니다.

```yml
!ImportValue BoostEC2PublicIp
```

Outputs에 정의하면 매번 불편하게 아이피를 찾아서 입력할 필요없이 간편하게 참조해서 사용할 수 있습니다. AWS Resource에서 어떤 값을 사용할 수 있는지 보려면 CloudFormation 페이지에서 "반환값" 항목을 보면 됩니다. ( [AWS CloudFormation: EC2 링크](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/aws-properties-ec2-instance.html))

![image-20191012001402018](/assets/2019-10-12/image-20191012001402018.png)

Outputs는 반드시 정의해야하는 속성은 아닙니다. 필요할 때 선택적으로 정의하면 됩니다.

지금까지 설명한 전체 예제는 다음과 같습니다.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: EC2 Instance Creation

Parameters:
  Environment:
    Description: Environment Name
    Type: String
    AllowedValues: [development, production]
    ConstraintDescription: must be development or production

Mappings:
  EnvironmentToInstanceType:
    development:
      instanceType: t3.micro
    production:
      instanceType: t3.small

Resources:
  EC2Server:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d097db2fb6e0f05e
      InstanceType: !FindInMap [EnvironmentToInstanceType, !Ref Environment, instanceType]
      KeyName: icelancer # EC2 접근에 필요한 키 페어
      SecurityGroupIds:
      - !Ref EC2SecurityGroup
      Tags:
      - Key: Name
        Value: !Ref AWS::StackName

  EC2SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: EC2 Basic Security Group. Created By CloudFormation.
      GroupName: cf-sg-ec2-basic
      SecurityGroupIngress:
        - CidrIp: 0.0.0.0/0
          FromPort: 80
          ToPort: 80
          IpProtocol: tcp
        - CidrIp: 0.0.0.0/0
          FromPort: 22
          ToPort: 22
          IpProtocol: tcp
Outputs:
  BoostEC2PublicIP: 
    Description: "Boost EC2 Public IP Address"
    Value: !GetAtt EC2Server.PublicIp

```

## CloudFormation Stack 만들기

CloudFormation Stack을 만드는 방법은 크게 2가지가 있습니다. 하나는 AWS Console에서 웹 UI로 만드는 방법이 있고 AWS CLI를 사용하는 방법이 있습니다. 이번에는 AWS Console에서 생성하는 방법에 대해 다루고 CLI는 다음 포스팅에서 다룰 예정입니다.

### AWS Console

1) CloudFormation 페이지에서 접근해서 "Create Stack" 버튼을 누릅니다.


![image-20191012002752398](/assets/2019-10-12/image-20191012002752398.png)



2) 정의한 템플릿 파일을 선택합니다.

![image-20191012003014115](/assets/2019-10-12/image-20191012003014115.png)



3) CloudFormation Stack 이름과 파라메터를 선택합니다.

![image-20191012003104717](/assets/2019-10-12/image-20191012003104717.png)



4) Configure stack options에서 Next를 누릅니다.



5) Review에서는 "Create Stack" 버튼을 누르면 스택 생성이 완료됩니다.

![image-20191012004034445](/assets/2019-10-12/image-20191012004034445.png)
