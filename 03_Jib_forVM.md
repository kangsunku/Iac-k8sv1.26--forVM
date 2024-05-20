어플리케이션을 컨테이너로 배포하기 위해서 일반적으로 Dockerfile을 작성하고 도커를 설치해 도커 이미지를 빌드하고 배포하는 과정을 진행해야한다.  
하지만 도커를 설치하지 않고 Dockerfile을 작성할 필요없이 Java 기반의 어플리케이션을 컨테이너로 쉽게 빌드하고 배포할 수 있는 방법이 있다.

바로 Google Cloud에서 제공하는 [Jib](https://cloud.google.com/java/getting-started/jib?hl=ko)을 사용하면 된다.

### Jib 빌드 흐름

Docker 빌드 흐름:  
![](https://velog.velcdn.com/images/yellowsunn/post/c64a5b0f-635e-4995-bd02-f23b5da15026/image.png)  
Jib 빌드 흐름:  
![](https://velog.velcdn.com/images/yellowsunn/post/acdd185d-5c7c-4c35-a513-8decd64ca274/image.png)

Maven이나 Gradle에서 Jib 플러그인을 사용하면 Docker 이미지를 빌드하는 각각의 단계를 구성할 필요없이 모든 단계를 Jib가 한번에 처리할 수 있다.

### Jib 플러그인 (Gradle)

해당 코드는 Kotlin 기반의 프로젝트에 맞게 작성되었으며 사용 방법은 [https://github.com/GoogleContainerTools/jib](https://github.com/GoogleContainerTools/jib) 를 참고하면 된다.

#### 빌드하기

build.gradle.kts

```kotlin
plugins { //... val jibVersion = "3.3.1" id("com.google.cloud.tools.jib") version jibVersion } configure<JibExtension> { from { image = "eclipse-temurin:17-jre-alpine" } to { image = "myregistry/myimage" tags = ["tag1", "latest"] } container { ports = listOf("8080") } }
```

간단한 필수 구성을 보자면 먼저 프로젝트를 도커로 빌드하기 위한 Base image가 필요하다. `from.image` 옵션을 통해서 Base image를 설정할 수 있다.  
그 다음 이미지를 빌드할 이미지를 `to.image` 옵션을 통해서 지정하면 된다.  
`container.ports` 옵션은 Docker의 Expose와 유사한 기능으로 노출하고자 하는 포트이다.

```null
gradle jib \ -Djib.from.image=eclipse-temurin:17-jre-alpine \ -Djib.to.image=myregistry/myimage \ -Djib.to.tags=tag1, latest \ -Djib.container.ports=8080
```

다음과 같이 `gradle jib` 실행 명령에 Jib 옵션을 추가로 파라미터로 전달할 수도 있다.

#### 이미지 저장소 설정하기

Base image를 가져오고 빌드한 이미지를 배포하는 컨테이너 레지스트리를 `credHelper`라는 옵션을 통해서 설정할 수도 있는데 기본 설정은 [Docker Hub Registry](https://hub.docker.com/)를 사용한다.

```gradle
configure<JibExtension> { from { image = 'aws_account_id.dkr.ecr.region.amazonaws.com/my-base-image' credHelper = 'ecr-login' } to { image = 'gcr.io/my-gcp-project/my-app' credHelper = 'gcr' auth { username = USERNAME password = PASSWORD } } }
```

위와 같이 base 이미지를 AWS ECR(Elastic Container Registry)에서 가져오고 배포는 GCR (Google Container Registry) 하는 등의 서로 다른 저장소를 설정할 수도 있다. 또 로그인이 필요한 경우 `from.auth`, `to.auth` 에 로그인 정보를 포함할 수도 있다.

### 젠킨스 배포 파이프라인

컨테이너 빌드, 배포 파이프라인 구성을 젠킨스 파이프라인으로 작성해서 테스트 해보았다.

```jenkinsfile
import java.text.SimpleDateFormat pipeline { agent any stages { stage('Checkout git repository') { steps { git branch: 'main', url: 'https://github.com/yellowsunn/simple-webservice.git' } } stage('Run test') { steps { sh './gradlew :my-api-server:test' } } stage('Get current time') { steps { script { def dateFormat = new SimpleDateFormat("yyyyMMddhhmmss") def date = new Date() timestamp = dateFormat.format(date) echo timestamp } } } stage('Build and Deploy image') { steps { script { withCredentials([usernamePassword(credentialsId: 'docker_hub_credentials', passwordVariable: 'password', usernameVariable: 'username')]) { sh """ ./gradlew :my-api-server:jib \ -Djib.to.tags=$timestamp \ -Djib.to.auth.username="$username" \ -Djib.to.auth.password="$password" """ } } } } } }
```

1.  git pull 프로젝트
2.  테스트 실행
3.  배포할 이미지 tag를 timstamp로 지정 (yyyyMMddhhmmss)
4.  Docker Hub 계정 정보, tag를 파라미터로 전달하여 jib 명령 실행 -> 도커 이미지 빌드 및 Docker Hub Registry 에 배포

#### 전체 플로우

![](https://velog.velcdn.com/images/yellowsunn/post/a3a57538-1c79-47b2-8e22-29d8bffc28fe/image.jpg)

### 실행 결과

#### 젠킨스 배포 실행 결과

![](https://velog.velcdn.com/images/yellowsunn/post/278a637c-fc9e-4df8-9f4f-249961b63324/image.png)

#### Docker Hub Registry 배포 확인

[https://hub.docker.com/r/yellowsunn/my-api-server/tags](https://hub.docker.com/r/yellowsunn/my-api-server/tags)  
![](https://velog.velcdn.com/images/yellowsunn/post/af62a09a-ae43-4e42-921a-f4835e58a848/image.png)

참고자료  
[https://cloud.google.com/java/getting-started/jib?hl=ko](https://cloud.google.com/java/getting-started/jib?hl=ko)  
[https://github.com/GoogleContainerTools/jib](https://github.com/GoogleContainerTools/jib)