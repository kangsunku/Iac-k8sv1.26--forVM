## Jenkins + ArgoCD 로 k8s CI/CD 파이프라인 구축하기

지난번에 '[Jib로 자바 컨테이너 빌드 및 배포하기](https://velog.io/@yellowsunn/Jib%EB%A1%9C-%EC%9E%90%EB%B0%94-%EC%BB%A8%ED%85%8C%EC%9D%B4%EB%84%88-%EB%B9%8C%EB%93%9C-%EB%B0%8F-%EB%B0%B0%ED%8F%AC%ED%95%98%EA%B8%B0)' 로 도커 이미지를 빌드하고 저장소에 배포하는 방법을 알아봤다.  
이번에는 이미지의 태그가 새로운 버전으로 업데이트 될때마다 실행중인 k8s 파드 역시 새로운 버전에 맞게 자동으로 배포하고자 한다.

### ArgoCD와 Kustomize를 활용한 CD(Continuos Deploy) 구성

#### ArgoCD

![](https://velog.velcdn.com/images/yellowsunn/post/cb05f082-f90c-41f7-8905-5fa31ecee38d/image.png)  
ArgoCD는 쿠버네티스를 위한 CD 툴이며 GitOps 방식으로 관리되는 manifest(yaml) 파일의 변경사항을 감시하여, 현재 배포된 상태를 Git에 정의해둔 Manfiest 상태와 동일하게 유지 시켜주는 역할을 수행한다.

이 방식을 통해서 Manifest에 정의된 이미지 버전을 수정할때마다 ArgoCD에서 자동으로 감지하여 새로운 버전으로 배포하도록 하고자 한다.

이미지 버전을 수정하기 위한 방법으로 **Kustomize** 를 활용하였다.

#### Kustomize

Kustomize는 쿠버네티스 리소스(manifest 파일)를 직접 변경하지 않고 필드를 재정의하여 새로운 쿠버네티스 리소스를 생성할 수 있는 도구이다.

```null
apiVersion: kustomize.config.k8s.io/v1beta1 kind: Kustomization resources: - deployment.yaml - service.yaml images: - name: my-api-server newTag: "old-version"
```

kustomization.yaml 파일이 다음과 같이 정의되어 있을 때 my-api-server 이미지의 태그를 변경하고자 하면 어떻게 해야할까?  
직접 파일에 접근해서 newTag 로 재정의된 버전을 수정하는 방법도 있긴 하지만 이방법으로는 자동으로 관리하기에 까다롭다.

Kustomize 명령을 활용하면 `kustomize edit set image my-api-server:new-version` 과 같은 명령어를 치면 아래와 같이 파일이 변경된 것을 확인할 수 있다.

```null
... images: - name: my-api-server newTag: "new-version"
```

이제 Kustomize를 활용해 이미지 버전을 수정하고 git에 push하면 자동으로 ArgoCD 에서 변경사항을 감지해 새로운 이미지 버전에 맞게 업데이트할 수 있다.

이미지 버전이 업데이트 되면, 자동으로 manifest 파일 역시 이미지 버전을 동일하게 업데이트하여 git 에 push 해두는 Jenkins 파이프라인을 구성해보자

### Jenkins 파이프라인

```jenkinsfile
// change-image-tag.jenkinsfile pipeline { agent any stages { stage('Pull git repository') { steps { git branch: 'main', url: 'https://github.com/yellowsunn/argocd-manifest' } } stage('Install kustomize') { steps { sh """ if [ ! -f "${env.WORKSPACE}/kustomize" ]; then curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash fi ${env.WORKSPACE}/kustomize version """ } } stage('Change imag tag') { steps { sh """ cd $baseDirectory ${env.WORKSPACE}/kustomize edit set image $imageName:$tag """ } } stage('Push to manifest repo') { steps { withCredentials([usernamePassword(credentialsId: '073c37ff-f218-4988-8490-8fbe46760674', passwordVariable: 'gitPassword', usernameVariable: 'gitUsername'), string(credentialsId: 'github_email', variable: 'gitEmail')]) { sh """ git config user.email $gitEmail git config user.name $gitUsername git add -A git commit -m '[jenkins] update image tag = $imageName:$tag' git push https://${gitUsername}:${gitPassword}@github.com/yellowsunn/argocd-manifest.git """ } } } } }
```

#### 파이프라인 실행준비

[이미지 빌드 및 배포 파이프라인](https://github.com/yellowsunn/local-iac/blob/main/jenkins/application/my-api-server-deploy.jenkinsfile)이 완료되면 마지막에 해당 파이프라인을 실행 요청

-   imageName, tag, baseDirectory(Kustomization경로)를 파라미터로 전달

#### 실행

1.  Manifest 파일을 관리하는 git repository pull
2.  kustomize 명령을 실행하기 위해 sh 파일을 다운로드 받는다.  
    (워크 스페이스가 캐시되어 이미 kustomize 스크립트 파일이 있는 경우 다운로드 받지 않음)
3.  kustomize 명령으로 이미지 태그 변경
4.  수정된 manifest 파일을 다시 push 한다

### 실행 결과

#### 젠킨스 파이프라인 실행 결과

![](https://velog.velcdn.com/images/yellowsunn/post/7792a577-dd28-42dc-8561-e8e073d5636c/image.png)

#### manifest 레포지토리 - git push 내역

-   [https://github.com/yellowsunn/argocd-manifest/tree/main/manifest/my-api-server](https://github.com/yellowsunn/argocd-manifest/tree/main/manifest/my-api-server)  
    ![](https://velog.velcdn.com/images/yellowsunn/post/c8756c30-2c69-41d6-8d2c-f48f6943ba81/image.png)

#### ArgoCD

git에 업데이트된 버전(20230225050400)에 맞게 파드가 다시 배포됨  
![](https://velog.velcdn.com/images/yellowsunn/post/616ce6cd-548b-4efd-83bf-2883fa3933c3/image.png)  
![](https://velog.velcdn.com/images/yellowsunn/post/3067b7ad-fa22-4fde-a874-04e800e21624/image.png)

### 전체 CI/CD 플로우

![](https://velog.velcdn.com/images/yellowsunn/post/bd2eeb38-c747-4edf-ae47-997dde1120f7/image.png)

참고자료  
[https://nayoungs.tistory.com/entry/ArgoCD%EB%9E%80-ArgoCD-%EA%B0%9C%EC%9A%94-%EB%B0%8F-%EC%84%A4%EC%B9%98](https://nayoungs.tistory.com/entry/ArgoCD%EB%9E%80-ArgoCD-%EA%B0%9C%EC%9A%94-%EB%B0%8F-%EC%84%A4%EC%B9%98)  
[https://malwareanalysis.tistory.com/402](https://malwareanalysis.tistory.com/402)  
[https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/](https://kubectl.docs.kubernetes.io/guides/introduction/kustomize/)  
[https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/)
