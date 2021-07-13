# Table of contents 
* 개발환경 
    * VSCODE 설치
    * JDK 설치
    * MAVEN 설치
    * DOCKER 설치
    * docker-compose 설치
    * GIT 설치
    * VSCODE Extension pack 설치
    * AWS Config


* 주로 사용하는 명령어 모음

* 개발 
    * 소스코드 다운로드
    * 소소코드 폴더 구조
    * 빌드 및 실행
    * eco-egy-gateway 
    * eco-egy-auth
    * eco-egy-admin
    * eco-egy-general

* 운영
    * AWS CodeCommit
    * AWS ECR
    * AWS CodeBuild
    * AWS EKS

    
---  


# 개발환경
- VSCODE 설치
    - 다운로드
      * https://code.visualstudio.com/download
    - JDK 다운로드 및 extension pack 설치(참고용)
     
    ```
     1. vscode 실행
     2. ctrl + shift + p  클릭 후  configure java runtime 실행 
     3. jdk11을 다운로드 한다.
     4. 다운로드한 압축파일을 해제 한 후 dev-tool (참고용)  폴더 아래로 이동 시킨다.         
     * 참고사이트 : https://allonsyit.tistory.com/10
    ```

- JDK 설치
    - Java version 확인 (참고용)
    ```
      $ java -version
        openjdk version "11.0.11" 2021-04-20
        OpenJDK Runtime Environment (build 11.0.11+9-Ubuntu-0ubuntu2.20.04)
        OpenJDK 64-Bit Server VM (build 11.0.11+9-Ubuntu-0ubuntu2.20.04, mixed mode, sharing)            
    ```

    
    - 환경변수 설정 (참고용)
    ```
    $ vim ~/.bashrc
      # 편집 및 추가
        export JAVA_HOME=/home/kafka/Dev-tool/jdk-11.0.11+9
        export PATH=$PATH:$JAVA_HOME/bin  
       
    $ source ~/.bashrc       
    $ echo $JAVA_HOME
    ```

- MAVEN 설치
    ```
    $ sudo apt install maven
    $ mvn -v
    	Apache Maven 3.6.3
	Maven home: /usr/share/maven
	Java version: 11.0.11, vendor: AdoptOpenJDK, runtime: /home/kafka/Dev-tool/jdk-11.0.11+9
	Default locale: en_US, platform encoding: UTF-8
	OS name: "linux", version: "5.8.0-59-generic", arch: "amd64", family: "unix"
    $ apt list maven    
    ```

- DOCKER 설치
    * https://kangwoo.kr/2020/07/25/%EC%9A%B0%EB%B6%84%ED%88%AC%EC%97%90-docker-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0/
    
    ```  
    $ sudo usermod -aG docker <your-username>
    or
    $ sudo chown root:docker /var/run/docker.sock 
    or
    $ sudo chmod 666 /var/run/docker.sock
    ```
- docker-compose 설치
   ```
   $ sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   $ sudo chmod +x /usr/local/bin/docker-compose
   $ sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
   $ docker-compose --version
     docker-compose version 1.29.2, build unknown
   * 참고사이트 : https://docs.docker.com/compose/install/
   ```
   
- GIT 설치
   ```
   패키지 리스트 업데이트
   $ sudo apt-get install git
    
   git 설치.
   $ sudo apt install git
    
   git의 버전 확인
   $ git --version
     git version 2.25.1
   push했을때 올라갈 내 정보를 입력해줍니다.
   $ git config --global user.name [이름]
   $ git config --global user.mail [메일 주소]
   ```

- VSCODE Extension pack 설치
   ```
   Java Extension Pack
   Maven for java
   Spring Boot Tools
   Spring Boot Extension Pack
   Spring Boot Dashboard
   Docker
   Kubernetes
   Kafka    
   ```

#  

- 소스코드 다운로드
  ```
    git clone https://git-codecommit.ap-northeast-2.amazonaws.com/v1/repos/eco_project
  ```

- 소스코드 폴더 구조
  
  eco_project
  1. eco-egy-lib
  ```
      com.skep.eco.common.config
      com.skep.eco.common.exception
      com.skep.eco.common.util
      com.skep.eco.core.controller
      com.skep.eco.core.dto
  ```
  2. micro/eco-egy-gateway
  ```
  ```
  3. micro/eco-egy-auth
  ```   
      kubernetes
      src/main/java
      	com.skep.eco.egy.auth.config
      	com.skep.eco.egy.auth.controller
      	com.skep.eco.egy.auth.handler
      	com.skep.eco.egy.auth.interceptor
      	com.skep.eco.egy.auth.persistence
      	com.skep.eco.egy.auth.security
      	com.skep.eco.egy.auth.service
      src/resources
        templates
      	application.yml
       target
       buildspec.yaml
       Dockerfile
       pom.xml
  ```
  4. micro/eco-egy-admin
  ```
      kubernetes
      src/main/java
      	com.skep.eco.egy.admin.config
      	com.skep.eco.egy.admin.controller
      	com.skep.eco.egy.admin.handler
      	com.skep.eco.egy.admin.interceptor
      	com.skep.eco.egy.admin.persistence
      	com.skep.eco.egy.admin.security
      	com.skep.eco.egy.admin.service
      src/resources
         templates
      	 application.yml
       target
       buildspec.yaml
       Dockerfile
       pom.xml
  ```
  5. micro/eco-egy-general       
  ```
      kubernetes
      src/main/java
      	com.skep.eco.egy.general.config
      	com.skep.eco.egy.general.controller
      	com.skep.eco.egy.general.handler
      	com.skep.eco.egy.general.interceptor
      	com.skep.eco.egy.general.persistence
      	com.skep.eco.egy.general.security
      	com.skep.eco.egy.general.service
      src/resources
         templates
      	 application.yml
       target
       buildspec.yaml
       Dockerfile
       pom.xml
  ```
  
- 소스코드 커밋
  eco_project 루트폴더 에서 실행함.  
  1. 최초 실행시
  ```
  git init
  git add .
  git commit -m "first commit"
  git branch -M main
  git remote add origin https://git-codecommit.ap-northeast-2.amazonaws.com/v1/repos/eco_project
  git push -u origin main
  ```
  2.소스코드 변경작업 후
  ```
  git add .
  git commit -m "commit"
  git push -u origin main
  
  ```
  
- 전체빌드 및 마이크로 서비스 실행
  eco_project 루트폴더 에서 실행함.
  ```
   mvn clean package
   docker system prune -a
   docker-compose down && docker-compose build --no-cache && docker-compose up
  ```
- eco-egy-gateway 
  ```
        
    ``` 
- eco-egy-auth
    ```
        
    ```
- eco-egy-admin
    ```
        
    ```
- eco-egy-general
    ```
        
    ```


# 운영
- AWS CodeCommit
    ```
        
    ```
- AWS ECR
    ```
        
    ``` 
- AWS CodeBuild
    ```
        
    ``` 
- AWS EKS
    ```
        
    ```
    
