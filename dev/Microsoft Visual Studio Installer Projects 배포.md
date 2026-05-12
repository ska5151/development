#pg언어 #c언어 

Tool : Visual Studio 2019
Language : C#

1. 상단메뉴의 확장 > 확장관리
    - Microsoft Visual Studio Installer Projects 검색 > 다운로드
2. Visual Studio 솔루션 우 클릭 > 추가 > 새 프로젝트
    - Setup Project 검색 > 다음 클릭
    - 프로젝트 이름 : setup 후 만들기
3. Application Folder 클릭 후 우측 빈 화면 우클릭 > Add > 프로젝트 출력 > 확인
4. 그럼 [기본 출력 from [프로젝트명] (Active)]가 생성되는데, 우 클릭 후 [Create Shortcut to...]를 클릭하면 파일 생성되는데 그 파일의 이름 변경
    - 이 파일의 이름이 실제 설치후 프로그램이름이 된다.
5. .ico파일(아이콘파일)을 복사 후 Application Folder에 붙여넣기
6. 4번의 파일을 똑같이 2개 더 생성 후 User's Desktop, User's Programs Menu에 드래그 삽입
7. User's Desktop의 우측 화면 파일 속성 > ICON을 클릭하여 팝업창 > Browse > Apllication Folder > [아이콘 지정] > ok 클릭
8. setup 솔루션 클릭 > 속성(F4)에 필요한 항목 기입
    - Author, Manufacturer, ProductName, Title에 프로그램명 입력
9. setup 솔루션 우클릭 > 속성
    - prerequisites... 클릭 > 필수 구성 요소를 설치하기 위한 설치 프로그램 만들기 체크
    - 구성 관리자 클릭 > [프로젝트명]의 빌드 체크
10. setup 솔루션 우클릭 > 빌드하면 setup 파일 생성
- msi파일로 배포

※ 프로그램 추가/수정 아이콘 수정

- 배포 프로젝트 속성 > AddRemoveProgramsIcon 수정

※ 이전 버전 삭제

- 배포 프로젝트 속성 > RemovePreviousVersions = True
    - false로 하면 프로그램 추가/수정 화면에서 이전버전 다 뜸…

※ msi 파일 관리자 권한 실행

1. setup project 경로 내 WiRunSql.vbs 삽입
    
    [WiRunSql.zip](https://prod-files-secure.s3.us-west-2.amazonaws.com/fe0f600a-d6b2-476f-b939-d71616f3c986/1c242fd7-908e-4b67-99a5-57f800f49333/WiRunSql.zip)
    
2. setup project
    - 속성 > PostBuildEvent

cscript //nologo "$(ProjectDir)WiRunSql.vbs" "$(BuiltOuputPath)" "UPDATE CustomAction SET CustomAction.Type=3073 WHERE CustomAction.Type=1025 AND CustomAction.Source='InstallUtil' AND CustomAction.Target='ManagedInstall'"
cscript //nologo "$(ProjectDir)WiRunSql.vbs" "$(BuiltOuputPath)" "UPDATE CustomAction SET CustomAction.Type=3585 WHERE CustomAction.Type=1537 AND CustomAction.Source='InstallUtil' AND CustomAction.Target='ManagedInstall'"

- 속성 > RunPostBuildEvent
    
    On successful build