# Table of contents 
* 개발환경 
    * VSCODE 설치
    * JDK 설치
    * MAVEN 설치
    * DOCKER 설치
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
    - VSCODE내 JDK 다운로드 및 extension pack 설치(참고용)
      * https://allonsyit.tistory.com/10

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
      sudo apt install maven
      mvn -v
      apt list maven    
    ```

- DOCKER 설치
    * https://kangwoo.kr/2020/07/25/%EC%9A%B0%EB%B6%84%ED%88%AC%EC%97%90-docker-%EC%84%A4%EC%B9%98%ED%95%98%EA%B8%B0/
    
    ```  
    sudo usermod -aG docker <your-username>
    or
    sudo chown root:docker /var/run/docker.sock 
    or
    sudo chmod 666 /var/run/docker.sock
    ```

- GIT 설치
    ```
    명령어를 입력하여 패키지 리스트를 업데이트합니다.
    sudo apt-get install git
    
    명령어를 입력하여 깃을 설치합니다.
    sudo apt install git
    
    git의 버전을 알 수 있습니다
    git --version
    
    push했을때 올라갈 내 정보를 입력해줍니다.
    git config --global user.name [이름]
    git config --global user.mail [메일 주소]
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
    ```
        
    ```
- 빌드 및 실행
    ```
        
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
    
