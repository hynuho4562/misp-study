### misp-study에 대해 공부해봅시다!
##### 이 정리한 글을 보면 MISP를 올바르게 설치할 수 있을 것입니다
<hr>

#### 깔기 위해서 3가지 조건들이 필요합니다
1. RedHat 로그인: [https://www.redhat.com/wapps/ugc/register.html?_flowId=register-flow&_flowExecutionKey=e1s1]
2. 기본 유닉스 CLI
3. 운영 체제: 레드햇 엔터프라이즈 리눅스 7.9
<hr>

레드햇 엔터프라이즈 리눅스 7.9 빌드에서 컴파일 도구와 Git 설치를 해줍니다<br>
``` sudo yum install git gcc bcc gcc-c++ cmake -y ```<br><br>

GitHub에서 설치 리포지토리 복제<br>
``` git clone https://github.com/MISP/MISP-RPM.git cd MISP-RPM ```<br><br>

여기까지 하셨다면 3가지 진행을 준비하여야 합니다
1. 엔터프라이즈 등록
2. 필요한 리포지토리를 포함하도록 yum을 업데이트해야 합니다.
3. 몇가지 패키지를 설치해야 합니다.

``` make prepare ```

만약 실패했다면 수동으로 실행할 수도 있습니다.<br><br>
이렇게 하면 올바른 리포지토리가 다음 단계에서 필요한 모든 rpm 패키지를 다운로드 할 수 있습니다!<br>

```
sudo subscription-manager register --auto-attach
sudo yum-config-manager --enable rhel-server-rhscl-7-rpms
sudo subscription-manager repos --enable rhel-7-server-optional-rpms
sudo yum install -y scl-utils ca-certificates git
```
<hr>

yum에서 올바른 리포지토리를 활성화했는지 확인해야 합니다.<br><br>

구성 요소를 설치하고 rpm를 빌드합니다.<br><br>
RPMS/ 결과는 디렉토리에서 찾을 수 있습니다.

``` make misp.rpm ```<br><br>

모든 빌드 및 전제 조건 패키지로 디렉토리 릴리스 생성

``` make release ```

<hr>

(선택사항) 활성화되지 않은 경우 RHEL 구독 및 소프트웨어 컬렉션을 활성화해야 합니다.

```
sudo subscription-manager register --auto-attach
sudo yum install yum-utils -y
sudo yum-config-manager --enable rhel-server-rhscl-7-rpms
sudo subscription-manager repos --enable rhel-7-server-optional-rpms
```

<br>

(필수) 아파치 웹 서버

```
sudo yum install -y httpd24 httpd24-mod_ssl
sudo systemctl enable httpd24-httpd
sudo systemctl restart httpd24-httpd
```

<br>

(필수) PHP 설치

```
sudo yum install -y rh-php73
```

<br>

PHP 런타임에 대한 권장 최댓 값 설정 ``` /etc/opt/rh/rh-php73/php.ini ```

```
- max_execution_time = 30 
+ max_execution_time = 300 
- memory_limit = 128M 
+ memory_limit = 2048M 
- post_max_size = 8M 
+ post_max_size = 50M 
- upload_max_filesize = 2M 
+ upload_max_filesize = 50M
```

<br>

다음 순서대로 패키지를 설치하고 필요한 경우 버전을 적용한다.

```
cd release/
```

MISP의 전제 조건

```
sudo yum install -y gtcaca-1.0+*.rpm libcaca*.rpm imlib2*.rpm
sudo yum install -y faup-1.6.0+*.rpm
sudo yum install -y ssdeep-libs*.rpm
```

<br>

MISP를 위한 사전 요구 사항 PHP 확장

```
export mispver="2.4.159–1"
sudo yum install -y misp-pear-crypt-gpg-$mispver.el7.x86_64.rpm \
misp-pecl-rdkafka-$mispver.el7.x86_64.rpm \
misp-pecl-redis-$mispver.el7.x86_64.rpm \
misp-pecl-ssdeep-$mispver.el7.x86_64.rpm \
misp-php-brotli-$mispver.el7.x86_64.rpm \
misp-python-virtualenv-$mispver.el7.x86_64.rpm
```

<br>

MISP RPM 설치

```
sudo yum install -y misp- $mispver .el7.x86_64.rpm
```

<br>

아파치 ```/httpd``` 구성

Apache VirtualHost 구성 예: ```/opt/rh/httpd24/root/etc/httpd/conf.d/misp.conf```

```
<VirtualHost _default_:80> 
DocumentRoot /var/www/MISP/app/webroot 
<Directory /var/www/MISP/app/webroot> 
옵션 -Indexes 
AllowOverride all 모두 승인 
필요 
</Directory> 
LogLevel warning 
ErrorLog /var/log/ httpd24/misp_error.log 
CustomLog /var/log/httpd24/misp_access.log 결합 
ServerSignature Off 
헤더는 항상 X-Content-Type-Options를 설정
 nosniff 헤더는 항상 X-Frame-Options를 설정
 SAMEORIGIN 헤더는 항상 "X-Powered-By" 를 설정 해제합니다  </VirtualHost>
 ```
 
 구성 변경 후
 
 ```
 sudo systemctl restart httpd
 ``` 
 
 <br>
 
 데이터베이스, 사용자 생성 및 권한 부여, 첫번째 명령은 MySQL / 셸을 열어야 합니다. 암호를 업데이트하고 기본값을 사용하지 않는걸 추천!
 
 ```
 sudo scl enable rh-mariadb105 mysql
 ```
 
 ```
 CREATE DATABASE misp;
 CREATE USER misp@'localhost' IDENTIFIED BY 'changeme';
 GRANT USAGE ON *.* to 'misp'@'localhost';
 GRANT ALL PRIVILEGES on misp.* to 'misp'@'localhost';
 FLUSH PRIVILEGES;
 exit;
```

초기 MISP스키마를 로드애햐 합니다 (위에서 설정한 비밀번호를 묻는 메시지가 표시됨)
```
sudo scl enable rh-mariadb105 'mysql -u misp -p misp' < /var/www/MISP/INSTALL/MYSQL.sql
```

<br>

#### MISP 구성<br><br>

데이터베이스 액세스를 구성합니다. (암호는 changeme 이전 데이터베이스 설정 단계에 따라 설정해야 함).

```
/var/www/MISP/app/Config/database.php
```

```
<?php
class DATABASE_CONFIG {
  public $default = array(
    'datasource' => 'Database/Mysql',
    'persistent' => false,
    'host' => 'localhost',
    'login' => 'misp',
    'port' => 3306,
    'password' => 'changeme',
    'database' => 'misp',
    'prefix' => '',
    'encoding' => 'utf8',
  );
}
```

<br>

기본 베어 구성 설정 (적절한 전체 설정에 대해서는 MISP 설명서를 확인해야 한다)

```
sudo cp -a /var/www/MISP/app/Config/core.default.php /var/www/MISP/app/Config/core.php 
sudo cp -a /var/www/MISP/app/Config/bootstrap .default.php /var/www/MISP/app/Config/bootstrap.php 
sudo cp -a /var/www/MISP/app/Config/config.default.php /var/www/MISP/app/Config/config .php 
sudo chmod o-rwx /var/www/MISP/app/Config/{config.php,database.php,email.php} 
sudo chown 아파치. /var/www/MISP/app/Config/{config.php,database.php,email.php}
```

<br>

Redis 서버 설치 - ``` rh-redis6-redis ```

```
sudo yum install -y rh-redis6-redis
sudo systemctl enable rh-redis6-redis
sudo systemctl start rh-redis6-redis
```
