# 소개

이 문서는 Local 에 mozilla 허브를 설치하고 실행하는 방법에 대한 내용이 담겨있습니다.

Windows 10 Pro(version 21H2) + WSL2 + Ubuntu 20.04.4 LTS  기반에서 작업이 진행되었습니다.


# 로컬로 Mozilla Hubs 설치

이 문서는 [albirrkarim/mozilla-hubs-installation-detailed](https://github.com/albirrkarim/mozilla-hubs-installation-detailed) 를 기반으로 제작되었습니다.

해당문서의 튜토리얼 영상 [youtube video](https://youtu.be/KH1T9u9DaCo). 

단, 튜토리얼 영상은 맥북으로 작성되어 MacOS 기반입니다. 그래서 이 문서와는 조금 차이가 있습니다. 주의하세요.

<br>

이것은 로컬에서 Mozilla 허브를 실행하는 것에 관한 것입니다. 설치 방벙을 단계별로 자세히 설명한 버전입니다.

기억하세요! 1시간 동안 해결할 수 없는 npm 또는 종속성에 문제가 있는 경우. PC를 다시 시작하면 됩니다.(100% 해결방법은 아닙니다.)

VPS 에서 Hubs를 설치하는 방법은 다음문서를 참고해주세요. [Hosting Mozilla Hubs on VPS](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/VPS_FOR_HUBS.md)

<br>
<br>

면책 조항 - 주의 !! 이 튜토리얼은 모범 사례가 아닐 수 있습니다 !!

<br>
<br>

# 요구사항:

### 하드웨어:

- 최소 8GB RAM
- 빠른 CPU 사용 권장

### 소프트웨어:

- 우분투 리눅스 20.04.4 LTS
- node.js가 설치되었습니다. 허브를 설치할 때 버전은 v16을 사용합니다.

### 필요한 지식

- Javascript
- React js
- Webpack dev server
- Elixir and phoenix
- Web Socket

# 개요

![시스템 오버뷰](/docs_img/System_Overview.png)

[figma](https://www.figma.com/) 로 만든 위의 이미지 는 [문서](https://hubs.mozilla.com/docs/system-overview.html) 에서 더 많은 설명을 읽을 수 있습니다.

또한 데이터베이스에 대한 소프트웨어 개요, 아키텍처 및 테이블을 만들려고 합니다. [figma 프로젝트](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1) 를 볼 수 있습니다

### 요약

Reticulum - 주요 호스트입니다. 위치, 회전, 개체의 상태를 동기화합니다. http 요청 및 websocket을 통해 클라이언트 브라우저와 통신합니다.

Dialog - 사용자의 비디오 및 오디오를 동기화 합니다. 웹 소켓을 통해 클라이언트 브라우저와 통신합니다.

Hubs, Spoke - static asset을 제공하고, Reticulum 이 이를 가져와 클라이언트 브라우저로 전달합니다.

postREST - 허브 관리자가 CRUD(읽기 업데이트 삭제 생성)와 같은 기본 작업을 수행하는 데 도움이 되는 서버입니다.

Hubs Admin - websocket을 사용하여 인증(로그인)을 위해 postgREST와 통신합니다. CRUD 목적의 허브 관리자는 http 요청(GET, POST 등)을 Reticulum 에 보낸 다음 Reticulum 은postgREST에 프록시 전달을 수행합니다.

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

PostgreSQL을 설치하려면 먼저 서버의 로컬 패키지 인덱스를 새로 고칩니다.
```
sudo apt update
```

그런 다음 몇 가지 추가 유틸리티 및 기능을 추가하는 `-contrib` 패키지와 함께 Postgres 패키지를 설치합니다.
```
sudo apt install postgresql postgresql-contrib
```

다음 명렁어로 postgresql 서비스를 시작합니다.
```
sudo service postgresql start
```

`* Starting PostgreSQL 12 database server       [OK]` 메시지가 나왔다면 정상적으로 postgresql 이 실행된것이다.

#### 주의
[튜토리얼](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart)에는 postgresql 서비스 시작을 위해 `sudo systemctl start postgresql.service` 를 명령어를 사용하는데, 다음과 같은 에러메시지가 나온다.
```
System has not been booted with systemd as init system (PID 1). Can't operate.
```

이 에러는 systemctl 을 사용해서 나오는것으로, WSL 우분투가 기본적으로 지원하지 않기 때문이다. 때문에 systemctl 을 사용할수가 없다.
우분투 리눅스에서 systemctl 을 사용 하기위해서는 추가로 작업을 하던지, 혹은 다른 방법(다른 명렁어)을 찾아야한다.



postgresql 의 실행을 확인하였다면.. 다음 작업으로 user 생성/password 변경합니다.

user: `postgres`

password : `postgres`

먼저 postgres 계정으로 psql 에 쉘에 접속합니다.

다음 명령어를 사용합니다.
```
sudo -i -u postgres
```
이때, root 권한의 비밀번호를 요구합니다. 비밀번호를 입력해줍니다.
postgres 계정으로 변경한 후, psql 쉘 실행

```
psql
```

`postgres` 계정의 비밀번호를 `postgres`로 바꾸기 위해 다음과 같이 psql 쉘 상에 입력한다.

```
ALTER USER postgres WITH PASSWORD 'postgres';
```

그리고 postgres 유저의 권한을 SUPERUSER 로 변경합니다.

```
ALTER USER postgres WITH SUPERUSER;
```

![img_00_01](https://user-images.githubusercontent.com/75593521/188348585-0c625408-d029-47c0-aa8d-ec9b772b144c.png)


#### Elixir and Erlang (Elixir 1.12 and erlang 버전 23)

이 [튜토리얼](https://www.pluralsight.com/guides/installing-elixir-erlang-with-asdf)을 따라 설치할 수 있습니다.

Elixir(버전 1.12)와 erlang(버전 23)의 버전에 주의해야합니다. 다른 버전을 설치 할 경우 지원을 하지 않거나 호환성에 문제가 있을 수 있습니다.

이 문서에서는 튜토리얼 영상에 맞춰 Elixir 1.12.3 과  erlang 23.3 버전으로 설치하도록 하겠습니다.

먼저 ASDF 를 복제합니다.

```
git clone https://github.com/asdf-vm/asdf.git ~/.asdf
```

그리고 다음을 명령어를 입력하여줍니다.

```
. $HOME/.asdf/asdf.sh
```

이제 ASDF 를 사용할 수 있습니다.

ASDF 를 이용하여 erlnag 과 elixir 설치를 시작합니다. 
면저 ASDF elixir 와 erlang 플러그인을 설치해야합니다.

```
asdf plugin add erlang
asdf plugin add elixir
```

플러그인 설치가 완료되면 erlang 23.3 을 설치합니다.
```
asdf install erlang 23.3
```


![img_04](https://user-images.githubusercontent.com/75593521/188382917-8f106c21-af59-46b2-8ed6-e72206ab2549.png)


erlang 설치를 시도하면 높은 확률로 필요로하는 패키지가 없다는 메시지와 함께 설치에 실패한다.
보통 이때 나오는 패키지는 `libssl-dev` `make` `autoconf` `automake` `libncurses5-dev` `gcc` 이다.
만약 추가로 요구되는 패키지가 있다면 인터넷 검색을 통해 해당 패키지를 설치하자.

필요로 하는 패키지를 설치해주자.

libssl-dev
```
sudo apt-get install libssl-dev
```

make
```
sudo apt install gcc make
```

autoconf
```
sudo apt install autoconf
```

automake
```
sudo apt install automake
```

libncurses5-dev
```
sudo apt install libncurses5-dev
```

gcc
```
sudo apt install gcc
```

이제 다시 ASDF 를 이용해 erlang 설치하면 문제없이 완료될것이다. erlang 설치는 보통 시간이 조금 걸리니 조금 기다리자.


![img_05](https://user-images.githubusercontent.com/75593521/188382970-50a80125-3287-4454-b5b7-c361b8018085.png)


erlang 설치가 완료되었다면 elixir 를 설치한다.
이때 elixir 의 설치에는 `unzip`이 필요하다. 커맨드 창에 `unzip` 을 입력했을때 unzip 을 찾을수 없다는 메시지가 나오면 unzip 을 설치한다.

```
sudo apt install unzip
```

`unzip -v` 로 unzip 이 제대로 설치됬는지 확인하자. 버전 정보가 나온다면 제대로 설치된것이다.
이제 다음 명령어로 elixir 를 설치한다.

 
```
asdf install elixir 1.12.3
```

erlang 과 달리 보통 빠르게 설치가 끝날것이다.

elixir 와 erlang 이 잘 설치가 됬는지 확인하자. `asdf list` 입력했을때 다음과같이 나오면 설치가 제대로 된 것이다.


![img_06](https://user-images.githubusercontent.com/75593521/188385289-117d375e-27e7-4597-a37c-50aa519e3cb5.png)


이제 프로젝트의 asdf 버전 설정을 해주어야 한다. reticulum 폴더로 이동하여 다음과 같이 입력하자.

```
asdf install
asdf local erlang 23.3
asdf locla elixir 1.12.3
```

`asdf install` 을 해당 폴더에서 입력해야 로컬 버전 설정이 가능해진다.

전역 설정도 가능하다.

```
asdf global erlang 23.3
asdf global elixir 1.12.3
```

만약 로컬 버전 설정이 없는 디렉토리라면 asdf 이용시 해당 버전으로 작동하게 된다.

<!-- **Ansible**

You can use `pip` to install. take a look at this [tutorial](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#from-pip) -->

### 1.1.3 다음 명령어를 실행

1. `mix deps.get`

#### 주의
`mix ecto.create` 를 실행하기전에 postgresql 이 온라인 상태인지 확인한다.

```
service postgresql status
```

만약 온라인 상태가 아니라면, postgresql 을 실행하여준다.

```
sudo service postgresql start
```


2. `mix ecto.create`


설치에 시간이 조금 걸린다. 명령어 수행이 완료되면 최종적으로 다음과 같은 메시지가 나타난다.

![img_09](https://user-images.githubusercontent.com/75593521/188405409-821a2987-129b-4ef3-9292-d9e54814b48f.png)

    - 2단계가 실패하면 `dev.exs`에 구성된 비밀번호와 일치하도록 `postgres` 의 비밀번호를 변경해야 할 수 있습니다.
    - `psql` 쉘 내에서 `ALTER USER postgres WITH PASSWORD 'postgres';`를 입력합니다.
    - `ret_dev` 데이터베이스가 존재하지 않는다는 오류가 발생하면 (psql 쉘을 다시 사용하여) `create database ret_dev;`를 입력합니다.

3.  프로젝트 디렉토리에서 `mkdir -p storage/dev`

### 1.1.4 로컬 Dialog 인스턴스에 대해 Reticulum 실행

1. `dev.exs`에서 Janus 호스트를 업데이트합니다:

`dev.exs`는 reticulum 프로젝트의 config 폴더안에 있습니다. 즉, `/reticulum/config/dev.exs`

```elixir
dev_janus_host = "localhost"
```

![img_12](https://user-images.githubusercontent.com/75593521/188417176-8aca9876-274d-4337-870f-69e36dd39f46.png)

2. `dev.exs`에서 Janus 포트를 업데이트합니다:

```elixir
config :ret, Ret.JanusLoadStatus, default_janus_host: dev_janus_host, janus_port: 4443
```
![img_11](https://user-images.githubusercontent.com/75593521/188417249-8eaa7f42-53b1-46bb-9870-49f829c62180.png)


3. `add_csp.ex`의 CSP 규칙에 dialog 메타 엔드포인트를 추가합니다:

`add_csp.ex`의 경로는 다음과 같습니다. `/reticulum/lib/ret_web/plugs/add_csp.ex`

```elixir
default_janus_csp_rule =
   if default_janus_host,
      do: "wss://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port}/meta",
      else: ""
```

![img_13](https://user-images.githubusercontent.com/75593521/188417309-e3af7a10-e277-4c5e-a989-a32dc59e724d.png)



4. Coturn 설치

참고 : [우분투에 Coturn 설치](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04)

다음 명령어를 사용하여 coturn 을 설치합니다.

```
sudo apt-get -y update
sudo apt-get install coturn
```

Dialog [구성 파일](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04) 편집`turnserver.conf` 및 Reticulum 데이터베이스의 _coturn_ 스키마를 사용하도록 PostgreSQL 데이터베이스 연결 문자열을 업데이트합니다:

다음 명령어로 turnserver.conf 설정 파일을 수정합니다.

혹은 폴더에 접근하여 윈도우즈 상에서 원하는 편집기로 수정해도 됩니다.

```
vi /etc/turnserver.conf
or
nano /etc/turnserver.conf
```

turnserver.conf 파일에 다음 내용을 붙여넣기 합니다.

```
psql-userdb="host=localhost dbname=ret_dev user=postgres password=postgres options='-c search_path=coturn' connect_timeout=30"
```

##### 이미지 추가

## 1.2 Dialog

Dialog 는 mediasoup RTC를 사용하여 오디오 및 비디오 실시간 통신을 처리합니다. 카메라 스트림과 같은 공유 화면에서 사용됩니다.

### 1.2.1 복제 및 종속성 가져오기

```bash
git clone https://github.com/mozilla/dialog.git
cd dialog
npm install
```

### 1.2.2 보안 키 설정

이 [코멘트](https://github.com/mozilla/hubs/discussions/3323#discussioncomment-1857495) 를 참고해주세요.

[generator online](https://travistidwell.com/jsencrypt/demo/)을 사용하여 RSA(공개 및 개인 키) 생성.

빈 파일 `perms.pub.pem`을 dialog 프로젝트의 certs 폴더안에 만들고 RSA 공개 키로 채웁니다.

```
mkdir certs
cd certs
vi perms.pub.pem
```

![RSA generator ](/docs_img/rsa.png)
![Paste](/docs_img/rsa_1.png)

`reticulum/config/dev.exs`의 reticulum 디렉토리로 이동하여 이전에 생성한 RSA 개인 키로 PermsToken을 변경합니다.

```elixir
config :ret, Ret.PermsToken, perms_key: "-----BEGIN RSA PRIVATE KEY----- paste here copyed key but add every line \n before END RSA PRIVATE KEY-----"
```
Public Key 와 달리 Private Key 는 약간의 수정이 필요하다.
모든줄에 `\n` 추가한다음, 한줄로 수정하여 `dev.exs` 에 붙여넣기 하여야 한다.

먼저 Private Key 복사한 다음 붙여넣기한다.

![img_15](https://user-images.githubusercontent.com/75593521/188557928-dcee72bd-2db9-4f17-a8fc-68c4f628a145.png)

그런 다음 모든 둘째줄부터 모든행의 앞에 `\n`을 넣어준다. 

![img_16](https://user-images.githubusercontent.com/75593521/188557966-1dab512a-02c0-4ba5-997e-cd625e925cec.png)

그 후, 전부 한줄로 만들어준다.

![img_17](https://user-images.githubusercontent.com/75593521/188557980-4df24ac4-9782-44e3-a25a-8e294b3f7594.png)

`dev.exs`의 `config :ret,  Ret.PermsToken, perms_key:` 라인을 찾아 복사한 후, 기존 줄은 주석처리하고 복사한 줄의 값은 공란으로 처리한다.

![img_18](https://user-images.githubusercontent.com/75593521/188558002-cf5beb07-6ecb-432c-a2ae-860361cff9fa.png)

값으로 방금 수정한 PRIVATE KEY 값을 입력한다.

![img_19](https://user-images.githubusercontent.com/75593521/188558011-22fd4645-b7c5-4e58-aaa4-3b6b36271dff.png)

이제 보안 키 설정이 완료되었습니다. 다음 단계로 넘어갑니다.

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

<!--
## 1.4 Hubs

이 [repo](https://github.com/mozilla/hubs)에는 허브 클라이언트 및 허브 관리자(hubs/admin)가 포함되어 있습니다.

![System Overview](/docs_img/hubs_overview.jpeg)

종속성 복제 및 설치

```
git clone https://github.com/mozilla/hubs.git
cd hubs
npm ci
```
-->


## 1.4 Hubs(larchiveum_hubs_reactjs)

이 [repo](https://github.com/geminisoft-vn/larchiveum_hubs_reactjs)에는 허브 클라이언트 및 허브 관리자(hubs/admin)가 포함되어 있습니다.

![System Overview](/docs_img/hubs_overview.jpeg)

종속성 복제 및 설치

```
git clone https://github.com/geminisoft-vn/larchiveum_hubs_reactjs
cd larchiveum_hubs_reactjs
npm ci
```

#### 주의

npm ci 를 통해 hubs 설치시 멈추는 현상이 발생할때가 있다.

다음과 같이...특히 emojione 부분에서 많이 멈춘다. 혹은 easyrtc.

![img_20](https://user-images.githubusercontent.com/75593521/188601108-c33bc301-51f3-4445-a8bb-7751c6775a95.png)

easyrtc의 경우는 package.json 파일을 최신으로 업데이트하면 근본적으로 해결된다. 

하지만 아래 방법으로도 해결이 어느정도 가능하다.

보통 이 문제의 원인은 node.js 와 npm 의 버전이 너무 낮기 때문이다.

`node -v` 와 `npm -v`로 버전을 확인해 본다.

정상적으로 작동되는 nodejs와 npm 의 버전은 

`node.js v16.16.0` 이상, `npm 8.14.0` 이상 이다. 

만약 위의 버전보다 낮아서 문제가 발생한다면 아래의 방법으로 해결해보자.

<br>

먼저 nodejs 부터 설치한다.

빌드 관련 툴 부터 설치한다. 안해도되지만, 하는것이 더 좋다.

```
sudo apt install -y build-essential
```

이제 nodejs 의 LTS(Long Term Support, 장기 지원) 버전을 설치한다.

```
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs
```

마찬가지로 npm 도 최신버전으로 설치한다.

```
npm install -g npm
```

이제 `npm ci`로 다시 설치를 시작해보자. 설치가 정상적으로 끝날것이다.  


![img_21](https://user-images.githubusercontent.com/75593521/188605127-657480ec-c01c-4764-91e3-2d8053c55d22.png)



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
