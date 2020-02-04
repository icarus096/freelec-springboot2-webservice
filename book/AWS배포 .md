# 스프링부트 AWS에 배포해 보자. 

1~5장 스프링부트 서비스 코드 개발, 6~7장 배포환경 구성(AWS)

이제 이들을 조합해 실제 서비스를 배포해 본다. 

## AWS EC2 서버에 프로젝트 배포 (step1)

git에서 clone 받아 build/deploy 하는 스크립트를 만들어보자. 

```bash
sudo yum install git

$ mkdir -p ~/app/step1; cd ~/app/step1
$ git clone https://github.com/icarus096/freelec-springboot2-webservice.git
// 테스트 검증 (5장 Security적용하기까지 잘 적용되었다면 통과해야함)
$ ./gladlew test 

//deploy.sh 작성 (STEP1 -> STEP2)
$ vim ~/app/git/deploy.sh

nohup 으로 java jar 실행
-> 에러가 난다. 이유는, property 파일이 없어서(커밋 예외 항목으로)
-> 배포보다 상위 폴더에 파일을 생성해보자. 
$ vi /home/ec2-user/app/application-oauth.properties
-> 파일 생성 후에 아래와 같이 상용 Property 파일을 포함해 실행하도록 한다. 
-> classpath: 가 붙으면 프로젝트내 resources 폴더를 기준으로 한다. 
$ nohup java -jar -Dspring.config.location=classpath:/application.properties,classpath:/application-real.properties,/home/ec2-user/app/application-oauth.properties,/home/ec2-user/app/application-real-db.properties \
    -Dspring.profiles.active=real \
    $JAR_NAME > $REPOSITORY/nohup.out 2>&1 &
```



DB에 Posts, spring session 관려 테이블2개 생성해준다. (Scheme.sql )

오류: longvar~ -> blob

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
spring.jpa.hibernate.ddl-auto=none

spring.datasource.url=jdbc:mariadb://rds주소:포트명(기본은 3306)/database명
spring.datasource.username=db계정
spring.datasource.password=db계정 비밀번호
spring.datasource.driver-class-name=org.mariadb.jdbc.Driver
```

