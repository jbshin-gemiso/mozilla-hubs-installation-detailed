
# Introduce

This document contains information on how to install and run mozilla hub in Local.

Windows 10 Pro (version 21H2) + WSL2 + Ubuntu 20.04.4 LTS based on the work was done.

# Install Mozilla Hubs locally

This document is based on [albirrkarim/mozilla-hubs-installation-detailed](https://github.com/albirrkarim/mozilla-hubs-installation-detailed).

Tutorial video [youtube video](https://youtu.be/KH1T9u9DaCo) of the document .

However, the tutorial video was written on a MacBook and is based on MacOS. So, it is a little different from this document. Be careful.

<br>

This is about running the Mozilla hub locally. This is a detailed step-by-step version of how to install.

Remember! If you have a problem with npm or a dependency that you can't solve for an hour. Just restart your PC. (Not a 100% solution.)

For how to install Hubs on VPS, please refer to the following article. [Hosting Mozilla Hubs on VPS](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/VPS_FOR_HUBS.md)

<br>
<br>

Disclaimer - Attention!! This tutorial may not be best practice!!

<br>
<br>

# Requirements:

### hardware:

- Minimum 8GB RAM
- Recommended to use fast CPU

### software:

- Ubuntu Linux 20.04.4 LTS
- node.js is installed. When installing the hub, the version is v16.

### Required knowledge

- Javascript
- React js
- Webpack dev server
- Elixir and phoenix
- Web Socket


# Overview

![System Overview](/docs_img/System_Overview.png)

Above image made with [figma](https://www.figma.com/) You can read more explanation at [Documentation](https://hubs.mozilla.com/docs/system-overview.html) there is.

You will also want to create a software overview, architecture, and tables for your database. You can see [figma project](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1)

### Summary

Reticulum - The main host. Synchronize the position, rotation, and state of objects. It communicates with the client browser via http requests and websockets.

Dialog - Synchronize the user's video and audio. It communicates with the client browser via websockets.

Hubs, Spokes - Provides static assets, Reticulum fetches them and passes them to client browsers.

postREST - A server that helps hub administrators perform basic operations such as create read update delete (CRUD).

Hubs Admin - Use websocket to communicate with postgREST for authentication (login). For CRUD purposes, the hub manager sends an http request (GET, POST, etc) to Reticulum, which then proxy forwards to postgREST.

# Attention!

There is Major Steps - [Cloning and Preparation](#1-cloning-and-preparation) -> [Setting up HOST](#2-setting-up-host) -> [Setting up HTTPS(SSL)](#3-setting-up-https-ssl) -> [Running](#4-running)

<br>

# 1. Cloning and Preparation


## 1.1 Reticulum

It's a backend server using elixir and phoenix.


### 1.1.1 Clone

```bash
git clone https://github.com/mozilla/reticulum.git
cd reticulum
```

### 1.1.2 Install  Requirements

#### Postgres database

Install  on [Linux](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart)

To install PostgreSQL, first refresh the server's local package index.

```
sudo apt update
```

Then install the Postgres package along with the `-contrib` package which adds some additional utilities and features.

```
sudo apt install postgresql postgresql-contrib
```

Start the postgresql service with the following command.

```
sudo service postgresql start
```

If the message `* Starting PostgreSQL 12 database server [OK]` appears, postgresql has been successfully executed.

#### Caution

[Tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart) has `sudo systemctl start postgresql.service` to start the postgresql service. command, the following error message appears:

```
System has not been booted with systemd as init system (PID 1). Can't operate.
```

This error is caused by using systemctl because WSL Ubuntu does not support it by default. Therefore, systemctl cannot be used.
In order to use systemctl in Ubuntu Linux, you need to do additional work or find another way (different command).


If you have verified that postgresql is running.. Create user/change password with the following tasks.

user: `postgres`

password : `postgres`

First, connect to the psql shell with the postgres account.

Use the following command:

```
sudo -i -u postgres
```

At this time, a password with root authority is requested. Enter your password.
After changing to the postgres account, run the psql shell

```
psql
```

And change the privileges of the postgres user to SUPERUSER.

```
ALTER USER postgres WITH SUPERUSER;
```

![img_00_01](https://user-images.githubusercontent.com/75593521/188348585-0c625408-d029-47c0-aa8d-ec9b772b144c.png)


#### Elixir and Erlang (Elixir 1.12 and erlang version 23)

You can install it by following this [tutorial](https://www.pluralsight.com/guides/installing-elixir-erlang-with-asdf).

Note the versions of Elixir (version 1.12) and erlang (version 23). If you install a different version, it may not be supported or there may be compatibility issues.

In this document, we will install Elixir 1.12.3 and erlang 23.3 in accordance with the tutorial video.

First, clone the ASDF.


```
git clone https://github.com/asdf-vm/asdf.git ~/.asdf
```

And enter the following command:

```
. $HOME/.asdf/asdf.sh
```

You can now use ASDF.

Start installing erlnag and elixir using ASDF.
If you want, you need to install ASDF elixir and erlang plugin.

```
asdf plugin add erlang
asdf plugin add elixir
```

After the plugin installation is complete, install erlang 23.3.

```
asdf install erlang 23.3
```

![img_04](https://user-images.githubusercontent.com/75593521/188382917-8f106c21-af59-46b2-8ed6-e72206ab2549.png)

When you try to install erlang, it is highly likely that the installation will fail with a message stating that the required package is missing.
The packages usually appear at this time are `libssl-dev` `make` `autoconf` `automake` `libncurses5-dev` `gcc`.
If there are additional required packages, install them through an Internet search.

Install the packages you need.

* libssl-dev
```
sudo apt-get install libssl-dev
```

* make
```
sudo apt install gcc make
```

* autoconf
```
sudo apt install autoconf
```

* automake
```
sudo apt install automake
```

* libncurses5-dev
```
sudo apt install libncurses5-dev
```

* gcc
```
sudo apt install gcc
```

Now, install erlang again using ASDF and it will be completed without any problems. Installing erlang usually takes a little while, so be patient.

![img_05](https://user-images.githubusercontent.com/75593521/188382970-50a80125-3287-4454-b5b7-c361b8018085.png)

After installing erlang, install elixir.
At this time, `unzip` is required to install elixir. If you enter `unzip` in the command window and get a message that unzip cannot be found, install unzip.

```
sudo apt install unzip
```

Check if unzip is installed properly with `unzip -v`. If the version information is displayed, it is installed properly.
Now install elixir with the following command.

```
asdf install elixir 1.12.3
```

Unlike erlang, the installation will usually finish quickly.

Check that elixir and erlang are installed properly. When you enter `asdf list`, the following is displayed, indicating that the installation was successful.

![img_06](https://user-images.githubusercontent.com/75593521/188385289-117d375e-27e7-4597-a37c-50aa519e3cb5.png)

Now you need to set the asdf version of the project. Go to the reticulum folder and type the following:

```
asdf install
asdf local erlang 23.3
asdf locla elixir 1.12.3
```

You must enter `asdf install` in the folder to set the local version.

Global settings are also possible.

```
asdf global erlang 23.3
asdf global elixir 1.12.3
```

If the directory does not have a local version setting, it will work with the corresponding version when using asdf.

<!-- **Ansible**

You can use `pip` to install. take a look at this [tutorial](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#from-pip) -->

### 1.1.3 Run this command

1. `mix deps.get`

#### Caution
Make sure postgresql is online before running `mix ecto.create`.

```
service postgresql status
```

If not online, run postgresql.


```
sudo service postgresql start
```

2. `mix ecto.create`

It will take some time to install. When the command execution is completed, the following message is finally displayed.

![img_09](https://user-images.githubusercontent.com/75593521/188405409-821a2987-129b-4ef3-9292-d9e54814b48f.png)

    - If step 2 fails, you may need to change the password in `postgres` to match the password configured in `dev.exs`.
    - Inside the `psql` shell, type `ALTER USER postgres WITH PASSWORD 'postgres';`.
    - If you get an error that the `ret_dev` database does not exist (using the psql shell again), type `create database ret_dev;`.
    
3. In the project directory `mkdir -p storage/dev`


### 1.1.4 Run Reticulum against a local Dialog instance

1. Update the Janus host in `dev.exs`:

`dev.exs` is located in the config folder of the reticulum project. i.e. `/reticulum/config/dev.exs`

```elixir
dev_janus_host = "localhost"
```

![img_12](https://user-images.githubusercontent.com/75593521/188417176-8aca9876-274d-4337-870f-69e36dd39f46.png)

2. Update Janus port in `dev.exs`:

```elixir
config :ret, Ret.JanusLoadStatus, default_janus_host: dev_janus_host, janus_port: 4443
```

![img_11](https://user-images.githubusercontent.com/75593521/188417249-8eaa7f42-53b1-46bb-9870-49f829c62180.png)

3. Add the dialog meta endpoint to the CSP rule in `add_csp.ex`:

The path of `add_csp.ex` is as follows. `/reticulum/lib/ret_web/plugs/add_csp.ex`


```elixir
default_janus_csp_rule =
   if default_janus_host,
      do: "wss://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port} https://#{default_janus_host}:#{janus_port}/meta",
      else: ""
```

![img_13](https://user-images.githubusercontent.com/75593521/188417309-e3af7a10-e277-4c5e-a989-a32dc59e724d.png)


4. install Coturn

Note: [Install Coturn on Ubuntu](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu -18-04)

Install coturn using the following command:

```
sudo apt-get -y update
sudo apt-get install coturn
```

Dialog [config file](https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18- 04) Edit `turnserver.conf` and update the PostgreSQL database connection string to use the _coturn_ schema of the Reticulum database:

Modify the turnserver.conf configuration file with the following command.

Alternatively, you can access the folder and edit it with the editor you want on Windows.

```
vi /etc/turnserver.conf
or
nano /etc/turnserver.conf
```

Paste the following into the turnserver.conf file.

```
psql-userdb="host=localhost dbname=ret_dev user=postgres password=postgres options='-c search_path=coturn' connect_timeout=30"
```

##### Add image

## 1.2 Dialog

Dialog uses mediasoup RTC to handle audio and video real-time communication. Used in shared screens such as camera streams.

### 1.2.1 Cloning and importing dependencies

```bash
git clone https://github.com/mozilla/dialog.git
cd dialog
npm install
```

### 1.2.2 Set Security Key

Please refer to this [comment](https://github.com/mozilla/hubs/discussions/3323#discussioncomment-1857495).

Generate public and private keys (RSA) using [generator online](https://travistidwell.com/jsencrypt/demo/).

Create an empty file `perms.pub.pem` inside your dialog project's certs folder and fill it with your RSA public key.

```
mkdir certs
cd certs
vi perms.pub.pem
```

![RSA generator ](/docs_img/rsa.png)
![Paste](/docs_img/rsa_1.png)

Go to the reticulum directory in `reticulum/config/dev.exs` and change the PermsToken with the RSA private key you created earlier.

```elixir
config :ret, Ret.PermsToken, perms_key: "-----BEGIN RSA PRIVATE KEY----- paste here copyed key but add every line \n before END RSA PRIVATE KEY-----"
```

Unlike the public key, the private key needs some modification.
After adding `\n` to every line, you need to edit it into one line and paste it into `dev.exs`.

First, copy and paste the private key.

![img_15](https://user-images.githubusercontent.com/75593521/18857928-dcee72bd-2db9-4f17-a8fc-68c4f628a145.png)

Then, put `\n` in front of every line from every second line.

![img_16](https://user-images.githubusercontent.com/75593521/188557966-1dab512a-02c0-4ba5-997e-cd625e925cec.png)

After that, make it all in one line.

![img_17](https://user-images.githubusercontent.com/75593521/188557980-4df24ac4-9782-44e3-a25a-8e294b3f7594.png)

After finding and copying the line `config :ret, Ret.PermsToken, perms_key:` of `dev.exs`, comment out the existing line and leave the value of the copied line blank.

![img_18](https://user-images.githubusercontent.com/75593521/188558002-cf5beb07-6ecb-432c-a2ae-860361cff9fa.png)

Enter the PRIVATE KEY value that was just modified as the value.

![img_19](https://user-images.githubusercontent.com/75593521/188558011-22fd4645-b7c5-4e58-aaa4-3b6b36271dff.png)

Security key setup is now complete. Go to the next step.

## 1.3 Spoke

Editor of Hubs. You can create and edit/distribute scenes.

### 1.3.1 Clone

![Mozilla Spoke](/docs_img/spoke.png)

```
git clone https://github.com/mozilla/Spoke.git
cd Spoke
yarn install
```

### 1.3.2 Set the base routes

I hope you know basically `react-router-dom` with the base url at the slash `/` in `localhost:9090`.

(I hope you know the basic `react-router-dom` with the default URL in slash `/` on `localhost:9090`)

Anyway, in the end, you will access it using `localhost:4000/spoke` as the spoke address.

So you need to set the base URL to `/spoke`.

Add the `ROUTER_BASE_PATH=/spoke` parameter to the `start` command in `package.json`.


![Mozilla Spoke](/docs_img/spoke_change.png)


```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.crt --key certs/key.pem
```

## 1.4 Hubs (larchiveum_hubs_reactjs)

This [repo](https://github.com/geminisoft-vn/larchiveum_hubs_reactjs) contains the hub client and hub admin (hubs/admin).

![System Overview](/docs_img/hubs_overview.jpeg)

Clone and install dependencies

```
git clone https://github.com/geminisoft-vn/larchiveum_hubs_reactjs
cd larchiveum_hubs_reactjs
npm ci
```

#### caution

When installing hubs through npm ci, sometimes it hangs.

As follows...especially the 'emojione' part stops a lot. or 'easyrtc'.

![img_20](https://user-images.githubusercontent.com/75593521/188601108-c33bc301-51f3-4445-a8bb-7751c6775a95.png)

In the case of 'easyrtc', it is fundamentally solved by updating the package.json file to the latest.

<br>

However, it can be solved with the method below.

Usually the cause of this problem is that the versions of node.js and npm are too low.

Check the version with `node -v` and `npm -v`.

The versions of nodejs and npm that work normally are

It is `node.js v16.16.0` or later, `npm 8.14.0` or later.

If the problem occurs because the version is lower than the above version, try the method below.

<br>

First, install nodejs.

Install the build-related tools first. You don't have to, but it's better to do it.

```
sudo apt install -y build-essential
```

Now install the LTS (Long Term Support) version of nodejs.

```
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt-get install -y nodejs
```

Similarly, install npm with the latest version.

```
npm install -g npm
```

Now let's start the installation again with `npm ci`. The installation will finish normally.

![img_21](https://user-images.githubusercontent.com/75593521/188605127-657480ec-c01c-4764-91e3-2d8053c55d22.png)

After installation, never type npm audit . It is unnecessary and may require reinstallation.


## 1.5 Hubs Admin

You can go to `hubs/admin` in the [hubs repo](#14-hubs) and then run it.


```
npm install
```

Likewise, never type npm audit . It is unnecessary and may require reinstallation.

## 1.6 API Server (Larchiveum API Server)
### 1.6.1 Install MySQL

install mysql server
```bash
sudo apt install mysql-server
```
start mysql service
```bash
sudo service mysql start
```
login to mysql by administrator
```bash
sudo mysql -u root -p
```
create mysql user
```bash
mysql> CREATE USER 'larchiveum'@'localhost' IDENTIFIED BY 'larchiveum';
```
grant privileges
```bash
mysql> GRANT ALL PRIVILEGES ON *.* TO 'larchiveum'@'localhost';
```
quit mysql
```bash
mysql> \q
```
login to mysql by user larchiveum 
```bash
sudo mysql -u larchiveum -p
```
create database
```bash
mysql> CREATE DATABASE larchiveum;
```
quit mysql
```bash
mysql> \q
```

### 1.6.2 Clone and project Larchiveum API Nodejs
clone project (Your git account must has permision to access this repository)
```bash
git clone https://github.com/jbshin-gemiso/mozilla-hubs-installation-detailed.git
```

* Nodejs with npm is already installed

install dependencies
```bash
npm install 
npm install -g pm2 
```
genarate API document
```bash
npm run genAPIDoc 
```

### 1.6.3 Import database
go to setup folder
```bash
cd larchiveum_api_nodejs/setup/
```
import database
```bash
sudo mysql -u larchiveum -p larchiveum < database.sql
```
check inport was successful or not?
```bash
sudo mysql -u larchiveum -p
mysql> SHOW TABLES;
```
![System Overview](/docs_img/1.6.3.check_import_databse.png)

# 2. Setting up HOST

We do not use the `hubs.local` domain. We use `localhost`

So you need to change all host configuration (hubs.local -> localhost) for reticulum, dialog, hubs, hubs admin, spoke.

See the tutorial [video](https://youtu.be/KH1T9u9DaCo?t=1482).
<br>

There are two main ways to replace `hubs.local` -> `localhost`.

1. How to access the folder and replace it with an editor such as Visual Studio Code or Sublime Text.

2. Replace all relevant words in a specific folder with Linux commands.


<br>

## 2.1 Substitution via editor

After accessing the folder on Linux, enter the following command.

```
explorer.exe .
```

This will make the folder appear as a Windows Explorer window. This will allow you to access the WSL2 Linux folder from Windows.

Now that you know the folder path, use an editor based on this and edit `hubs.local` to `localhost`.

<br>

## 2.2 Substitution via Linux commands

After accessing the folder on Linux, enter a command to replace a specific string of all documents in the folder. The command is:

```
$ find ./ -type f xargs sed -i 's/{replace string}/{new string}/g'

Ex) If you want to change Test to test...

$ find ./ -type f xargs sed -i 's/TEST/test/g'
```

# 3. Setting up HTTPS (SSL)

All servers must come with HTTPS. For this reason, you must create a certificate and key file.

## 3.1 Generating certificate and making it trust

Open a terminal in the reticulum directory,

Generate a certificate and key file by running the following command:

```bash
mix phx.gen.cert
```

It will generate key `selfsigned_key.pem` and certificate `selfsigned.pem` in the `priv/cert` folder

Create a `self signed key.pem` key and a certificate `self signed.pem` in the `priv/cert` folder.

Rename `selfsigned_key.pem` to `key.pem`.

Rename `selfsigned.pem` to `cert.pem`.

<br>

#### You now have the `key.pem` and `cert.pem` files.


First, add the certificate as a trusted root certificate in Linux. It is written based on Ubuntu Linux.

<br>

For a `cert.pem` certificate, you must change it to a `cert.crt` certificate.

```bash
openssl x509 -in cert.pem -inform pem -out cert.crt
```

Copy the cert.crt file to the directory below. (create the directory if it doesn't exist)

```bash
sudo cp cert.crt /usr/share/ca-certificates/extra
```
Have Ubuntu add the certificate as trusted.

```bash
sudo dpkg-reconfigure ca-certificates
```

Then you will be asked whether you want to add a new certificate as follows. Choose `yes`.

![img_22](https://user-images.githubusercontent.com/75593521/188849275-b5f150b7-fa7f-4b54-a8a3-c06dec6870f0.png)

The added `extra/cert.crt` is unchecked. Press the 'Space bar' key to check it, and then press Enter to complete the registration.

![img_23](https://user-images.githubusercontent.com/75593521/188849631-38efdd83-3371-46b2-a482-9ee668202c30.png)


Upon successful completion, the following message appears.

![img_24](https://user-images.githubusercontent.com/75593521/188850184-80895ac1-4c1c-437a-adc5-7ed9d8526e2f.png)


Now the cert.pem (cert.crt) certificate has been added to Linux as a trusted certificate.

If you do the same on Windows, the warning message will not appear when accessing the browser on Windows.

![Https mozilla hubs](/docs_img/cert.png)

Select and copy `cert.crt` and `key.pem`. The next step will deploy these two files to Hubs, Hubs Admin, Spoke, Dialog and Reticulum.

So, first set in Reticulum.

## 3.2 Setting https for reticulum

You need to set the path to the certificate and key files in `config/dev.exs`.

![img_25](https://user-images.githubusercontent.com/75593521/188854832-1b030bd4-5c7b-43e7-b02c-7d849df6dfcd.png)

<!--
![Https mozilla hubs](/docs_img/cert_1.png)
-->

## 3.3 Setting HTTPS for Hub

Copy the following [files](#now-we-have-keypem-and-certpem-files) to the 'hubs/certs' folder.

We run the hub with `npm run local`.

So you need to add an extra parameter to `package.json`.

`--https --cert certs/cert.crt --key certs/key.pem`

like this picture

![ssl hubs](/docs_img/ssl_hubs.png)


## 3.4 Setting HTTPS for hubs admin

Copy the following [files](#now-we-have-keypem-and-certpem-files) to the `hubs/admin/certs` folder.

We run the hub (Admin) with `npm run local`.

So add an extra parameter to `package.json`.

`--https --cert certs/cert.crt --key certs/key.pem`

like this picture

![ssl hubs admin](/docs_img/ssl_hubs_admin.png)


## 3.5 Setting HTTPS for Spoke


Copy the following [files](#now-we-have-keypem-and-certpem-files) to the `Spoke/certs` folder.


Spoke execution with `yarn start`.

So change the `start` command.

![ssl hubs admin](/docs_img/ssl_spoke.png)

And modify it like this:

#### Caution - This operation was done beforehand in 1.3.2. Just double-check that it has been corrected correctly.

```
cross-env NODE_ENV=development ROUTER_BASE_PATH=/spoke BASE_ASSETS_PATH=https://localhost:9090/ webpack-dev-server --mode development --https --cert certs/cert.crt --key certs/key.pem
```

Brief Description::

BASE_ASSETS_PATH = Run spokes on localhost:9090 by default.

## 3.6 Setting https for Dialog

Copy the following [files](#now-we-have-keypem-and-certpem-files) to the `dialog/certs` folder.

Rename `cert.crt` to `fullchain.pem`.

Rename `key.pem` to `privkey.pem`.

![ssl hubs dialog](/docs_img/ssl_dialog_1.png)

# 4. Running


Open 5 terminals. For execution of reticulum, dialog, spoke, hubs, hubs admin respectively..

![Running preparation](/docs_img/ss.png)

## 4.1 Run reticulum

Enter the following command

```bash
iex -S mix phx.server
```

## 4.2 Run dialog

Edit the `start` command in package.json with:

```
MEDIASOUP_LISTEN_IP=127.0.0.1 MEDIASOUP_ANNOUNCED_IP=127.0.0.1 DEBUG=${DEBUG:='*mediasoup* *INFO* *WARN* *ERROR*'} INTERACTIVE=${INTERACTIVE:='true'} node index.js
```

For giving params `MEDIASOUP_LISTEN_IP` and `MEDIASOUP_ANNOUNCED_IP`

![Running preparation](/docs_img/run_dialog.png)

Start the dialog server with the following command:

```
npm run start
```

`127.0.0.1` is the default IP of localhost on Mac/Linux. You can see the IP with the following command:

```bash
sudo nano /etc/hosts
```

## 4.3 Run Spoke

Use the following command

```bash
yarn start
```

## 4.4 Run hubs and hubs admin.


Both run with the following command

```bash
npm run local
```

## 4.5 Run postgREST server

More information on this can be found [here](https://github.com/mozilla/hubs-ops/wiki/Running-PostgREST-locally).

Download postREST. You must proceed from the directory where reticulum is installed.

```
sudo apt install libpq-dev
wget https://github.com/PostgREST/postgrest/releases/download/v9.0.0/postgrest-v9.0.0-linux-static-x64.tar.xz
tar -xf postgrest-v9.0.0-linux-static-x64.tar.xz
```

on reticulum iex

Paste the following command and run it.
```
jwk = Application.get_env(:ret, Ret.PermsToken)[:perms_key] |> JOSE.JWK.from_pem(); JOSE.JWK.to_file("reticulum-jwk.json", jwk)
```


Then it will create reticulum-jwk.json in the reticulum directory.

Now use the following command to create the reticulum.conf file.


```
nano reticulum.conf
```

or

```
vi reticulum.conf
```

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

Be careful. Among the above, absolute_path_to_your_file must be written directly.

```
ex) jwt-secret = "@/home/ubuntu/hubs/reticulum/reticulum-jwk.json"
```


If you work up to this point, the following two files will exist in the reticulum folder:

```
/
   postgrest
   reticulum.conf
```

then run postREST with


```
postgrest reticulum.conf
```

At this time, if it says that the postgrest path cannot be found, enter the full directory path before postgrest.

It means the path to the directory where the postgrest and `reticulum.conf` files are located, that is, the path to reticulum .

```
ex) /home/ubuntu/hubs/reticulum/postgrest reticulum.conf
```

<br>
<br>

When everything is running, you can finally connect to Hubs.

With lock symbol (SSL security)

Hubs

[https://localhost:4000](https://localhost:4000)

Hubs admin

[https://localhost:4000/admin](https://localhost:4000/admin)

Spoke

[https://localhost:4000/spoke](https://localhost:4000/spoke)

# 5. Additional Settings

## 5.1 Account Settings

When you log in for the first time, the mail login page will appear. First, enter your email address here.

Then it changes to a page asking you to complete email authentication, but the problem is that no email settings (SMTP, etc.) have been made.

Enter `https://localhost:4000?skipadmin` into the address bar to access the Hubs main page.

Now, after trying to log in by entering an email, if you return to the terminal running in reticulum, you can see the authentication link (address) that should be delivered to the original email.

![img_26](https://user-images.githubusercontent.com/75593521/189044884-360aa2d1-166b-4076-b1d5-d309b613f0e2.png)

Email verification is possible by entering the address into an Internet browser and accessing it. If authentication is successful, you can see that you are logged in when you return to the hub page.

Now set the email address to the 'admin' account.

Return to the terminal where reticulum is running and enter the following command:

```
Ret.Account |> Ret.Repo.all() |> Enum.at(0) |> Ecto.Changeset.change(is_admin: true) |> Ret.Repo.update!()
```

![img_27](https://user-images.githubusercontent.com/75593521/189046618-1287be71-c342-48b9-96d4-da9a63b3c821.png)

You are registered with an admin account, and you can now access the admin page.


![img_28](https://user-images.githubusercontent.com/75593521/189047044-9e6794a5-f6ee-4169-9cd8-72ebb79b313a.png)

## 5.1 룸 생성 오류 

When you click the Create Room button to open a room, nothing happens.

This is because the additional code required for Larchiveum.link is written. maybe...

The problem is with the following script.

`larchiveum_hubs_reactjs/src/react-components/home/CreateRoomButton.js`

Modify the contents of the script as follows.

```
import React from "react";
import { FormattedMessage } from "react-intl";
import { createAndRedirectToNewHub } from "../../utils/phoenix-utils";
import { Button } from "../input/Button";
import { useCssBreakpoints } from "react-use-css-breakpoints";

export function CreateRoomButton() {
  const breakpoint = useCssBreakpoints();

  return(
    <Button
      thick={breakpoint === "sm" || breakpoint === "md"}
      xl={breakpoint !== "sm" && breakpoint !== "md"}
      preset="landing"
      onClick={e => {
        e.preventDefault();
        createAndRedirectToNewHub(null, null, false);
      }}
    >
      <FormattedMessage id="create-room-button" defaultMessage="Create Room" />
    </Button>
  );
}
```

If there is no problem and the room is opened normally, there is no need to modify it.

## 5.2 Spoke error fix
<br>

![img_30](https://user-images.githubusercontent.com/75593521/190036522-14cecea3-9306-4540-8d1c-2b0fc1f3c60f.png)

<br>

When you first enter a spoke, you cannot create a project. It is because you are referring to the wrong path.

If the spoke is running, stop it and edit the `.env.defaults` file in the directory where the spoke is installed as follows.

![img_31](https://user-images.githubusercontent.com/75593521/190036635-bead2fbc-fa6c-4691-8d48-712b81e3a06f.png)

<br>

Now, if you run the spoke again and reconnect to the spoke, you can create a project normally.

When creating the project, the following error appears.

![img_32](https://user-images.githubusercontent.com/75593521/190037355-32ca58b6-6f56-4a38-a991-ac6cfe19fe54.png)

<br>

After downloading the object file directly by entering the link address included in the error, save it with a `.glb` extension.

This file was downloaded directly because `Terrain_Crater1.glb` included in the project could not be downloaded.

Now, if you upload the downloaded `glb` file to the project of the spoke and register it in the project, the object will be displayed in the editor normally.

The existing `Terrain_Crater1.glb` (with a warning mark) is deleted from the hierarchy view.

Adjust the position of the object a little and register the scene by clicking the 'Publish Scene' button.

Now, a scene that can be used when creating a room is registered and can be used.

<br>
<br>




## read it once : 

[Hosting Mozilla Hubs on VPS](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/VPS_FOR_HUBS.md)

[The Problem I Still Faced](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/PROBLEM_UNSOLVED.md)

[The Problem I Faced and I Already Solved](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/PROBLEM_SOLVED.md)

[Tips for Modification](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/HOW_TO_MODIFY.md)

[Overview System With Figma](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1)

[Experience Sharing About Hosting on Other Server](https://github.com/albirrkarim/mozilla-hubs-installation-detailed/blob/main/EXPERIENCE.md)

