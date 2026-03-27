# 01. N-tier Application 구성 (Spring Boot + MySQL)

- Docker를 활용하여 **Spring Boot 백엔드 서버와 MySQL DB를 컨테이너로 구성**하는 실무 아키텍처
- 서비스 간 통신은 **사용자 정의 네트워크 + 컨테이너 이름(DNS)** 으로 처리
![](./images/Pasted%20image%2020260326171219.png)


## 0. 프로젝트 디렉터리 구성
```
mkdir -p /home/ubuntu/docker/08.ntier cd /home/ubuntu/docker/08.ntier mkdir -p data/{mysql,db-init,db-conf}
```

## 1. git repository 생성 후 git clone

- `infra-basic-backend` 이름으로 git repository 생성
![](./images/Pasted%20image%2020260326164923.png)

- 로컬에 git clone
![](./images/Pasted%20image%2020260326164858.png)


## 2. Spring Boot 프로젝트

- 2-1. 프로젝트 세팅
- ![](./images/Pasted%20image%2020260326171551.png)
- ![](./images/Pasted%20image%2020260326171555.png)

```
src/main/java/com/example/backend
├── controller    (Presentation Layer: 외부 요청 접수 및 응답)
├── service       (Application Layer: 비즈니스 로직 처리)
├── repository    (Data Access Layer: DB 데이터 접근)
└── entity        (Domain Model: 데이터베이스 테이블 매핑)
```

- Entity
```java
package com.example.backend.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Entity
@Getter
@NoArgsConstructor
public class Dept {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer deptno;

    private String dname;
    private String loc;
}
```

- Repository
``` java
package com.example.backend.repository;

import com.example.backend.entity.Dept;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

@Repository
public interface DeptRepository extends JpaRepository<Dept, Integer> {
    // 기본적인 CRUD 메서드가 자동으로 생성됩니다.
}
```

- Service
```java
package com.example.backend.service;

import com.example.backend.entity.Dept;
import com.example.backend.repository.DeptRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // 성능 최적화를 위해 읽기 전용 설정
public class DeptService {

    private final DeptRepository deptRepository;

    public List<Dept> findAll() {
        return deptRepository.findAll();
    }
}
```

- Controller
```java
package com.example.backend.controller;

import com.example.backend.entity.Dept;
import com.example.backend.service.DeptService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api/depts")
@RequiredArgsConstructor
public class DeptController {

    private final DeptService deptService;

    @GetMapping
    public List<Dept> getAll() {
        return deptService.findAll();
    }
}
```

- application.properties
```
spring.application.name=infra-basic-backend
server.port=8080

# ▶DB

# MySQL에 접속하기 위한 JDBC 드라이버
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# DB 접속 주소
# ㄴ jdbc:mysql:// →  mysql 프로토콜
# ㄴ localhost → DB 위치 (내 컴퓨터)
# ㄴ 3306 → MySQL 기본 포트
# ㄴ app → 데이터베이스 이름
# ㄴ serverTimezone=Asia/Seoul → 시간설정
spring.datasource.url=jdbc:mysql://localhost:3306/app?serverTimezone=Asia/Seoul

# DB 로그인 계정 정보
spring.datasource.username=app
spring.datasource.password=app


# ▶JPA

# JPA가 DDL(테이블 생성 쿼리) 생성 여부
spring.jpa.generate-ddl=true

# * DB 테이블 자동 관리 옵션
# ㄴ create: 실행할때마다 새로 생성 (기존 삭제)
# ㄴ create-drop: 종료 시 삭제
# ㄴ update: 변경된 부분만 반영
# ㄴ validate: 검증만
# ㄴ none: 아무것도 안함
spring.jpa.hibernate.ddl-auto=update

spring.jpa.properties.hibernate.format_sql=true
spring.jpa.show-sql=true
```
↳ 스프링부트에서 로컬 MySQL을 사용할 때는 
DB가 같은 호스트에서 실행되므로 `localhost`를 사용한다.
(spring.datasource.url=jdbc:mysql://**localhost**:3306/app?serverTimezone=Asia/Seoul)

↳ **하지만, 이번 실습에서는 MySQL을 도커 컨테이너로 실행하므로,
애플리케이션에서 DB에 접근할 때 `localhost` 대신
`docker-compose에 정의된 MySQL 컨테이너의 서비스 이름`을 사용해야 한다.**

- 2-2) git push

## 3. 우분투에 git clone

- 3-1. (git 설치되지 않은 경우) git 설치
```
▶ which git을 했을때 아무것도 출력되지 않음 = git 설치 X 상태

▶ git 설치
sudo apt update
sudo apt install git -y

▶ git 설치 확인 
ubuntu@ubuntu:~/docker/08.ntier$ git --version
git version 2.43.0
```

- 3-2. git clone
```
git clone <레포지토리 주소>
```

```
ubuntu@ubuntu:~/docker/08.ntier$ git clone https://github.com/Containerxox/infra-basic-backend.git
Cloning into 'infra-basic-backend'...
remote: Enumerating objects: 35, done.
remote: Counting objects: 100% (35/35), done.
remote: Compressing objects: 100% (20/20), done.
remote: Total 35 (delta 1), reused 29 (delta 0), pack-reused 0 (from 0)
Receiving objects: 100% (35/35), 13.80 KiB | 1.38 MiB/s, done.
Resolving deltas: 100% (1/1), done.
ubuntu@ubuntu:~/docker/08.ntier$ ls
data  infra-basic-backend
ubuntu@ubuntu:~/docker/08.ntier$ tree
.
├── data
│   ├── db-conf
│   │   └── my.cnf
│   ├── db-init
│   │   └── app.sql
│   └── mysql
└── infra-basic-backend
    ├── infra-basic-backend
    │   ├── mvnw
    │   ├── mvnw.cmd
    │   ├── pom.xml
    │   └── src
    │       ├── main
    │       │   └── java
    │       │       └── com
    │       │           └── example
    │       │               └── backend
    │       │                   ├── controller
    │       │                   │   └── DeptController.java
    │       │                   ├── entity
    │       │                   │   └── Dept.java
    │       │                   ├── InfraBasicBackendApplication.java
    │       │                   ├── repository
    │       │                   │   └── DeptRepository.java
    │       │                   └── service
    │       │                       └── DeptService.java
    │       └── test
    │           └── java
    │               └── com
    │                   └── example
    │                       └── backend
    │                           └── InfraBasicBackendApplicationTests.java
    └── README.md

22 directories, 12 files
```


## 4. 우분투에 application.properties 생성
`.gitignore`에 의해 `application.properties`는 
깃헙에 업로드 되지 않는다. 

그래서 우분투에서 직접 `application.properties`를 만들어줘야 한다.

```
ubuntu@ubuntu:~/docker/08.ntier$ mkdir -p /home/ubuntu/docker/08.ntier/infra-basic-backend/infra-basic-backend/src/main/resources

ubuntu@ubuntu:~/docker/08.ntier$ vi /home/ubuntu/docker/08.ntier/infra-basic-backend/infra-basic-backend/src/main/resources/application.properties

4. application.properties 복붙
(주의: spring.datasource.url=jdbc:mysql://<docker-compose의 mysql서비스명>:3306/app?serverTimezone=Asia/Seoul 으로 변경해줘야 한다.!)

- 방법1) docker-compose에서 mysql컨테이너의 서비스명을 db로 설정했으니, db로 직접 수정해주면 된다!!!!

- 방법2) .env파일을 생성해서 변수화  
```

- 방법 1) application.properties
```
  spring.application.name=infra-basic-backend
  server.port=8080
   
  # DB
  spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
  spring.datasource.url=jdbc:mysql://db:3306/app?serverTimezone=Asia/Seoul
  spring.datasource.username=app
 spring.datasource.password=app
  
 # JPA
 spring.jpa.generate-ddl=true
 spring.jpa.hibernate.ddl-auto=update
 spring.jpa.properties.hibernate.format_sql=true
 spring.jpa.show-sql=true
```

- 방법 2) application.properties
```
 spring.application.name=infra-basic-backend
  server.port=8080
   
  # DB
  spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
  spring.datasource.url=jdbc:mysql://${MYSQL_HOST}:3306/${MYSQL_DATABASE}?serverTimezone=Asia/Seoul
  spring.datasource.username=${MYSQL_USER}
 spring.datasource.password=${MYSQL_PASSWORD}
  
 # JPA
 spring.jpa.generate-ddl=true
 spring.jpa.hibernate.ddl-auto=update
 spring.jpa.properties.hibernate.format_sql=true
 spring.jpa.show-sql=true
```

## 5. DB 초기화 SQL (app.sql)

- `emp`, `dept` 테이블 생성 & 데이터 삽입
```sql
cat > data/db-init/app.sql << 'EOF'
USE app;

DROP TABLE IF EXISTS emp;
DROP TABLE IF EXISTS dept;

CREATE TABLE IF NOT EXISTS dept (
  DEPTNO int NOT NULL,
  DNAME varchar(14) DEFAULT NULL,
  LOC varchar(13) DEFAULT NULL,
  PRIMARY KEY (DEPTNO)
);

INSERT INTO dept VALUES (10, 'ACCOUNTING', 'NEW YORK');
INSERT INTO dept VALUES (20, 'RESEARCH', 'DALLAS');
INSERT INTO dept VALUES (30, 'SALES', 'CHICAGO');
INSERT INTO dept VALUES (40, 'OPERATIONS', 'BOSTON');

CREATE TABLE IF NOT EXISTS emp (
  EMPNO int NOT NULL,
  ENAME varchar(10) DEFAULT NULL,
  JOB varchar(9) DEFAULT NULL,
  SAL double DEFAULT NULL,
  DEPTNO int DEFAULT NULL,
  PRIMARY KEY (EMPNO),
  KEY fk_emp (DEPTNO)
);

INSERT INTO emp VALUES (7369, 'SMITH', 'CLERK', 800, 20);
INSERT INTO emp VALUES (7499, 'ALLEN', 'SALESMAN', 1600, 30);
INSERT INTO emp VALUES (7521, 'WARD', 'SALESMAN', 1250, 30);
INSERT INTO emp VALUES (7839, 'KING', 'PRESIDENT', 5000, 10);
INSERT INTO emp VALUES (7902, 'FORD', 'ANALYST', 3000, 20);

ALTER TABLE emp
  ADD CONSTRAINT fk_emp FOREIGN KEY (DEPTNO) REFERENCES dept (DEPTNO)
  ON DELETE SET NULL ON UPDATE CASCADE;

COMMIT;
EOF
```

- 생성 및 구조 확인
```bash
ubuntu@ubuntu:~/docker/08.ntier$ tree
.
└── data
    ├── db-conf
    ├── db-init
    │   └── app.sql
    └── mysql
```

## 6. MySQL 설정 파일 (my.cnf)

```bash
cat > data/db-conf/my.cnf << 'EOF'
[client]
default-character-set=utf8mb4

[mysql]
default-character-set=utf8mb4

[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
EOF
```

- 생성 및 구조 확인
```bash
ubuntu@ubuntu:~/docker/08.ntier$ tree
.
└── data
    ├── db-conf
    │   └── my.cnf
    ├── db-init
    │   └── app.sql
    └── mysql

```

## 7. Spring Boot Dockerfile
Dockerfile 이용하여 
Step 1) 자바 프로젝트 빌드 → app.jar 생성 
Step 2) /app/target/ 임시경로에 생성된 app.jar를 생성될 컨테이너에 Copy → 컨테이너에서 java -jar app.jar 자바 프로젝트 실행

```
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

# Stage 2: 실행 이미지
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

주의) **`pom.xml`와 `src` 폴더가 존재하는 경로에 Dockerfile 생성해주어야 한다!**
경로 → `/home/ubuntu/docker/08.ntier/infra-basic-backend/infra-basic-backend/Dockerfile`
```
ubuntu@ubuntu:~/docker/08.ntier$ tree
.
├── data
│   ├── db-conf
│   │   └── my.cnf
│   ├── db-init
│   │   └── app.sql
│   └── mysql
└── infra-basic-backend
    ├── infra-basic-backend
    │   ├── Dockerfile   ← 여기 있어야 함 !!!!
    │   ├── mvnw
    │   ├── mvnw.cmd
    │   ├── pom.xml
    │   └── src
    │       ├── main
    │       │   ├── java
    │       │   │   └── com
    │       │   │       └── example
    │       │   │           └── backend
    │       │   │               ├── controller
    │       │   │               │   └── DeptController.java
    │       │   │               ├── entity
    │       │   │               │   └── Dept.java
    │       │   │               ├── InfraBasicBackendApplication.java
    │       │   │               ├── repository
    │       │   │               │   └── DeptRepository.java
    │       │   │               └── service
    │       │   │                   └── DeptService.java
    │       │   └── resources
    │       │       └── application.properties
    │       └── test
    │           └── java
    │               └── com
    │                   └── example
    │                       └── backend
    │                           └── InfraBasicBackendApplicationTests.java
    └── README.md

23 directories, 14 files
```

## 7. .env 생성

- `/home/ubuntu/docker/08.ntier`경로에 `.env`생성
```
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=app
MYSQL_USER=app
MYSQL_PASSWORD=app
MYSQL_HOST=db
```

## 8. .dockerignore 생성

`.dockerignore`파일은 **build context 폴더 안**"에 둬야 한다 !
( context 밖이면 적용되지 않음)

> .dockerignore 파일은 Dockerfile이 아니라 "context 기준 폴더 안"에 둬야 한다!

- 이유
	- 도커는 build context 기준으로 파일을 보내고
	- `.dockerignore`도 그 기준으로만 적용됨

그래서 `docker-compose.yml` 파일이 아래처럼 설정되어 있다면,
```
backend:
  build:
    context: ./infra-basic-backend/infra-basic-backend
```

그럼 `.dockerignore` 파일의 위치는 아래 경로에 있어야 한다.
```
`/home/ubuntu/docker/08.ntier/infra-basic-backend/infra-basic-backend/.dockerignore
```

- 구조
```
08.ntier/  
	├── data/  
	└── infra-basic-backend/  
		└── infra-basic-backend/  
			├── Dockerfile  
			├── .dockerignore ✅ 여기  
			├── pom.xml  
			└── src/
```

- .dockerignore
```
# git
.git
.gitignore

# build
target/
build/

# IDE
.idea
.vscode

# logs
*.log

# OS
.DS_Store

# test
src/test/
```

## 9. docker-compose.yml 생성
참고)
- `.env`는 docker-compose.yml에서 변수 치환용
- `environment`는 컨테이너 내부에서 실제로 사용하는 환경 변수
```
.env
  ↓ (값 제공)
docker-compose.yml (${변수 치환})
  ↓ (environment에 넣음)
컨테이너 내부 환경변수 생성
  ↓
Spring Boot가 읽음 (${...})
```

- `docker-compose.yml` 
```yml
services:
  db:
    container_name: mysql-server
    image: mysql:8.0
    ports:
      - "3306:3306"
    volumes:
      - ./data/mysql:/var/lib/mysql
      - ./data/db-init:/docker-entrypoint-initdb.d
      - ./data/db-conf:/etc/mysql/conf.d
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    networks:
      - app-network
    healthcheck:
      test: ['CMD-SHELL', 'mysqladmin ping -h 127.0.0.1 -u root --password=$$MYSQL_ROOT_PASSWORD']
      interval: 10s
      timeout: 5s
      retries: 20

  backend:
    container_name: backend_server
    build: 
      context: ./infra-basic-backend/infra-basic-backend
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      MYSQL_HOST: ${MYSQL_HOST}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

- `docker-compose config`를 통해
환경변수(.env)가 잘 치환됐는지 + YAML 문법이 맞는지 확인
```bash
ubuntu@ubuntu:~/docker/08.ntier$ docker-compose config
name: 08ntier
services:
  backend:
    build:
      context: /home/ubuntu/docker/08.ntier/infra-basic-backend/infra-basic-backend
      dockerfile: Dockerfile
    container_name: backend_server
    depends_on:
      db:
        condition: service_healthy
        required: true
    environment:
      MYSQL_DATABASE: app
      MYSQL_HOST: db
      MYSQL_PASSWORD: app
      MYSQL_USER: app
    networks:
      app-network: null
    ports:
      - mode: ingress
        target: 8080
        published: "8080"
        protocol: tcp
  db:
    container_name: mysql-server
    environment:
      MYSQL_DATABASE: app
      MYSQL_PASSWORD: app
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: app
    healthcheck:
      test:
        - CMD-SHELL
        - mysqladmin ping -h 127.0.0.1 -u root --password=$$MYSQL_ROOT_PASSWORD
      timeout: 5s
      interval: 10s
      retries: 20
    image: mysql:8.0
    networks:
      app-network: null
    ports:
      - mode: ingress
        target: 3306
        published: "3306"
        protocol: tcp
    volumes:
      - type: bind
        source: /home/ubuntu/docker/08.ntier/data/mysql
        target: /var/lib/mysql
        bind:
          create_host_path: true
      - type: bind
        source: /home/ubuntu/docker/08.ntier/data/db-init
        target: /docker-entrypoint-initdb.d
        bind:
          create_host_path: true
      - type: bind
        source: /home/ubuntu/docker/08.ntier/data/db-conf
        target: /etc/mysql/conf.d
        bind:
          create_host_path: true
networks:
  app-network:
    name: 08ntier_app-network
    driver: bridge
```


## 9. docker-compose.yml을 통해 서비스 시작

```
ubuntu@ubuntu:~/docker/08.ntier$ docker-compose up -d
```


## 10. 접속

```
localhost:8080/api/depts
```

- 브라우저
![](./images/Pasted%20image%2020260326233643.png)
- postman
![](./images/Pasted%20image%2020260326233818.png)

(확인 사항)
- 해당 URL로 접속했을 때 dept 테이블의 모든 데이터가 출력되는지

> 모든 설정이 정상이라면,
> 로컬의 8080 포트로 접속 시 우분투의 8080 포트를 거쳐 
> Spring Boot 컨테이너로 연결된다.

>그리고 Spring Boot는 같은 Docker 네트워크(app-network)에 있는 
>db 컨테이너에 접근하여
> dept 테이블의 데이터를 조회한 뒤 결과를 반환한다.

```
로컬 8080 → 우분투 8080 → backend 컨테이너 → db 컨테이너 → 응답
```


🟪 전체 구조
```
ubuntu@ubuntu:~/docker/08.ntier$ tree -a -L 3
.
├── data
│   ├── db-conf
│   │   └── my.cnf
│   ├── db-init
│   │   └── app.sql
│   └── mysql
├── docker-compose.yml
├── .env
└── infra-basic-backend
    ├── .git
    │   ├── branches
    │   ├── config
    │   ├── description
    │   ├── HEAD
    │   ├── hooks
    │   ├── index
    │   ├── info
    │   ├── logs
    │   ├── objects
    │   ├── packed-refs
    │   └── refs
    ├── .gitignore
    ├── infra-basic-backend
    │   ├── Dockerfile
    │   ├── .dockerignore
    │   ├── .gitattributes
    │   ├── .gitignore
    │   ├── mvnw
    │   ├── mvnw.cmd
    │   ├── pom.xml
    │   └── src
    └── README.md
```


## 11. Docker Compose 실무 패턴: build에서 image로 전환하는 이유

실무에서는 `docker-compose.yml`에서 `build.context`를 사용해 각 개발 환경에서 직접 이미지를 빌드하는 방식은 지양하는 편이다. 이는 사용자마다 개발 환경이 다르기 때문에 빌드 결과가 달라지거나 예상치 못한 오류가 발생할 수 있기 때문이다.

따라서 일반적으로는 `docker build -t <이미지명>:<태그> .` 명령어를 통해 표준화된 환경(CI/CD 등)에서 이미지를 미리 빌드하고, 이를 컨테이너 레지스트리에 저장한 뒤 사용하는 방식을 따른다.

이후 `docker-compose.yml`에서는 `build` 대신 `image:`를 사용하여, 미리 생성된 이미지를 기반으로 컨테이너를 실행한다.

즉, Dockerfile을 통해 이미지 생성 방식을 고정하고, 실행 시에는 동일한 이미지를 사용함으로써 환경 차이에 따른 문제를 최소화하는 것이 실무적인 접근 방식이다.

1. Dockerfile을 통해 image 생성
```
ubuntu@ubuntu:~/docker/08.ntier$ cd infra-basic-backend/infra-basic-backend/

ubuntu@ubuntu:~/docker/08.ntier/infra-basic-backend/infra-basic-backend$ docker build -t backend:v1.0 .
```

2. image가 생성됬는지 확인
```
ubuntu@ubuntu:~/docker/08.ntier/infra-basic-backend/infra-basic-backend$ docker images 
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
backend      v1.0      89bfe5a76099   13 seconds ago   239MB
```

3. 레지스트리(Docker hub)에 생성한 image 올리기

3-1.  Docker Hub 접속 → 레포지토리 생성
![](./images/Pasted%20image%2020260327120005.png)

3-2. docker hub에 image 올리기
```
< image에 Hub 태그 추가 >
예) docker tag <로컬이미지명> <DockerHub_ID>/<Repository명>:<태그>
▶ docker tag backend:v1.0 containerxox/infra-basic-backend:v1.0


< 확인 >
ubuntu@ubuntu:~/docker/08.ntier$ docker images
REPOSITORY                         TAG       IMAGE ID       CREATED          SIZE
containerxox/infra-basic-backend   v1.0      89bfe5a76099   17 minutes ago   239MB
backend                            v1.0      89bfe5a76099   17 minutes ago   239MB


< image push >
예) docker push <DockerHub_ID>/<Repository명>:<태그>
▶ docker push containerxox/infra-basic-backend:v1.0
```

3-3.  image가 Docker hub에 push 된 것을 확인
![](./images/Pasted%20image%2020260327120630.png)

추가) 
- 서버에서  image를 pull하고 싶다면
```
docker pull containerxox/infra-basic-backend:v1.0
```


4. docker-compose.yml 수정
```yml
services:
  db:
    container_name: mysql-server
    image: mysql:8.0
    ports:
      - "3306:3306"
    volumes:
      - ./data/mysql:/var/lib/mysql
      - ./data/db-init:/docker-entrypoint-initdb.d
      - ./data/db-conf:/etc/mysql/conf.d
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    networks:
      - app-network
    healthcheck:
      test: ['CMD-SHELL', 'mysqladmin ping -h 127.0.0.1 -u root --password=$$MYSQL_ROOT_PASSWORD']
      interval: 10s
      timeout: 5s
      retries: 20

  backend:
    container_name: backend_server
        image: backend:v1.0  ← 수정한 부분 ✅
    ports:
      - "8080:8080"
    environment:
      MYSQL_HOST: ${MYSQL_HOST}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

5. docker-compose.yml을 통해 서비스 시작
```
ubuntu@ubuntu:~/docker/08.ntier$ docker-compose up -d 
```
