# AWS 환경 구축

스프링 부트와 AWS로 혼자 구현하는 웹 서비스 - AWS 세팅 (Chapter 6,7)

## AWS 장점

* 1년간 대부분 무료

* 개발자가 할일을 대신 해준다. (백업,모니터링,로그관리,클러스터링 등)

* 많은 기업이 AWS로 이전 중이라 이직시 도움이 된다. 

* 사용자가 많아 커뮤니티 자료가 많다. 

  

## AWS EC2 (Elastic Compute Cloud)

* 무료티어에서는 t2.micro만 가능 (vCPU 1Core, Mem 1GB) 
* vCPU는 물리 CPU의 절반 정도 성능
* 월 750시간 제한 (초과시 비용 부과, 24시간*31일=744시간, 1대 사용시 24시간 가능)

### 생성

* 서울 리전 (northeast-2)
* AWS management Console -> ec2 -> 인스턴스 시작 
* AMI(Amazon Machine Image) - **Amazon Linux 1** OS 선택 (CentOS 6 기반, 2는 7)
  * 아마존이 개발하고 있어 지원이 쉽다, 레드햇 기반, AWS 서비스 상성이 좋음, Amazon 개발 리포지터리 사용으로 yum 이 빠름. 
* 인스턴스 유형 > 프리티어 > t2.micro
* 스토리지 추가 > 30GB (프리티어 최대) 
* 태그 추가 > ec2 instance 이름 설정 (Name: free-springboot2)
* 보안그룹 (방화벽) 설정: 인바운드 규칙, 허용 IP 등을 설정
* 인스턴스 접근을 위한 pem(비밀키) 다운로드 
* IP/도메인 생성 확인 (IP는 시작시마다 변경이 되는 IP가 발급된다. EIP를 할당받아 고정시켜야 함)

#### EIP(Elastic IP, 탄력적IP) 할당

* ec2 instance > 카테고리: 탄력적IP > 새 주소 할당 > 작업>주소연결 
* 인스턴스 ID를 연결해 준다. 
* <u>EIP는 연결하지않으면 과금되니 조심</u>하자. 

### ssh 접속 (mac)

기본적인 접속 방법

```bash
ssh -i {pem키위치} {EC2-EIP주소}

# ~/.ssh 로 pem 복사해두면 ssh 실행시자동으로 pem파일을 읽는다. 
cp ...pem ~/.ssh
chmod 600 ~/.ssh/pem이름
vim ~/.ssh/config

```

~/.ssh/config

```bash
Host springboot-aws
  HostName 13.125.76.186
  User ec2-user
  IdentityFile ~/.ssh/iiofirstkeypair.pem
```

접속

```bash
chmod 700 ~/.ssh/config 
ssh springboot-aws
```



### Amazon Linux 1 서버 세팅

1. Java 8 설치

   ```
   sudo yum install -y java-1.8.0-openjdk-devel.x86_64
   sudo /usr/sbin/alternatives --config java
   --> 1.8로 선택
   sudo yum remove java-1.7xxx
   java -version 
   ```

   

2. Timezone 변경

   ```bash
   sudo rm /etc/localtime
   sudo ln -s /usr/share/zoneinfo/Asia/Seoul /etc/localtime
   date #확인
   ```

3. Hostname 설정

   ```bash
   sudo vim /etc/sysconfig/network 
   HOSTNAME=springboot-aws
   
   sudo reboot
   
   /etc/hosts 에 호스트명 등록
   127.0.0.1 springboot-aws
   
   curl springboot-aws
   -> 호스트명등록실패: Could not resolve host...
   -> 성공: Failed to Connect.. (서비스가 없으니까)
   
   ```



## AWS RDS (Database)

Relational Database Service에는 오라클, MSSQL, PostgreSQL, Aurora등이 있다. 

MariaDB는 MySQL의 오픈소스 버전으로 MySQL보다 향상된성능, 활성화된 커뮤니티, 기능과 스토리지 엔진이ㅡ 다양화 등의 장점이 있다. 

### 인스턴스 생성

1. 데이터베이스생성 > 표준생성 > MariaDB > 프리티어 > 스토리지: 20GB > DB명/유저명 설정
2. 퍼블릭 액세스: 예 (이후 보안그룹에서 IP지정) 
3. 파라미터 설정 후 > 데이터베이스 -> 수정 > 파라미터그룹 설정 및 선택(하단 참조) > 즉시적용
4. 재부팅

#### 파라미터 설정

카테고리 > 파라미터 그룹 > 생성 > 인스턴스버전 선택 > 편집

1. 타임존(time_zone) > Asia/Seoul 로 설정
2. Character Set > utf8mb4 (char..로 검색해서 모두 변경해 준다.)
3. max connection > 150 (프리티어에선 60개 가능하나 넉넉하게)



### 접속

RDS 보안그룹에 본인 PC IP를 추가한다. 

1. 데이터베이스 보안그룹 선택
2. 다른 창으로 **EC2에 사용된 보안그룹** 의 그룹ID를 복사 
3. 복사된 보안그룹ID(EC2)와 본인PC IP를 **RDS 보안그룹의 인바운드** 로 추가 (MYSQL/Aurora 선택하면 3306자동 선택됨)



##### IntelliJ Database Plugin 설치

디비 클라이언트는 Workbench, SQLyog, Sequal Pro, DataGrip 등이 있다. 여기선 IntelliJ Database Plugin으로 접속 시도한다. 

> Database Navigator > Install
>
> Action 검색(CMD+Shift+a) > Database Browser를 실행

AWS RDS 정보 > **엔드포인트** 를 확인 > 카피 (database명, user명 등도 기록해둔다.)

New Database: 

* Database Name: mysql (혹은 mariadb)
* User / pw : 설정한 마스터 id/pw 

현재 character_set, collation 확인하여 다르면 기본값 변경

```sql
show variables like 'c%'; //현재 character_set, collation 확인

//utf8mb4가 아니면 변경해준다.
alter database mysql
character set = 'utf8mb4'
collate = 'utf8mb4_general_ci';

# 타임존 확인
select @@time_zone, now();
```

table 생성테스트(한글)

```sql
create database springbootaws; #Database를 만든다. 
use springbootaws;

create table test (
	id bigint(20) not null auto_increment,
	content varchar(255) default null,
	primary key(id)
) engine=InnoDB;

insert into test(content) values('테스트');
select * from test;
```



## EC2에서 MySQL에 접근

```bash
# mysql client 설치
sudo yum install mysql 

# mysql -u {user} -p -h {hostname}
mysql -u icarus -p -h springboot-aws.cygftujktqll.ap-northeast-2.rds.amazonaws.com
#PW -> 계정비번과 같음
```



## AWS IAM Key 발급

* 외부 서비스 접근을 위해, 접근권한을 가진 key를 생성해 사용
* IAM(Identity and Access Management) : 서비스접근방식과 권한 관리
* Travis CI -> S3 and CodeDeploy에 접근시 IAM 키를 사용함. 

### 권한 사용자 발급

* IAM > 사용자 > 사용자추가 > "deploy용 사용자명", 프로그래밍방식 선택 > 기존정책직접연결 > s3full, CodeDeployFull 검색/체크 > Tag지정. Name: springboot-aws-travis-deploy 
* 생성하면, 액세스key ID/비밀 액세스 키 두개를 구할 수 있다. 이것을 TravisCI에 설정저장
* AWS _Access_key, AWS_Secret_key

### IAM 역할 생성

* EC2가 CodeDeploy를 연동 받을 수 있게 역할을 생성
* 역할: AWS서비스에만 할당가능 (EC2, CodeDeploy, SQS 등)
* 사용자: AWS외에 사용할 수 있는 권한 (로컬 PC, IDC서버 등)
* IAM > 역할 > 역할만들기 > AWS서비스->EC2 > 정책: EC2RoleForA 선택 > 태그 지정 > 생성
* EC2 > 인스턴스 우클릭: 인스턴스설정 -> IAM역할 연결/바꾸기 > "위의 role 선택" > EC2 재부팅 (해야 반영됨)

## AWS S3 버킷 생성

Simple Storage Service (S3) : 파일서버. 첨부파일 저장, 빌드파일 저장

S3 > 버킷 만들기 > "springboot-aws-build" > 다음:미설정 > 다음:모든 퍼블릭 차단 > 버킷만들기 