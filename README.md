# 1. 이중화 - LBaaS 미사용 (Apache 사용)

## Apache & Tomcat 연동 이유
### 정적 콘텐츠 서비스 효율이 뛰어남
* 웹서버가 이미지 파일이나 동영상 등 정적 콘텐츠를 제공하는 데 더 성능이 뛰어남
* 특정상황(톰캣에 APR native와 sendFile 사용)에서는 톰캣이 정적 콘텐츠 처리도 더 빠른 경우도 있음
  
### 클러스터링
* L4 없이도 Apache 서버와 연계해서 사용하면 톰캣을 클러스터로 손쉽게 구성

### 모듈 기반의 확장성
* Apache 서버의 다양한 모듈로 서버의 기능 확장(mod_headers, mod_rewrite 등)
### 가상 호스트
* 하나의 서버에 여러 개의 가상 호스트를 설정해 가상 호스트마다 다른 웹 어플리케이션 서비스를 제공
* 예) app.example.com -> Tomcat / word.example.com -> PHP
### 보안
* 1024 이하의 포트는 루트 사용자만 사용해야 함 -> 보안상 문제
	* Apache 웹 서버는 루트로 구동해도 자신의 프로세스를 포크한 후 apache 실행 그룹과 계정으로 전환됨
* 웹 서버는 외부망과 내부망 사이 위치해야 하는데, 주로 DB 서버와 연동하는 톰캣을 웹 서버로 사용한다면 데이터베이스 서버로 바로 연결하는 것을 허용해야 하므로 데이터 유출될 수 있음
* SELinux 사용하여 아파치 웹 서버는 8080, 8009 포트에만 접근할 수 있어 웹 서버 공격으로 인한 피해 최소화

### 연계 방식
* mod_jk
	* URL, 콘텐츠 별로 유연한 매핑
	* 전용 바이너리 프로토콜이기에 mod_proxy보다 빠름
	* 톰캣의 AJP 커넥터와 연계
* mod_proxy
	* 별도 모듈 설치 필요없음
	* HTTP Reverse Proxy로 동작 -> HTTP를 제공하는 모든 WAS에 적용 가능
* mod_jk / mod_proxy 연동 후 8080 포트는 닫아야함
### SELinux 설정

* 톰캣의 커넥터가 사용하는 포트가 SELinux가 httpd에 허용한 포트인지 확인
* SELinux에서 허용된 포트가 아니면 차단되므로, 포트가 차단되었는지 확인
## 아키텍처


  ![](https://lh3.googleusercontent.com/Z-mbRYsywg3sKU5c5hSHHH6d6KThU2zloLW7FL4NiW1WF5ovuryDiRQ0OPpBcktOEQfeEokg--iU)

  

## Apache 설치

  
```
$mkdir /etc/httpd

$cd /etc/httpd

$yum install gcc gcc-c++ httpd-devel openssl-devel

$wget  [http://mirror.apache-kr.org//httpd/httpd-2.4.37.tar.gz](http://mirror.apache-kr.org//httpd/httpd-2.4.37.tar.gz)

$tar xzvf httpd-2.4.37.tar.gz

$cd httpd-2.4.37/srclib

$wget  [http://mirror.apache-kr.org/apr/apr-1.6.5.tar.gz](http://mirror.apache-kr.org//apr/apr-1.5.2.tar.gz)

$wget  [http://mirror.apache-kr.org/apr/apr-util-1.6.1.tar.gz](http://mirror.apache-kr.org/apr/apr-util-1.6.1.tar.gz)

$tar xvfz apr-1.6.5.tar.gz

$tar xvfz apr-util-1.6.1.tar.gz

$mv apr-1.6.5 apr

$mv apr-util-1.6.1 apr-util

$wget  [http://downloads.sourceforge.net/project/pcre/pcre/8.41/pcre-8.41.tar.bz2](http://downloads.sourceforge.net/project/pcre/pcre/8.41/pcre-8.41.tar.bz2)

$tar xvf pcre-8.41.tar.bz2

$cd /etc/httpd/httpd-2.4.37/srclib/pcre-8.41ma

$./configure

$make

$make install

  

$cd /etc/httpd-2.4.37

$yum install openssl-devel

$./configure -prefix=/etc/httpd -enable-so -enable-rewrite -enable-ssl -enable-mods-shared=all -enable-modules=shared -enable-mpms-shared=all -with-included-apr -enable-unique-id

$make

$make install
```
  

## Tomcat 설치 및 포트 설정

  
```
$wget  [http://apache.tt.co.kr/tomcat/tomcat-7/v7.0.91/bin/apache-tomcat-7.0.91.tar.gz](http://apache.tt.co.kr/tomcat/tomcat-7/v7.0.91/bin/apache-tomcat-7.0.91.tar.gz)

$tar xvfz apache-tomcat-7.0.91.tar.gz

$mv apache-tomcat-7.0.91 /etc/tomcat7-1

$tar xvfz apache-tomcat-7.0.91.tar.gz

$mv apache-tomcat-7.0.91 /etc/tomcat7-2
```
  

### #1 Tomcat

  
```
$vim /etc/tomcat7-1/conf/server.xml
```
  

* Shutdown 포트 설정
```html
<Server port="8105" shutdown="SHUTDOWN">
```

  

* LB 수신 포트 설정
```html
<Connector port="8109" protocol="AJP/1.3" redirectPort="8443"/>
```


* 세션 클러스터 설정

  

![](https://lh3.googleusercontent.com/CnrOpAngTFLa-zD0jjl1E61hkj0l7GE8RZO6cLsQuwQwAByCyQXDU7oPE-yFefLLGdipD1MqC9Jw)

* 이중화된 Tomcat 실행경로 설정  
```
$vim /etc/tomcat7-1/bin/catalina.sh
```
```
export CATALINA_HOME=DIR_OF_TOMCAT
export TOMCAT_HOME=DIR_OF_TOMCAT
export CATALINA_BASE=DIR_OF_TOMCAT
```  

  

* #2 Tomcat - 포트 번호 및 경로 설정 이외의 과정 동일

  
  

##  mod_jk 설치 및 연동

  

### Tomcat Connectors 설치

  
```
$wget  [http://apache.mirror.cdnetworks.com/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.46-src.tar.gz](http://apache.mirror.cdnetworks.com/tomcat/tomcat-connectors/jk/tomcat-connectors-1.2.46-src.tar.gz)

$tar xvfz tomcat-connectors-1.2.46-src.tar.gz

$cd tomcat-connectors-1.2.46-src/native

$./configure --with-apxs=/usr/sbin/apxs

$make

$make install
```
  

* mod_jk와 httpd.conf를 같은 경로에 위치시키기
```
$mv /root/tomcat-connectors-1.2.41-src/native/apache-2.0/mod_jk.so /etc/httpd/conf/mod_jk.so
```
  


### Apache 설정

  
```
$vi /etc/httpd/conf/httpd.conf
```
  


  
```
LoadModule jk_module /etc/httpd/conf/mod_jk.so

JkWorkersFile /etc/httpd/conf/workers.properties

JkLogFile /etc/httpd/logs/mod_jk.log

JkShmFile /etc/httpd/logs/mod_jk.shm

JkMount /* worker1

JkMountFile /etc/httpd/conf/uriworkermap.properties
```
  
  

## Tomcat Worker 설정

  


### workers.properties 설정

  
```
$vi /etc/httpd/conf/workers.properties
```


```
worker.list=balancer

  

worker.worker1.type=ajp13

worker.worker1.host=localhost

worker.worker1.port=8109 # 포트번호

worker.worker1.lbfactor=2 # 서버 밸런스 비율

  

worker.worker2.type=ajp13

worker.worker2.host=localhost

worker.worker2.port=8209 # 포트번호

worker.worker2.lbfactor=1 # 서버 밸런스 비율

  

worker.balancer.type=lb

worker.balancer.balance_workers=worker1,worker2

worker.balancer.sticky_session=TRUE
```
  
  


##  mod_ssl 설치 및 설정

  
  
```
$yum install mod_ssl
```
  


###  인증서 생성 및 conf 파일 생성

  
  
```
$openssl genrsa -out ca.key 1024

$openssl req -new -key ca.key -out ca.csr

$openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
```
  
  
```
$vim /etc/httpd/conf/ssl.conf
```
  
  
  
```
SSLCertificateFile /etc/pki/tls/certs/ca.crt

SSLCertificateKeyFile /etc/pki/tls/private/ca.key
```
  


### httpd.conf 파일 수정

```
<VirtualHost *:443>
	SSLEngine on
	SSLCertificateFile /etc/pki/tls/certs/ca.crt
	SSLCertificateKeyFile /etc/pki/tls/private/ca.key
	ServerAdmin USER_ACCOUNT
	DocumentRoot /var/www/html
	ServerName DNS_NAME
	ErrorLog logs/ssl_error_log
	CustomLog logs/ssl_log common
</VirtualHost>
```

  
  


## MySQL 설치

  
  
```
$rpm -ivh  [https://dev.mysql.com/get/mysql57-community-release-el6-11.noarch.rpm](https://dev.mysql.com/get/mysql57-community-release-el6-11.noarch.rpm)

$yum install mysql-community-server

$service mysqld start
```
  


### 초기 root 계정 비밀번호 설정

  
  
```
$grep 'temporary password' /var/log/mysqld.log  
$mysql -u root -p

$alter user 'root'@'localhost' identified by '';
```
  


### 포트 번호 및 Character Set 변경, buffer pool size, default storage engine 설정

  
  
```
$vim /etc/my.cnf



port=13306

character-set-server = utf8

innodb_buffer_pool_size=2048M

default-storage-engine=InnoDB
```
  
  


### ※MyISAM과 InnoDB의 차이점

  

* MyISAM

  

	* MyISAM은 데이터 모델 디자인이 단순하고, 속도가 빠름(특히 Select 구문) / Full-text Indexing 가능

	* 데이터 무결성에 대한 보장이 없음, 트랜잭션에 대한 지원이 없음

	* Table-level lock을 사용하여 쓰기 작업의 속도가 느림

	* 조회작업이 많은 DB일 경우 주로 사용됨

	  


* InnoDB
	
	* 트랜잭션 지원, 데이터 무결성 보장

	* Commit, Rollback, 장애복구, row-level locking, 외래키 등 지원

	* 변경 작업(Insert, Update, Delete)에 대한 속도가 빠름

	* 시스템 자원 소모가 크고, Full-Text Indexing 불가

	* 높은 퍼포먼스를 유지해야하는 서비스에 적합

  


### Database Lock

  

* 공유하고 있는 리소스의 접근 경쟁을 제어

  

* Table Level Lock 

	
	* 테이블 자원에 엑세스하여 데이터를 읽거나 쓸 때 다른 세션에서는 테이블 자원에 대한 엑세스를 제한(Lock이 걸린 테이블의 ALTER, DROP 방지)

	* MyISAM에서 사용하는 락 단위

  


* Row Level Lock

	* 특정 쿼리 구문에 대하여 다른 유저들이 변경할 수 없도록

	* InnoDB에서 사용하는 락 단위


## MySQL 이중화

  


### Master Server 설정

  
```
$vim /etc/my.cnf

log-bin=mysql-bin

server-id=1  // Master DB의 server-id가 Slave DB의 server-id보다 높아야함
```
 



  


### MySQL 접속 후 Replication User 생성


  
```
CREATE USER 'irteam'@'%' identified by '';
GRANT REPLICATION SLAVE ON *.* TO 'irteam'@'ip_address';
SHOW MASTER STATUS;
```
  
  


###  Slave Server 설정

  
```
$vim /etc/my.cnf

  

log-bin=mysql-bin

server-id=2
```



### MySQL 접속 후 연동 설정

```
change master to
	master_host='MASTER_IP',
	master_user='MASTER_USER'
	master_password='',
	master_log_file='mysql-bin from show master status;'
	master_log_pos=number from show master status;
	master_connect_retry=number of retries;
```
 
 
* 설정 후 SLAVE 시작 - START SLAVE

  



## Master-Slave 상태 확인

  


### Master에서 Slave의 접속 상태 확인

```
$show slave hosts; 
```
 

![](https://lh3.googleusercontent.com/qe0msZEkQfpSeoT9ca2SeBy-3KXbYFv0lTKCO9RMSh2BmD50AC4u1LVKdrooK0m112UBrvCrwYK0)
  

### Slave에서 접속 상태 확인


```
$show slave status \G;
```
![](https://lh3.googleusercontent.com/41Iydb5nHpWNL7KVmSzpuu5YOUwNOujzGO2foSq1cvJQ-sJV7qpiRsfNX9bKgTQPx4PEuQhNquNA)


## Java Servlet 페이지 작성 및 MySQL 테이블 출력

  
  

### MySQL Connector를 각 톰캣에 설치

  
  
```
$wget  [https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz](https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz)

$tar xzvf mysql-connector-java.5.1.47.tar.gz

$cd mysql-connector-java.5.1.47

$cp mysql-connector-java.5.1.47-bin.jar /etc/tomcat7-1/lib

$cp mysql-connector-java.5.1.47-bin.jar /etc/tomcat7-2/lib
```
  


### MySQL DB 테이블 생성 및 데이터 입력

  
  
```
>create table member (idx not null auto_increment primary key, name char(20), team char(20));
```



### Java Servlet 코드 작성 및 컴파일


  
```
$vim /etc/tomcat7-1/webapps/ROOT/WEB-INF/classes/textConnection.java

$javac -classpath /etc/jdk1.8.0_131/jre/lib/ext/servlet-api.jar textConnection.java
```


### #2 톰캣에 복사

  
```
$cp -r /etc/tomcat7-1/webapps/ROOT/WEB-INF/classes /etc/tomcat7-2/webapps/ROOT/WEB-INF
```
  

### web.xml 상에서 Servlet Mapping

  
```
$vim /etc/tomcat7-1/webapps/ROOT/WEB-INF/web.xml
```

![enter image description here](https://lh3.googleusercontent.com/yOgLHzaUYALfJB5NhHYh2mdHRZ-b-v0OasBjj0cOGjiW8xIjiDXTgHWIBYpSTFxu_3XAr5g0gwU0)
  

## Session Clustering 설정 및 Test

```
$vim /etc/tomcat7-1/webapps/ROOT/WEB-INF/web.xml

$vim /etc/tomcat7-2/webapps/ROOT/WEB-INF/web.xml
```
  

* web-app 태그 내에 <distributable/> 문구 각각 추가


* #1 Worker Shutdown 후 #2 Worker의 Session ID값 확인 시 동일함을 알 수 있음

  
  

# 2. 이중화 - LBaaS 사용

## 아키텍처

  ![](https://lh3.googleusercontent.com/BpPMVFuXQqJfZn6eKkCqfFy5Fzq82HV4R_o0ZMK8Ozck-AeCkVMMHqaULfUHDnVknY-ijUy23UVk)
  

* Master DB가 동작하지 않을때 DB 변경작업을 수행할 수 없음.

  

* 두 대의 Master DB가 Active-Standby로 운영되고, Active 상태의 Master DB가 다른 Slave DB와 Master-Slave 관계 형성.

  

* Active 상태의 Master DB가 동작하지 않을 때 Standby 상태의 Master DB가 Active되고, Slave DB와 Master-Slave 관계 형성.

  

* MySQL MMM으로 구현하였고, 2대의 Master DB와 Slave DB 상태를 MMM Monitor 인스턴스로 관찰하였다.

  

## nGrinder를 활용한 부하분산 확인

  

* vUser 수 5000

  

### Load Balancer에 2개의 Apache가 연결되어 있을 때

  
![enter image description here](https://lh3.googleusercontent.com/knt_F7SNpJAFd5yvfzIJQJu2iurr-XHLCqFbKgx759ZScJpdDTlpFyK4za78wCSRT70GF5-hXicZ)
  

### Load Balancer에 1개의 Apache를 제거하였을 때

  
![enter image description here](https://lh3.googleusercontent.com/epW3mLCnEIdVGIANObmblZ_EhUl4mq1A7W5reLnlmYVHyvTdhtm58bZ2macm60t6ctwZpipz8rBI)

  

* 하나의 Apache를 Load Balancer에서 제거하였을 때
	- TPS 변화폭 급증

	- 특정 대역에서 오류가 급증하는 현상 발생

  

  

## #1 Apache 설정

  
```
$vim /etc/httpd/conf/workers.properties
```
  

```
worker.list=worker1

worker.worker1.type=ajp13

worker.worker1.host=#1 Tomcat IP 주소

worker.worker1.port=8109
```
  
```
$vim /etc/httpd/conf/httpd.conf
```
  

```
LoadModule jk_module /etc/httpd/mod_jk.so

JkWorkersFile /etc/httpd/conf/workers.properties

JkLogFile /etc/httpd/logs/mod_jk.log

JkShmFile /etc/httpd/logs/mod_jk.shm

JkMount /* worker1
```
  

### mod_ssl 설치 및 설정

  
```
$yum install mod_ssl
```
  

### 인증서 생성 및 conf 파일 생성

  
```
$openssl genrsa -out ca.key 1024

$openssl req -new -key ca.key -out ca.csr

$openssl x509 -req -days 365 -in ca.csr -signkey ca.key -out ca.crt
```
  
```
$vim /etc/httpd/conf/ssl.conf
```
  
  
```
SSLCertificateFile /etc/pki/tls/certs/ca.crt

SSLCertificateKeyFile /etc/pki/tls/private/ca.key
```
  

### httpd.conf 파일 수정
```
<VirtualHost *:443>
	SSLEngine on
	SSLCertificateFile /etc/pki/tls/certs/ca.crt
	SSLCertificateKeyFile /etc/pki/tls/private/ca.key
	ServerAdmin USER
	DocumentRoot /var/www/html
	ServerName SERVER_NAME
	ErrorLog logs/ssl_error_log
	CustomLog logs/error_log common
</VirtualHost>
```

  

## #1 Tomcat Worker 설정

  
```
$vim /etc/tomcat7-1/conf/server.xml
```
  

* Shutdown 포트 설정
```html
<Server port="8105" shutdown="SHUTDOWN">
```

  

* LB 수신 포트 설정

```html
<Connector port="8109" protocol"AJP/1.3" redirectPort="8443" />
```  

  

* 세션 클러스터 설정 - LBaaS 사용 전과 다르게 Receiver의 IP주소를 톰캣의 IP주소로 할당

  
![](https://lh3.googleusercontent.com/z7opWW_hndBABXmjxdQdvc6jqePhE2DQuWjUTwQ36-sxonGZgndhXmJ5tNbNZW2vHNi00Bjpn80D)

```
$vim /etc/tomcat7-1/bin/catalina.sh

export CATALINA_HOME=TOMCAT_FOLDER
export TOMCAT_HOME=TOMCAT_FOLDER
export CATALINA_BASE=TOMCAT_FOLDER
```


## #2 Apache 설정

  

* #1 Apache 설정 과정과 동일

  

## #2 Tomcat Worker 설정

  

* #1 Tomcat Worker 설정 과정과 동일

  

## Load Balancer 생성 후 #1 Apache와 #2 Apache 연결

  

## MySQL 이중화(Dual Master Replication, MMM 활용)

  

* 서비스 운영에 적합한 이중화 구조

* Master Server의 Health Check, 자동으로 Failover 수행

* Master DB의 장애 발생 시 Master 간의 Replication 깨지지 않음

  

### Insert Query 수행 시 Master Active & Master Slave 동기화

  

* Master Active - Query를 Binary Log에 기록 -> 실행

* Master Slave - Binary Log 감시 -> 새로운 Log 인지 -> Slave의 Relay Log로 Import -> 실행

  

* MHA와는 다르게 Failover 대상이 Master Active / Master Slave로 정해져 있음

  

### Failover 과정

  

* Master Active에 장애 발생 시 Master Active를 읽기 모드로 변경, Session Kill, VIP(Write 권한) 회수

* Master Standby에서 복제 지연 확인, 복제 재구성(Master Standby와 Slave를 Master-Slave 관계로 형성), VIP(Write 권한) 할당

  

### MMM의 단점

  

* Multi Slave로 구성되어 있을 시 Replication Crash 가능성 존재

* Insert Query가 Bin Log에 저장 -> Query 전달 후 응답을 기다림.

  

* Master Standby만 응답을 전송할 경우 MMM Monitor에서 Failover 진행

* MMM은 Failover 대상이 정해져있기 때문에 Master Standby가 Master Active 역할 수행(Insert Query 미수행 상태)

  

* 새로운 Master Active는 유실된 데이터를 복구하기 위해 Insert Query를 수행하고, 이를 Slave에 전달

* Slave에는 이미 Insert되어있었기 때문에 중복키 오류로 인해 복제가 깨짐.

  

### MySQL 설치

  
```
$rpm -ivh  [https://dev.mysql.com/get/mysql57-community-release-el6-11.noarch.rpm](https://dev.mysql.com/get/mysql57-community-release-el6-11.noarch.rpm)

$yum install mysql-community-server

$service mysqld start
```
  

### 서버 구성

  
|  | IP | hostname | server-id |
|--|--|--|--|
| Monitoring Server | 192.168.0.64 |mon  |  |
| #1 Master | 192.168.0.65 | db1 | 1 |
| #2 Master | 192.168.0.71 | db2 | 2 |
| Slave | 192.168.0.63 | db3 | 3 |  

### Virtual IP 구성

| IP | R/W |
|--|--|
| 192.168.0.100 | Write, Active상태인 master에 할당 |
| 192.168.0.101 | Read |
| 192.168.0.102 | Read |
| 192.168.0.103 | Read |

### MySQL Server 설정

  

* Master Server 설정
	* server-id가 중복되지 않게 할당
  
```
$vim /etc/my.cnf
```
	
* Slave Server 설정

  
![](https://lh3.googleusercontent.com/bn5m3kdy8RLbspGN2qbk12Ct7OYGDiE01cHoxq6V1nhQv-X-LbNqtezoDyGvVOZeLapIxOy8wmdU)
  

### MySQL user 생성

  
| userID | Privileges | IP | 비고 |
|--|--|--|--|
| mmm_monitor | replication client | 192.168.0.% | 각 DB 서버의 상태 확인 |
| irteam | super, replication client process | 192.168.0.% | DB 버의 Agent 계정 |


### #1 Master, #2 Master 간의 Dual Replication 및 Master-Slave 설정 및 작동상태 확인

  

*  Master가 #2 Master를 바라보도록 설정

  
```
change master to master_host='192.168.0.65', master_user='irteam', master_port=13306, master_password='Goodboy12!', master_log_file='mysql-bin.000012', master_log_pos=1746;

>start slave;
```
  

  

* Master가 #1 Master를 바라보도록 설정

  
```
change master to master_host='192.168.0.71', master_user='irteam', master_port=13306, master_password='Goodboy12!', master_log_file='mysql-bin.000013', master_log_pos=1039;

>start slave;
```
  


* Active Master(#1)와 Master-Slave 관계 형성

  
```
change master to master_host='192.168.0.71', master_user='irteam', master_port=13306, master_password='Goodboy12!', master_log_file='mysql-bin.000013', master_log_pos=1039;

>start slave;
```
  
  

### MMM 설치 및 설정

  

* 3개의  DB 서버 및 모니터링 서버에 공통적으로 설치

  
```
$yum install mysql-mmm
```
  

* 3개의 DB 서버 및 모니터링 서버에 공통 설정

  
```
$vim /etc/mysql-mmm/mmm_common.conf
```
  
```
active_master_role writer

  

<host default>

cluster_interface eth0

  

pid_path /var/run/mmmd_agent.pid

bin_path /usr/lib/mysql-mmm/

  

mysql_port 13306

replication_user irteam

replication_password

  

agent_user mmm_agent

agent_password

</host>

  

<host db1>

ip 192.168.0.65

mode master

peer db2

</host>

  

<host db2>

ip 192.168.0.71

mode master

peer db1

</host>

  

<host db3>

ip 192.168.0.63

mode slave

</host>

  

<role writer>

hosts db1, db2

ips 192.168.0.100

mode exclusive

</role>

  

<role reader>

hosts db1, db2, db3

ips 192.168.0.101, 192.168.0.102, 192.168.0.103

mode balanced

</role>
```
  

* Monitoring Agent 설정

  
```
$vim /etc/mysql-mmm/mmm_common.conf
```
  
![](https://lh3.googleusercontent.com/8KxndumLhiPgbqkvLhA5cGwV12l9mLzAjxb39Jy1gTnmOoz2CawxusdMSAhPd1TdM9hTAdir0N9I)

  

### MMM 시작 및 확인

  

* DB 서버에 이미 MMM Agent들이 실행되고 있어서, 프로세스들을 종료시킨 후 재시작.

* MMM Monitor에서 DB 서버 상태 확인
  

  ![](https://lh3.googleusercontent.com/qo0xOba5R8JNB8I1wVW9nGFm1lkXSuZnsInvL2fO_nH46wNEqxFxhHpJE7Zjq9Y5xMj1oZX1Ok7j)

* 이미 한 번의 Failover를 수행한 상태이므로, db2가 writer 권한을 갖고있는 Active Master가 됨

* db2 인스턴스를 강제종료(장애 상황) 후 MMM Monitor를 통한 상태 확인

  ![](https://lh3.googleusercontent.com/FjZ-SwAdcUN4HEo44RbXYt2q8-JMQuyU6wjN-WKu15LK6wY8tDuu5ebNmd4QSQKV-z2P0jyGkvis)
  

* db2에 있던 writer 권한이 db1으로 넘어감

* db2 인스턴스 재시작 후 MMM Monitor를 통한 상태 확인
![](https://lh3.googleusercontent.com/CktZRdiSxWF-xUD88qUdvBpl9e1-OrogzPGF2mA5Z0pAsjkTUkL-mWzYXjzqnBG9YA6LaxfXjEUh)
![](https://lh3.googleusercontent.com/8N_TJm8fdFSjLa-kTTszHSFBoir3KILziLXxJa78BVo33gR5aAgPeNBcQPDMiOBJ2UCoxRkCUyVN)
### MMM 구성의 한계

  

* DHCP 방식을 채택하고 있을 시 Virtual IP를 할당할 수 없음.

 
### DHCP(Dynamic Host Configuration Protocol)

  

* IP를 필요로하는 인스턴스에 할당해주고, 사용하지 않으면 반환받아 다른 인스턴스가 사용할 수 있게 함.

#### DHCP 임대

* DHCP Discover

  

	* IP 주소를 할당받기 위해 클라이언트가 호스의 Mac 주소를 기반으로 네트워크의 각 호스트에 Discover 패킷을 Broadcast

	* 호스트 중 자신의 Mac 주소와 일치하는 호스트가 Discover 패킷을 받음

  

* DHCP Offer

  

	* Discover 패킷을 받은 DHCP 서버가 Discover 패킷을 보낸 호스트의 Mac 주소를 담은 Offer 패킷을 Broadcast

  

* DHCP Request

  

	* Request 패킷을 Broadcast하여 IP를 할당받을것을 요구

  

* DHCP ACK

  

	* DHCP 서버 내에서 할당 가능한 IP 주소를 찾아 Broadcast하여 클라이언트에 전달

  

#### DHCP 갱신

  

* 임대 기간 도래 전 임대 기간을 연장

#### DHCP 반환

  

* 더이상 사용하지 않는 IP를 반환

  

### MMM 운영 중 장애 시

  

* SE, DBA에서 장애 인지 → 개발 부서에 공유 → DB Connection을 Slave DB 로 바꿈

* 결국, 수동으로 배포가 필요

  

* MMM-Monitor에서 각 DB Agent들의 상태를 관찰할 수 있으므로, 로그 모니터링 
*  (mmm_mond.log)을 통해 DB Connection을 형성하고 있던 Master Active의 상태가  **Online이 아닐 시**

* Master Standby로 DB Connection을 형성할 수 있는 버전을 배포

  

### 기타 DB 이중화 방안

  

#### MHA

  
![](https://lh3.googleusercontent.com/Ene-EaHYouCaom2OgKBARtnC5JzRPtU6zML68kwwSz8hnggBg0zntw7aWEOlZuocoO1T1gng4MHK)

  

* MMM과는 달리 별도의 Agent를 설치할 필요 없음. MHA Monitor에서 각 DB Agent로 명령을 날리는 형식

* Master와 Slave가 단방향 복제로 구성

  

* Master DB에서 장애 발생 시 MHA Monitor에서 Master와 Slave간의 복제를 끊음

* 나머지 DB 서버들로 복제를 재구성

* 장애가 발생한 DB 서버가 정상 가동되었을 때 복제를 재구성하는 작업 필요

  

* Failover 대상이 고정되어 있지 않음. Failover 시 각 Agent의 DB 상태를 확인하여 신규 Master로 승격.

* Binary Log와 Relay Log를 비교하여 차이나는 데이터 추출, Agent에 데이터 업데이트

  

* Application이 Virtual IP를 바라보고 있어야 하기 때문에 구축 불가.

  

#### MHA + DNS 방식

  

* Virtual IP를 할당할 수 없기 때문에 DNS 구축을 고려.

* DB Connection을 IP 주소에서 도메인 주소로 변경.

* Caching으로 인한 지연 소요

  

#### PowerDNS

  

* 오픈소스 기반 구성, MySQL Database에서 도메인 관리

  

* 설치

  
```
$yum -y install pdns pdns-backend-mysql
```
  

* MySQL 계정 및 스키마 생성

  
```
mysql> mysql - u root -p

mysql> create database pdns_production;

mysql> grant all on pdns_production.* to 'pdns'@'IP_addr' identified by '';
```
  

* domain, record, supermaster, cryptokey, tsigkey 등의 테이블을 생성한 후 도메인과 IP 주소를 일일이 입력해야 함.

* Supermaster : 특정 도메인의 Master-Slave 관계에서 Master 역할 수행

* cryptokey : API 호출을 통해 DNSSEC(공개키 암호화방식 + DNS에 도입) key를 변경하는 역할 수행

  

* 각 테이블의 속성에 대한 지식 필요.

* 내부망에 국한된 도메인 관리이기 때문에 부적합 판단

  

#### 외부 DNS 서버 활용

  

#### 기타 HW 이중화 방식

  

* 평상시 Standby 서버를 사용할 수 없고, Failover 용도로만 사용.

* Disk 각각의 백업 서버가 필요.

* OS와 HW에 대한 지식 필요 - 유지보수 및 장애 대응의 어려움

  

#### Shared Disk 방식

  

* Master Active와 Standby가 하나의 Disk를 공유

* Master Active가 MySQL DB를 띄우고 VIP로 서비스 중 장애가 발생 시 Master Standby로 Disk 활성화, VIP로 서비스.

* Shared Disk와 RHCS 솔루션 필요하므로 비용 부담

  

#### Disk Replication 방식

  

* Shared Disk 보다 비용 절감

* Master Active와 Standby가 각각의 Disk를 바라보고 있음

* Master Active에서 일어나는 Transaction 데이터를 Network를 통해 Master Standby로 공유

* Network Latency로 인한 성능 문제

  

#### MySQL Clustering

  

* **VIP를 할당할 수 없기에**  MMM, MHA로 DB 이중화를 구성할 수 없음.

* Master, Slave로 Replication을 구성할 시 Read Agent와 Write Agent 분리 필요

* 데이터 복제가 비동기 방식으로 진행. Slave에 변경 사항이 반영될 때까지 일정 시간 소요 → 순간의 데이터 불일치 발생 가능성

  

* Clustering 구성 방법 중 오픈소스로 구현된 Galera Cluster가 있음

* Active-Active 방식, 모든 DB 노드에서 Read/Write 가능 (특정 Master의 장애발생 시 운영 차질 방지)

  

* Galera Cluster 구조

  

![](https://t1.daumcdn.net/cfile/tistory/21521734564C8FF535)

  

* 특정 노드에서 Insert나 Update 등의 쿼리가 발생 시 WSREP, Garela Replication Module이 다른 MySQL Node로 데이터를 복제

  

* WSREP Module : DB Node의 변화나 Transaction을 감지

* Garela Replication Module : WSREP API를 실행, 다른 DB Node로 복제

  

* 데이터 복제 프로세스

  

![](https://t1.daumcdn.net/cfile/tistory/266B6234564C8FDF1F)

  

* Node1(Master)에서 트렌젝션 발생 후 Commit 실행 → Node2(Slave)에 복제 요청 → Node2(Slave)에서 OK 신호를 Node1(Master)로 전송한 뒤 복제 수행

  

* 성능

  

	* Master 측에서 복제 요청을 보내고, 요청에 대한 OK 신호를 전송해야하기 때문에 Replication 구조보다 느림

  

* 단점

  

	* 장애 전파

  

		* 장애를 다른 DB 노드로 전파시킬 가능성 존재.

		* 복제를 받는 DB 노드에서 장애가 발생하여 복제 요청에 대한 응답을 하지 못할 시 From DB 노드에서 대기 현상 발생

  

	* 스케일링의 한계

  

		* 모든 노드에 데이터를 복제하고 트렌젝션을 끝내기 때문에, 전체 DB 노드 수가 증가할 수록 복제 시간 소모 증가

 

### Session Clustering 설정 및 Test

  

### #1 Worker의 Session ID값 공유

  

LBaaS를 사용하지 않았을 때와 다르게 Tomcat Worker를 하나씩 종료하고 redirect 시 잠깐의 Time Interval 발생

 

## 3. 빌드 및 배포

* TOAST의 Deploy를 활용한 빌드 및 배포 진행

  

* Java Servlet을 #1 Tomcat, #2 Tomcat에 배포 및 빌드

  

### 배포 시나리오 구성

  

### 버전 관리

  

* Annotated Tag

  

	* 태그를 만든 사람의 이름, 이메일, 만든 날짜, 태그 메시지, 서명정보 포함

  

* Lightweight Tag

  

	* 특정 커밋에 대한 포인터

  

* Git의 Lightweight Tag를 활용하여 특정 Servlet Page 별 버전 관리

  
  

* Shell Script 상에서의 Rollback

  

	* Git Clone 수행 시 가장 최신의 버전(v2.0)이 업로드됨.

	* 해당 폴더에서 이전 버전으로 회귀

  

		* $git checkout v1.0

## 4. 모니터링

### 모니터링 도구 비교

  

* 오픈소스 기반의 Pinpoint, Scouter, Zabbix를 비교

  

* Pinpoint는 복수의 JDK 설치와 환경설정의 복잡성 / Zabbix는 PHP, mariaDB 등의 설치 필요

  

* Scouter는 기구성된 톰캣과 JDK만으로 구현이 가능. APM 설치를 위해 다른 부차적인 것들이 필요하지 않음.

  

### 모니터링 지표

  

* Transaction이 자주 발생하는 서비스인지, 조회가 자주 발생하는 서비스인지에 따라 Instance, Apache, Tomcat, DB를 모니터링하는 지표 또한 다름

  

* 본 과제는 Transaction은 발생하지 않고 조회 Query가 자주 발생하는 서비스

  

### Scouter 설치(Agent, Collector)

  
```
$wget  [https://github.com/scouter-project/scouter/releases/download/v2.0.1/scouter-all-2.0.1.tar.gz](https://github.com/scouter-project/scouter/releases/download/v2.0.1/scouter-all-2.0.1.tar.gz)

$ tar xvzf  [scouter-all-2.0.1.tar.gz](https://github.com/scouter-project/scouter/releases/download/v2.0.1/scouter-all-2.0.1.tar.gz)
```
  

### Scouter 설치(Client)

  

* 윈도우 상에 zip 파일 다운로드 후 압축 해제

  

### LBaaS 미사용 인스턴스 모니터링

  

### Host Agent 설정

  
```
$vim /etc/scouter/agent.host/conf/scouter.conf
```
```  
net_collector_ip=Public_IP_Addr

net_collector_udp_port={PORT_NUMBER}

net_collector_tcp_port={PORT_NUMBER}
```
  

### Tomcat 설정

  

```
$vim /etc/tomcat7-1/conf/scouter.conf

  

  

net_collector_ip=192.168.0.70 # collector ip주소
```
  

### Scouter Server Port (Default : 6100)

* net_collector_udp_port={PORT_NUMBER}

* net_collector_tcp_port={PORT_NUMBER}

  

### Scouter Name(Default : tomcat1)

* obj_name=Tomcat1

* trace_interservice_enabled=true

  

* profile_http_parameter_enabled=true

* profile_http_header_enabled=true

  

* 각각의 Tomcat lib 폴더에 scout.agent.jar 파일 복사

  
```
$vim /etc/tomcat7-1/bin/catalina.sh

export AGENT_HOME="/etc/scouter/scouter/agent.java"

  

if [ "$1" = "start" -o "$1" = "run" ]; then

export JAVA_OPTS="$JAVA_OPTS -javaagent:$CATALINA_HOME/lib/scouter.agent.jar"

export JAVA_OPTS="$JAVA_OPTS -Dscouter.config=$CATALINA_HOME/conf/scouter.conf"

fi
```
  

### Server 설정

  
```
$vim /etc/scouter/server/conf/scouter.conf

db_dir=/etc/scouter/server/database

log_dir=/etc/scouter/server/logs

net_udp_listen_port={PORT_NUMBER}

net_tcp_listen_port={PORT_NUMBER}
```
  

### Scouter와 Telegraf 연동

  

* 목적

  

	* 기본 Scouter 설정 상에서 확인할 수 없는 Apache와 MySQL 모니터링 구현

	* MySQL과 Apache의 모니터링 데이터 수집

  

* 문제점

  

	* 단순히 Scouter와 Telegraf를 연동하는 것이 아니라, Telegraf에서 모니터링 데이터를 수집하기 위해 InfluxDB로 데이터 저장 필요

  

### Tomcat, Host, Server 실행 후 Client에서 로그인

  

* Instance

    ![](https://lh3.googleusercontent.com/I4wGL52iTrE5B7Yal8ujQLg99F4oGMj_Qp7Vz6ryU4u1VdpPGpR7QdW4zWTRa4FMNJuZJCm1hNxP)

* Apache

  

	* Telegraf와 연동하여 Apache를 모니터링함

  

* Tomcat

  ![](https://lh3.googleusercontent.com/fmbrnWN33fU8Ddhi39gjCE_iKl-d5mvAKVQzAgX11b3RUH36SKqO5uv-VJcIso4hRf-KhO_DcDW0)
  

