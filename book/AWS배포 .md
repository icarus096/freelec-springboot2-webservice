# 스프링부트 AWS에 배포해 보자. 

1~5장 스프링부트 서비스 코드 개발, 6~7장 배포환경 구성(AWS)

이제 이들을 조합해 실제 서비스를 배포해 본다. 

## AWS EC2 서버에 프로젝트 배포 (step1)

git에서 clone 받아 build/deploy 하는 스크립트를 만들어보자. 

```bash
sudo yum install git

$ mkdir -p ~/app/step1; cd ~/app/step1
$ git clone https://github.com/icarus096/freelec-springboot2-webservice.git
#테스트 검증 (5장 Security적용하기까지 잘 적용되었다면 통과해야함)
$ ./gladlew test 

#deploy.sh 작성 (STEP1 -> STEP2)
$ vim ~/app/git/deploy-step1.sh

#nohup 으로 java jar 실행
# 에러가 난다. 이유는, (커밋 예외 항목이므로) property 파일이 없어서
# 배포보다 상위 폴더에 파일을 생성해보자. 
$ vi /home/ec2-user/app/application-oauth.properties

-> 파일 생성 후에 아래와 같이 상용 Property 파일을 포함해 실행하도록 한다. 
-> classpath: 가 붙으면 프로젝트내 resources 폴더를 기준으로 한다. 

# test
$ nohup java -jar -Dspring.config.location=classpath:/application.properties,classpath:/application-real.properties,/home/ec2-user/app/application-oauth.properties,/home/ec2-user/app/application-real-db.properties \
    -Dspring.profiles.active=real \
    springboot-aws.jar > springbootaws.log 2>&1 &

# server local test
$ java -jar -Dspring.config.location=classpath:/application-real.properties,/home/ec2-user/app/application-oauth.properties,/home/ec2-user/app/application-real-db.properties \
    -Dspring.profiles.active=real \
    springboot-aws.jar
    
$ curl localhost:8080
$ while sleep 1; do curl localhost:8080; done
```

http://ec2-13-125-76-186.ap-northeast-2.compute.amazonaws.com:8080/

> DB에 Posts, spring session 관려 테이블2개 생성해준다. (Scheme.sql )
>
> 오류: longvar~ -> blob

```sql
create table posts (id bigint not null auto_increment, created_date datetime, modified_date datetime, author varchar(255), content TEXT not null, title varchar(500) not null, primary key (id)) engine=InnoDB;
create table user (id bigint not null auto_increment, created_date datetime, modified_date datetime, email varchar(255) not null, name varchar(255) not null, picture varchar(255), role varchar(255) not null, primary key (id)) engine=InnoDB;

CREATE TABLE SPRING_SESSION (
	PRIMARY_ID CHAR(36) NOT NULL,
	SESSION_ID CHAR(36) NOT NULL,
	CREATION_TIME BIGINT NOT NULL,
	LAST_ACCESS_TIME BIGINT NOT NULL,
	MAX_INACTIVE_INTERVAL INT NOT NULL,
	EXPIRY_TIME BIGINT NOT NULL,
	PRINCIPAL_NAME VARCHAR(100),
	CONSTRAINT SPRING_SESSION_PK PRIMARY KEY (PRIMARY_ID)
);

CREATE UNIQUE INDEX SPRING_SESSION_IX1 ON SPRING_SESSION (SESSION_ID);
CREATE INDEX SPRING_SESSION_IX2 ON SPRING_SESSION (EXPIRY_TIME);
CREATE INDEX SPRING_SESSION_IX3 ON SPRING_SESSION (PRINCIPAL_NAME);

CREATE TABLE SPRING_SESSION_ATTRIBUTES (
	SESSION_PRIMARY_ID CHAR(36) NOT NULL,
	ATTRIBUTE_NAME VARCHAR(200) NOT NULL,
	ATTRIBUTE_BYTES LONGVARBINARY NOT NULL,
	CONSTRAINT SPRING_SESSION_ATTRIBUTES_PK PRIMARY KEY (SESSION_PRIMARY_ID, ATTRIBUTE_NAME),
	CONSTRAINT SPRING_SESSION_ATTRIBUTES_FK FOREIGN KEY (SESSION_PRIMARY_ID) REFERENCES SPRING_SESSION(PRIMARY_ID) ON DELETE CASCADE
);

```



Build.gladle에 mariadb plugin 설치

```
compile("org.mariadb.jdbc:mariadb-java-client")
```

application-real.properties 생성 (real profile 생성)

-> 보안상 문제될 파일은 넣지 않음.  (아래 2개 파일은 서버에만 위치하도록 한다. github 공개금지)

application-oauth.properties

```properties
spring.security.oauth2.client.registration.google.client-id=구글클라이언트ID
spring.security.oauth2.client.registration.google.client-secret=구글클라이언트시크릿
spring.security.oauth2.client.registration.google.scope=profile,email

# registration
spring.security.oauth2.client.registration.naver.client-id=네이버클라이언트ID
spring.security.oauth2.client.registration.naver.client-secret=네이버클라이언트시크릿
spring.security.oauth2.client.registration.naver.redirect-uri={baseUrl}/{action}/oauth2/code/{registrationId}
spring.security.oauth2.client.registration.naver.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.naver.scope=name,email,profile_image
spring.security.oauth2.client.registration.naver.client-name=Naver

# provider
spring.security.oauth2.client.provider.naver.authorization-uri=https://nid.naver.com/oauth2.0/authorize
spring.security.oauth2.client.provider.naver.token-uri=https://nid.naver.com/oauth2.0/token
spring.security.oauth2.client.provider.naver.user-info-uri=https://openapi.naver.com/v1/nid/me
spring.security.oauth2.client.provider.naver.user-name-attribute=response


```

application-real-db.properties

```properties
# JPA 테이블 자동생성 하지 않음. !주의: 잘못하면 기존 테이블이 모조리 교체될 수 있음. 
spring.jpa.hibernate.ddl-auto=none 

spring.datasource.url=jdbc:mariadb://rds주소:포트명(기본은 3306)/database명
spring.datasource.username=db계정
spring.datasource.password=db계정 비밀번호
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver
```



### EC2에 소셜 로그인하기

* 체크; 8080이 열려있는지 확인: 보안그룹 > ec2 > 8080인바운드 확인
* ec2 퍼블릭 DNS명 확인 

##### 구글에 EC2 주소 등록

http://console.cloud.google.com/home/dashboard > API 및 서비스 > 사용자 인증정보

> OAuth 동의 화면 탭 > 승인된 도메인 -> EC2 퍼블릭 DNS 입력

사용자인증정보 탭 > 등록한 서비스명 클릭 > 

> 퍼블릭 DNS 주소에 :8080/login/oauth2/code/google 주소 추가하여 승인된 리디렉션 URI에 등록

##### 네이버에 EC2 주소 등록

Https://developers.naver.com/apps/#/myapps > 프로젝트 이동 > API 설정 > PC웹 항목: 서비스 URL(8080빼고)과 Callback URL(http://pubDNS:8080/login/oauth2/code/naver) 2개 수정



## Travis CI 배포 자동화

Git에 PUSH되면 자동으로 테스트와 빌드 수행 (CI - Continous Integration, 지속적통합)

이 빌드 결과를 자동으로 운영서버에 무중단 배포 (CD - Continous Deployment, 지속적 배포)

마틴파울러 블로그(http://bit.ly/2Yv0vFp) CI 규칙참조

* 모든 소스 코드가 살아있고 누구든 현재의 소스에 접근할 수 있는 단일 지점을 유지할 것
* 빌드 프로세스를 자동화해서 누구든 소스로부터 시스템을 빌드하는단일 명령을 사용 가능
* 테스팅을 자동화해서 단일 명령어로 언제든지시스템에 대한 건전한 테스트 수트를 실행 가능
* 누구나 현재 실행 파일을 얻으면 지금까지 가장 완전한 실행 파일을 얻었다는 확신을 하게 할 것

테스트자동화가 중요하다: 백명석님의 클린코더스 TDD편 참고 http://bit.ly/2xtKinX

###### Travis CI는..

> 깃허브 제공 무료 CI 서비스 
>
> AWS는 CodeBuild를 제공하나 유료임(빌드시간만큼 부과)

###### Travis CI 세팅

> https://travis-ci.org > 계정명 -> Settings > CI에 연결할 Github 프로젝트 선택
>
> 해당 프로젝트를 선택하면 히스토리를 볼 수 있다. 
>
> Travis CI 세팅은 이게 끝. 자세한 사항은 **`.travis.yml`** 파일에서 한다. 

notification 에 이메일 변경하고 push 하자. 히스토리 페이지에서 실시간 로그를 이메일로 결과를 받아볼 수 있다. 



## Travis CI 와 S3 연동

실제 배포는 AWS CodeDeploy 이용한다. S3는 jar 파일 전달용으로 사용.

##### AWS IAM Key 발급

* 외부 서비스 접근을 위해, 접근권한을 가진 key를 생성해 사용

* IAM(Identity and Access Management) : 서비스접근방식과 권한 관리

* Travis CI -> S3 and CodeDeploy에 접근시 IAM 키를 사용함. 

  

##### TravisCI용 권한 사용자 발급

* IAM > 사용자 > 사용자추가 > "deploy용 사용자명", **프로그래밍방식** 선택 > 기존정책직접연결 > **s3full, CodeDeployFull** 검색/체크 > Tag지정. Name: springboot-aws-travis-deploy 

* 생성하면, 액세스key ID/비밀 액세스 키 두개를 구할 수 있다. 이것을 TravisCI에 설정저장

* AWS _Access_key, AWS_Secret_key 저장 (생성시에만 다운로드 가능)

* Travisci project home > more option> setting > Environment Variable 등록

  * AWS_ACCESS_KEY

  * AWS_SECRET_KEY

    

##### AWS S3 버킷 생성

Simple Storage Service (S3) : 파일서버. 첨부파일 저장, 빌드파일 저장

S3 > 버킷 만들기 > "springboot-aws-build" > 다음:미설정 > 다음:모든 퍼블릭 차단 > 버킷만들기 



##### IAM 역할 생성

* EC2가 CodeDeploy를 연동 받을 수 있게 역할을 생성
* 역할: AWS서비스에만 할당가능 (EC2, CodeDeploy, SQS 등)
* 사용자: AWS외에 사용할 수 있는 권한 (로컬 PC, IDC서버 등)
* IAM > 역할 > 역할만들기 > AWS서비스->EC2 > 정책: EC2RoleForA 선택 > 태그 지정 > 생성
* EC2 > 인스턴스 우클릭: 인스턴스설정 -> IAM역할 연결/바꾸기 > "위의 role 선택" > EC2 재부팅 (해야 반영됨)

