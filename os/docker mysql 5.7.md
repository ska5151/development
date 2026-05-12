#linux

# 이미지 생성
```bash
docker pull mysql:5.7
```

# 컨테이너 실행
```bash
docker run -d \
  --name mysql-5.7 \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD=your_password \
  -v ~/mysql_data:/var/lib/mysql \
  mysql:5.7
```
  
# Docker 컨테이너 재시작 정책 설정
```bash
docker update --restart unless-stopped mysql-5.7
```

# wsl docker 자동 시작
```bash
crontab -e
@reboot sudo service docker start
```