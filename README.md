MRS/Nebula 배포 운영 가이드 (Windows/PowerShell)

0) 용어/구조 요약

배포 레포: mc-launcher-distro (GitHub)

배포 루트(로컬): D:\mcLauncherRoot

런처 원격 설정:
exports.REMOTE_DISTRO_URL = "https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/distribution.json"

파일 내려줄 BASE_URL(rawgit로 통일):
https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/

폴더 트리(핵심만):

D:\mcLauncherRoot
 ├─ distribution.json      ← Nebula가 생성
 ├─ servers/
 │   └─ <ServerId-Version>/
 │       ├─ servermeta.json
 │       ├─ files/                ← 인스턴스 루트에 복사될 일반 파일
 │       │   ├─ options.txt
 │       │   ├─ config/...
 │       │   ├─ shaderpacks/*.zip
 │       │   └─ resourcepacks/*.zip
 │       └─ fabricmods|forgemods|neoforgemods/
 │           ├─ required/*.jar
 │           ├─ optionalon/*.jar
 │           └─ optionaloff/*.jar
 └─ repo/                     ← Nebula가 생성(라이브러리/버전 캐시)

1) 캐시 초기화(깨끗한 재빌드가 필요할 때)
1-1) Nebula 캐시 무효화 + 재생성(권장)
Set-Location "C:\Users\user\Nebula" ; Set-Content .env "ROOT=D:\mcLauncherRoot`nBASE_URL=https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/`nINSTALL=false`nDISCARD_OUTPUT=false`nINVALIDATE_CACHE=true" ; npm run start -- generate distro

1-2) 런처 로컬 캐시/인스턴스 초기화(강제 리셋)

런처가 Helios 계열이면 보통 다음 경로 중 하나를 씀.

Remove-Item -Recurse -Force "$env:APPDATA\Helios Launcher" -ErrorAction SilentlyContinue ; Remove-Item -Recurse -Force "$env:APPDATA\MRS Launcher" -ErrorAction SilentlyContinue


완전 초기화라 저장 데이터가 날아간다. 필요할 때만.

2) 새 서버 추가(10분 루틴)
2-1) 서버 골격 생성(예: Fabric 1.21.8)
Set-Location "C:\Users\user\Nebula" ; npm run start -- generate server FabricServer-1.21.8 1.21.8 --fabric 0.17.2


Forge/NeoForge도 가능:
--forge <로더버전> / --neoforge <로더버전>

2-2) 파일 배치(폴더에 그냥 복붙)

servers/FabricServer-1.21.8/files/

options.txt, config/**/*, shaderpacks/*.zip, resourcepacks/*.zip

servers/FabricServer-1.21.8/fabricmods/required/*.jar (혹은 forgemods/neoforgemods)

servermeta.json에 서버 설명/주소/로더 버전/모듈 등을 적절히 설정

쉐이더 자동적용: files/config/iris.properties 에 shaderPack=<파일명>.zip

2-3) distribution.json 생성
Set-Location "C:\Users\user\Nebula" ; npm run start -- generate distro

2-4) GitHub로 배포(필수 파일 강제 추가 → 푸시)
Set-Location "D:\mcLauncherRoot" ; git add -A ; git commit -m "publish distro" ; git push


zip/jar가 무시될 경우(404 방지용 강제 추가):

Set-Location "D:\mcLauncherRoot" ; git add -f distribution.json ; git add -f repo/** ; git add -f servers/**/files/**/*.zip ; git add -f servers/**/fabricmods/**/*.jar ; git commit -m "force add zips/jars" ; git push


리모트가 앞서서 거절되면 로컬로 덮어쓰기:

git add -A ; git commit -m "force overwrite with local distro" ; git push -f origin master

3) distribution.json만 다시 만들 때
Set-Location "C:\Users\user\Nebula" ; npm run start -- generate distro


.env의 BASE_URL이 반드시 https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/ 여야 JSON 속 URL이 전부 raw로 나오고, 런처에서 404/ECONNREFUSED가 안 뜬다.

4) 빠른 진단/트러블슈팅
4-1) 404 팝업이 뜰 때(필요 파일 누락 확인)

distribution.json 안 모든 URL을 HEAD로 검사(200이 아닌 것만 출력):

$dist="https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/distribution.json"; $j=Invoke-RestMethod $dist; $bad=@(); foreach($s in $j.servers){ foreach($m in $s.modules + $m.subModules){ if($m -and $m.artifact -and $m.artifact.url){ try{$code=(Invoke-WebRequest -UseBasicParsing -Method Head -TimeoutSec 10 -Uri $m.artifact.url).StatusCode}catch{$code=$_.Exception.Response.StatusCode}; if($code -ne 200){ $bad+=("$code`t$($m.type)`t$($m.id)`t$($m.artifact.url)") } } } }; $bad -join "`n"


범인 URL이 나오면:

그 파일이 레포에 있는지 확인(대소문자/경로 주의)

없으면 아래 강제 추가 & 푸시:

Set-Location "D:\mcLauncherRoot" ; git add -f repo/** ; git add -f servers/**/files/**/*.zip ; git add -f servers/**/fabricmods/**/*.jar ; git commit -m "add missing assets" ; git push

4-2) ECONNREFUSED가 뜰 때

distribution.json 속 URL이 http://localhost:8080로 남아있을 때 발생.

해결: .env의 BASE_URL을 raw 주소로 고치고 재생성:

Set-Location "C:\Users\user\Nebula" ; Set-Content .env "ROOT=D:\mcLauncherRoot`nBASE_URL=https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/`nINSTALL=false`nDISCARD_OUTPUT=false`nINVALIDATE_CACHE=true" ; npm run start -- generate distro

4-3) mods 폴더 에러(Bad archive/Archive read error)

fabricmods|forgemods|neoforgemods 폴더엔 .jar/.zip만 두기.

안내용 .txt/.keep 넣으면 Nebula가 모드로 간주하고 깨짐 → 폴더를 완전 비우거나 진짜 모드만.

4-4) 브랜치/경로 확인

브라우저에서 아래 3개가 모두 열려야 정상:

https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/distribution.json

https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/servers/FabricServer-1.21.8/files/options.txt

https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/repo/versions/1.21.8-fabric-0.17.2/1.21.8-fabric-0.17.2.json

5) 런처 브랜딩(텍스트/링크 변경)

좌상단 타이틀 등: app/assets/lang/_custom.toml

[ejs.app]
title = "ME Launcher"


변경 후:

npm run build ; npm start

6) 권장 .gitignore (배포 레포용)

zip/jar 무시 금지 — 런처가 원격에서 내려받기 위해 꼭 필요

# OS/에디터
.DS_Store
Thumbs.db
desktop.ini
.vscode/
.idea/

# Node/로그
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Temp
*.tmp
*.swp
*.bak

7) 자주 쓰는 원라인 모음

## distribution 재생성(캐시 무효화 포함)
Set-Location "C:\Users\user\Nebula" ; Set-Content .env "ROOT=D:\mcLauncherRootnBASE_URL=https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/`nINSTALL=false`nDISCARD_OUTPUT=false`nINVALIDATE_CACHE=true
" ; npm run start -- generate distro`

## 필수 파일 강제 푸시
Set-Location "D:\mcLauncherRoot" ; git add -f distribution.json ; git add -f repo/** ; git add -f servers/**/files/**/*.zip ; git add -f servers/**/fabricmods/**/*.jar ; git commit -m "publish distro" ; git push

## 리모트 덮어쓰기(주의)
git add -A ; git commit -m "force overwrite with local distro" ; git push -f origin master

## 404 범인 URL 탐지
$dist="https://raw.githubusercontent.com/MukEllie/mc-launcher-distro/master/distribution.json"; $j=Invoke-RestMethod $dist; $bad=@(); foreach($s in $j.servers){ foreach($m in $s.modules + $m.subModules){ if($m -and $m.artifact -and $m.artifact.url){ try{$code=(Invoke-WebRequest -UseBasicParsing -Method Head -TimeoutSec 10 -Uri $m.artifact.url).StatusCode}catch{$code=$_.Exception.Response.StatusCode}; if($code -ne 200){ $bad+=("$code`t$($m.type)`t$($m.id)`t$($m.artifact.url)") } } } }; $bad -join "`n"