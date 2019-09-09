---
title: "Quick Start"
date: 2019-09-09T15:00:37+08:00
draft: false
---

Bigfile sticks to ease of use, all you have to do is install MySQL and create a database. If you are docker fan, then everything is even simpler. You can start a mysql like this:

    docker run -it --rm -p 33306:3306 -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=bigfile  mysql:5.7.21 --character-set-server=utf8 --collation-server=utf8_general_ci

Here you need to set the character set to `utf8`. Next, you need to download the binary executable from [release page](https://github.com/bigfile/bigfile/releases). Then just start the server you want to run. All data tables will be synchronized when the server is started.

### Configuration

Bigfile has a default configuration, Here it is:

```yaml
database:
  driver: mysql
  host: localhost
  user: root
  password: root
  port: 3306
  dbName: bigfile
log:
  console:
    enable: true
    level: info
    format: '%{color:bold}[%{time:2006/01/02 15:04:05.000}] %{pid} %{level:.5s} %{color:reset} %{message}'
  file:
    enable: true
    path: storage/logs/bigfile.log
    level: warn
    format: '[%{time:2006/01/02 15:04:05.000}] %{pid} %{longfile} %{longfunc} %{callpath} ▶ %{level:.4s} %{message}'
    maxBytesPerFile: 52428800
http:
  apiPrefix: /api/bigfile
  accessLogFile: storage/logs/bigfile.http.access.log
  limitRateByIPEnable: false
  limitRateByIPInterval: 1000
  limitRateByIPMaxNum: 100
  corsEnable: false
  corsAllowOrigins:
    - '*'
  corsAllowMethods:
    - PUT
    - PATCH
    - DELETE
  corsAllowHeaders:
    - Origin
  corsExposeHeaders:
    - 'Content-Length'
  corsAllowCredentials: true
  corsAllowAllOrigins: false
  corsMaxAge: 3600
chunk:
  rootPath: storage/chunks
```

Bigfiel will find configuration file in `current directory`, `user home` and `/etc/bigfile`. You need to place a file named `.bigfile.yml` in one of the directories. 

If you just want to start Bigfile, then the only thing you need to set is to set up MySQL. We provide an entry on the command line and set up the database without a configuration file. like this:

        bigfile --db-host 127.0.0.1 --db-user root --db-pass root  --db-name bigfile  --db-port 33306 http:start 


### All in Docker

1. pull image: 

        docker pull bigfile/bigfile

2. create a network

        docker network create bigfile

3. start mysql
    
         docker run -it --rm --network bigfile --name bigfile-mysql -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=bigfile mysql:5.7.21 --character-set-server=utf8 --collation-server=utf8_general_ci

4. start http server

        docker run --rm --network bigfile -p 10985:10985  bigfile/bigfile --db-host bigfile-mysql http:start