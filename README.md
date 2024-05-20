# local-iac
로컬 환경에 필요한 인프라를 코드로 정의해놓은 레포지토리

## Vagrant
Vagrant는 Vagrantfile을 작성해서 가상머신(기본값 Virtualbox) 위에 자동으로 OS를 설치하고 인프라를 프로비저닝할 수 있다.

1. [로컬에 쿠버네티스 클러스터 환경 구축하기](https://github.com/kangsunku/Iac-k8sv1.26--forVM/blob/main/01_local_k8s_forVM.md)
2. [가상머신에 OpenVpn 서버 설치하기](https://github.com/kangsunku/Iac-k8sv1.26--forVM/blob/main/02_local_openVPN%20Server.md)
3. [Jib로 자바 컨테이너 빌드 및 배포하기](https://github.com/kangsunku/Iac-k8sv1.26--forVM/blob/main/03_Jib_forVM.md)
4. [Jenkins + ArgoCD 로 k8s CI/CD 파이프라인 구축하기](https://github.com/kangsunku/Iac-k8sv1.26--forVM/blob/main/04_Jenkins_ArgoCD_k8s_CICD.md)

   
### 실행 예시
* Vagrantfile 이 있는 경로에서 `vagrant up` 명령어 실행
```bash
$ cd vagrant/k8s_1_26
$ vagrant up
```

* 가상머신 종료 및 이미지 제거
```bash
$ vagrant destroy -f
```

## 정의한 인프라
![image](https://user-images.githubusercontent.com/43487002/225531403-03bc9a4c-5059-484a-835b-4eea68bc6690.png)
