# 스프링 부트와 AWS로 혼자 구현하는 웹 서비스 - AWS 세팅

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
* AMI(Amazon Machine Image) - Amazon Linux 1 OS 선택 (CentOS 6 기반, 2는 7)
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
* EIP는 연결하지않으면 과금되니 조심하자. 

### ssh 접속 (mac)

기본적인 접속 방법

```
ssh -i {pem키위치} {EC2-EIP주소}

# ~/.ssh 로 pem 복사해두면 ssh 실행시자동으로 pem파일을 읽는다. 
cp ...pem ~/.ssh
chmod 600 ~/.ssh/pem이름
vim ~/.ssh/config

```

~/.ssh/config

```shell
Host springboot-aws
  HostName 13.125.76.186
  User ec2-user
  IdentityFile ~/.ssh/iiofirstkeypair.pem
```

chmod 700 ~/.ssh/config

접속

```
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

   ```
   sudo rm /etc/localtime
   sudo ln -s /usr/share/zoneinfo/Asia/Seoul /etc/localtime
   date (확인)
   ```

3. Hostname 설정

   ```
   sudo vim /etc/sysconfig/network 
   HOSTNAME=호스트이름설정
   
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

1. 표준생성 > MariaDB > 프리티어 > 스토리지: 20GB > DB명/유저명 설정
2. 퍼블릭 액세스: 예 (이후 보안그룹에서 IP지정) 
3. 파라미터 설정 후 > 데이터베이스 -> 수정 > 파라미터그룹을 아래설정한 그룹으로 선택 > 즉시적용
4. 재부팅

#### 파라미터 설정

카테고리 > 파라미터 그룹 > 인스턴스버전 선택 > 편집

1. 타임존(time_zone) > Asia/Seoul 로 설정
2. Character Set > utf8mb4 (char..로 검색해서 모두 변경해 준다.)
3. max connection > 150 (프리티어에선 60개 가능하나 넉넉하게)

### 접속

RDS 보안그룹에 본인 PC IP를 추가한다. 

1. 데이터베이스 보안그룹 선택
2. 다른 창으로 **EC2에 사용된 보안그룹** 의 그룹ID를 복사 
3. 복사된 보안그룹ID와 본인PC IP를 **RDS 보안그룹의 인바운드** 로 추가 (MYSQL/Aurora 선택하면 3306자동 선택됨)

##### IntelliJ Database Plugin

디비 클라이언트는 Workbench, SQLyog, Sequal Pro, DataGrip 등이 있다. 

RDS 정보 > **엔드포인트** 를 확인 > 카피 

> Database Navigator > Install
>
> Action 검색(CMD+Shift+a) > Database Browser를 실행

