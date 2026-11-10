---
title: "Iac , Pulumi"
date: 2026-09-23
tags: []
velog_url: https://velog.io/@da2un/Iac-Pulumi
---

# IaC & Pulumi

Ready GSM 상용환경에 적용된 IaC 구조를 이해하기 위해 IaC와 Pulumi에 대해 알아보겠습니다.

# IaC

IaC(Infrastructure as Code)는 **서버, 네트워크, 데이터베이스 같은 인프라를 코드로 정의하고 관리하는 방식**입니다.

기존에는 AWS 콘솔에 들어가 직접 서버를 만들고 설정했다면, IaC에서는 이런 설정을 코드로 작성하고 코드를 통해 인프라를 생성하거나 변경할 수 있습니다.

## 왜 필요한가?

수동으로 인프라를 관리하면 같은 환경을 다시 만들 때 설정을 처음부터 다시 해야 하고, 사람이 직접 설정하기 때문에 실수가 발생할 수 있습니다.

IaC를 사용하면 인프라 설정이 코드로 남기 때문에 **같은 환경을 다시 구성하기 쉽고, 설정을 확인하거나 변경하기도 편리합니다.**

# IaC의 핵심 개념

## 명령형

명령형은 **어떻게 작업할 것인지**를 순서대로 작성하는 방식입니다.

예를 들어 서버를 만들고 보안 그룹을 설정하는 과정을 직접 순서대로 작성합니다.

```bash
서버 생성
보안 그룹 생성
포트 허용
서버에 보안 그룹 연결
```

즉, **작업 과정**을 정의합니다.

## 선언형

선언형은 **최종적으로 어떤 상태가 되어야 하는지**를 작성하는 방식입니다.

예를 들어

```text
EC2 1대가 있어야 함
인스턴스 타입은 t3.micro여야 함
```

처럼 원하는 결과를 정의합니다.

IaC 도구가 현재 상태와 비교하여 필요한 작업을 수행합니다.

## State

State는 IaC 도구가 **현재 관리하고 있는 인프라의 상태와 정보를 저장해두는 것**입니다.

선언형에서는 원하는 상태만 코드로 작성하기 때문에 현재 인프라가 어떤 상태인지 확인할 정보가 필요합니다.

예를 들어 코드에서는 EC2 3대가 필요하지만 State에는 EC2 2대가 있다면, 부족한 EC2 1대를 생성할 수 있습니다.

## 멱등성

멱등성이란 **같은 작업을 여러 번 실행해도 결과가 동일하게 유지되는 성질**입니다.

예를 들어 원하는 상태가 EC2 1대라면 같은 IaC 코드를 여러 번 실행해도 EC2가 계속 추가되는 것이 아니라 이미 원하는 상태라면 추가 작업을 하지 않습니다.

```text
첫 번째 실행 → EC2 1대 생성
두 번째 실행 → 변경사항 없음
세 번째 실행 → 변경사항 없음
```

# Pulumi

Pulumi는 **프로그래밍 언어를 사용하여 클라우드 인프라를 코드로 정의하고 관리하는 IaC 도구**입니다.

TypeScript, Python, Go, C#, Java 등의 언어를 사용할 수 있습니다.

## Pulumi 동작 방식

Pulumi 코드를 실행하면 코드에 작성된 리소스를 확인하고 기존 State와 비교하여 변경사항을 계산합니다.

그 후 AWS 같은 클라우드 제공업체의 API를 통해 실제 인프라를 생성하거나 수정하고, 변경된 내용을 State에 저장합니다.

## State의 Input과 Output

### Input

`Input`은 리소스를 생성하거나 설정할 때 **입력으로 사용하는 값**입니다.

예를 들어 EC2의 인스턴스 타입이나 AMI 등이 있습니다.

```text
instanceType = "t3.micro"
ami = "ami-xxxx"
```

### Output

`Output`은 리소스가 생성된 후 결정되는 **결과값**입니다.

예를 들어 EC2가 생성된 후 AWS에서 할당된 인스턴스 ID나 Public IP 등이 있습니다.

이 Output은 다른 리소스의 Input으로 사용할 수도 있습니다.

# Pulumi Project / Stack 관리

## Project

Project는 Pulumi 인프라 코드와 설정을 관리하는 단위입니다.

새로운 Project를 만들 때 `pulumi new`를 사용합니다.

```bash
pulumi new <템플릿>
```

## Stack

Stack은 하나의 Project에서 사용하는 **독립적인 환경**입니다.

예를 들어 `dev`, `stage`, `prod`처럼 환경을 나누어 관리할 수 있습니다.

```bash
pulumi stack init dev
pulumi stack ls
pulumi stack select prod
pulumi stack rm dev
```

# Pulumi 핵심 명령어

## `pulumi preview`

```bash
pulumi preview
```

실제 인프라를 변경하지 않고 **어떤 변경사항이 발생하는지 미리 확인**합니다.

## `pulumi up`

```bash
pulumi up
```

Pulumi 코드에 정의된 내용을 실제 인프라에 **적용**합니다.

## `pulumi destroy`

```bash
pulumi destroy
```

현재 Stack에서 관리하고 있는 **인프라 리소스를 삭제**합니다.

## `pulumi state`

```bash
pulumi state <명령>
```

Pulumi의 **State를 확인하거나 관리**할 때 사용합니다.

예를 들어 특정 리소스를 State에서 삭제할 수 있습니다.

```bash
pulumi state delete <resource>
```

## `pulumi refresh`

```bash
pulumi refresh
```

실제 클라우드의 상태를 확인하여 **Pulumi State를 실제 인프라 상태에 맞게 갱신**합니다.

