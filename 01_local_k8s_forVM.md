## 로컬에 쿠버네티스 클러스터 환경 구축하기

## 개요

최근 사내에서 빈스톡 기반에서 EKS 환경으로 대규모 서비스 전환 작업이 이루어지면서 쿠버네티스에 대한 관심이 생겼다. 그렇게 인프런에서 한 [쿠버네티스 강의](https://inf.run/3wZr)를 신청해 수강하던 중 신세계를 맛보고 말았다!

필자는 개인 프로젝트도 사내 환경과 비슷한 인프라 환경을 구성하면 좋을 것 같다고 생각했는데 이에 대한 해답을 줄 수 있는 도구를 발견했다.

바로 **Vagrant** 라는 녀석이다.

## Vagrant

![](https://velog.velcdn.com/images/yellowsunn/post/1ddaccb7-47b2-4bf2-a2c8-b3f092715fba/image.png)  
Vagrant는 가상 머신 환경을 구축하고 관리할 수 있는 도구이다. 수동으로 가상머신의 자원을 할당하고 OS 이미지를 설치하고 초기 세팅들을 설정할 필요없이, 인프라를 코드로 작성(IaC)해두면 자동으로 인프라를 프로비저닝 할 수 있다.

> 프로비저닝(provisioning): 사용자의 요구의 맞게 시스템 자원을 할당, 배치, 배포해 두었다가 필요 시 시스템을 즉시 사용할 수 있는 상태로 미리 준비해두는 것을 말한다.

Vagrant는 [공식 사이트](https://developer.hashicorp.com/vagrant/intro)에서도 언급하듯이 '개발 환경'을 손쉽고 빠르게 자동화하고 테스트 해볼 수 있는 도구이다. 실제 운영 환경에서 서버에 가상 머신을 띄운 다음 인프라 환경을 구성하는 시도는 거의 없다고 봐야 할테니 로컬 환경에서 테스트하는 용도에 적합하다고 할 수 있다.

## Vagrant를 통해 가상머신에 쿠버네티스 클러스터 구축하기

Vagrant를 이용하면 로컬에서 가상머신에 kubeadm 클러스터 환경을 손쉽게 구성할 수 있다.  
(이에 대한 자료는 수강하던 [쿠버네티스 강의](https://inf.run/3wZr)에서 저자가 설정한 [코드](https://github.com/sysnet4admin/_Lecture_k8s_learning.kit/blob/main/ch1/1.5/k8s-min-5GiB-wo-add-nodes/Vagrantfile)를 최신 버전에 가깝게 다시 수정함)

### 구성

**k8s 클러스터 (Virtualbox)**

-   master-node: 192.168.50.20
-   worker-node: 192.168.50.21-192.168.50.23 (3대)

**Version**

-   Vagrant box: sysnet4admin/CentOS-k8s
    -   OS: CentOS7
-   kubernetes version: 1.26.1
    -   Calico version: 3.25.0
    -   MetalLB version: 0.12.1
    -   Helm version: 3.11.0
    -   Metrics-Server version: 3.8.3
-   Docker version: 20.10.23
-   Containerd version: 1.6.15

### 사전준비

> vagrant, virtualbox 설치

### 실행하기

```bash
$ git clone https://github.com/yellowsunn/local-iac.git
```

```bash
$ cd local-iac/vagrant/k8s_1_26
```

```bash
$ vagrant up
```

### Vagrantfile

```yml
Vagrant.configure("2") do |config| # 마스터 노드 config.vm.define "m-k8s-1.26", primary: true do |cfg| cfg.vm.box = "sysnet4admin/CentOS-k8s" cfg.vm.provider "virtualbox" do |vb| vb.name = "m-k8s-1.26" vb.cpus = 4 vb.memory = 4096 vb.customize ["modifyvm", :id, "--groups", "/k8s-1.26"] end cfg.vm.host_name = "m-k8s" cfg.vm.network "private_network", ip: "192.168.50.20" cfg.vm.network "forwarded_port", guest: 22, host: 60020, auto_correct: true, id: "ssh" cfg.vm.synced_folder "../data", "/vagrant", disabled: true cfg.vm.provision "shell", path: "k8s_env_build.sh", args: N cfg.vm.provision "shell", path: "k8s_pkg_cfg.sh", args: [ k8s_V, docker_V, ctrd_V ] cfg.vm.provision "shell", path: "master_node.sh" end ### 워커 노드 (1..N).each do |i| config.vm.define "w#{i}-k8s-1.26" do |cfg| cfg.vm.box = "sysnet4admin/CentOS-k8s" cfg.vm.provider "virtualbox" do |vb| vb.name = "w#{i}-k8s-1.26" vb.cpus = 2 vb.memory = 2048 vb.customize ["modifyvm", :id, "--groups", "/k8s-1.26"] end cfg.vm.host_name = "w#{i}-k8s" cfg.vm.network "private_network", ip: "192.168.50.2#{i}" cfg.vm.network "forwarded_port", guest: 22, host: "6020#{i}", auto_correct: true, id: "ssh" cfg.vm.synced_folder "../data", "/vagrant", disabled: true cfg.vm.provision "shell", path: "k8s_env_build.sh", args: N cfg.vm.provision "shell", path: "k8s_pkg_cfg.sh", args: [ k8s_V, docker_V, ctrd_V ] cfg.vm.provision "shell", path: "work_nodes.sh" end end end
```

### Vagrant 실행 결과

![](https://velog.velcdn.com/images/yellowsunn/post/4b87db93-ed8f-4c7c-abb9-d1015e8acebe/image.png)  
`vagrant up` 명령을 실행하고 작업이 완료되는 것을 기다리면 VirtualBox에 실행중인 4대의 VM 인스턴스를 확인할 수 있다.

![](https://velog.velcdn.com/images/yellowsunn/post/2d1af0d0-baf5-4d20-b26f-b2498de1c812/image.png)  
마스터 노드(m-k8s-1.26) 서버에 접속하여 `$ kubectl get nodes` 명령어로 클러스터에 구성되어 있는 모든 노드가 확인 가능하다. ROLES가 **control-plan**인 클러스터의 상태와 설정 값을 저장하고 관리하는 마스터 노드다.

### 전체적인 구성도

![](https://velog.velcdn.com/images/yellowsunn/post/270e88c3-8724-47c0-8742-d527e63928bc/image.jpg)

참고자료  
[https://ko.wikipedia.org/wiki/%ED%94%84%EB%A1%9C%EB%B9%84%EC%A0%80%EB%8B%9D](https://ko.wikipedia.org/wiki/%ED%94%84%EB%A1%9C%EB%B9%84%EC%A0%80%EB%8B%9D)  
[https://github.com/sysnet4admin/\_Lecture\_k8s\_learning.kit](https://github.com/sysnet4admin/_Lecture_k8s_learning.kit)  
[https://developer.hashicorp.com/vagrant/intro](https://developer.hashicorp.com/vagrant/intro)
