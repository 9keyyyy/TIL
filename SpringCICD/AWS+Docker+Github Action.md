## AWS EC2 인스턴스 생성하기
<img width="751" alt="스크린샷 2023-04-18 오후 7 41 39" src="https://user-images.githubusercontent.com/62213813/232753620-993fcaf8-9e8d-436b-b22e-224eca304441.png">
Amazone Linux 이미지를 프리티어로 사용함
</br>
</br>
<img width="613" alt="스크린샷 2023-04-18 오후 7 44 56" src="https://user-images.githubusercontent.com/62213813/232753937-afb2d6cf-9327-4dee-b055-f092453ab752.png">
키페어 생성해서 저장해놓기 (지워버리지 않도록 유의하기)

그 외에는 스토리지만 20GB로 변경하고 인스턴스 생성함



## AWS EC2 인스턴스에 도커 설치하기
키페어가 있는 폴더 내부에서 아래의 명령어를 통해 ssh 접속
```
ssh -i "키페어 파일이름" "퍼블릭 DNS 주소"
```

이후 아래의 과정을 통해 docker, docker-compose 설치
```
//도커 설치
sudo yum install docker -y

//도커 실행
sudo service docker start

//도커 상태 확인
systemctl status docker.service

//도커 관련 권한 추가
sudo chmod 666 /var/run/docker.sock
docker ps

//최신 버전 docker-compose 설치
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

//권한 추가
sudo chmod +x /usr/local/bin/docker-compose

//버전 확인
docker-compose --version

```


## Github-Actions 스크립트 파일 생성
Github repository - Actions - Java with Gradle 선택

<img width="793" alt="스크린샷 2023-04-18 오후 7 53 36" src="https://user-images.githubusercontent.com/62213813/232756072-577caf4e-94fd-4616-b212-6e6f6bd86eae.png">


이후 `gradle.yml`이라는 파일을 생성하게 됨
Spring 배포를 위해 작성한 코드는 아래와 같음

```s
name: Java CI with Gradle

on:
  push:
    branches: [ "main" , "dev" ]
  pull_request:
    branches: [ "main" , "dev" ]

permissions:
  contents: read

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'

        
    - name: make application-prod.yml
      run: |
        cd ./src/main/resources
        touch ./application-prod.yml
        echo "${{ secrets.APPLICATION_PROD }}" > ./application-prod.yml
        
        
    - name: Grant execute permission for gradlew
      run: chmod +x gradlew
      
    - name: Build with Gradle
      run: ./gradlew build -x test
      
    - name: Docker build
      run: |
        docker login -u ${{ secrets.DOCKER_USERNAME }} -p ${{ secrets.DOCKER_PASSWORD }}
        docker build -t app .
        docker tag app ${{ secrets.DOCKER_USERNAME }}/sejongmate:latest
        docker push ${{ secrets.DOCKER_USERNAME }}/sejongmate:latest
    
    - name: Deploy
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }} # EC2 인스턴스 퍼블릭 DNS
        username: ec2-user
        key: ${{ secrets.PRIVATE_KEY }} # pem 키
        # 도커 작업
        script: |
          docker pull ${{ secrets.DOCKER_USERNAME }}/sejongmate:latest
          docker stop $(docker ps -a -q)
          docker run -d --log-driver=syslog -p 8080:8080 ${{ secrets.DOCKER_USERNAME }}/sejongmate:latest
          docker rm $(docker ps --filter 'status=exited' -a -q)
          docker image prune -a -f

```
위 코드를 순서대로 설명해보면, 아래와 같음
1. 배포를 위한 application-prod.yml 파일 생성하기 : 개발에서 사용되는 rds 주소 및 비밀번호를 노출할 수 없기 때문에 해당 파일은 .gitignore 해두고 배포 시 secret 키 이용해서 생성함
2. jar 파일 빌드 : 빌드 전 권한 설정해주기
3. docker build : docker hub 로그인 -> build -> push
4. 배포 : docker hub에서 image pull -> 이전에 올라와 있던 것 stop -> docker run -> 사용 중이 아닌 이미지 삭제


### Github Action 비밀키 생성법
Github Repository > Settings > Secrets and variables > Actions > New repository secret 버튼을 통해 생성 가능
<img width="793" alt="스크린샷 2023-04-18 오후 8 06 56" src="https://user-images.githubusercontent.com/62213813/232758903-e62b98f0-1277-4425-8bcd-464f5b534edd.png">


## 배포 완료 확인
<img width="655" alt="스크린샷 2023-04-18 오후 8 08 45" src="https://user-images.githubusercontent.com/62213813/232759234-c2e94106-efba-4334-9d40-e17c6d2a87df.png">