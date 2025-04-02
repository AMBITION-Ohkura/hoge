# Skima

## 開発環境構築手順
#### Clone
```
$ git clone https://github.com/visualworks-inc/skima.git
```

#### Config
```
$ cd skima/app/config
$ cp -r development_docker development
```

#### WSL Environment
```
$ sudo sh -c "echo 'nameserver 192.168.1.1' > /etc/resolv.conf"
$ sudo rm -rf /etc/wsl.conf
$ sudo vi /etc/wsl.conf
[network]
generateResolvConf = false

$ export DOCKER_BUILDKIT=0
$ export COMPOSE_DOCKER_CLI_BUILD=0
```

#### Payment Setting
```
$ vi skima/app/config/development/nppay.php
'transaction_prefix' => 'local-[Your Name]',
```

#### Global Setting
```
$ vi skima/app/config/development/constant.php
'skima_global' => [
    'price_multiplier' => [
        'endpoint' => 'https://backend-nginx:443',
        'api_ssl_verification_disabled' => true,
    ],
    'url' => 'https://local.wskima.com:8443',
    'secret_token' => 'GzaiUuiRXZvpBUwFkZC9OoFMHRVKTytswah7SovZOKuee8dYivfglFAvaoYPa8UI',
],
```

#### Start Up
```
$ docker-compose up -d

※if failed to run Build function: centos:7: failed to authorize: failed to fetch anonymous token Error
$ sudo vi /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
$ docker-compose up -d
```

#### Migrate
```
$ docker ps -a
$ docker exec -it [app container id] bash
# cd var/www/html
# php oil refine migrate
# exit
```

#### URL
## Top
https://localhost/

## Admin
https://localhost/management

## Minio
http://localhost:9001/login

## Mail
http://localhost:18025/
