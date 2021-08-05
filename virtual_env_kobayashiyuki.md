# 環境構築作業手順書  
* バージョン一覧  
  * Dockerを使用しました
  * php 7.3
  * DB MySQL5.7
  * Webサーバ- Apache
  * Laravel 6.0
### 下記にて仮装環境手順説明  
**マックOSにてDocker、php 等の環境は既にインストール済みとして説明します。何卒ご理解ください。**
### まずはdockerで動かすためのLaravel バージョン6.0で指定したララベルアプリを作っていきます。
  * コマンドラインにて
``cd 任意のアプリを作りたいディレクトリ
``
その後指定したディレクトリ上で
`` composer create-project --prefer-dist laravel/laravel laravel_app "6.*"
 ``コマンドにてバージョンを指定し今回は**laravel laravel_app**という名前のアプリを作成します。 
  今回使用するのはバージョン６.０ということなので**6.0**とバージョンを指定しています
 <br>
### その後dockerを使用してLaravelの環境構築をするにあたって便利な laradock というものを使用します。
* まずは ``
 git clone https://github.com/laradock/laradock.git ``というコードを先ほどlaravel_appを作ったディレクトリ直下で打ちgit cloneで便利なlaradockを作業用ディレクトリに追加します。
その後`` cd laradock ``をコマンド上で打ちlaradockに移動します
### プロジェクトごとにdockerマシンを区別して使うための準備をしていきます。
`` cp env.example .env ``コマンドを打ち
その中に`` DATA_PATH_HOST `` の設定をLaravelのプロジェクト名と同じにしておきます。
``  vi env.example ``で中を覗いてみましょう。
その中に
```
### Paths #################################################

# Point to the path of your applications code on your host
APP_CODE_PATH_HOST=../

# Point to where the `APP_CODE_PATH_HOST` should be in the container
APP_CODE_PATH_CONTAINER=/var/www

# You may add flags to the path `:cached`, `:delegated`. When using Docker Sync add `:nocopy`
APP_CODE_CONTAINER_FLAG=:cached

# Choose storage path on your machine. For all storage systems
DATA_PATH_HOST=~/.laradock/data

### Drivers ################################################
```
とあるのでその中の
``
DATA_PATH_HOST=~/.laradock/data
``
とあるので以下に編集をしましょう
``
DATA_PATH_HOST=~/.laradock-laravel_app/data  
``
* ここから大切な`` docker-compose.yml``のファイルを編集していきます。
※この後に行う`` build ``コマンドなどを行う上でこのファイルを参照するのでインデントなどが崩れてしまうとエラーを起こすので注意してください。
* `` docker-compose.yml``では作成するコンテナの作業ディレクトリを指定します。
※ここでいうコンテナとは自分のOSのなかにある仮装OSのことです。
``
vi docker-compose.yml
``にてファイルを開き下記の様に
``
working_dir
``の行を追加し、その後のパスを追記してください。
 ```
 ### Workspace Utilities ##################################
    workspace:
      build:
        context: ./workspace
        args:
          - LARADOCK_PHP_VERSION=${PHP_VERSION}

         # 省略

          - no_proxy
        working_dir: /var/www/laravel_app # この行を追加
        volumes:
          - ${APP_CODE_PATH_HOST}:${APP_CODE_PATH_CONTAINER}${APP_CODE_CONTAINER_FLAG}
          - ./php-worker/supervisord.d:/etc/supervisord.d
 ```
ここまで編集が終わったら前準備は完了です。
### laradockにあるもので必要なものだけをbuildしましょう
* 今回必要となるコンテナは、以下です。それ以外のコンテナは作成する必要はありません。
  * workspace
  * mysql
  * apache
  * nginx
  * php-fpm

*　コマンドを実行しコンテナの作成を行いますが
僕との約束です。以下を守りましょう。
``docker-compose`` というコマンドは コマンドを実行したディレクトリの ``docker-compose.yml`` ファイルの内容に従って処理を行いますので、
**必ず** ``docker-compose.yml`` がある**ディレクトリに移動してコマンドを実行しましょう。**

* それでは``docker-compose.yml``のあるディレクトリにて
```
$ docker-compose build workspace mysql apache2 nginx php-fpm
```
を打ち込みましょう！
初めて``build``する際は時間がかなり掛かります。

5分後...
コンテナの作成が終了したら、ここで一旦コンテナを起動するコマンドを下記で実行してみましょう。
```
$ docker-compose up -d workspace apache2 mysql
```
それでは
``
$ docker-compose ps
``コマンドで変更したコンテナがどうなっているのか見てみましょう！
```
NAME                          COMMAND                  SERVICE             STATUS              PORTS
laradock_apache2_1            "/opt/docker/bin/ent…"   apache2             created             
laradock_docker-in-docker_1   "dockerd-entrypoint.…"   docker-in-docker    running             2375/tcp, 2376/tcp
laradock_mysql_1              "docker-entrypoint.s…"   mysql               running             0.0.0.0:3306->3306/tcp, :::3306->3306/tcp, 33060/tcp
laradock_nginx_1              "/docker-entrypoint.…"   nginx               running             0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:81->81/tcp, :::81->81/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp
laradock_php-fpm_1            "docker-php-entrypoi…"   php-fpm             running             9000/tcp, 0.0.0.0:9003->9003/tcp, :::9003->9003/tcp
laradock_workspace_1          "/sbin/my_init"          workspace           running             0.0.0.0:2222->22/tcp, :::2222->22/tcp, 0.0.0.0:3000->3000/tcp, :::3000->3000/tcp, 0.0.0.0:3001->3001/tcp, :::3001->3001/tcp, 0.0.0.0:4200->4200/tcp, :::4200->4200/tcp, 0.0.0.0:8001->8000/tcp, :::8001->8000/tcp, 0.0.0.0:8080->8080/tcp, :::8080->8080/tcp
```
上記の様に変更したコンテナ5つ全て``running``となっていれば成功したということです。

ここでもし、大量の``WARN[0000]``の警告を吐いても焦らず自分が
``docker-compose.yml``のあるディレクトリにいるか確認をしてください。
そして、何かコンテナに変更を加えた際は**必ず**``
$ docker-compose ps
``コマンドで変更されているかチェックする癖をつけましょう。

ではここで一度[http://localhost](http://localhost)を確認しましょう。
どうでしょうか、みんなが大好きなlaravelのWLCOMは表示されましたか。
答えはノーですよね。
### それではWLCOMを表示させるためにApacheの設定ファイルを編集していきましょう！
対象ファイルは、``laradock/apache2/sites/default.apache.conf`` です。
``$ cd 対象のディレクトリ``でへ``default.apache.conf``のあるディレクトリへ移動しましょう。
その後``$ vi default.apache.conf``にて中身をみてください
```
<VirtualHost *:80>
  ServerName laradock.test　　１
  DocumentRoot /var/www/　　　２　
  Options Indexes FollowSymLinks

  <Directory "/var/www/">　　３　
    AllowOverride All
    <IfVersion < 2.4>
      Allow from all
    </IfVersion>
    <IfVersion >= 2.4>
      Require all granted
    </IfVersion>
  </Directory>

  ErrorLog /var/log/apache2/error.log
  CustomLog /var/log/apache2/access.log combined
</VirtualHost>
```
とあると思いますが
１、２、３の部分を変更しApacheの設定をしていきます
上記の3行を下記の3行に変更してください

1. ``ServerName localhost``
2. ``DocumentRoot /var/www/laravel_app/public/``　
3. ``<Directory "/var/www/laravel_app/public">　``
すると
```
<VirtualHost *:80>
  ServerName localhost
  DocumentRoot /var/www/laravel_app/public/
  Options Indexes FollowSymLinks

  <Directory "/var/www/laravel_app/public">
    AllowOverride All
    <IfVersion < 2.4>
      Allow from all
    </IfVersion>
    <IfVersion >= 2.4>
      Require all granted
    </IfVersion>
  </Directory>

  ErrorLog /var/log/apache2/error.log
  CustomLog /var/log/apache2/access.log combined
</VirtualHost>
```
上記の様になっていると思うので同じ様に編集をしてください。
そして、あなたは今、コンテナであるファイルに変更を加えました。
変更を加えたらもう一度``build``をし直します
ですので``build``をする際は``docker-compose.yml``のあるファイルで実行しましょう！

そのディレクトリに戻れたら``$ docker-compose build apache2``を実行したのち、
``$ docker-compose up -d workspace apache2``を実行してみましょう。
<br>
※``docker-compose up`` で起動した場合、Logを吐き出し続け入力を受け付けない状態となり不便なので、バックグラウンドで起動をし続けられるよう -d オプションを付けて実行しましょう。
### DBの設定偏
最初に作ったバージョン6.0のlaravel_appディレクトリ直下に``.env``ファイルがあります。
では``.env``ファイルを``$ vi``で開きファイルを編集しましょう
```
DB_CONNECTION=mysql
DB_HOST=mysql        # 変更
DB_PORT=3306
DB_DATABASE=default  # 変更
DB_USERNAME=root     # 変更
DB_PASSWORD=root     # 変更
```
上記の様にファイルを変更できたら早速動作の確認としてマイグレーションを実行していきましょう
* 実行方法としてコンテナ内に入ってコマンドを実行する方法と、コンテナの外からコンテナの中に対してコマンドを実行する2つの方法がありますのでどちらも試してみましょう。
`1. コンテナの中に入りコマンドを実行する方法`
  * では``$ docker-compose exec サービス名 bash``という記述方法で
  ``サービス名``の部分を今回は``workspace``で指定しましょう。<br>
    ``$ docker-compose exec workspace bash``を実行。
    するとコマンドの様子が``root@XXXXXXXXX:/#``の様になりましたか。
    この``XXXXXXXXX``部分はご自身の環境によって変わります。
    <br>
    コンテナ内にログインしたら``php artisan migrate``を実行してみましょう。
    マイグレーションが正常に実行できましたでしょうか？？
    実行できたらもう一つの方法を試すために一度 
    ``php artisan migrate:reset`` を実行してテーブルを削除して``exit``でコンテナから抜けましょう。
    
  ` 2. コンテナの外からコンテナ内にコマンドを実行する方法 `
  * では次にコンテナ外からマイグレーションを実行してみましょう。
  ``docker-compose.yml``があるディレクトリにて
  ``$ docker-compose exec workspace php artisan migrate``を実行してください。
  その後ログインなどDBに接続できれば成功です！
  