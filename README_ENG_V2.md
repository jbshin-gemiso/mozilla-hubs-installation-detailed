
# Introduce

This document contains information on how to install and run mozilla hub in Local.

Windows 10 Pro (version 21H2) + WSL2 + Ubuntu 20.04.4 LTS based on the work was done.

# Install Mozilla Hubs locally

This document is based on [albirrkarim/mozilla-hubs-installation-detailed](https://github.com/albirrkarim/mozilla-hubs-installation-detailed).

Tutorial video [youtube video] of the document (https://youtu.be/KH1T9u9DaCo).

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


# summary

![System Overview](/docs_img/System_Overview.png)

Above image made with [figma](https://www.figma.com/) You can read more explanation at [Documentation](https://hubs.mozilla.com/docs/system-overview.html) there is.

You will also want to create a software overview, architecture, and tables for your database. You can see [figma project](https://www.figma.com/file/h92Je1ac9AtgrR5OHVv9DZ/Overview-Mozilla-Hubs-Project?node-id=0%3A1)

### summary

Reticulum - The main host. Synchronize the position, rotation, and state of objects. It communicates with the client browser via http requests and websockets.

Dialog - Synchronize the user's video and audio. It communicates with the client browser via websockets.

Hubs, Spokes - Provides static assets, Reticulum fetches them and passes them to client browsers.

postREST - A server that helps hub administrators perform basic operations such as create read update delete (CRUD).

Hubs Admin - Use websocket to communicate with postgREST for authentication (login). For CRUD purposes, the hub manager sends an http request (GET, POST, etc) to Reticulum, which then proxy forwards to postgREST.

# fist!

Main Steps - [Cloning and Preparation](#1-cloning-and-preparation) -> [Setting up HOST](#2-setting-up-host) -> [Setting up HTTPS (SSL)](#3-setting -up-https-ssl) -> [Running](#4-runing)

#1. Cloning and Preparation

## 1.1 Reticulum

It's a backend server using elixir and phoenix.


### 1.1.1 Clone

```bash
git clone https://github.com/mozilla/reticulum.git
cd reticulum
```

### 1.1.2 Installation Requirements

#### Postgres database

Installing on [Linux] (https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart)

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

[Tutorial] (https://www.digitalocean.com/community/tutorials/how-to-install-postgresql-on-ubuntu-20-04-quickstart) has `sudo systemctl start postgresql.service` to start the postgresql service. command, the following error message appears:

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

You can install it by following this [tutorial] (https://www.pluralsight.com/guides/installing-elixir-erlang-with-asdf).

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

### 1.1.3 Run the following command

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


### 1.1.4 Running Reticulum on local Dialog instance

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


4. Coturn 설치

Note: [Install Coturn on Ubuntu] (https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu -18-04)

Install coturn using the following command:

```
sudo apt-get -y update
sudo apt-get install coturn
```

Dialog [config file] (https://ourcodeworld.com/articles/read/1175/how-to-create-and-configure-your-own-stun-turn-server-with-coturn-in-ubuntu-18- 04) Edit `turnserver.conf` and update the PostgreSQL database connection string to use the _coturn_ schema of the Reticulum database:

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

Please refer to this [comment] (https://github.com/mozilla/hubs/discussions/3323#discussioncomment-1857495).

Generate public and private keys (RSA) using [generator online] (https://travistidwell.com/jsencrypt/demo/).

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

### 1.3.2 Set default path

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

This [repo] (https://github.com/geminisoft-vn/larchiveum_hubs_reactjs) contains the hub client and hub admin (hubs/admin).

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
<br>

