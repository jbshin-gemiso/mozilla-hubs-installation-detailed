# 소개

이 리포지토리에는 mozilla 허브에 대한 기술 지식이 포함되어 있습니다.
서로 도와줍시다.

# 로컬로 Mozilla Hub 설치

이 문서는 albirrkarim/mozilla-hubs-installation-detailed 를 기반으로 제작되었습니다.
해당문서의 튜토리얼 영상 [youtube video](https://youtu.be/KH1T9u9DaCo). 
단, 튜토리얼 영상은 맥북으로 작성되어 MacOS 기반입니다. 그래서 이 문서와는 조금 차이가 있습니다. 주의하세요.

이것은 로컬에서 Mozilla 허브를 실행하는 것에 관한 것입니다. 이것은 내가 하는 일을 단계별로 자세히 설명한 버전입니다.
기억하다! 1시간 동안 해결할 수 없는 npm 또는 종속성에 문제가 있는 경우. PC를 다시 시작하면 됩니다. 날 믿어.

VPS 에서 Hubs를 설치하는 방법은 다음문서를 참고해주세요. [Hosting Mozilla Hubs on VPS](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/VPS_FOR_HUBS.md)

<br>
<br>

면책 조항 - 이 튜토리얼은 모범 사례가 아닐 수 있습니다.

<br>
<br>

# 요구사항:

### Hardware:

- 최소 8GB RAM
- 빠른 CPU 사용 권장
- 우분투 20

### 소프트웨어

- 노드 js가 설치되었습니다. 허브를 설치할 때 버전은 v16을 사용합니다.

### 지식

나는 당신이 이미 알고 있다고 가정합니다. 그렇지 않다면 먼저 기술을 향상시켜야 합니다.

![Up skill](/docs_img/excercise.gif)

- Javascript
- React js
- Webpack dev server
- Elixir and phoenix
- Web Socket

# 개요

![시스템 오버뷰](/docs_img/System_Overview.png)

[figma](https://www.figma.com/) 로 만든 위의 이미지 는 [문서](https://hubs.mozilla.com/docs/system-overview.html) 에서 더 많은 설명을 읽을 수 있습니다

또한 데이터베이스에 대한 소프트웨어 개요, 아키텍처 및 테이블을 만들려고 합니다. 내 [figma 프로젝트](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1) 를 볼 수 있습니다

### 요약

Reticulum - 주요 호스트입니다. 그것은 위치, 회전, 개체의 상태를 동기화합니다. http 요청 및 websocket을 통해 클라이언트 브라우저와 통신합니다.

Dialog - 동기화 비디오 및 오디오 사용자. 웹 소켓을 통해 클라이언트 브라우저와 통신합니다.

Hubs, Spoke - 정적 자산을 제공한 다음 레티큘럼이 이를 가져와 클라이언트 브라우저로 전달합니다.

postREST - 허브 관리자가 CRUD(읽기 업데이트 삭제 생성)와 같은 기본 작업을 수행하는 데 도움이 되는 서버입니다.

Hubs Admin - websocket을 사용하여 인증(로그인)을 위해 postgREST와 통신합니다. CRUD 목적의 허브 관리자는 http 요청(GET, POST 등)을 레티큘럼에 보낸 다음 프록시를 postgREST에 전달하는 레티큘럼을 보냅니다.

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

그리고 권한을 변경

```
ALTER USER postgres WITH SUPERUSER
```

#### Elixir and Erlang (Elixir 1.12 and erlang 버전 23)

이 [튜토리얼](https://www.pluralsight.com/guides/installing-elixir-erlang-with-asdf)을 따라 설치할 수 있습니다.

Elixir와 erlang의 버전에 주의하십시오.

<!-- **Ansible**

You can use `pip` to install. take a look at this [tutorial](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#from-pip) -->

### 1.1.3 다음 명령어를 실행

1. `mix deps.get`
2. `mix ecto.create`
    - 2단계가 실패하면 `dev.exs`에 구성된 비밀번호와 일치하도록 `postgres` 역할의 비밀번호를 변경해야 할 수 있습니다.
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

3. `add_csp.ex`의 CSP 규칙에 Dialog 메타 엔드포인트를 추가합니다:

```elixir
default_janus_csp_rule =
   if default_janus_host,
      do: "wss://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port}/meta",
      else: ""
```

4. coturn 설치 및 관리 방법은 구글에서 찾아보세요.

[우분투에 coturn 설치](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04)

5. Dialog [구성 파일](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04) 편집`turnserver.conf` 및 Reticulum 데이터베이스의 _coturn_ 스키마를 사용하도록 PostgreSQL 데이터베이스 연결 문자열을 업데이트합니다:

```
psql-userdb="host=localhost dbname=ret_dev user=postgres password=postgres options='-c search_path=coturn' connect_timeout=30"
```

## 1.2 Dialog

Using mediasoup RTC will handle audio and video real-time communication. like camera stream, share screen.

### 1.2.1 Clone and get dependencies

```bash
git clone https://github.com/mozilla/dialog.git
cd dialog
npm install
```

### 1.2.2 Setting up secret key

thanks to this [comment](https://github.com/mozilla/hubs/discussions/3323#discussioncomment-1857495)

Generate RSA (Public and Private key) with [generator online](https://travistidwell.com/jsencrypt/demo/)

make empty file `perms.pub.pem` and fill it with RSA Public key

![RSA generator ](/docs_img/rsa.png)
![Paste](/docs_img/rsa_1.png)

Goto reticulum directory on `reticulum/config/dev.exs` change PermsToken with the RSA private key that you generate before.

```elixir
config :ret, Ret.PermsToken, perms_key: "-----BEGIN RSA PRIVATE KEY----- paste here copyed key but add every line \n before END RSA PRIVATE KEY-----"
```

## 1.3 Spoke

In here you can create/edit the scenes/buildings whatever you call it.

### 1.3.1 Clone

![Mozilla Spoke](/docs_img/spoke.png)

```
git clone https://github.com/mozilla/Spoke.git
cd Spoke
yarn install
```

### 1.3.2 Set the base routes

I hope you know the basic `react-router-dom` with the default URL in  slash `/` on `localhost:9090`

But in the end, we will access the spoke on `localhost:4000/spoke`

So we must set the base URL to `/spoke`

Add the `ROUTER_BASE_PATH=/spoke` params to the `start` command on `package.json`


![Mozilla Spoke](/docs_img/spoke_change.png)

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.pem --key certs/key.pem
```

## 1.4 Hubs

In this [repo](https://github.com/mozilla/hubs) contains the hubs client and hubs admin (hubs/admin)

![System Overview](/docs_img/hubs_overview.jpeg)

Clone and install dependencies

```
git clone https://github.com/mozilla/hubs.git
cd hubs
npm ci
```

## 1.5 Hubs Admin

from the [hubs repo](#14-hubs) you can move to `hubs/admin` then run

```
npm install
```

# 2. Setting up HOST

We are not using `hubs.local` domain. we use `localhost`

so change every host configuration on reticulum, dialog, hubs, hubs admin, spoke.

# 3. Setting up HTTPS (SSL)

All the servers must serve with HTTPS. you must generate a certificate and key file

## 3.1 Generating certificate and making it trust

Open terminal in reticulum directory

run command

```bash
mix phx.gen.cert
```

It will generate key `selfsigned_key.pem` and certificate `selfsigned.pem` in the `priv/cert` folder

Rename `selfsigned_key.pem` to `key.pem`

Rename `selfsigned.pem` to `cert.pem`

#### Now we have `key.pem` and `cert.pem` file

In Mac OS, I don't know in windows or Linux. please find it yourself

Open the `cert.pem` on the tab system find that certificate then clicks twice and change to always trust.

![Https mozilla hubs](/docs_img/cert.png)

Select the `cert.pem` and `key.pem` and copy it. next step we will distribute those two files into hubs, hubs admin, spoke, dialog, and reticulum.

Oke first set up in the reticulum.

## 3.2 Setting https for reticulum

On the `config/dev.exs` We must be setting the path for the certificate and key file.

![Https mozilla hubs](/docs_img/cert_1.png)

## 3.3 Setting HTTPS for hubs

Paste [that](#now-we-have-keypem-and-certpem-file) file into `hubs/certs`

We run hubs with `npm run local` right?
so add additional params on `package.json`

`--https --cert certs/cert.pem --key certs/key.pem`

Like this picture

![ssl hubs](/docs_img/ssl_hubs.png)

## 3.4 Setting HTTPS for hubs admin

Paste [that](#now-we-have-keypem-and-certpem-file) file into `hubs/admin/certs`

We run hubs with `npm run local` right?
so add additional params on `package.json`

`--https --cert certs/cert.pem --key certs/key.pem`

Like this picture

![ssl hubs admin](/docs_img/ssl_hubs_admin.png)

## 3.5 Setting HTTPS for spoke

Paste [that](#now-we-have-keypem-and-certpem-file) file into `spoke/certs`

We run spoke with `yarn start` right?
So change the `start` command

![ssl hubs admin](/docs_img/ssl_spoke.png)

With this

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.pem --key certs/key.pem
```

Short description:

BASE_ASSETS_PATH = basicaly we run the spoke on localhost:9090

## 3.6 Setting https for dialog

Paste [that](#now-we-have-keypem-and-certpem-file) file into `dialog/certs`

rename `cert.pem` to `fullchain.pem`

rename `key.pem` to `privkey.pem`

![ssl hubs dialog](/docs_img/ssl_dialog_1.png)

# 4. Running

Open five terminals. for each reticulum, dialog, spoke, hubs, hubs admin.

![Running preparation](/docs_img/ss.png)

## 4.1 Run reticulum

with command

```bash
iex -S mix phx.server
```

## 4.2 Run dialog

Edit the `start` command on the package.json with 

```
MEDIASOUP_LISTEN_IP=127.0.0.1 MEDIASOUP_ANNOUNCED_IP=127.0.0.1 DEBUG=${DEBUG:='*mediasoup* *INFO* *WARN* *ERROR*'} INTERACTIVE=${INTERACTIVE:='true'} node index.js
```

For giving params `MEDIASOUP_LISTEN_IP` and `MEDIASOUP_ANNOUNCED_IP`

![Running preparation](/docs_img/run_dialog.png)

Start dialog server with command:
```
npm run start
```

`127.0.0.1` is the default IP of localhost on Mac / Linux you can look at the IP with this command:

```bash
sudo nano /etc/hosts
```

## 4.3 Run spoke

with command

```bash
yarn start
```

## 4.4 Run hubs and hubs admin

Each with command

```bash
npm run local
```

## 4.5 Run postgREST server

More about this is in [this](https://github.com/mozilla/hubs-ops/wiki/Running-PostgREST-locally)

Download postREST

```
sudo apt install libpq-dev
wget https://github.com/PostgREST/postgrest/releases/download/v9.0.0/postgrest-v9.0.0-linux-static-x64.tar.xz
tar -xf postgrest-v9.0.0-linux-static-x64.tar.xz
```

On reticulum iex

paste this
```
jwk = Application.get_env(:ret, Ret.PermsToken)[:perms_key] |> JOSE.JWK.from_pem(); JOSE.JWK.to_file("reticulum-jwk.json", jwk)
```

then it will create `reticulum-jwk.json` in your reticulum directory

Make `reticulum.conf` file 

```
nano reticulum.conf
```
and paste 

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

```
/
   postgrest
   reticulum.conf
```

then run  postREST with

```
postgrest reticulum.conf
```
<br>
<br>

Urraaaa, Now you can access

with lock symbol (SSL secure)

Hubs

[https://localhost:4000](https://localhost:4000)

Hubs admin

[https://localhost:4000/admin](https://localhost:4000/admin)

Spoke

[https://localhost:4000/spoke](https://localhost:4000/spoke)


<br>
<br>

[Paypal](https://paypal.me/AlbirrKarim)

<a href='https://ko-fi.com/Q5Q0BC92X' target='_blank'><img height='36' style='border:0px;height:36px;' src='https://cdn.ko-fi.com/cdn/kofi3.png?v=3' border='0' alt='Buy Me a Coffee at ko-fi.com' /></a>


## Also read:

[Hosting Mozilla Hubs on VPS](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/VPS_FOR_HUBS.md)

[The Problem I Still Faced](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/PROBLEM_UNSOLVED.md)

[The Problem I Faced and I Already Solved](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/PROBLEM_SOLVED.md)

[Tips for Modification](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/HOW_TO_MODIFY.md)

[Overview System With Figma](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1)

[Experience Sharing About Hosting on Other Server](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/EXPERIENCE.md)
