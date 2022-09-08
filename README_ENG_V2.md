
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


