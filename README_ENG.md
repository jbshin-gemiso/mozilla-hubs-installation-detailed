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

# Attention!

major step - [Cloning and Preparation](#1-cloning-and-preparation) -> [Setting up HOST](#2-setting-up-host) -> [Setting up HTTPS (SSL)](#3-setting-up-https-ssl) -> [Running](#4-runing)

# 1. Cloning and preparation

## 1.1 Reticulum

It's a backend server that uses elixir and phoenix.

### 1.1.1 Clone

```bash
git clone https://github.com/mozilla/reticulum.git
cd reticulum
```

### 1.1.2 Install requirement

#### Postgres Database

install on [linux ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart)

Install on mac

With brew for installing CLI Postgres

```
brew install postgres
```

그런 다음 user 생성/password 변경

Then create user/change password

user: `postgres`

password : `postgres`

And change the privileges of the postgres user to SUPERUSER

```
ALTER USER postgres WITH SUPERUSER
```

#### Elixir and Erlang (Elixir 1.12 and erlang version 23)

You can install those with follow [tutorial](https://www.pluralsight.com/guides/installing-elixir-erlang-with-asdf)

Note the versions of Elixir (version 1.12) and erlang (version 23). If you install a different version, it may not be supported or there may be compatibility issues.


<!-- **Ansible**

You can use `pip` to install. take a look at this [tutorial](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#from-pip) -->

### 1.1.3 run this command

1. `mix deps.get`
2. `mix ecto.create`
    - If step 2 fails, you may need to change the password for the `postgres` role to match the password configured `dev.exs`.
    - From within the `psql` shell, enter `ALTER USER postgres WITH PASSWORD 'postgres';`
    - If you receive an error that the `ret_dev` database does not exist, (using psql again) enter `create database ret_dev;`
3.  From the project directory `mkdir -p storage/dev`

### 1.1.4 Run Reticulum against a local Dialog instance

1. Update the Janus host in `dev.exs`:

```elixir
dev_janus_host = "localhost"
```

2. Update the Janus port in`dev.exs`:

```elixir
config :ret, Ret.JanusLoadStatus, default_janus_host: dev_janus_host, janus_port: 4443
```

3. Add the Dialog meta endpoint to the CSP rules in `add_csp.ex`:

```elixir
default_janus_csp_rule =
   if default_janus_host,
      do: "wss://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port}/meta",
      else: ""
```

4. Find on google how to install coturn, and manage it

[install coturn on ubuntu](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04)

5. Edit the Dialog [configuration file](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18-04) `turnserver.conf` and update the PostgreSQL database connection string to use the coturn schema from the Reticulum database:

```
psql-userdb="host=localhost dbname=ret_dev user=postgres password=postgres options='-c search_path=coturn' connect_timeout=30"
```

## 1.2 Dialog

Using mediasoup RTC will handle audio and video real-time communication. like camera stream, share screen..

### 1.2.1 Clone and get dependencies

```bash
git clone https://github.com/mozilla/dialog.git
cd dialog
npm install
```

### 1.2.2 Setting up secret key

Please refer to this [comment](https://github.com/mozilla/hubs/discussions/3323#discussioncomment-1857495)

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

### 1.3.2  Set the base routes

I hope you know the basic `react-router-dom` with the default URL in  slash `/` on `localhost:9090`

But in the end, we will access the spoke on `localhost:4000/spoke`

So we must set the base URL to `/spoke`

Add the `ROUTER_BASE_PATH=/spoke` params to the `start` command on `package.json`


![Mozilla Spoke](/docs_img/spoke_change.png)

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.pem --key certs/key.pem
```

## 1.4 Hubs

In this [repo](https://github.com/mozilla/hubs)contains the hubs client and hubs admin (hubs/admin).

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

so change every host configuration(hubs.local -> localhost) on reticulum, dialog, hubs, hubs admin, spoke.

See the tutorial [video](https://youtu.be/KH1T9u9DaCo?t=1482)
<br>

There are two main ways to replace `hubs.local` -> `localhost`

1. Access that folder and replace it with an editor such as Visual Studio Code or Sublime Text.

2. Replaces all the corresponding words in a specific folder with a Linux command.

<br>

## 2.1 Substitution via editor

After accessing the folder on Linux, enter the following command.

```
explorer.exe .
```

This will pop up the folder into a Windows Explorer window. So, now you can access the WSL2 Linux folder from Windows.

Now that you know the folder path, use an editor based on this and edit `hubs.local` to `localhost`.

<br>

## 2.2 Substitution via Linux commands

After accessing the folder on Linux, enter a command to replace a specific string of all documents in the folder. The command is:

```
$ find ./ -type f |xargs sed -i 's/{old String}/{new String}/g'

ex) If you want to change TEST to test...

$ find ./ -type f |xargs sed -i 's/TEST/test/g'
```

<br>

# 3. Setting up HTTPS (SSL)

All the servers must serve with HTTPS. you must generate a certificate and key file

## 3.1 Generating certificate and making it trust

Open terminal in reticulum directory,

Generate the certificate and key file by run command.


```bash
mix phx.gen.cert
```


It will generate key `selfsigned_key.pem` and certificate `selfsigned.pem` in the `priv/cert` folder

Rename `selfsigned_key.pem` to `key.pem`

Rename `selfsigned.pem` to `cert.pem`

<br>

#### Now we have`key.pem` and `cert.pem` file.


First, you need to register the certificate as a trusted root certificate in Ubuntu Linux.
<br>

For a `cert.pem` certificate, you must change it to a `cert.crt` certificate.

```bash
openssl x509 -in cert.pem -inform pem -out cert.crt
```

Copy the `cert.crt` file to the directory below. (create the directory if it doesn't exist)

```bash
cp cert.crt /usr/share/ca-certificates/extra
```

Have Ubuntu linux add the certificate as trusted.

```bash
dpkg-reconfigure ca-certificates
```


Now the `cert.pem (cert.crt)` certificate has been added to Linux as a trusted certificate.

윈도우에도 동일한 작업을 해주면 윈도우에서 브라우저 접속시 경고메시지가 안뜨게 됩니다.
If you do the same on Windows, the warning message will not appear when accessing the browser on Windows.

---- 윈도우에 인증서 등록하는 방법 추가... 윈도우즈 10기준


![Https mozilla hubs](/docs_img/cert.png)

Select the `cert.crt` and `key.pem` and copy it. next step we will distribute those two files into Hubs, Hubs admin, Spoke, Dialog, and Reticulum.

OK, first set up in the Reticulum.

## 3.2 Setting https for reticulum

On the `config/dev.exs` We must be setting the path for the certificate and key file.

![Https mozilla hubs](/docs_img/cert_1.png)

## 3.3 Setting HTTPS for Hubs

Paste that [file](#now-we-have-keypem-and-certpem-file) into `hubs/certs`

We run hubs with `npm run local`

so add additional params on `package.json`

`--https --cert certs/cert.crt --key certs/key.pem`

Like this picture

![ssl hubs](/docs_img/ssl_hubs.png)

## 3.4 Setting HTTPS for Hubs admin

Paste that [file](#now-we-have-keypem-and-certpem-file) into `hubs/admin/certs`

We run Hubs admin with `npm run local`

so add additional params on `package.json`

`--https --cert certs/cert.crt --key certs/key.pem`

Like this picture

![ssl hubs admin](/docs_img/ssl_hubs_admin.png)

## 3.5 Setting HTTPS for spoke


Paste that [file](#now-we-have-keypem-and-certpem-file) into `spoke/certs`

We run spoke with `yarn start`

So change the `start` command

![ssl hubs admin](/docs_img/ssl_spoke.png)

And modify it like this:

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.pem --key certs/key.pem
```

Short description:

BASE_ASSETS_PATH = basicaly we run the spoke on localhost:9090

## 3.6 Setting https for dialog

Paste that [file](#now-we-have-keypem-and-certpem-file) into `dialog/certs`

rename `cert.crt` to `fullchain.pem`

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

Paste the following command and run it.
```
jwk = Application.get_env(:ret, Ret.PermsToken)[:perms_key] |> JOSE.JWK.from_pem(); JOSE.JWK.to_file("reticulum-jwk.json", jwk)
```

then it will create `reticulum-jwk.json` in your reticulum directory

Make `reticulum.conf` file 

```
nano reticulum.conf
```
and paste 

And paste the following content into the created file.

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

At this time, if it says that the postgrest path cannot be found, enter the full directory path before postgrest.(Path to the directory where the `reticulum.conf` file is located)

```
ex) /home/ubuntu/hubs/reticulum/postgrest reticulum.conf
```

<br>
<br>

When everything is running, you can finally connect to Hubs.

with lock symbol (SSL secure)

Hubs

[https://localhost:4000](https://localhost:4000)

Hubs admin

[https://localhost:4000/admin](https://localhost:4000/admin)

Spoke

[https://localhost:4000/spoke](https://localhost:4000/spoke)


<br>
<br>




## Also read : 

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
