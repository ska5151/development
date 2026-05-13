#linux #docker #apache #php #mysql

# Docker 컨테이너 생성 및 접속

```bash
# 컨테이너 생성 및 백그라운드 실행 (이름: balju_setting)
# SSH를 위해 -p 2222:22 옵션이 추가
docker pull centos:7.9.2009
docker run -itd -p 80:80 -p 3306:3306 -p 2222:22 --privileged --restart=unless-stopped -e "TZ=Asia/Seoul" --name balju_setting centos:7.9.2009 bash

# 컨테이너 내부 쉘(bash)로 접속
docker exec -it balju_setting bash
```

==사전 준비 (의존성 패키지 설치)==
```bash
# 1. CentOS 7 EOL로 인한 Mirror 주소 Vault로 변경 (필수)
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

# 2. 패키지 캐시 정리 및 업데이트
yum clean all
yum update -y

# 3. EPEL 저장소 설치 (oniguruma-devel 등 확장 라이브러리 설치용)
yum install -y epel-release

# 4. 필수 빌드 도구 및 라이브러리 설치
yum install -y gcc gcc-c++ make cmake wget tar nano perl \
pcre-devel openssl-devel expat-devel libxml2-devel sqlite-devel \
oniguruma-devel libcurl-devel zlib-devel ncurses-devel \
libaio numactl-libs
```

# MySQL 5.7.44 설치

==다운로드 및 압축 해제==
```bash
mkdir -p /usr/local/src
cd /usr/local/src

wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.44-linux-glibc2.12-x86_64.tar.gz
tar -zxvf mysql-5.7.44-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.44-linux-glibc2.12-x86_64 /usr/local/mysql
```

==MySQL 사용자 생성 및 DB 초기화==
```bash
# mysql 유저 및 그룹 생성
groupadd mysql
useradd -r -g mysql -s /bin/false mysql

# 디렉토리 권한 설정
cd /usr/local/mysql
chown -R mysql:mysql .

# DB 초기화 (★ 화면 마지막에 출력되는 root 임시 비밀번호를 반드시 메모하세요!)
bin/mysqld --initialize --user=mysql
-- root@localhost: _5TaI#g7+DR2

# 접속
/usr/local/mysql/support-files/mysql.server start
/usr/local/mysql/bin/mysql -u root -p --socket=/tmp/mysql.sock

# root 비밀번호 변경
ALTER USER 'root'@'localhost' IDENTIFIED BY 'kb1177@';
# root 계정의 외부 접속을 허용하는 경우
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'kb1177@' WITH GRANT OPTION;
# 변경된 권한 적용
FLUSH PRIVILEGES;
```

==MySQL 백그라운드 실행 (systemctl 대체)==
```bash
# MySQL 데몬 백그라운드 실행
bin/mysqld_safe --user=mysql &

# 환경 변수 등록
echo 'export PATH=$PATH:/usr/local/mysql/bin' >> ~/.bashrc
source ~/.bashrc
```

# Apache 2.2.34 설치 (소스 컴파일)

==다운로드 및 압축 해제==
```bash
cd /usr/local/src
wget https://archive.apache.org/dist/httpd/httpd-2.2.34.tar.gz
tar -zxvf httpd-2.2.34.tar.gz
cd httpd-2.2.34
```

==컴파일 및 설치==
```bash
./configure \
--prefix=/usr/local/apache2 \
--enable-so \
--enable-rewrite \
--with-included-apr

make
make install
```

# PHP 8.1.27 설치 (소스 컴파일)

==다운로드 및 압축 해제==
```bash
cd /usr/local/src
wget https://www.php.net/distributions/php-8.1.27.tar.gz
tar -zxvf php-8.1.27.tar.gz
cd php-8.1.27
```

==컴파일 및 설치==
```bash
./configure \
--prefix=/usr/local/php \
--with-apxs2=/usr/local/apache2/bin/apxs \
--with-config-file-path=/usr/local/php/etc \
--with-mysqli=mysqlnd \
--with-pdo-mysql=mysqlnd \
--with-zlib \
--with-openssl \
--enable-mbstring \
--enable-xml \
--enable-sockets

# php.ini 설정 파일 복사
mkdir -p /usr/local/php/etc
cp php.ini-production /usr/local/php/etc/php.ini
```

# Apache - PHP 연동 설정

```bash
vi /usr/local/apache2/conf/httpd.conf

1. ServerName 설정 (경고 메시지 방지, 파일 상단 부근에 추가)
ServerName localhost:80

2. PHP 모듈 로드 확인 (LoadModule 들이 모여있는 곳에 보통 자동 추가되어 있습니다. 만약 없다면 직접 추가하세요.)
LoadModule php_module modules/libphp.so

3. DirectoryIndex 설정 변경 (index.php 추가)
<IfModule dir_module>
    DirectoryIndex index.php index.html
</IfModule>

4. PHP 핸들러 추가 (맨 밑에 추가)
<FilesMatch \.php$>
    SetHandler application/x-httpd-php
</FilesMatch>
```

# 서비스 시작 및 테스트

==Apache 서버 실행==
```bash
/usr/local/apache2/bin/apachectl start
```

==테스트 파일 생성==
```bash
echo "<?php phpinfo(); ?>" > /usr/local/apache2/htdocs/info.php
```

==브라우저 접속 확인==
```bash
http://localhost/info.php
```

# phpMyAdmin 설치 및 접속 방법

==다운로드 및 압축 해제==
```bash
cd /usr/local/src

# PHP 8.1과 호환되는 최신 phpMyAdmin 다운로드
wget https://files.phpmyadmin.net/phpMyAdmin/5.2.1/phpMyAdmin-5.2.1-all-languages.tar.gz

# 압축 해제 후 Apache DocumentRoot로 이동
tar -zxvf phpMyAdmin-5.2.1-all-languages.tar.gz
mv phpMyAdmin-5.2.1-all-languages /usr/local/apache2/htdocs/phpmyadmin
```

==환경 설정 파일 구성==
```bash
cd /usr/local/apache2/htdocs/phpmyadmin

# 샘플 설정 파일을 복사하여 실제 설정 파일 생성
cp config.sample.inc.php config.inc.php

# 1. 쿠키 인증을 위한 blowfish_secret 암호화 키 설정 (아무 문자열이나 32자리 이상)
sed -i "s/\$cfg\['blowfish_secret'\] = '';/\$cfg\['blowfish_secret'\] = 'my_secret_key_12345678901234567890';/" config.inc.php

# 2. 소켓 경로 문제를 회피하기 위해 접속 호스트를 127.0.0.1 (TCP)로 명시적 변경
sed -i "s/'localhost'/'127.0.0.1'/" config.inc.php
```

==브라우저에서 접속 확인==
```bash
http://localhost/phpmyadmin
```

# PHP 프레임워크 라우팅 (Rewrite) 설정

CodeIgniter, Laravel 등의 프레임워크를 위해 `.htaccess` 라우팅을 활성화하는 방법입니다.
`D:\APM\Apache24\conf\httpd.conf` 파일을 열어 수정

==Apache 설정 수정==
- D:\APM\Apache24\conf\httpd.conf 파일을 열어 수정
```conf
1. mod_rewrite 모듈 활성화: 아래 줄의 주석(#)을 제거합니다.
LoadModule rewrite_module modules/mod_rewrite.so

2. 구버전 접근 제어 호환 모듈 활성화 (500 에러 방지용): 구형 프레임워크의 .htaccess에서 사용하는 Order, Allow 명령어 오류(Invalid command 'Order')를 방지하기 위해 아래 줄의 주석(#)을 제거합니다.
LoadModule access_compat_module modules/mod_access_compat.so

3. DocumentRoot 권한 수정: <Directory "D:/APM/Apache24/htdocs"> 블록 안에서 AllowOverride None을 AllowOverride All로 변경합니다.
<Directory "D:/APM/Apache24/htdocs">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

# 기타 세팅

==.htaccess 파일 생성==
```bash
vi /usr/local/apache2/htdocs/.htaccess

# 붙여넣기
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteBase /

    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule ^(.*)$ index.php/$1 [L]
</IfModule>
```

==timezone==
```bash
# php.ini > php/etc/php.ini
date.timezone = Asia/Seoul
```

==엑셀 내려받기 에러==
```bash
yum install php81-php-zip php81-php-pecl-zip libzip -y
```

# 재시작

```bash
/usr/local/mysql/bin/mysqld_safe --user=mysql &
/usr/local/apache2/bin/apachectl restart
```
