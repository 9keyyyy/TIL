# 도커를 사용한 서버 배포

**⚙️ 개발 환경**
- 도커는 이미 깔려 있는 상태 (Mac OS)
- AWS EC2도 Amazone Linux 2
- Spring boot
- Java 17
- Gradle

</br></br>

## 📍 도커 파일 생성
**Dockerfile**
```
FROM openjdk:17-alpine

ARG JAR_FILE=/build/libs/sejongmate-0.0.1-SNAPSHOT.jar

COPY ${JAR_FILE} /sejongmate.jar

ENTRYPOINT ["java","-jar","-Dspring.profiles.active=prod", "/sejongmate.jar"]
```
</br>

- open jdk java17 버전의 환경 구성
- build 되는 시점에 JAR_FILE 경로에 jar 파일 생성
- JAR_FILE을 sejongmate.jar에 복사
- jar 파일 실행 명령 (여기서 `-Dspring.profiles.active=prod` 옵션은 application.yml을 개발 환경에서 따로 분리한 것)

</br></br>

## 📍 Gradle 빌드 및 도커 Image build/push

이때, docker hub에 계정이 있고, repository가 있어야함

</br></br>

프로젝트 폴더에서
```
./gradlew clean build -x test
```
통해 build/libs 위치에 jar 파일 생성
</br></br>

```
docker build --build-arg DEPENDENCY=build/dependency -t 9keyyyy/sejongmate --platform linux/amd64 .
```
위의 명령어를 통해 dockerfile을 docker image로 빌드
</br></br>

```
docker push 9keyyyy/sejongmate
```
docker hub에 푸시

</br></br>

## 📍 도커 이미지를 통해 Springboot Application 배포
터미널로 pem 키가 있는 폴더 내부에서 아래와 같은 명령어를 통해 EC2 ssh 접속

```
ssh -i "키페어 파일이름" "퍼블릭 DNS 주소"
```
</br></br>

아래의 명령어를 통해 서버 배포

```
sudo docker run -p 8080:8080 9keyyyy/sejongmate
```

</br></br>

배포 완료
<img width="655" alt="스크린샷 2023-04-18 오후 9 12 48" src="https://user-images.githubusercontent.com/62213813/232773794-90e9abfb-7e41-4539-9ea8-892577776ac3.png">
