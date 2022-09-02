# Introduction

This document contains information on how to install and run Mozilla Hubs in Local.

Windows 10 Pro (version 21H2) + WSL2 + Ubuntu 20.04.4 LTS based on the work was done.


# Installing Mozilla Hubs Locally


This document is based on [albirrkarim/mozilla-hubs-installation-detailed](https://github.com/albirrkarim/mozilla-hubs-installation-detailed).

Tutorial video [youtube video] of the document (https://youtu.be/KH1T9u9DaCo).

However, the tutorial video was written on a MacBook and is based on MacOS. So, it is a little different from this document. Be careful. 

In particular, most of the image examples are currently based on MacOS.

<br>

This article is about running the Mozilla Hub locally. This is a detailed step-by-step version of how to install.

Remember! If you run into a problem with npm or a dependency that you can't solve for an hour... restarting your PC will fix it (not a 100% solution).


VPS 에서 Hubs를 설치하는 방법은 다음문서를 참고해주세요. [Hosting Mozilla Hubs on VPS](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/VPS_FOR_HUBS.md)

For how to install Hubs on VPS, please refer to the following article. [Hosting Mozilla Hubs on VPS](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/VPS_FOR_HUBS.md)

<br>
<br>

Disclaimer - Attention!! This tutorial may not be best practice!!

<br>
<br>

# Requirements:

### Hardware:

- at least 8GB of RAM
- recommended using fast CPU

### Software:

- Ubuntu 20.04.4 LTS
- Node js installed. when I install this hubs I use v16

### Knowledge

- Javascript
- React js
- Webpack dev server
- Elixir and phoenix
- Web Socket

# Overview

![](/docs_img/System_Overview.png)

[figma](https://www.figma.com/) 로 만든 위의 이미지는 [문서](https://hubs.mozilla.com/docs/system-overview.html) 에서 더 많은 설명을 읽을 수 있습니다.

또한 데이터베이스에 대한 소프트웨어 개요, 아키텍처 및 테이블을 만들고 있습니다. [figma 프로젝트](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1) 에서 볼 수 있습니다.

Above image made with [figma](https://www.figma.com/) you can read more explanation in [document](https://hubs.mozilla.com/docs/system-overview.html) there is.

also creating a software overview, architecture and tables for the database. [figma project](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1)

### Summary

'Reticulum' is the main host. it sync position, rotation, state of object. Comunicates with client browser through http request and websocket.

'Dialog' sync video and audio user. comunicates with clients browser through websocket.

'Hubs', 'Spoke' serve static assets then reticulum takes it and forward to client browser.

'postREST' is a server that help hubs Admin to doing basic task like CRUD (create read update delete)

'Hubs Admin' use websocket to comunicates with postgREST for authentication (login). for CRUD purpose hubs admin send http request (GET, POST, etc) to reticulum then reticulum doing proxy pass to postgREST.


<br/>
<br/>

# 주목!

주요 단계 - [Cloning and Preparation](#1-cloning-and-preparation) -> [Setting up HOST](#2-setting-up-host) -> [Setting up HTTPS (SSL)](#3-setting-up-https-ssl) -> [Running](#4-runing)

# 1. 복제 및 준비

## 1.1 Reticulum

엘릭서와 피닉스를 사용하는 백엔드 서버입니다.

### 1.1.1 Clone

```bash
git clone https://github.com/mozilla/reticulum.git
cd reticulum
```

### 1.1.2 설치 요구 사항

#### Postgres 데이타베이스

[리눅스](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart)에 설치하기

Mac 에 설치하기.

CLI Postgres 설치를 위한 brew 사용

```
brew install postgres
```

그런 다음 user 생성/password 변경

user: `postgres`

password : `postgres`

그리고 postgres 유저의 권한을 SUPERUSER 로 변경합니다.

```
ALTER USER postgres WITH SUPERUSER
```

#### Elixir and Erlang (Elixir 1.12 and erlang 버전 23)

이 [튜토리얼](https://www.pluralsight.com/guides/installing-elixir-erlang-with-asdf)을 따라 설치할 수 있습니다.

Elixir(버전 1.12)와 erlang(버전 23)의 버전에 주의해야합니다. 다른 버전을 설치 할 경우 지원을 하지 않거나 호환성에 문제가 있을 수 있습니다.


<!-- **Ansible**

You can use `pip` to install. take a look at this [tutorial](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#from-pip) -->

### 1.1.3 다음 명령어를 실행

1. `mix deps.get`
2. `mix ecto.create`
    - 2단계가 실패하면 `dev.exs`에 구성된 비밀번호와 일치하도록 `postgres` 의 비밀번호를 변경해야 할 수 있습니다.
    - `psql` 쉘 내에서 `ALTER USER postgres WITH PASSWORD 'postgres';`를 입력합니다.
    - `ret_dev` 데이터베이스가 존재하지 않는다는 오류가 발생하면 (psql 쉘을 다시 사용하여) `create database ret_dev;`를 입력합니다.
3.  프로젝트 디렉토리에서 `mkdir -p storage/dev`

### 1.1.4 로컬 Dialog 인스턴스에 대해 Reticulum 실행

1. `dev.exs`에서 Janus 호스트를 업데이트합니다:

```elixir
dev_janus_host = "localhost"
```

2. `dev.exs`에서 Janus 포트를 업데이트합니다:

```elixir
config :ret, Ret.JanusLoadStatus, default_janus_host: dev_janus_host, janus_port: 4443
```

3. `add_csp.ex`의 CSP 규칙에 dialog 메타 엔드포인트를 추가합니다:

```elixir
default_janus_csp_rule =
   if default_janus_host,
      do: "wss://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port}/meta",
      else: ""
```

4. Coturn 설치 및 관리 방법은 구글에서 찾아보세요.

[우분투에 Coturn 설치](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04)

5. Dialog [구성 파일](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04) 편집`turnserver.conf` 및 Reticulum 데이터베이스의 _coturn_ 스키마를 사용하도록 PostgreSQL 데이터베이스 연결 문자열을 업데이트합니다:

```
psql-userdb="host=localhost dbname=ret_dev user=postgres password=postgres options='-c search_path=coturn' connect_timeout=30"
```

## 1.2 Dialog

Dialog 는 mediasoup RTC를 사용하여 오디오 및 비디오 실시간 통신을 처리합니다. 카메라 스트림과 같은 공유 화면에서 사용됩니다.

### 1.2.1 복제 및 종속성 가져오기

```bash
git clone https://github.com/mozilla/dialog.git
cd dialog
npm install
```

### 1.2.2 비밀 키 설정

이 [코멘트](https://github.com/mozilla/hubs/discussions/3323#discussioncomment-1857495) 를 참고해주세요.

[generator online](https://travistidwell.com/jsencrypt/demo/)을 사용하여 RSA(공개 및 개인 키) 생성.

빈 파일 `perms.pub.pem`을 만들고 RSA 공개 키로 채웁니다.

![RSA generator ](/docs_img/rsa.png)
![Paste](/docs_img/rsa_1.png)

`reticulum/config/dev.exs`의 reticulum 디렉토리로 이동하여 이전에 생성한 RSA 개인 키로 PermsToken을 변경합니다.

```elixir
config :ret, Ret.PermsToken, perms_key: "-----BEGIN RSA PRIVATE KEY----- paste here copyed key but add every line \n before END RSA PRIVATE KEY-----"
```

## 1.3 Spoke

Hubs 의 편집기 입니다. 씬을 생성하고 편집/배포할 수 있습니다.

### 1.3.1 복제

![Mozilla Spoke](/docs_img/spoke.png)

```
git clone https://github.com/mozilla/Spoke.git
cd Spoke
yarn install
```

### 1.3.2 기본 경로 설정

`localhost:9090`의 슬래시 `/`에 기본 URL이 있는 `react-router-dom`을 기본적으로 알고 있기를 바랍니다.

(I hope you know the basic `react-router-dom` with the default URL in  slash `/` on `localhost:9090`)

어쨋든 결국, 스포크 주소로 `localhost:4000/spoke` 를 사용하여 액세스 할것입니다.

따라서 기본 URL을 `/spoke`로 설정해야 합니다.

`package.json`의 `start` 명령에 `ROUTER_BASE_PATH=/spoke` 매개변수를 추가합니다.


![Mozilla Spoke](/docs_img/spoke_change.png)

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.pem --key certs/key.pem
```

## 1.4 Hubs

이 [repo](https://github.com/mozilla/hubs)에는 허브 클라이언트 및 허브 관리자(hubs/admin)가 포함되어 있습니다.

![System Overview](/docs_img/hubs_overview.jpeg)

종속성 복제 및 설치

```
git clone https://github.com/mozilla/hubs.git
cd hubs
npm ci
```

## 1.5 Hubs Admin

[hubs repo](#14-hubs)에서 `hubs/admin`으로 이동한 다음 실행할 수 있습니다.

```
npm install
```

# 2. 호스트 설정

우리는 `hubs.local` 도메인을 사용하지 않습니다. 우리는 `localhost`를 사용합니다

따라서 reticulum, dialog, hubs, hubs admin, spoke의 모든 호스트 구성을 변경(hubs.local -> localhost)해야합니다.

튜토리얼 [동영상](https://youtu.be/KH1T9u9DaCo?t=1482) 을 참조하세요. 
<br>

`hubs.local`  -> `localhost` 의 치환에는 크게 두가지 방법이 있습니다.

1. 해당폴더에 접근하여 비주얼 스튜디오 코드나 서브라임 텍스트와 같은 편집기로 치환하는 방법.

2. 리눅스의 명령어로 특정폴더내 모든 해당단어를 치환.

<br>

## 2.1 편집기를 통한 치환

리눅스 상에서 해당 폴더에 접근한 후, 다음과 같은 명령어를 입력합니다.

```
explorer.exe .
```

이렇게 하면 해당 폴더가 윈도우 탐색기 창으로 나타납니다. 이를 통해 WSL2 리눅스 폴더를 윈도우에서 접근할 수 있습니다.

이제 폴더경로를 알았으니 이를 기준으로 편집기를 이용하여, `hubs.local` 을 `localhost`로 수정합니다.

<br>

## 2.2 리눅스 명령어를 통한 치환

리눅스 상에서 해당 폴더로 접근 한후 폴더내 모든 문서의 특정 문자열을 치환하는 명령어를 입력한다. 명령어는 다음과 같습니다.

```
$ find ./ -type f |xargs sed -i 's/{바꿀문자열}/{새로운문자열}/g'

예) Test 를 test 로 바꾸려 한다면...

$ find ./ -type f |xargs sed -i 's/TEST/test/g'
```

<br>

# 3. HTTPS(SSL) 설정

모든 서버는 HTTPS와 함께 제공되어야 합니다. 때문에 인증서와 키 파일을 반드시 생성해야 합니다.

## 3.1 인증서 생성 및 신뢰 만들기

reticulum 디렉토리에서 터미널 열고,

다음 명령을 실행하여 인증서와 키 파일을 생성합니다.

```bash
mix phx.gen.cert
```


It will generate key `selfsigned_key.pem` and certificate `selfsigned.pem` in the `priv/cert` folder

`priv/cert` 폴더에 `self signed key.pem` 키와 인증서 `self signed.pem`을 생성합니다.

`selfsigned_key.pem`의 이름을 `key.pem`으로 바꿉니다.

`selfsigned.pem`의 이름을 `cert.pem`으로 바꿉니다.

<br>

#### 이제 `key.pem` 및 `cert.pem` 파일이 있습니다.


먼저 인증서를 리눅스에 신뢰할수 있는 Root 인증서로 추가합니다. Ubuntu 리눅스를 기준으로 작성되었습니다.
<br>

cert.pem 인증서의 경우, cert.crt 인증서로 반드시 변경해야 합니다.

```bash
openssl x509 -in cert.pem -inform pem -out cert.crt
```

cert.crt 파일을 아래 디렉토리로 복사한다. (디렉토리가 없으면 생성)

```bash
cp cert.crt /usr/share/ca-certificates/extra
```
Ubuntu가 인증서를 신뢰할 수 있는 것으로 추가하도록 한다.

```bash
dpkg-reconfigure ca-certificates
```

이제 cert.pem(cert.crt) 인증서가 신뢰할 수 있는 인증서로 리눅스에 추가되었습니다.


윈도우에도 동일한 작업을 해주면 윈도우에서 브라우저 접속시 경고메시지가 안뜨게 됩니다.


![Https mozilla hubs](/docs_img/cert.png)

`cert.crt`과 `key.pem`을 선택하고 복사합니다. 다음 단계에서는 이 두 파일을 Hubs, Hubs Admin, Spoke, Dialog 및 Reticulum 에 배포합니다.

그럼, 먼저 Reticulum 에서 설정합니다.

## 3.2 레티큘럼에 대한 https 설정

On the `config/dev.exs` We must be setting the path for the certificate and key file.

`config/dev.exs`에서 인증서와 키 파일의 경로를 설정해야 합니다.

![Https mozilla hubs](/docs_img/cert_1.png)

## 3.3 허브용 HTTPS 설정

'hubs/certs' 폴더에 다음 [파일](#now-we-have-keypem-and-certpem-file)들을 복사합니다.

우리는 `npm run local`로 허브를 실행합니다.

그래서 `package.json`에 추가 매개변수를 추가해야합니다.

`--https --cert certs/cert.crt --key certs/key.pem`

이 사진처럼

![ssl hubs](/docs_img/ssl_hubs.png)

## 3.4 허브 관리자용 HTTPS 설정

`hubs/admin/certs` 폴더에 다음 [파일](#now-we-have-keypem-and-certpem-file)들을 복사합니다.

우리는 `npm run local`로 허브(Admin)를 실행합니다.

그래서 `package.json`에 추가 매개변수를 추가하십시오.

`--https --cert certs/cert.crt --key certs/key.pem`

이 사진처럼

![ssl hubs admin](/docs_img/ssl_hubs_admin.png)

## 3.5 스포크용 HTTPS 설정


`spoke/certs` 폴더에 다음 [파일](#now-we-have-keypem-and-certpem-file)들을 복사합니다.


'yarn start'로 스포크 실행합니다.

따라서 `start` 명령을 변경하십시오.

![ssl hubs admin](/docs_img/ssl_spoke.png)

그리고 다음과 같이 수정합니다.

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.pem --key certs/key.pem
```

간단한 설명::

BASE_ASSETS_PATH = 기본적으로 localhost:9090에서 스포크를 실행합니다.

## 3.6 Dialog에 대한 https 설정

`dialog/certs` 폴더에 다음 [파일](#now-we-have-keypem-and-certpem-file)들을 복사합니다.

cert.crt`을 `fullchain.pem`으로 이름을 바꿉니다.

key.pem`을 `privkey.pem`으로 이름을 바꿉니다.

![ssl hubs dialog](/docs_img/ssl_dialog_1.png)

# 4. 실행

Open five terminals. for each reticulum, dialog, spoke, hubs, hubs admin.

5개의 터미널을 엽니다. 각각 reticulum, dialog, spoke, hubs, hubs admin 의 실행을 위해서.. 

![Running preparation](/docs_img/ss.png)

## 4.1 reticulum 실행

다음 커맨드 입력

```bash
iex -S mix phx.server
```

## 4.2 dialog 실행

다음을 사용하여 package.json의 `start` 명령을 편집합니다.

```
MEDIASOUP_LISTEN_IP=127.0.0.1 MEDIASOUP_ANNOUNCED_IP=127.0.0.1 DEBUG=${DEBUG:='*mediasoup* *INFO* *WARN* *ERROR*'} INTERACTIVE=${INTERACTIVE:='true'} node index.js
```

For giving params `MEDIASOUP_LISTEN_IP` and `MEDIASOUP_ANNOUNCED_IP`

매개변수 `MEDIASOUP_LISTEN_IP` 및 `MEDIASOUP_ANNOUNCED_IP` 제공


![Running preparation](/docs_img/run_dialog.png)

다음 명령어로 다이얼로그 서버 시작:
```
npm run start
```

`127.0.0.1`은 Mac/Linux에서 localhost의 기본 IP입니다. 다음 명령으로 IP를 볼 수 있습니다:

```bash
sudo nano /etc/hosts
```

## 4.3 spoke 실행

다음 명령어 사용

```bash
yarn start
```

## 4.4 hubs 와 hubs admin 실행.


둘 다 다음 명령어로 실행

```bash
npm run local
```

## 4.5 postgREST 서버 실행

이에 대한 자세한 내용은 [여기](https://github.com/mozilla/hubs-ops/wiki/Running-PostgREST-locally)에 있습니다.

postREST 다운로드

```
sudo apt install libpq-dev
wget https://github.com/PostgREST/postgrest/releases/download/v9.0.0/postgrest-v9.0.0-linux-static-x64.tar.xz
tar -xf postgrest-v9.0.0-linux-static-x64.tar.xz
```

reticulum iex 상에서

다음 명령어를 붙여넣기 한 후 실행 합니다.
```
jwk = Application.get_env(:ret, Ret.PermsToken)[:perms_key] |> JOSE.JWK.from_pem(); JOSE.JWK.to_file("reticulum-jwk.json", jwk)
```

then it will create `reticulum-jwk.json` in your reticulum directory

그러면 reticulum 디렉토리에 `reticulum-jwk.json`이 생성됩니다.

Make `reticulum.conf` file 

`reticulum.conf` 파일 만들기

```
nano reticulum.conf
```
and paste 

그리고 만들어진 파일에 아래의 내용을 붙여넣기 합니다.

```
# reticulum.conf
db-uri = "postgres://postgres:postgres@localhost:5432/ret_dev"
db-schema = "ret0_admin"
db-anon-role = "postgres_anonymous"
jwt-secret = "@/absolute_path_to_your_file/reticulum-jwk.json"
jwt-aud = "ret_perms"
role-claim-key = ".postgrest_role"
```

then the folder looks like this (contain two files)

폴더는 다음과 같습니다(두 개의 파일 포함).

```
/
   postgrest
   reticulum.conf
```

then run  postREST with

그런 다음 postREST를 실행합니다.

```
postgrest reticulum.conf
```

이때, postgrest 경로를 찾을 수 없다고 나오면, postgrest 앞에 전체 디렉토리 경로를 입력합니다.(reticulum.conf 파일이 있는 디렉토리 경로)

예) /home/ubuntu/hubs/reticulum/postgrest reticulum.conf

<br>
<br>

전부 실행이 완료되면, 드디어 Hubs 에 접속이 가능합니다.

잠금 기호 포함(SSL 보안)

Hubs

[https://localhost:4000](https://localhost:4000)

Hubs admin

[https://localhost:4000/admin](https://localhost:4000/admin)

Spoke

[https://localhost:4000/spoke](https://localhost:4000/spoke)


<br>
<br>




## 한번 읽어보세요 : 

[Hosting Mozilla Hubs on VPS](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/VPS_FOR_HUBS.md)

[The Problem I Still Faced](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/PROBLEM_UNSOLVED.md)

[The Problem I Faced and I Already Solved](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/PROBLEM_SOLVED.md)

[Tips for Modification](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/HOW_TO_MODIFY.md)

[Overview System With Figma](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1)

[Experience Sharing About Hosting on Other Server](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/EXPERIENCE.md)




## Special Thanks : 

[albirrkarim](https://github.com/albirrkarim)

[Paypal](https://paypal.me/AlbirrKarim)

<a href='https://ko-fi.com/Q5Q0BC92X' target='_blank'><img height='36' style='border:0px;height:36px;' src='https://cdn.ko-fi.com/cdn/kofi3.png?v=3' border='0' alt='Buy Me a Coffee at ko-fi.com' /></a>
