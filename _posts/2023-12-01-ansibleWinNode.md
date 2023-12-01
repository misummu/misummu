---
layout: single
title: "Windows 운영체제를 Ansible 로 관리하기"
categories: Devops
tag: [Ansible, Vagrant, Windows]
---





## 소개



앤서블이 리눅스뿐만 아니라 윈도우 운영체제까지 관리 할 수 있는 것은 앤서블이  많은 관심을 받고 앤서블의 커뮤니티가 활발하게 운영되는 이유 중 하나이다. 하지만 앤서블이나 베이그런트의 버전에 따라서 모듈의 이름이 달라지거나 작동이 안되는 경우가 있다. 따라서 추후 실습을 다시 할 때는 아래의 베이그런트와 앤서블의 버전으로 똑같이 다운받거나 앤서블의 공식 문서를 확인해서 모듈의 이름이나 기능이 달라지진 않았는지 확인 해야 한다.



	Ansible version 2.9.27




```bash
[vagrant@ansible-server ~]$ ansible --version
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/vagrant/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Apr  2 2020, 13:16:51) [GCC 4.8.5 20150623 (Red Hat 4.8.5-39)]
```




	Vagrant version 2.3.8 dev






```
c:\HashiCorp>vagrant version
Installed Version: 2.3.8.dev
Latest Version: 2.4.0

To upgrade to the latest version, visit the downloads page and
download and install the latest version of Vagrant from the URL
below:

  https://www.vagrantup.com/downloads.html

If you're curious what changed in the latest release, view the
CHANGELOG below:

  https://github.com/hashicorp/vagrant/blob/v2.4.0/CHANGELOG.md

```





## 윈도우 노드 추가하기



### Vagrantfile 



```sh
Vagrant.configure("2") do |config|

  #==============#

  # Windows node #

  #==============#

  #Ansible-Node

  config.vm.define "ansible-node" do |cfg|
     cfg.vm.box = "gusztavvargadr/windows-server"
     cfg.winrm.max_tries = 300
     cfg.winrm.retry_delay = 2
     cfg.vm.provider "vmware_fusion" do |vb|
       vb.name = "Ansible-Node03"
       vb.customize ['modifyvm', :id, '--clipboard', 'bidirectional']
       vb.gui = false
     end
     cfg.vm.host_name = "ansible-node"
     cfg.vm.network "public_network", ip: "192.168.10.13"
     cfg.vm.network "forwarded_port", guest: 22, host: 60013, auto_correct: true, id: "ssh"
     cfg.vm.synced_folder "../data", "/vagrant", disabled: true
     cfg.vm.provision "shell", inline: "netsh advfirewall set allprofiles state off"
  end  

  #================#

  # Ansible Server #

  #================#

  config.vm.define "ansible-server" do |cfg|
    cfg.vm.box = "centos/7"
    cfg.vm.provider "vmware_fusion" do |vb|
      vb.name = "Ansible-Server"
    end
    cfg.vm.host_name = "ansible-server"
    cfg.vm.network "public_network", ip: "192.168.10.10"
    cfg.vm.network "forwarded_port", guest: 22, host: 60010, auto_correct: true, id: "ssh"
    cfg.vm.synced_folder "../data", "/vagrant", disabled: true
    cfg.vm.provision "shell", inline: "yum install epel-release -y"
    cfg.vm.provision "shell", inline: "yum install ansible -y"
    cfg.vm.provision "file", source: "ansible_env_ready.yml",
      destination: "ansible_env_ready.yml"
    cfg.vm.provision "shell", inline: "ansible-playbook ansible_env_ready.yml"
    cfg.vm.provision "shell", path: "add_ssh_auth.sh", privileged: false
  end
end
```







### ansible_env_ready.yml







```yml
---
- name: Setup for the Ansible's Environment
  hosts: localhost
  gather_facts: no

  tasks:
    - name: Add "/etc/ansible/hosts"
      blockinfile:
        path: /etc/ansible/hosts
        block: |
          192.168.10.176 ansible_connection=winrm ansible_user=vagrant ansible_port=5985
          
    - name: Install epel-release
      yum:
        name: epel-release
        state: present
        
    - name: Install pip
      yum:
        name: python-pip
        state: present

### 생략 ###

```



파이썬 코드에서 WinRM을 사용하기 위해서는 **pywinrm** 이라는 파이썬 모듈을 설치 해야 하는데 이 모듈은 **pip**를 통해서 설치가 가능하다. **pip** 또한 기본 CentOS에 포함되어 있지 않고 **epel-release**에 포함되어 있기 때문에 위의 순서대로 설치한다. 하지만 위 파일에서 **pywinrm**을 설치 하면 오류가 생기기 때문에 ansible-server의 터미널에서 직접 설치한다.





### pywinrm 설치



ansible-server의 터미널로 들어와서 다음과 같이 입력한다.



```sh
sudo yum install -y python2-winrm.noarch
```



이 명령어를 실행해도 파이썬의 버전 문제로 오류가 생기지만 앞으로의 실습에서 필요한 모듈은 설치 되기 때문에 신경 쓰지 않아도 된다.

WinRM을 설치하면 윈도우 노드에 win_ping 모듈을 이용해서 통신을 확인할 수 있다.





### 결과

![pong](D:\github_blog\misummu.github.io\images\2023-12-01-ansibleWinNode\pong-1701420459176-14.png)

## Nginx 서비스 설치 및 실행하기





### nginx_install.yml





**사용한 모듈**

- win_file
  
  nginx 를 저장할 디렉터리를 생성한다.
  
- win_get-url
  1. nginx-1.14.0.zip 파일을 다운로드 받는다.
  2. 윈도우 노드에서 보여지는 웹 메인 페이지를 다운로드 받는다.

- win_unzip
  
  다운로드 받은 파일의 압축을 푼다.
  
- win_chocolatey
  
  nssm (the Non-Sucking Service Manager) 를 설치한다.
  
- win_nssm
  
  nginx.exe 를 서비스로 등록한다.
  
- win_service
  
  등록된 서비스를 다시 시작한다.





```yml
---
- name: Install nginx on Windows
  hosts: Windows
  gather_facts: no

  tasks:
    - name: create directory
      win_file:
        path: C:\nginx
        state: directory

    - name: download nginx
      win_get_url:
        url: http://nginx.org/download/nginx-1.14.0.zip
        dest: C:\nginx\nginx-1.40.0.zip

    - name: unzip nginx
      win_unzip:
        src: C:\nginx\nginx-1.40.0.zip
        dest: C:\nginx
        delete_archive: yes

    - name: install NSSM
      win_chocolatey:
        name: NSSM

    - name: download new index.html
      win_get_url:
        url: https://www.nginx.com
        dest: C:\nginx\nginx-1.14.0\html\index.html

    - name: nginx service on by NSSM
      win_nssm:
        name: nginx
        application: C:\nginx\nginx-1.14.0\nginx.exe
        state: present

    - name: restart nginx service
      win_service:
        name: nginx
        state: restarted
```





### 결과



윈도우 노드의 주소(192.168.10.176)로 접속하게 되면 Nginx의 홈페이지를 볼 수 있다.



![access](D:\github_blog\misummu.github.io\images\2023-12-01-ansibleWinNode\access.png)





## 윈도우 노드의 시간대 변경하기



### timezone.yml

**사용한 모듈**

- win_timezone



```yml
---
- name: Setup windows timezone
  hosts: Windows
  gather_facts: no

  tasks:
    - name: set timezone to 'Korea Standard Time'
      win_timezone: timezone='Korea Standard Time'
```



### 결과



플레이북 실행 전



![current_timezone](D:\github_blog\misummu.github.io\images\2023-12-01-ansibleWinNode\current_timezone.png)



실행 후



![change_timezone](D:\github_blog\misummu.github.io\images\2023-12-01-ansibleWinNode\change_timezone.png)



## 윈도우 NFS 클라이언트 구성하기



### nfs.yml

**사용한 모듈**

- win_feature
  
  NFS 클라이언트를 위한 기능을 활성화한다.
  
- win_mapped_drive
  
  네트워크 드라이브를 매핑한다.
  
- win_reboot
  
  NFS 클라이언트 설정을 적용하도록 시스템을 다시 시작한다.





```yml
---

- name: Setup for nfs Server
  hosts: localhost
  gather_facts: no

  tasks:
    - name: make nfs_shared directory
      file:
        path: /home/vagrant/nfs_shared
        state: directory
        mode: 0777

    - name: configure /etc/exports
      become: yes
      lineinfile:
        path: /etc/exports
        line: /home/vagrant/nfs_shared@192.168.10.0/24(rw,sync)

    - name: nfs service restart
      become: yes
      service:
        name: nfs
        state: restarted
        
- name: Setup for nfs windows clients
  hosts: Windows
  gather_facts: no

  tasks:
    - name: mount feature on
      win_feature:
        name: NFS-Client
        state: present
        
    - name: mount nfs_shared
      win_mapped_drive:
        letter: Z
        path: \\192.168.10.10/home/vagrant/nfs_shared

    - name: windows reboot
      win_reboot:
```



### 결과



윈도우 노드의 cmd에서 아래 명령을 입력하여 마운트가 됐는지 확인한다.



```
net use
```



Z: 드라이브에 마운트가 된 것 과 ansible-server 터미널에서 사전에 넣어둔 nfstestingfile 파일이 보인다.



![nfs_good](D:\github_blog\misummu.github.io\images\2023-12-01-ansibleWinNode\nfs_good.png)