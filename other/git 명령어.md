#git #명령어

```bash
# 유저이름 설정
git config --global [user.name](http://user.name) "your_name"

# 유저 이메일 설정하기
git config --global user.email "your_email"

# 머지 옵션
git config --global pull.rebase false

# 대소문자 구분
git config —global core.ignorecase false

# 정보 확인하기
git config --list

# 초기화
git init

# 내 branch에 소스코드 업데이트하기
git add .
git commit -m "first commit"
git push origin 브렌치이름

# 상태 확인 (선택사항)
git status

# 히스토리 만들기
git commit -m "first commit"

# Github repository랑 내 로컬 프로젝트랑 연결
git remote add origin https://github.com/bitnaGithub/firstproject.git

# 잘 연결됬는지 확인 (선택사항)
git remote -v

# Github로 올리기
git push origin master

# 소스코드 다운로드
git clone 주소 폴더이름

# branch 생성
git checkout -b feature-01
git push origin feature-01

# branch 삭제
git branch -d 브렌치이름
git push origin --delete 브렌치이름

# 원격 branch 업데이트
git fetch --prune

# 마스터 branch에 소스 가져오기(pull)
git pull origin master

# A branch에서 B branch 머지하기
git merge B

# A branch로 이동
git checkout A

# A branch 와 B branch 차이
git diff —name-only A B

# 다른 branch 있는 commit 가져오기
git cherry-pick <commit 주소>

# commit 되돌리기
git revert <commit 주소>
```