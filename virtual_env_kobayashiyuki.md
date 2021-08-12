# 環境構築作業手順書 
|  使った物 |  バージョンETC  |
| ---- | --- |
|  仮想環境  |  DOKCER  |
| php | 7.3 |
|  Webサーバー |  Apache  |
| Laravel | 6.0|

### 下記にて仮想環境手順説明  
**マックOSにてDocker、php 等の環境は既にインストール済みとして説明します。何卒ご理解ください。**
## 目次
1. [Laravelアプリの雛形を作成](#section1)
1. [laradockをクローンする](#section2)
1. [プロジェクト別でDockerマシンを区別するための準備をする](#section3)
1. [laradockでbuildなどDockerのコマンドで操作する](#section4)
1. [Apacheの設定ファイルを編集しlaravelのWELLCOME画面を表示させる](#section5)
1. [DBの設定を編集しユーザー登録などをできるようにする](#section6)
1. [参考サイト](#section7)
1. [所感](#section8)
<a id="section1"></a>

##  1. Dockerで動かすためのLaravelプロジェクトをインストール 　
  * コマンドラインにて
``cd 任意のアプリを作りたいディレクトリ
``
その後指定したディレクトリ上で
`` composer create-project --prefer-dist laravel/laravel laravel_app "6.*"
 ``コマンドにてバージョンを指定し,
 
 今回は**laravel laravel_app**という名前のアプリを作成します。 

  今回使用するのはバージョン6.0ということなので6.*とバージョンを指定しています

 <a id="section2"></a>

### 2. Dockerを使用してLaravelの環境構築をするにあたって便利な laradock を使用します。
*laladockとはlaravelアプリをDockerで動かす際に必要なファイルやイメージが用意されたファイルのことを指す。*
* まずは 
``` shell
$ git clone https://github.com/laradock/laradock.git 
 ```
 というコードを先ほどlaravel_appを作ったディレクトリ直下にて実行。
 便利な``laradock``を作業用ディレクトリに追加します。
その後

```　shell
$　cd　laradock　
```
を実行し``laradock``ディレクトリに移動します

<a id="section3"></a>

### 3. プロジェクトごとにdockerマシンを区別して使うための準備をしていきます。
``` shell
$ cp env.example .env 
``` 
実行後
その中に記載されている`` DATA_PATH_HOST `` の部分をLaravelのプロジェクト名と同じにしておきます。
では早速``$  vi env.example ``で中を覗いてみましょう。
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
``DATA_PATH_HOST=~/.laradock/data``
とあるので以下に編集をしましょう
``DATA_PATH_HOST=~/.laradock-laravel_app/data  ``
* ここから大切な`` docker-compose.yml``のファイルを編集していきます。
※この後に行う`` build ``コマンドなどを行う上でこのファイルを参照するのでインデントなどが崩れてしまうとエラーを起こすので注意してください。
* `` docker-compose.yml``では作成するコンテナの作業ディレクトリを指定します。
※ここでいうコンテナとは自分のOSのなかにある仮想OSのことです。
``` shell
$ vi docker-compose.yml
```
にてファイルを開き下記の様に
``working_dir``の行を追加し、その後のパスを追記してください。
 ```　
 ### Workspace Utilities ##################################
    workspace:
      build:
        context: ./workspace
        args:
          - LARADOCK_PHP_VERSION=${PHP_VERSION}

         # 省略

          - no_proxy
        working_dir: /var/www/laravel_app # この行は本来ないので追加
        volumes:
          - ${APP_CODE_PATH_HOST}:${APP_CODE_PATH_CONTAINER}${APP_CODE_CONTAINER_FLAG}
          - ./php-worker/supervisord.d:/etc/supervisord.d
 ```
ここまで編集が終わったら前準備は完了です。
<a id="section4"></a>

### 4. laradockにあるもので必要なものだけをbuildしましょう
* 今回必要となるコンテナは、以下です。それ以外のコンテナは作成する必要はありません。
  * workspace
  * mysql
  * apache
  * nginx
  * php-fpm

*　コマンドを実行しコンテナの作成を行いますが
僕との約束です。以下を守りましょう。
``docker-compose`` というコマンドは コマンドを実行したディレクトリの ``docker-compose.yml`` ファイルの内容に従って処理を行いますので、
**必ず** ``docker-compose.yml`` がある
**ディレクトリに移動してコマンドを実行しましょう。**

* それでは``docker-compose.yml``のあるディレクトリにて
``` shell
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
``` shell
$ docker-compose ps
```
コマンドで変更したコンテナがどうなっているのか見てみましょう！
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
そして、何かコンテナに変更を加えた際は
**必ず**
``` shell
$ docker-compose ps
```
上記コマンドで変更されているかチェックする癖をつけましょう。

ではここで一度[http://localhost](http://localhost)を確認しましょう。
どうでしょうか、みんなが大好きなlaravelのWELLCOMEは表示されましたか。
*答えはノーですよね。*
<a id="section5"></a>
### 5. WELLCOME画面を表示させるためにApacheの設定ファイルを編集していきましょう！
対象ファイルは、``laradock/apache2/sites/default.apache.conf`` です。
その後
``` shell
$　vi パス名/default.apache.conf
```
にて中身をみてください
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

そのディレクトリに戻れたら
``` shell
$ docker-compose build apache2
```
を実行したのち、
``` shell
$ docker-compose up -d workspace apache2
```
を実行してみましょう。
<br>
※``docker-compose up`` で起動した場合、Logを吐き出し続け入力を受け付けない状態となり不便なので、バックグラウンドで起動をし続けられるよう -d オプションを付けて実行しましょう。
<a id="section6"></a>

### 6. DBの設定をしよう
最初に作った**バージョン6.0のlaravel_appディレクトリ直下に**``.env``ファイルがあります。
ではその``.env``ファイルを
```shell
$ vi .envファイルのあるパス
```
で開きファイルを編集しましょう
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
では
``` shell
  $ docker-compose exec サービス名 bash
```
  という記述方法で
  ``サービス名``の部分を今回は``workspace``で指定しましょう。
``` shell
$ docker-compose exec workspace bash
```
を実行。
するとコマンドの様子が``root@XXXXXXXXX:/#``の様になりましたか。
この``XXXXXXXXX``部分はご自身の環境によって変わります。
<br>
コンテナ内にログインしたら
``` shell
root@XXXXXXXXX:php artisan migrate
```
を実行してみましょう。
マイグレーションが正常に実行できましたでしょうか？？
実行できたらもう一つの方法を試すために一度 
``` shell
root@XXXXXXXXX:php artisan migrate:reset
```
を実行してテーブルを削除して``exit``でコンテナから抜けましょう。
<br>
` 2. コンテナの外からコンテナ内にコマンドを実行する方法 `
では次にコンテナ外からマイグレーションを実行してみましょう。``docker-compose.yml``があるディレクトリにて
``` shell
$ docker-compose exec workspace php artisan migrate
```
を実行してください。
その後ログインなどDBに接続できれば成功です！
<a id="section7"></a>

## 7. 参考サイト
* [マークダウン記法のqiit](https://qiita.com/shizuma/items/8616bbe3ebe8ab0b6ca1)  コレを見ればなんとなくマークダウン記法が描ける様になります！
* [マークダウン記法ページ内リンク](https://shinshin86.hateblo.jp/entry/2020/04/08/224318)
* [dockerのコンテナとは](https://tech-lab.sios.jp/archives/18811)  コンテナと言われ簡単に説明がしたかったのでこちらのサイトを参考にしました。

<a id="section8"></a>

## 8. 所感
* 書簡として私の場合本来Vagrantでやる部分を記憶に新しいDockerでやらせていただいたので、とても難しいや分からない！ということでは無かったです。
* 基本的にカリキュラムも丁寧なため、しっかり文字を読み自分なりに落とし込めさえすれば、大幅に道が逸れるということはなかったです。
何よりメンターさんがいたことと、勢いに任せた部分があったので。
* **ここから大事なことを書きます！**
この学習の上でかなり勉強になったのは何かエラーが会った際は
一度``docker-compose down``で一度ダウンさせ、``buildまたはup``をすると
解決する場面が多々あったので、それは何かあったら思い出してみます。
* また、今回``laravel 6.0``にて認証やログイン機能を作れなかったのでそちらも挑戦したかったです。
* またマークダウン記法の確認がV.SCODEのみでできることも驚きました！
* 追記
``docker-compose down``について調べました。
調べてもなんとなくしか分からなかったですが、
``docker-compose down``はupで作成したコンテナ・ネットワーク・ボリューム・イメージを削除する。とあるので一度downで削除し、再構築をすることで、全てがリセットされ最新の変更に変わるため、治ったのではないかと思いました。
[docker-compose downの参考サイト](https://qiita.com/yusuke_mrmt/items/e05d7914065824384a6b)