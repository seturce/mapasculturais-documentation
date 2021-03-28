# Mapas Culturais  - Instalação Centos

Este repositório irá armazenar as configurações necessárias para efetuas a instalaçao passo-a-passo da aplicação mapasculturais tendo como base esta [documentação](http://docs.mapasculturais.org/mc_deploy/), tivemos que fazer algumas mudanças do tutorial original para que fosse possível a sua instalaçao.

## 1. Configuração da máquina ( EC2 amazon )

* CENTOS 7
* vCPUS 2
* 4GB RAM
* SSD 50GB

_Essa máquina foi criada apenas para efetuarmos testes iniciais e fazer a primeira instalação, o procedimento será o mesmo em sua máquina de produção, para ter acesso aos requisitos mínimos acesse para rodar em produção acesse [este Link](https://github.com/mapasculturais/mapasculturais#hardware-requisitos-para-instala%C3%A7%C3%A3o)_

## 2. Softwares Requeridos

### Atualize os repositórios de referência de sua máquina
  
```console
  root@server# cd ~
  root@server# yum update -y
  root@server# yum install epel-release -y
```

### Instale as dependências
  
```console
  root@server# yum install yum-utils -y
  root@server# yum install git curl gcc-c++ make zip unzip -y 
```

### Instale a versão stable mais nova do nodejs
  
```console
  root@server# curl -sL https://rpm.nodesource.com/setup_10.x | sudo -E bash -
  root@server# yum install nodejs -y
```

### Verificando se foi instalada a versão mais recente do NodeJS e do NPM
  
```console
  root@server# node -v
  root@server# npm -v
```

### Instalando os minificadores de código Javascript, CSS e SASS
  
```console
  root@server# npm install -g uglify-js uglifycss autoprefixer terser postcss
  root@server# yum install rubygem-sass -y
```
  
### Instale o nginx

```console
  root@server# yum install nginx -y
  root@server# systemctl start nginx
  root@server# systemctl enable nginx
```

### Instale o php7.2, php7.2-fpm, extensões do php utilizadas no sistema e gerenciador de dependências do PHP Composer

```console
  root@server# vi /etc/yum.repos.d/CentOS-Base.repo
```

```txt
    [base]
      exclude=php*
    [updates]
      exclude=php*
```

```console
  root@server# wget https://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/e/epel-release-7-12.noarch.rpm
  root@server# wget http://rpms.remirepo.net/enterprise/remi-release-7.rpm
  root@server# yum -y install epel-release
  root@server# rpm -Uvh epel-release-latest-7.noarch.rpm
  root@server# rpm -Uvh remi-release-7.rpm
  root@server# yum-config-manager --enable remi-php72
  root@server# yum install php php-fpm php-pdo php-json php-common php-cli php-xml php-pgsql php-mbstring php-mcrypt php-pecl-apcu php-pecl-imagick php-opcache php-doctrine-orm php-pecl-zip php-mysql -y
  root@server# systemctl start php-fpm
  root@server# systemctl enable php-fpm
  root@server# cd ~
  root@server# url -sS https://getcomposer.org/installer | php
  root@server# mv composer.phar /usr/bin/composer
```

### PHP Configs

```console
  root@server# vi /etc/php.ini
```console

```txt
  variables_order = "EGPCS"
```

## 3. Baixando o projeto e configurando

### Adicionando o usuario da aplicação e download do projeto, tema, e plugins através do github

```console
  root@server# useradd -G nginx -d /srv/mapas -m mapas
  root@server# su - mapas
  mapas@server# git clone https://github.com/secultce/mapasculturais.git
  mapas@server# cd ~/mapasculturais
  mapas@server# git checkout production
  mapas@server# cd ~/mapasculturais/src/protected/application/themes
  mapas@server# git clone https://github.com/secultce/theme-Ceara.git Ceara
  mapas@server# cd ~/mapasculturais/src/protected/application/plugins
  mapas@server# git clone https://github.com/secultce/plugin-MultipleLocalAuth.git MultipleLocalAuth
```

### Com o usuário criado, crie a pasta para os assets, para os uploads e para os uploads privados (arquivos protegidos, como anexos de inscrições em oportunidades)

```console
  root@server# su - mapas
  mapas@server# mkdir ~/mapasculturais/src/assets
  mapas@server# mkdir ~/mapasculturais/src/files
  mapas@server# mkdir ~/mapasculturais/private-files
  mapas@server# mkdir ~/mapasculturais/private-files/sessions  
```

### Configurando o projeto no nginx

```console
  root@server# mkdir /var/log/mapasculturais
  root@server# chown mapas:nginx /var/log/mapasculturais
  root@server# mkdir /etc/nginx/sites-available
  root@server# mkdir /etc/nginx/sites-enabled

  root@server# vim /etc/nginx/nginx.conf
    // Agora adicione include da pasta "sites-enabled" no fim do arquivo
      include /etc/nginx/sites-enabled/*.conf;
    // Comente todo o codigo de configuração do "server"
      #  server {
      # ...
      # ...
      # }
```

#### Precisamos criar o virtual host do nginx para a aplicação. Para isto crie, como root, o arquivo /etc/nginx/sites-available/mapas.conf

```console
  root@server# vim /etc/nginx/sites-available/mapas.conf 
```

#### Configurações sem domínio

```txt
  server {
    listen 80 default_server;
    listen [::]:80 default_server;  
    server_name mapacultural.secult.ce.gov.br;
    access_log /var/log/nginx/mapas.access.log;
    error_log  /var/log/nginx/mapas.error.log;

    client_max_body_size 10M;

    root /srv/mapas/mapasculturais/src/;
    index index.php;

    location / {  
      try_files $uri $uri/ /index.php?$args;
    }	

    location ~ /files/.*\.php$ {
      deny all;
      return 403;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|woff|pdf|odt|xls)$ {
      expires 1w;
      log_not_found off;
    }

    location ~ \.php$ {
      try_files $uri =404;
      include fastcgi_params;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      fastcgi_pass unix:/var/run/php-fpm/mapas.sock;
      client_max_body_size 10M;
    }

    charset utf-8;
  }
```

#### Configurações com domínio

```txt
  server {
    set $site_name meu.dominio.gov.br;

    listen *:80;
    server_name meu.dominio.gov.br;
    access_log   /var/log/mapasculturais/nginx.access.log;
    error_log    /var/log/mapasculturais/nginx.error.log;

    index index.php;
    root  /srv/mapas/mapasculturais/src/;

    location / {
      try_files $uri $uri/ /index.php?$args;
    }
    
    location ~ /files/.*\.php$ {
      return 80;
    }
    

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|woff)$ {
            expires 1w;
            log_not_found off;
    }

    location ~ \.php$ {
      try_files $uri =404;
      include fastcgi_params;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      fastcgi_pass unix:/var/run/php-fpm/mapas.sock;
      client_max_body_size 0;
    }

    charset utf-8;
  }

  server {
    listen *:80;
    server_name www.meu.dominio.gov.br;
    return 301 $scheme://meu.dominio.gov.br$request_uri;
  }
  ```

```console
  root@server# ln -s /etc/nginx/sites-available/mapas.conf /etc/nginx/sites-enabled/mapas.conf
```

### Configurações pool do php7.2-fpm: Crie o arquivo ```/etc/php-fpm.d/mapa.conf```. Muita atenção na alteração da linha ```listen = /var/run/php-fpm/mapas.sock``` pois o nome do arquivo .sock precisará ser extamente como foi configurado do arquivo ```/etc/nginx/sites-available/mapas.conf```. O diretório ```/var/run/php/``` podrá mudar dependendo da versão que estiver trabalhando

```console
  root@server# vi /etc/php-fpm.d/mapa.conf
```

```txt
  [mapas]
  listen = /var/run/php-fpm/mapas.sock
  listen.owner = mapas
  listen.group = nginx
  user = mapas
  group = nginx
  catch_workers_output = yes
  pm = dynamic
  pm.max_children = 20
  pm.start_servers = 5
  pm.min_spare_servers = 5
  pm.max_spare_servers = 10
  pm.max_requests = 500
  chdir = /srv/mapas

  php_admin_value[error_log] = /dados/mapas/logs/php.error.log
  php_admin_flag[log_errors] = on
  php_value[display_errors] = 'stderr'
  php_value[memory_limit]= 6144M
  php_admin_value[upload_max_filesize] = 10M
  php_admin_value[post_max_size] = 10M

  clear_env = no

  env["APP_MODE"] = "production"
```

## 4. Banco de dados

### Instale o postgresql, postgis e inicializa o servico postgresql
  
```console
  root@server# yum-config-manager --enable pgdg96
  root@server# yum -y install https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
  root@server# yum install postgresql96 postgresql96-server postgresql96-contrib postgresql96-libs -y
  root@server# yum install postgis24_96
  root@server# /usr/pgsql-9.6/bin/postgresql96-setup initdb
  root@server# systemctl start postgresql-9.6
  root@server# systemctl status postgresql-9.6
  root@server# systemctl enable postgresql-9.6
  root@server# vi /var/lib/pgsql/9.6/data/pg_hba.conf
```

```txt
  // Edite a linha
    host    all             all             127.0.0.1/32            ident
  //para 
    host    all             all             127.0.0.1/32            md5
```

```console
  root@server# systemctl restart postgresql-9.6
```

### Criando usuario e database

```console
  root@server# su - postgres 
  postgres@server# createuser mapas
  postgres@server# createdb --owner mapas mapas
  postgres@server# psql 
  psql@server# alter user "mapas" with encrypted password 'mapas';
  psql@server# grant all privileges on database mapas to "mapas";
  psql@server# \c mapas
  psql@server# CREATE EXTENSION postgis;
  psql@server# CREATE EXTENSION unaccent;
  psql@server# \q
  postgres@server# psql -U mapas -d mapas -f /srv/mapas/mapasculturais/db/schema.sql
  postgres@server# psql -U mapas -d mapas -f /srv/mapas/mapasculturais/db/initial-data.sql
  postgres@server# exit
```

## 5. Configurações do sistema
  
### Primeiro crie um arquivo de configuração copiando o arquivo de template de configuração. Este arquivo está preparado para funcionar com este guia, utilizando o método de autenticação Fake
  
```console
  root@server# su - mapas
  mapas@server# cp mapasculturais/src/protected/application/conf/config.template.php mapasculturais/src/protected/application/conf/config.php
```

### Autenticação - MultipleLocalAuth

  _Após a instalção do Mapa Cultural teremos várias possibilidades para autenticação, mas iremos abordar aqui apenas a autenticação utilizando MultipleLocalAuth que dará ao usuário a possibilidade de fazer a autenticação utilizando contas Locais, Gooogle, Facebook e afins. 
  Após os procedimentos descritos abaixo iremos sair do modo de 'Autenticação Fake'. 
  Para isso iremos utilizar as informações já disponibilidas no repositório [MultipleLocalAuth](https://github.com/secultce/MultipleLocalAuth) e adicionar alguns detalhes com o objetivo de facilitar a implementação_
  
#### Agora, Faça a edição do arquivo de configuração do mapas utilizando vi ou o editor de sua preferência.
  
```console
mapas@server# vi /srv/mapas/mapasculturais/src/protected/application/conf/config.php
```
  
#### Neste momento teremos três etapas_
  
  1- Ativar o plugin
  2- Configurar MultipleLocalAuth como seu Provider de autenticação
  3- Configurar as chaves das redes sociais
  
##### 1- Ativar o plugin
  
###### Ainda com o arquivo ```config.php``` aberto adicione a seguinte configuração: 
  
```php
  'plugins' => [
      // ... outros plugins
      'MultipleLocalAuth' => [
          'namespace' => 'MultipleLocalAuth',
      ],
  ],
```

_Nota: A linha acima ficará dentro do array de configurações do arquivo ```config.php``` que estamos editando neste momento._
  
##### 2- Configurar MultipleLocalAuth como seu Provider de autenticação
  
###### Procure a linha com o código
  
  ```'auth.provider' => 'Fake'```,
  
###### comente e adicione uma nova ou altere para 
  
  ```'auth.provider' => '\MultipleLocalAuth\Provider'```
  
##### 3- Configurar as chaves das redes sociais
  
###### Defina a configuração auth.config para definir as estratégias utilizadas e as chaves dos serviços:
  
```php
  //'auth.provider' => 'Fake'
  'auth.provider' => '\MultipleLocalAuth\Provider'
  'auth.config' => [
    'salt' => 'LT_SECURITY_SALT_SECURITY_SALT_SECURITY_SALT_SECURITY_SALT_SECU', // string to salt crypto password
    'timeout' => '24 hours', //timeout
    'enableLoginByCPF' => true, //login using CPF
    'metadataFieldCPF' => 'documento', //metadata key in agent_meta to CPF value
    'userMustConfirmEmailToUseTheSystem' => false, //confirm email after new user 
    'passwordMustHaveCapitalLetters' => true, //validate strongest passwords Must Have Capital Letters
    'passwordMustHaveLowercaseLetters' => true, //validate strongest passwords Must Have Lower Letters
    'passwordMustHaveSpecialCharacters' => true, //validate strongest passwords Must Have Special Characters
    'passwordMustHaveNumbers' => true, //validate strongest passwords Must Have Numbers
    'minimumPasswordLength' => 7, //validate  passwords length 
    'google-recaptcha-secret' => '6LeIxAcTAAAAAGG-vFI1TnRWxMZNFuojJ4WifJWe', //google recaptcha
    'google-recaptcha-sitekey' => '6LeIxAcTAAAAAJcZVRqyHh71UMIEGNQ_MXjiZKhI', //google recaptcha
    'sessionTime' => 7200, // session time by user in seconds
    'numberloginAttemp' => '5', // login attempts with error before block user by X seconds
    'timeBlockedloginAttemp' => '900', // block time user after overcoming login attempts with error
    'strategies' => [
         'Facebook' => [
           'app_id' => 'SUA_APP_ID',
           'app_secret' => 'SUA_APP_SECRET', 
           'scope' => 'email'
         ],

        'LinkedIn' => [
          'api_key' => 'SUA_API_KEY',
          'secret_key' => 'SUA_SECRET_KEY',
          'redirect_uri' => URL_DO_SEU_SITE . '/autenticacao/linkedin/oauth2callback',
          'scope' => 'r_emailaddress'
        ],
        'Google' => [
          'client_id' => 'SEU_CLIENT_ID',
          'client_secret' => 'SEU_CLIENT_SECRET',
          'redirect_uri' => URL_DO_SEU_SITE . '/autenticacao/google/oauth2callback',
          'scope' => 'email'
        ],
        'Twitter' => [
            'app_id' => 'SUA_APP_ID', 
            'app_secret' => 'SUA_APP_SECRET', 
        ],

      ]        
  ],
```
  
_Nota: Em nosso projeto utilizamos apenas a autenticação Padrão e Google, portanto foi necessário gerar as chaves utilizando a ferramenta [Google Console Developers](https://console.developers.google.com/apis/credentials), após gerar as chaves basta adiciona-las na configurações citadas acima._  
  
### Para finalizar, precisamos executar um script que entre outras coisas compila e minifica os assets, faz o download das dependecias do composer, otimiza o autoload de classes do composer e roda atualizações do banco
  
```console
  root@server# su - mapas
  mapas@server# ./mapasculturais/scripts/deploy.sh
```

### Reiniciando serviços php-fpm e nginx

```console
  root@server# systemctl restart php-fpm
  root@server# systemctl restart nginx
```

## 6. Configurações do serviço de atualização de permissões do sistema

_Nota: O Mapa Cultural possui um serviço (script bash) que atualiza as permissão de acesso (autorização) dos usuários esse serviço precisa rodar em background para manter as autorizações atualizadas, permitindo que novos agentes tenham permissão de inscrição em editais, administrar entidades do sistema, ou até mesmo os avaliadores terem permissão de realizar e visualizar avaliações em editais

### Criar o script que vai rodar em segundo plano

```console
  root@server# su - mapas
  mapas@server# mkdir /srv/mapas/scripts
  mapas@server# vi /srv/mapas/scripts/recreate-pending-pcache-cron.sh
```

```txt
  !/bin/bash

  while [ true ]; do
    /srv/mapas/mapasculturais/scripts/recreate-pending-pcache.sh
    
    if [ -z "$PENDING_PCACHE_RECREATION_INTERVAL" ]; then
      sleep 60
    else
      sleep $PENDING_PCACHE_RECREATION_INTERVAL
    fi

  done
```

```console
chmod +x /srv/mapas/scripts/recreate-pending-pcache-cron.sh
```

### Configurar o daemon do servidor

```console
  mapas@server# vi /etc/systemd/system/recreate-pcache.service
```

```txt
  [Unit]
  Description=Pending Cache

  [Service]
  Type=simple
  WorkingDirectory=/srv/mapas/scripts
  User=mapas
  Restart=always
  RestartSec=10
  ExecStart=/srv/mapas/scripts/recreate-pending-pcache-cron.sh

  [Install]
  WantedBy=multi-user.target
```

```console
  root@server# systemctl daemon-reload
  root@server# systemctl start recreate-pcache.service
```

#### Testar se o serviço esta rodando sem erros

```console
  root@server# systemctl status recreate-pcache.service -l
```

### Pronto!! Após o procedimento você deverá ser capaz de acessar o sistema com servidor web, php, base de dados, autenticação e serviço de permissoes funcionando


## 7. Redis Server

_Nota: Utilizaremos o redis para armazenar as sessões do php

### NO SERVIDOR REDIS

#### Instalação

```console
  root@server# yum install epel-release -y
  root@server# yum update
  root@server# yum install redis -y
  root@server# vi /etc/redis.conf
```

```txt
  Na linha , Adicionar o IP da rede do servidor redis
  bind 127.0.0.1 IP DO SERVIDOR

  Na linha , Adicionar a senha do servidor redis
  requirepass SENHA
```

```console
  root@server# systemctl enable redis
  root@server# systemctl start redis
  root@server# systemctl status redis
  root@server#
  root@server#
  root@server#
```

#### Testar portas

```console
  root@server# netstat -tlpn
  root@server# ss -an | grep 6379
```

#### Se for preciso alterar o firewall com zona

```console
  root@server# firewall-cmd --permanent --new-zone=redis
  root@server# firewall-cmd --permanent --zone=redis --add-port=6379/tcp
  root@server# firewall-cmd --permanent --zone=redis --add-source=client_server_private_IP
  root@server# firewall-cmd --reload
```

#### Se for preciso alterar o firewall sem zona

```console
  root@server# firewall-cmd --permanent --zone=public --add-port=6379/tcp 
  root@server# irewall-cmd  --reload
```

#### Se for preciso alterar o firewall com iptables

```console
  root@server# iptables -A INPUT -i lo -j ACCEPT
  root@server# iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
  root@server# iptables -A INPUT -p tcp -s client_servers_private_IP/32 --dport 6379 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
  root@server# iptables -P INPUT DROP
```

#### Testando Redis com senha de autenticação

_Nota: Utilizamos o redis-cli para acessar o servidor redis , será preciso informar a senha que configuramos anteriormente, em seguida execute o comando ping, se tudo estiver correto o servidor responderá com "PONG"

```console
  root@server# redis-cli -h IP DO SERVIDOR -p 6379
  root@server# AUTH SENHA
  root@server# OK
  root@server# ping
  root@server# PONG
```

### NO CLIENTE

#### Instalar um client redis ( No caso do servidor redis ser diferente do servidor do sistema)

```console
  root@server# npm install -g redis-cli
```

#### Testar o acesso com senha ( No caso do servidor redis ser diferente do servidor do sistema)

```console
  root@server# rdcli -h IP DO SERVIDOR -p 6379 -a SENHA
  redis$ ping
  redis$ PONG
```

#### Configurar o php com redis

```console
  root@server# vi /etc/php.ini
```

```txt
  session.save_handler = redis
  session.save_path = "tcp://REDIS_IP_SERVER:6379?auth=REDIS_SENHA"
```

#### Configurar o php-fpm com redis

```console
  root@server# vi /etc/php-fpm.d/mapa.conf
```

##### Adicionar no fim do arquivo

```txt
[mapas]

php_admin_value[session.save_path] = "tcp://REDIS_IP_SERVER:6379?auth=REDIS_SENHA"
php_admin_value[session.save_handler ] = redis
env["SESSIONS_SAVE_PATH"] = "tcp://REDIS_IP_SERVER:6379?auth=REDIS_SENHA"

```

#### Alterar script do servico de permissoes com configuracoes do redis

```console
  root@server# vi /srv/mapas/scripts/recreate-pending-pcache-cron.sh
```

```txt
  #!/bin/bash

  export APP_MODE='development'
  export SESSIONS_SAVE_PATH='tcp://172.20.12.98:6379?auth=209P#WGEVZe4jFLz@21'

  while [ true ]; do
      /srv/mapas/mapasculturais/scripts/recreate-pending-pcache.sh
      if [ -z "$PENDING_PCACHE_RECREATION_INTERVAL" ]; then
          sleep 5
      else
          sleep $PENDING_PCACHE_RECREATION_INTERVAL
      fi
  done
```

```console
  root@server# systemctl daemon-reload
  root@server# systemctl restart recreate-pcache.service
  root@server# systemctl status recreate-pcache.service -l
```
