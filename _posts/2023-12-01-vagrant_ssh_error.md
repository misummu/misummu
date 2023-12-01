---
layout: single
title: "Vagrant SSH 접속이 안되는 문제"
categories: Devops
tag: [Ansible, Vagrant]
---

### 상황

vagrant suspend 이후 다음날 vagrant resume 을 했을때 SSH 접속이 안되는 오류이다.
suspend와 resume의 구조적 문제점이 아니라 vagrantfile과 yml 파일 그리고 가정용 KT 라우터에서 받아오는 유동적인 공인 IP 설정으로 생긴 관리적 문제로 보인다. 게다가 문제 해결중 Vagrant의 공개키까지 삭제하면서 상황이 더 복잡해진다. 

### 문제

```
The provider for this Vagrant-managed machine is reporting that it
is not yet ready for SSH. Depending on your provider this can carry
different meanings. Make sure your machine is created and running and
try again. Additionally, check the output of `vagrant status` to verify
that the machine is in the state that you expect. If you continue to
get this error message, please view the documentation for the provider
you're using.
```

### 원인

앞서 상황 설명에서 말한 것처럼 여러가지 설정 파일과 문제를 해결하기 위해 설치했던 플러그인이 얽혀있어 정확한 원인 파악이 어렵다.  따라서 처음 발견했던 오류를 문제1, 그리고 문제1을 해결하다가 새로 생긴 오류를 문제2로 나누어 해결 방안을 찾는다.

#### 문제1 : 복합적인 원인

#### 문제2: ~/.ssh/authorized_keys(vagrant 공개키)를 삭제


### 해결

#### 문제1:

가상 머신이 설치되어 있는 폴더 (C:\HashiCorp\.vagrant\machines/) 안에 있는 private_key를
insecurevag_private_key로 이름을 바꾼다. 가상머신이 동작하고 있는 상황이라면 vagrant halt 명령어를 실행하여 가상머신을 종료함과 동시에 새로운 개인키를 받아낸다.
![vagrant ssh 인증 해결](\images\2023-12-01-vagrant_ssh_error\vagrant ssh 인증 해결.png)

#### 문제2:

VMware로 가상머신에 직접 들어가서 문제점을 수정한다.

삭제한 공개키를 curl 명령어를 이용해 다시 받아온다.

```
curl https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant.pub
```


### 참고

Vagrant 공개키
https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant.pub