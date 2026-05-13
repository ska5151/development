#win #apache #php #mysql 

# 디렉토리 구조 준비 및 사전 설치

==관리를 쉽게 하기 위해 D:\APM 폴더를 만들고 그 안에 모든 프로그램을 구성합니다.==
1. `D:\APM` 폴더를 생성합니다.
2. Visual C++ 재배포 가능 패키지(VC15, VS16) 설치:
	Apache와 PHP 8.1이 Windows에서 동작하려면 C++ 런타임이 필요합니다.
  - [Microsoft 공식 다운로드 (x64)](https://www.google.com/search?q=https://aka.ms/vs/16/release/vc_redist.x64.exe)에서 다운로드 후 설치합니다.

# MySQL 5.7.44 설치 (Windows ZIP)

==다운로드 및 압축 해제==
1. MySQL Archives에서 `mysql-5.7.44-winx64.zip` 파일을 다운로드합니다.
  - [다운로드 링크](https://www.google.com/search?q=https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.44-winx64.zip)
2. 다운로드한 파일의 압축을 풀고, 폴더 이름을 `mysql`로 변경하여 `D:\APM\mysql` 에 위치시킵니다.

==환경 설정 파일 생성 (my.ini)==
- D:\APM\mysql 폴더 안에 my.ini 라는 텍스트 파일을 새로 만들고 아래 내용을 적어 저장
```ini
[mysqld]
basedir=D:/APM/mysql
datadir=D:/APM/mysql/data
port=3306
character-set-server=utf8
```

==DB 초기화 및 서비스 등록 (관리자 권한 CMD)==
```bash
:: D 드라이브 경로로 이동하기 위해 /d 옵션 사용
cd /d D:\APM\mysql\bin

:: 1. DB 초기화 (실행 후 화면에 임시 비밀번호가 나오니 반드시 메모하세요!)
mysqld --initialize --console

:: 2. Windows 서비스로 MySQL 등록
mysqld --install MySQL57

:: 3. MySQL 서비스 시작
net start MySQL57
```

# PHP 8.1.27 설치

==다운로드 및 압축 해제==
1. PHP for Windows 페이지에서 `PHP 8.1.27 (VS16 x64 Thread Safe)`의 ZIP 파일을 다운로드합니다.
  - [다운로드 링크](https://www.google.com/search?q=https://windows.php.net/downloads/releases/archives/php-8.1.27-Win32-vs16-x64.zip)
2. 압축을 풀고 폴더 이름을 `php`로 변경하여 `D:\APM\php` 에 위치시킵니다.

==환경 설정 파일 구성 (php.ini)==
1. `D:\APM\php` 폴더에서 `php.ini-production` 파일을 복사하여 `php.ini` 로 이름을 바꿉니다.
2. 메모장 등 에디터로 `php.ini`를 열고 아래 항목들의 맨 앞 주석(`;`)을 지우고 경로를 수정합니다.
```ini
; 확장 디렉토리 활성화
extension_dir = "D:/APM/php/ext"
extension_dir = "C:\company\project\발주나라\apm"

; 필요한 확장 모듈 활성화 (주석 해제)
extension=curl
extension=mbstring
extension=mysqli
extension=openssl
extension=pdo_mysql
```

# Apache 2.4 설치 및 PHP 연동

==다운로드 및 압축 해제==
1. Apache Lounge에서 Windows용 Apache 2.4 바이너리를 다운로드합니다.
  - [다운로드 링크](https://www.google.com/search?q=https://www.apachelounge.com/download/VS16/binaries/httpd-2.4.58-win64-VS16.zip) (버전은 다를 수 있음)
2. 압축 파일 내부의 `Apache24` 폴더를 통째로 `D:\APM\Apache24` 로 복사합니다.

==환경 설정 파일 수정 (httpd.conf)==
- D:\APM\Apache24\conf\httpd.conf 파일을 열고 아래 내용들을 수정/추가
```conf
1. 서버 루트 경로 수정 (파일 상단)
Define SRVROOT "D:/APM/Apache24"

2. 서버 이름 설정
ServerName localhost:80

3. DirectoryIndex 수정 (index.php 추가)
<IfModule dir_module>
    DirectoryIndex index.php index.html
</IfModule>

4. PHP 8.1 모듈 연동 (파일 맨 아래에 통째로 복사해서 붙여넣기)
# PHP 8.1 연동
LoadModule php_module "D:/APM/php/php8apache2_4.dll"
AddHandler application/x-httpd-php .php
PHPIniDir "D:/APM/php"

5. htdocs
<Directory "${SRVROOT}/htdocs"> ...
AcceptPathInfo On 추가
```

==Apache Windows 서비스 등록 및 실행==
```bash
:: D 드라이브 경로로 이동
cd /d D:\APM\Apache24\bin

:: Windows 서비스로 Apache 등록
httpd -k install

:: Apache 서비스 시작
httpd -k start
```

==연동 테스트==
1. `D:\APM\Apache24\htdocs` 폴더에 `info.php` 파일을 만들고 `<?php phpinfo(); ?>` 를 입력해 저장합니다.
2. 웹 브라우저에서 `http://localhost/info.php` 에 접속하여 보라색 PHP 정보 화면이 나오면 성공입니다.

# phpMyAdmin 설치

1. phpMyAdmin 공식 사이트에서 최신버전 ZIP 파일을 다운로드합니다.
    - [다운로드 링크](https://www.google.com/search?q=https://files.phpmyadmin.net/phpMyAdmin/5.2.1/phpMyAdmin-5.2.1-all-languages.zip)
2. 압축을 풀고 폴더 이름을 `phpmyadmin`으로 변경하여 `D:\\APM\\Apache24\\htdocs\\phpmyadmin` 경로에 넣습니다.
3. `config.sample.inc.php` 파일을 복사하여 `config.inc.php` 로 이름을 바꿉니다.
4. 파일을 열고 쿠키 암호화 키를 적당히 32자 이상 입력합니다.
```php
$cfg['blowfish_secret'] = 'my_secret_key_12345678901234567890';
```
5. 브라우저에서 http://localhost/phpmyadmin 으로 접속 후, root 계정과 2.3 단계에서 발급받은 임시 비밀번호로 로그인합니다.

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

==Apache 재시작==
```bash
cd /d D:\APM\Apache24\bin
httpd -k restart
```

