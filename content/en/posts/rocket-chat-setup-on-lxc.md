---
title: "Rocket.chat Setup on LXC Container"
date: 2022-09-05
tags: [Blog, Setup, Homelab, Proxmox, LXC, Rocket.chat]
---

## Rocket.chat Setup
This a basic guide on how we can install Rocket.chat on Ubuntu LXC.

This is a part 1 of 2 part series. In this part, we will look into how to install MongoDB, NodeJS & Rocket.Chat and also configure them.

In the next part, we will see how to reverse proxy the localhost:3000 to a domain and install SSL certificate.

## Requirements
* Ubuntu: `20.04`
* NodeJS: `14.19.3`
* NPM: `6.14.17`
* Mongodb: `5.0`


## Installing Pre-requisites (NodeJS, NPM, & MongoDB)
### NodeJS & NPM
We will be using nvm to install NodeJS and NPM.

Steps:
Install NVM
```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | shell
```
List all available NodeJS version
```shell
nvm ls-remote
```
Install specified NodeJS version, in our case it v14.19.3.
```shell
nvm install v14.19.3
```
Check NPM version
```shell
npm -v
6.14.17
```

### MongoDB
We will be adding MongoDB from official repo, which is not in default Ubuntu LXC source list.

Steps:

Install gnupg

```shell
sudo apt install gnupg
```

Import mongodb key

```shell
wget -qO - https://www.mongodb.org/static/pgp/server-5.0.asc | sudo apt-key add -
```

Create the list file

```shell
/etc/apt/sources.list.d/mongodb-org-5.0.list
```

Add repo url to source list file

```shell
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/5.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-5.0.list
```

Update local package db

```shell
sudo apt-get update
```

Install MongoDB

```shell
sudo apt-get install -y mongodb-org
```

## Installing Rocket.Chat

Install dependencies

```shell
sudo apt install -y curl build-essential graphicsmagick
```

Download Rocket.Chat binary:

```shell
curl -L https://releases.rocket.chat/latest/download -o /tmp/rocket.chat.tgz
```

Extract the archive

```shell
tar -xzf /tmp/rocket.chat.tgz -C /tmp
```

Install npm packages:

```shell
cd /tmp/bundle/programs/server && npm install --production
```

> **NOTE:** If you're using the `root` account (not recommended), you'll have to use `sudo npm install --unsafe-perm --production` instead of the above.

Copy built package to /opt folder

```shell
sudo mv /tmp/bundle /opt/Rocket.Chat
```

## Configure the Rocket.Chat service

Add the rocketchat user

```shell
sudo useradd -M rocketchat && sudo usermod -L rocketchat
```

Set the right permissions on the Rocket.Chat folder

```shell
sudo chown -R rocketchat:rocketchat /opt/Rocket.Chat
```

Save node path in variable to pass it on service file

```shell
NODE_PATH=$(which node)
```

Create & save the systemd service file

```shell
cat << EOF |sudo tee -a /lib/systemd/system/rocketchat.service
[Unit]
Description=The Rocket.Chat server
After=network.target remote-fs.target nss-lookup.target nginx.service mongod.service
[Service]
ExecStart=$NODE_PATH /opt/Rocket.Chat/main.js
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=rocketchat
User=rocketchat
[Install]
WantedBy=multi-user.target
EOF
```

### Passing environment variables

Pass some environment variables to the running process. 

For more information of configuring via environment variables read [this article](https://docs.rocket.chat/quick-start/environment-configuration/environment-variables).

Run:

```
sudo systemctl edit rocketchat
```

It will open in vim or nano based on your system. Now, write the following in the file

```
[Service]
Environment=ROOT_URL=http://localhost:3000
Environment=PORT=3000
Environment=MONGO_URL=mongodb://localhost:27017/rocketchat?replicaSet=rs01
Environment=MONGO_OPLOG_URL=mongodb://localhost:27017/local?replicaSet=rs01
```

Change the values as you need. Save and exit.

### MongoDB Configuration

MongoDB config file should look something like the following:

```yaml
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  engine: wiredTiger

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 127.0.0.1

processManagement:
  fork: true
  timeZoneInfo: /usr/share/zoneinfo

replication:
  replSetName: rs01
```

Start MongoDB with the following command:

```shell
sudo systemctl enable --now mongod
```

If MongoDB is already started, do a restart of the service

```shell
sudo systemctl restart mongod
```

Create the replicaset:

```shell
mongo --eval "printjson(rs.initiate())"
```

You can start Rocket.Chat now.

```shell
sudo systemctl enable --now rocketchat
```

## Related Links
1. [Rocket.Chat Doc](https://docs.rocket.chat/quick-start/deploying-rocket.chat/other-deployment-methods/manual-installation/debian-based-distros/ubuntu)
2. [Proxmox VE](https://www.proxmox.com/en/proxmox-ve)
3. [NVM](https://github.com/nvm-sh/nvm)
4. [MongoDB Doc](https://www.mongodb.com/docs/v5.0/tutorial/install-mongodb-on-ubuntu/)