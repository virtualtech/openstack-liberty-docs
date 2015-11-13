Title: OpenStack構築手順書 Liberty版
Company: 日本仮想化技術
Version:0.9.1

#OpenStack構築手順書 Liberty版

<div class="title">
バージョン：0.9.1 (2015/11/13作成)<br>
日本仮想化技術株式会社
</div>

<!-- BREAK -->

##変更履歴

|バージョン|更新日|更新内容|
|:---|:---|:---|
|0.9.0|2015/11/02|Liberty版執筆開始|
|0.9.1|2015/11/13|Liberty版Beta1|


````
筆者注:このドキュメントはKilo版をベースに編集中です。提案や誤りの指摘は
Issue登録か、日本仮想化技術までメールにてお願いします。
https://github.com/virtualtech/openstack-liberty-docs/issues
````

<!-- BREAK -->

##目次
<!--TOC max3-->

<!-- BREAK -->

#Part.1 OpenStack 構築編
<br>
本章は、OpenStack Foundationが公開している公式ドキュメント「OpenStack Installation Guide for Ubuntu 14.04」の内容から、「Block Storage Service」までの構築手順を翻訳したものです。
OpenStackをUbuntu Server 14.04 ベースで構築する手順を解説しています。
Canonical社が提供するCloud Archiveリポジトリーを使って、OpenStackの最新版Libertyを導入しましょう。

<!-- BREAK -->

## 1. 構築する環境について

### 1-1 環境構築に使用するOS

本書はCanonicalのUbuntu ServerとCloud Archiveリポジトリーのパッケージを使って、OpenStack Libertyを構築する手順を解説したものです。

Ubuntu Serverでは新しいハードウェアのサポートを積極的に行うディストリビューションです。そのため、Linux KernelのバージョンをTrustyの場合は14.04.2以降のLTSのポイントリリースごとに、スタンダート版のUbuntuと同様のバージョンに置き換えてリリースしています。
- <https://wiki.ubuntu.com/Kernel/LTSEnablementStack>

一般的な利用では特に問題ありませんが、OpenStackとSDNのソリューションを連携した環境を作る場合などに、Linux KernelやOSのバージョンを考慮しなくてはならない場合があります。
また、Trustyの場合はLinux Kernel v3.13以外のバージョンのサポート期間は18ヶ月になっています。期限が切れた後はLinux Kernel v3.13にダウングレードするか、手動で新しいLinux Kernelをインストールすることができるメタパッケージを導入する必要がありますので注意してください。

本書ではサポート期間が長く、Trustyの初期リリースの標準カーネルであるLinux Kernel v3.13を使うために、以下のURLよりイメージをダウンロードしたUbuntu Server 14.04.1 LTS(以下Ubuntu Server)のイメージを使ってインストールします。インストール後`apt-get dist-upgrade`を行って最新のアップデートを適用した状態にしてください。Trustyではこのコマンドを実行してもカーネルのバージョンがアップグレードすることはありません。

本書は3.13.0-68以降のバージョンのカーネルで動作するUbuntu 14.04.3を想定しています。

- <http://old-releases.ubuntu.com/releases/14.04.1/ubuntu-14.04.1-server-amd64.iso>

<!-- BREAK -->

### 1-2 作成するサーバー（ノード）

本書はOpenStack環境をSQL,Controller,Computeの3台のサーバー上に構築することを想定しています。

SQL           | コントローラー | コンピュート 
------------- | -------------- | --------------
MariaDB       | RabbitMQ       | Linux KVM
              | NTP            | Nova Compute
              | Keystone       | Linux Bridge Agent
              | Glance
              | Nova
              | Neutron Server
              | Linux Bridge Agent
              | L3 Agent
              | DHCP Agent
              | Metadata Agent
              | Cinder
               

<!-- BREAK -->

### 1-3 ネットワークセグメントの設定

今回は2つのネットワークセグメントを用意し構成しています。

+ 内部ネットワーク(Instance Tunnels)
  ネットワークノードとコンピュートノード間のトンネル用に使用するネットワーク。インターネットへの接続は行えなくても構いません。

+ 外部ネットワーク(Management)
  外部との接続に使用するネットワーク。構築中はaptコマンドを使って外部リポジトリからパッケージなどをダウンロードするため、インターネット接続が必要となります。

OpenStack稼働後は、仮想マシンインスタンスに対しFloating IPアドレスを割り当てることで、外部ネットワークへ接続することができます。

なお、各種APIを外部公開する際にも使用できますが、今回の手順ではAPIの公開は行いません。

IPアドレスは以下の構成で構築されている前提で解説します。

|-|外部ネットワーク|内部ネットワーク|
|:---|:---|:---|
|インターフェース|eth0|eth1|
|ネットワーク|10.0.0.0/24|192.168.0.0/24|
|ゲートウェイ|10.0.0.1|なし|
|ネームサーバー|10.0.0.1|なし|

<!-- BREAK -->

### 1-4 各ノードのネットワーク設定

各ノードのネットワーク設定は以下の通りです。

+ sqlノード

|インターフェース|eth0|
|:---|:---|:---|
|IPアドレス|10.0.0.100|
|ネットマスク|255.255.255.0|
|ゲートウェイ|10.0.0.1|
|ネームサーバー|10.0.0.1|

+ controllerノード

|インターフェース|eth0|eth1|
|:---|:---|:---|
|IPアドレス|10.0.0.101|192.168.0.101|
|ネットマスク|255.255.255.0|255.255.255.0|
|ゲートウェイ|10.0.0.1|なし|
|ネームサーバー|10.0.0.1|なし|

+ computeノード

|インターフェース|eth0|eth1|
|:---|:---|:---|
|IPアドレス|10.0.0.102|192.168.0.103|
|ネットマスク|255.255.255.0|255.255.255.0|
|ゲートウェイ|10.0.0.1|なし|
|ネームサーバー|10.0.0.1|なし|

<!-- BREAK -->

### 1-5 Ubuntu Serverのインストール

#### 1-5-1 インストール

3台のサーバーに対し、Ubuntu Serverをインストールします。要点は以下の通りです。

+ 優先ネットワークインターフェースをeth0に指定  
 + インターネットへ接続するインターフェースはeth0を使用するため、インストール中はeth0を優先ネットワークとして指定します。
+ OSは最小インストール
 + パッケージ選択ではOpenSSH serverのみ選択


【インストール時の設定パラメータ例】

|設定項目|設定例|
|:---|:---|
|初期起動時のLanguage|English|
|起動|Install Ubuntu Server|
|言語|English - English|
|地域の設定|other→Asia→Japan|
|地域の言語|United States - en_US.UTF-8|
|キーボードレイアウトの認識|No|
|キーボードの言語|Japanese→Japanese|
|優先するNIC|eth0: Ethernet|
|ホスト名|それぞれのノード名(sql, controller, compute)|
|ユーザ名とパスワード|フルネームで入力|
|アカウント名|ユーザ名のファーストネームで設定される|
|パスワード|任意のパスワード|
|Weak password（出ない場合も）|Yesを選択|

<!-- BREAK -->

|設定項目|設定例|
|:---|:---|
|ホームの暗号化|任意|
|タイムゾーン|Asia/Tokyoであることを確認|
|パーティション設定|Guided - use entire disk and set up LVM|
|パーティション選択|sdaを選択|
|パーティション書き込み|Yesを選択|
|パーティションサイズ|デフォルトのまま|
|変更の書き込み|Yesを選択|
|HTTP proxy|環境に合わせて任意|
|アップグレード|No automatic updatesを推奨|
|ソフトウェア|OpenSSH serverのみ選択|
|GRUB|Yesを選択|
|インストール完了|Continueを選択|

```
筆者注:
Ubuntuインストール時に選択した言語がインストール後も使われます。
Ubuntu Serverで日本語の言語を設定した場合、標準出力や標準エラー出力が文字化けしたり、作成されるキーペア名が文字化けするなど様々な問題が起きますので、言語は英語を設定されることを推奨します。
```


#### 1-5-2 プロキシーの設定

外部ネットワークとの接続にプロキシーの設定が必要な場合は、aptコマンドを使ってパッケージの照会やダウンロードを行うために次のような設定をする必要があります。

- システムのプロキシー設定

```
# vi /etc/environment
http_proxy="http://proxy.example.com:8080/"
https_proxy="https://proxy.example.com:8080/"
```

- APTのプロキシー設定

```
# vi /etc/apt/apt.conf
Acquire::http::proxy "http://proxy.example.com:8080/";
Acquire::https::proxy "https://proxy.example.com:8080/";
```

より詳細な情報は下記のサイトの情報を確認ください。

- <https://help.ubuntu.com/community/AptGet/Howto>
- <http://gihyo.jp/admin/serial/01/ubuntu-recipe/0331>

<!-- BREAK -->

### 1-6 Ubuntu Serverへのログインとroot権限

Ubuntuはデフォルト設定でrootユーザーの利用を許可していないため、root権限が必要となる作業は以下のように行ってください。

+ rootユーザーで直接ログインできないので、インストール時に作成したアカウントでログインする。
+ root権限が必要な場合には、sudoコマンドを使用する。
+ rootで連続して作業したい場合には、sudo -iコマンドでシェルを起動する。

### 1-7 設定ファイル等の記述について

+ 設定ファイルは特別な記述が無い限り、必要な設定を抜粋したものです。
+ 特に変更の必要がない設定項目は省略されています。
+ [見出し]が付いている場合、その見出しから次の見出しまでの間に設定を記述します。
+ コメントアウトされていない設定項目が存在する場合には、値を変更してください。多くの設定項目は記述が存在しているため、エディタの検索機能で検索することをお勧めします。
+ 特定のホストでコマンドを実行する場合はコマンドの冒頭にホスト名を記述しています。

【設定ファイルの記述例】

```
controller# vi /etc/glance/glance-api.conf ←コマンド冒頭にこのコマンドを実行するホストを記述

[database] ←この見出しから次の見出しまでの間に以下を記述
#connection = sqlite:////var/lib/glance/glance.sqlite          ← 既存設定をコメントアウト
connection = mysql://glance:password@controller/glance         ← 追記


[keystone_authtoken] ← 見出し
#auth_host = 127.0.0.1 ← 既存設定をコメントアウト
auth_host = controller ← 追記

auth_port = 35357
auth_protocol = http
auth_uri = http://controller:5000/v2.0 ← 追記
admin_tenant_name = service ← 変更
admin_user = glance ← 変更
admin_password = password ← 変更
```
<!-- BREAK -->

## 2. OpenStackインストール前の設定

OpenStackパッケージのインストール前に各々のノードで以下の設定を行います。

+ ネットワークデバイスの設定
+ ホスト名と静的な名前解決の設定
+ リポジトリーの設定とパッケージの更新
+ Chronyサーバーのインストール（コントローラーノードのみ）
+ Chronyクライアントのインストール
+ Python用MySQL/MariaDBクライアントのインストール
+ MariaDBのインストール（SQLノードのみ）
+ RabbitMQのインストール（コントローラーノードのみ）

### 2-1 ネットワークデバイスの設定

各ノードの/etc/network/interfacesを編集し、IPアドレスの設定を行います。

#### 2-1-1 SQLノードのIPアドレスの設定

```
sql# vi /etc/network/interfaces

auto eth0
iface eth0 inet static
      address 10.0.0.100
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1
```

<!-- BREAK -->

#### 2-1-2 コントローラーノードのIPアドレスの設定

```
controller# vi /etc/network/interfaces

auto eth0
iface eth0 inet static
      address 10.0.0.101
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1

auto eth1
iface eth1 inet static
      address 192.168.0.101
      netmask 255.255.255.0
```

#### 2-1-3 コンピュートノードのIPアドレスの設定

```
compute1# vi /etc/network/interfaces

auto eth0
iface eth0 inet static
      address 10.0.0.102
      netmask 255.255.255.0
      gateway 10.0.0.1
      dns-nameservers 10.0.0.1

auto eth1
 iface eth1 inet static
       address 192.168.0.102
       netmask 255.255.255.0
```

#### 2-1-4 ネットワークの設定を反映

各ノードで変更した設定を反映させるため、ホストを再起動します。

```
# shutdown -r now
```

<!-- BREAK -->

### 2-2 ホスト名と静的な名前解決の設定

各ノードの/etc/hostsに各ノードのIPアドレスとホスト名を記述します。127.0.1.1の行はコメントアウトします。

#### 2-2-1 各ノードのホスト名の設定

各ノードのホスト名をhostnamectlコマンドを使って設定します。反映させるためには一度ログインしなおす必要があります。

（例）controllerの場合

```
# hostnamectl set-hostname controller
# cat /etc/hostname
controller
```

#### 2-2-2 各ノードの/etc/hostsの設定

すべてのノードで127.0.1.1の行をコメントアウトします。
またホスト名で名前引きできるように設定します。

（例）controllerの場合

```
# vi /etc/hosts
127.0.0.1 localhost
#127.0.1.1 controller ← 既存設定をコメントアウト
#ext
10.0.0.100 sql
10.0.0.101 controller
10.0.0.102 compute
#int
192.168.0.100 sql-int
192.168.0.101 controller-int
192.168.0.102 compute-int
```

<!-- BREAK -->

### 2-3 リポジトリーの設定とパッケージの更新

コントローラーノードとコンピュートノードで以下のコマンドを実行し、Liberty向けUbuntu Cloud Archiveリポジトリを登録します。

```
# apt-get install -y software-properties-common
# add-apt-repository cloud-archive:liberty Ubuntu Cloud Archive for OpenStack liberty
 More info: https://wiki.ubuntu.com/ServerTeam/CloudArchive
Press [ENTER] to continue or ctrl-c to cancel adding it    ← Enterキーを押す...
Importing ubuntu-cloud.archive.canonical.com keyring
OK
Processing ubuntu-cloud.archive.canonical.com removal keyring
OK
```

各ノードのシステムをアップデートします。Ubuntuではパッケージのインストールやアップデートの際にまず`apt-get update`を実行してリポジトリーの情報の更新が必要です。そのあと`apt-get -y dist-upgrade`でアップグレードを行います。カーネルの更新があった場合は再起動してください。

なお、`apt-get update`は頻繁に実行する必要はありません。日をまたいで作業する際や、コマンドを実行しない場合にパッケージ更新やパッケージのインストールでエラーが出る場合は実行してください。以降の手順では`apt-get update`を省略します。

```
# apt-get update && apt-get -y dist-upgrade
```


### 2-4 OpenStackクライアントのインストール

コントローラーノードでOpenStackクライアントをインストールします。依存するパッケージは全てインストールします。

```
controller# apt-get install python-openstackclient
```

<!-- BREAK -->

### 2-5 時刻同期サーバーのインストールと設定

#### 2-5-1 時刻同期サーバーChronyの設定

各ノードで時刻を正確にするために時刻同期サーバーのChronyをインストールします。

```
# apt-get install -y chrony
```

#### 2-5-2 コントローラーノードの時刻同期サーバーの設定

コントローラーノードで公開NTPサーバーと同期するNTPサーバーを構築します。
適切な公開NTPサーバー(ex.ntp.nict.jp etc..)を指定します。ネットワーク内にNTPサーバーがある場合はそのサーバーを指定します。

```
# vi /etc/chrony/chrony.conf
...
server NTP_SERVER iburst
```

内容変更した場合は設定を適用するため、NTPサービスを再起動します。

```
controller# service chrony restart
```

<!-- BREAK -->

#### 2-5-3 その他ノードの時刻同期サーバーの設定

SQLノードとコンピュートノードでcontrollerノードと同期するNTPサーバーを構築します。

```
compute# vi /etc/chrony/chrony.conf

#server 0.debian.pool.ntp.org offline minpoll 8 #デフォルト設定はコメントアウト
#server 1.debian.pool.ntp.org offline minpoll 8
#server 2.debian.pool.ntp.org offline minpoll 8
#server 3.debian.pool.ntp.org offline minpoll 8

server controller iburst
```

設定を適用するため、NTPサービスを再起動します。

```
network# service chrony restart
```

#### 2-5-4 NTPサーバーの動作確認

構築した環境でコマンドを実行して、各NTPサーバーが同期していることを確認します。

- 公開NTPサーバーと同期しているコントローラーノード

```
controller# chronyc sources
chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* chobi.paina.jp                2   8    17     1    +47us[ -312us] +/-   13ms
^- v157-7-235-92.z1d6.static     2   8    17     1  +1235us[+1235us] +/-   45ms
^- edm.butyshop.com              3   8    17     0  -2483us[-2483us] +/-   82ms
^- y.ns.gin.ntt.net              2   8    17     0  +1275us[+1275us] +/-   35ms
```

- コントローラーノードと同期しているその他ノード

```
compute# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* lib-controller                3   6    77    25   -509us[-1484us] +/-   13ms
```

<!-- BREAK -->

### 2-6 Python用MySQL/MariaDBクライアントのインストール

コントローラーノードとコンピュートノードでPython用のMySQL/MariaDBクライアントをインストールします。

```
# apt-get install -y python-pymysql
```

Python MySQLライブラリーはMariaDBと互換性があります。

<!-- BREAK -->

## 3. SQLノードの事前設定

### 3-1 MariaDBのインストール

SQLノードにデータベースサーバーのMariaDBをインストールします。

#### 3-1-1 パッケージのインストール

apt-getコマンドでmariadb-serverパッケージをインストールします。

```
sql# apt-get install -y mariadb-server
```

インストール中にパスワードの入力を要求されますので、MariaDBのrootユーザーに対するパスワードを設定します。
本例ではパスワードとして「password」を設定します。

#### 3-1-2 MariaDBの設定を変更

MariaDBの設定ファイルmy.cnfを開き以下の設定を変更します。

+ バインドアドレスをeth0に割り当てたIPアドレスへ変更
+ 文字コードをUTF-8へ変更

別のノードからMariaDBへアクセスできるようにするためバインドアドレスを変更します。加えて使用する文字コードをutf8に変更します。

※文字コードをutf8に変更しないとOpenStackモジュールとデータベース間の通信でエラーが発生します。

```
sql# vi /etc/mysql/my.cnf

[mysqld]
#bind-address = 127.0.0.1                   ← 既存設定をコメントアウト
bind-address = 10.0.0.100                   ← 追記(sqlノードのIPアドレス)
default-storage-engine = innodb             ← 追記
innodb_file_per_table                       ← 追記
collation-server = utf8_general_ci          ← 追記
init-connect = 'SET NAMES utf8'             ← 追記
character-set-server = utf8                 ← 追記
```

<!-- BREAK -->

#### 3-1-3 MariaDBサービスの再起動

変更した設定を反映させるためMariaDBのサービスを再起動します。

```
sql# service mysql restart
```

#### 3-1-4 MariaDBデータベースのセキュア化

mysql_secure_installationコマンドを実行すると、データベースのセキュリティを強化できます。必要に応じて設定を行ってください。

+ rootパスワードの入力

```
sql# mysql_secure_installation
In order to log into MariaDB to secure it, we'll need the current
password for the root user.  If you've just installed MariaDB, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.
Enter current password for root (enter for none):  password　← MariaDBのrootパスワードを入力
```

+ rootパスワードの変更

```
Setting the root password ensures that nobody can log into the MariaDB
root user without the proper authorisation.
You already have a root password set, so you can safely answer 'n'.
Change the root password? [Y/n]  n
```

+ anonymousユーザーの削除

```
By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.
Remove anonymous users? [Y/n]  y
```

<!-- BREAK -->

+ リモートからのrootログインを禁止

本例ではデータベースの操作は全てsqlノード上で行うことを想定しているため、リモートからのrootログインは禁止に設定します。必要に応じて設定してください。

```
Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.
Disallow root login remotely? [Y/n] y
```

+ testデーターベースの削除

```
By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.
Remove test database and access to it? [Y/n] y
```

+ 権限の再読み出し

```
Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.
Reload privilege tables now? [Y/n] y
....
Thanks for using MariaDB!
```

<!-- BREAK -->


#### 3-1-5 MariaDBクライアントのインストール

SQLノード以外のノードに、インストール済みのMariaDBと同様のバージョンのMariaDBクライアントをインストールします。

````
# apt-get install -y mariadb-client-5.5 mariadb-client-core-5.5
````

### 3-2 mytopのインストール

データベースの状態を確認するため、データベースパフォーマンスモニターツールのmytopをインストールします。

```
sql# apt-get install -y mytop
```

利用するには、sqlノードで次のように実行します。ロードアベレージやデータのin/outなどの情報を確認できます。

```
sql# mytop --prompt
Password: password   　← MariaDBのrootパスワードを入力
```

<!-- BREAK -->

## 4. コントローラーノードのインストール前設定

### 4-1 RabbitMQのインストール

OpenStackは、オペレーションやステータス情報を各サービス間で連携するためにメッセージブローカーを使用しています。OpenStackではRabbitMQ、Qpid、ZeroMQなど複数のメッセージブローカーサービスに対応しています。
本書ではRabbitMQをインストールする例を説明します。

#### 4-1-1 パッケージのインストール

rabbitmq-serverパッケージをインストールします。
標準リポジトリーにある最新版をインストールします。

```
controller# apt-get update
controller# apt-cache policy rabbitmq-server
rabbitmq-server:
  Installed: (none)
  Candidate: 3.5.4-1~cloud0
  Version table:
     3.5.4-1~cloud0 0
        500 http://ubuntu-cloud.archive.canonical.com/ubuntu/ trusty-updates/liberty/main amd64 Packages
     3.2.4-1 0
        500 http://ja.archive.ubuntu.com/ubuntu/ trusty/main amd64 Packages
controller# apt-get install -y rabbitmq-server=3.2.4-1  ← 11/4/2015 時点の最新版
controller# apt-mark hold rabbitmq-server               ← バージョンを固定
```

<!-- BREAK -->

#### 4-1-2 openstackユーザーと権限の設定

RabbitMQにアクセスするためのユーザーとしてopenstackユーザーを作成し、必要なパーミッションを設定します。次はRabbitMQのパスワードをpasswordにする例です。

```
controller# rabbitmqctl add_user openstack password
controller# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 4-1-3 待ち受けIPアドレス・ポートとセキュリティ設定の変更

以下の設定ファイルを作成し、RabbitMQの待ち受けポートとIPアドレスを定義します。

+ 待ち受け設定の追加

```
controller# vi /etc/rabbitmq/rabbitmq-env.conf

RABBITMQ_NODE_IP_ADDRESS=10.0.0.101    ← controllerのIPアドレス
RABBITMQ_NODE_PORT=5672
HOSTNAME=controller
```

以下の設定ファイルを作成し、localhost以外からもRabbitMQへアクセスできるように設定します。

+ リモート認証の許可

```
controller# vi /etc/rabbitmq/rabbitmq.conf

[{rabbit, [{loopback_users, []}]}].
```

<!-- BREAK -->

#### 4-1-4 RabbitMQサービス再起動と確認

+ ログの確認

メッセージブローカーサービスが正常に動いていないと、OpenStackの各コンポーネントは正常に動きません。RabbitMQサービスの再起動と動作確認を行い、確実に動作していることを確認します。

```
controller# service rabbitmq-server restart
controller# tailf /var/log/rabbitmq/rabbit@controller.log
...
=INFO REPORT==== 4-Nov-2015::11:55:21 ===
started TCP Listener on 10.0.0.101:5672   ←待受IPとポートを確認
=INFO REPORT==== 4-Nov-2015::11:55:21 ===
Server startup complete; 0 plugins started.
```

※新たなエラーが表示されなければ問題ありません。

+ デフォルトユーザーguestでRabbitMQのWeb管理画面にアクセス

次のように実行して、RabbitMQの管理画面を有効化します。

```
controller# rabbitmq-plugins enable rabbitmq_management
controller# service rabbitmq-server restart
```

ブラウザーで下記URLの管理画面にアクセスします。ユーザー:guest パスワード:guestでログインできればRabbitMQサーバー自体は正常です。

```
http://controller-node-ipaddress:15672
```

作成したopenstackユーザーでリモートからRabbitMQの管理画面にログインできないのは正常です。これはopenstackユーザーにadministrator権限が振られていないためです。ユーザー権限は「rabbitmqctl list_users」コマンドで確認、任意のユーザーに管理権限を設定するには「rabbitmqctl set_user_tags openstack administrator」のように実行するとログイン可能になります。

<!-- BREAK -->


### 4-2 環境変数設定ファイルの作成

#### 4-2-1 admin環境変数設定ファイルの作成

adminユーザー用環境変数設定ファイルを作成します。

```
controller# vi ~/admin-openrc.sh

export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export PS1='\u@\h \W(admin)\$ '
```

#### 4-2-2 demo環境変数設定ファイルの作成

demoユーザー用環境変数設定ファイルを作成します。

```
controller# vi ~/demo-openrc.sh
 
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=demo
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=password
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export PS1='\u@\h \W(demo)\$ '
```

<!-- BREAK -->

## 5. Keystoneインストールと設定（コントローラーノード）

各サービス間の連携時に使用する認証サービスKeystoneのインストールと設定を行います。

### 5-1 データベースの作成

Keystoneで使用するデータベースを作成します。
SQLサーバー上でMariaDBにデータベースkeystoneを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

### 5-2 データベースの確認

SQLノードにユーザーkeystoneでログインしデータベースの閲覧が可能であることを確認します。

```
sql# mysql -u keystone -p
Enter password:  ← MariaDBのkeystoneパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| keystone           |
+--------------------+
2 rows in set (0.00 sec)
```

<!-- BREAK -->

### 5-3 admin_tokenの決定

Keystoneのadmin_tokenに設定するトークン文字列を次のようなコマンドを実行して決定します。出力される結果はランダムな英数字になります。

```
controller# openssl rand -hex 10
45742a05a541f26ddee8
```

### 5-4 パッケージのインストール

Keystoneのインストール時にサービスの自動起動が行われないようにするため、以下のように実行します。

```
controller# echo "manual" > /etc/init/keystone.override
```

apt-getコマンドでkeystoneパッケージをインストールします。

```
controller# apt-get install -y keystone apache2 libapache2-mod-wsgi memcached python-memcache
```

<!-- BREAK -->

### 5-5 Keystoneの設定を変更

keystoneの設定ファイルを変更します。

```
controller# vi /etc/keystone/keystone.conf

[DEFAULT]
admin_token = 45742a05a541f26ddee8   ← 追記(5-1-3で出力されたキーを入力)
log_dir = /var/log/keystone          ← 設定されていることを確認
verbose = True        ← 追記(詳細なログを出力する)
...
[database]
#connection = sqlite:////var/lib/keystone/keystone.db    ← 既存設定をコメントアウト
connection = mysql+pymysql://keystone:password@sql/keystone  ← 追記
...
[memcache]...servers = localhost:11211  ← アンコメント
...
[token]
...
provider = uuid            ← アンコメント
driver = memcache          ← 追記 
...
[revoke]...driver = sql               ← アンコメント
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/keystone/keystone.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 5-6 データベースに表を作成

```
controller# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

<!-- BREAK -->

### 5-7 Apache Webサーバーの設定

+ controllerノードの/etc/apache2/apache2.confのServerNameにcontrollerノードのホスト名を設定します。

```
# Global configuration
#
ServerName controller
...
```

+ コントローラーノードで/etc/apache2/sites-available/wsgi-keystone.confを作成して、次の内容を記述します。

```
Listen 5000Listen 35357<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined
    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined
    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>
```

<!-- BREAK -->

+ バーチャルホストでIdentity serviceを有効に設定します。

```
controller# ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
```


### 5-8 サービスの再起動とDBの削除

+ Apache Webサーバーを再起動します。

```
controller# service apache2 restart
```

+ パッケージインストール時に作成される不要なSQLiteファイルを削除します。

```
controller# rm /var/lib/keystone/keystone.db
```

<!-- BREAK -->

### 5-9 サービスとAPIエンドポイントの作成

以下コマンドでサービスとAPIエンドポイントを設定します。

+ 環境変数の設定

```
controller# export OS_TOKEN=45742a05a541f26ddee8  ← 追記(5-1-3で出力されたキーを入力)
controller# export OS_URL=http://controller:35357/v3
controller# export OS_IDENTITY_API_VERSION=3
```

+ サービスを作成

```
controller# openstack service create \
  --name keystone --description "OpenStack Identity" identity
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Identity               |
| enabled     | True                             |
| id          | 492157c4ba4c432995a6ebbf579b8654 |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
```

+ APIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  identity public http://controller:5000/v2.0
controller# openstack endpoint create --region RegionOne \
  identity internal http://controller:5000/v2.0
controller# openstack endpoint create --region RegionOne \
  identity admin http://controller:35357/v2.0
```

<!-- BREAK -->

### 5-10 プロジェクト・ユーザー・ロールの作成

以下コマンドで認証情報（プロジェクト・ユーザー・ロール）を設定します。


+ adminプロジェクトの作成

```
controller# openstack project create --domain default \
  --description "Admin Project" admin
```

+ adminユーザーの作成

```
controller# openstack user create --domain default --password-prompt admin
User Password: password  #adminユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | f96730bfc0b24c30aea7e16393ca5cde |
| name      | admin                            |
+-----------+----------------------------------+
```

+ adminロールの作成
```
controller# openstack role create admin
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 3952dbf71af946a0be607fb282a2a176 |
| name  | admin                            |
+-------+----------------------------------+
```

+ adminプロジェクトとユーザーにadminロールを追加します。
```
controller# openstack role add --project admin --user admin admin
```

<!-- BREAK -->

+ serviceプロジェクトを作成
```
controller# openstack project create --domain default \
 --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 9307593a3c904114aa4266c0c9cd3087 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | None                             |
+-------------+----------------------------------+
```

+ demoプロジェクトの作成
```
controller# openstack project create --domain default \
 --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | b93bc87be9ed41219e8f9c8b5b74dea2 |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | None                             |
+-------------+----------------------------------+
```

+ demoユーザーの作成
```
controller# openstack user create --domain default \
 --password-prompt demo
User Password: password  #demoユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | 3d98e96da46e4d43a81a1e355dc2d36f |
| name      | demo                             |
+-----------+----------------------------------+
```

<!-- BREAK -->

+ userロールの作成
```
controller# openstack role create user
+-------+----------------------------------+
| Field | Value                            |
+-------+----------------------------------+
| id    | 5e1472c6480c474f9c0e255ce2b0d0d4 |
| name  | user                             |
+-------+----------------------------------+
```

+ demoプロジェクトとdemoユーザーにuserロールを追加します。
```
controller# openstack role add --project demo --user demo user
```

<!-- BREAK -->

### 5-11 Keystoneの動作を確認

他のサービスをインストールする前にIdentityサービスが正しく構築、設定されたか動作を検証します。

+ セキュリティを確保するため、一時認証トークンメカニズムを無効化します。
  + /etc/keystone/keystone-paste.iniを開き、[pipeline:public_api]と[pipeline:admin_api]と[pipeline:api_v3]セクション（訳者注:..のpipeline行）から、admin_token_authを取り除きます。

```
[pipeline:public_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension user_crud_extension public_service
...
[pipeline:admin_api]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension s3_extension crud_extension admin_service
...
[pipeline:api_v3]
pipeline = sizelimit url_normalize request_id build_auth_context token_auth json_body ec2_extension_v3 s3_extension simple_cert_extension revoke_extension federation_extension oauth1_extension endpoint_filter_extension endpoint_policy_extension service_v3
```

+ Keystoneへの作成が完了したら環境変数をリセットします。

```
controller# unset OS_TOKEN OS_URL
```

<!-- BREAK -->

動作確認のためadminおよびdemoテナントに対し認証トークンを要求してみます。
admin、demoユーザーのパスワードを入力する必要があります。

+ adminユーザーとして、Identity バージョン 2.0 API から管理トークンを要求します。

```
controller# openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name admin --os-username admin --os-auth-type password \
  token issue
Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-11-04T06:35:49.316276Z      |
| id         | 98026d8fb75745bdab492e0d35ff2afd |
| project_id | e7a1bb2fb5a1488fbdd761136d0d9daa |
| user_id    | f96730bfc0b24c30aea7e16393ca5cde |
+------------+----------------------------------+
```

正常に応答が返ってくると、/var/log/apache2/keystone_access.logにHTTP 200と記録されます。正常に応答がない場合は/var/log/apache2/keystone_error.logを確認しましょう。

```
...
10.0.0.101 - - [23/Jun/2015:09:55:21 +0900] "GET / HTTP/1.1" 300 789 "-" "python-keystoneclient"
10.0.0.101 - - [23/Jun/2015:09:55:24 +0900] "POST /v2.0/tokens HTTP/1.1" 200 1070 "-" "python-keystoneclient"
```

<!-- BREAK -->

+ adminユーザーで管理ユーザー専用のコマンドを使って、作成したプロジェクトを表示できることを確認します。

```
controller# openstack --os-auth-url http://controller:35357/v3   --os-project-domain-id default --os-user-domain-id default --os-project-name admin --os-username admin --os-auth-type password project list
Password:
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 9307593a3c904114aa4266c0c9cd3087 | service |
| b93bc87be9ed41219e8f9c8b5b74dea2 | demo    |
| e7a1bb2fb5a1488fbdd761136d0d9daa | admin   |
+----------------------------------+---------+
```

+ adminユーザーでユーザーを一覧表示して、先に作成したユーザーが含まれることを確認します。

```
controller# openstack --os-auth-url http://controller:35357/v3   --os-project-domain-id default --os-user-domain-id default --os-project-name admin --os-username admin --os-auth-type password user list
Password:
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 3d98e96da46e4d43a81a1e355dc2d36f | demo  |
| f96730bfc0b24c30aea7e16393ca5cde | admin |
+----------------------------------+-------+
```

+ adminユーザーでユーザーを一覧表示して、先に作成したロールが含まれることを確認します。

```
controller# openstack --os-auth-url http://controller:35357/v3   --os-project-domain-id default --os-user-domain-id default --os-project-name admin --os-username admin --os-auth-type password role list
Password:
+----------------------------------+-------+
| ID                               | Name  |
+----------------------------------+-------+
| 3952dbf71af946a0be607fb282a2a176 | admin |
| 5e1472c6480c474f9c0e255ce2b0d0d4 | user  |
+----------------------------------+-------+
```

<!-- BREAK -->

+ demoユーザーとして、Identity バージョン 3 API から管理トークンを要求します。

```
controller# openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name demo --os-username demo --os-auth-type password \
  token issue
Password:
+------------+----------------------------------+
| Field      | Value                            |
+------------+----------------------------------+
| expires    | 2015-11-04T06:42:24.143091Z      |
| id         | e1320cf3e1ca49b18448e43f1b873f05 |
| project_id | b93bc87be9ed41219e8f9c8b5b74dea2 |
| user_id    | 3d98e96da46e4d43a81a1e355dc2d36f |
+------------+----------------------------------+
```

<!-- BREAK -->


## 6. Glanceのインストールと設定

### 6-1 データベースの作成

MariaDBにデータベースglanceを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
 IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

### 6-2 データベースの確認

ユーザーglanceでログインしデータベースの閲覧が可能であることを確認します。

```
sql# mysql -u glance -p
Enter password: ← MariaDBのglanceパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| glance             |
+--------------------+
2 rows in set (0.00 sec)

```

<!-- BREAK -->

### 6-3 認証情報の作成

以下コマンドで認証情報を作成します。

+ 環境変数の設定

admin-openrc.shを読み込むと次のように出力が変化します。

```
controller# source admin-openrc.sh
controller ~(admin)#
```

+ glanceユーザーの作成

```
controller# openstack user create --domain default --password-prompt glance
User Password: password  #glanceユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | 6f51630f9dcb4a4aaca5e1c6f5805dd6 |
| name      | glance                           |
+-----------+----------------------------------+
```

+ adminロールをglanceユーザーとserviceプロジェクトに追加

```
controller# openstack role add --project service --user glance admin
```

<!-- BREAK -->

+ サービスの作成

```
controller# openstack service create --name glance \--description "OpenStack Image service" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image service          |
| enabled     | True                             |
| id          | 827aac89e9c2450191af2ddfffcfd687 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

+ サービスエンドポイントの作成

```
controller# openstack endpoint create --region RegionOne \
  image public http://controller:9292
controller# openstack endpoint create --region RegionOne \
  image internal http://controller:9292
controller# openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```

<!-- BREAK -->


### 6-4 パッケージのインストール

apt-getコマンドでglanceとglanceクライアントパッケージをインストールします。

```
controller# apt-get install -y glance python-glanceclient
```


### 6-5 Glanceの設定を変更

Glanceの設定を行います。glance-api.conf、glance-registry.confともに、[keystone_authtoken]に追記した設定以外のパラメーターはコメントアウトします。

```
controller# vi /etc/glance/glance-api.conf

[DEFAULT]
...
notification_driver = noop    ← アンコメント
...
[database]
#sqlite_db = /var/lib/glance/glance.sqlite         ← 既存設定をコメントアウト
connection = mysql+pymysql://glance:password@sql/glance   ← 追記
...
[glance_store]
...
default_store =file                                 ← 設定されていることを確認
filesystem_store_datadir = /var/lib/glance/images/  ← 設定されていることを確認
...
[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = glance
password = password        ← glanceユーザーのパスワード(5-2で設定したもの)
...
[paste_deploy]
...
flavor = keystone          ← 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-api.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

```
controller# vi /etc/glance/glance-registry.conf

[DEFAULT]
...
notification_driver = noop    ← アンコメント
...
[database]
#sqlite_db = /var/lib/glance/glance.sqlite             ← 既存設定をコメントアウト
connection = mysql+pymysql://glance:password@sql/glance   ← 追記

[glance_store]
...
default_store =file                                 ← 設定されていることを確認
filesystem_store_datadir = /var/lib/glance/images/  ← 設定されていることを確認

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = glancepassword = password              ← glanceユーザーのパスワード(5-2で設定したもの)
...
[paste_deploy]
flavor = keystone                ← 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/glance/glance-registry.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 6-6 データベースにデータを登録

下記コマンドにてglanceデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "glance-manage db_sync" glance
```

<!-- BREAK -->

### 6-7 Glanceサービスの再起動

設定を反映させるため、Glanceサービスを再起動します。

```
controller# service glance-registry restart && service glance-api restart
```

### 6-8 動作の確認と使用しないデータベースファイルの削除

サービスの再起動後、ログを参照しGlance RegistryとGlance APIサービスでエラーが起きていないことを確認します。

```
controller# tailf /var/log/glance/glance-api.log
controller# tailf /var/log/glance/glance-registry.log
```

インストール直後は作られていない場合が多いですが、コマンドを実行してglance.sqliteを削除します。

```
controller# rm /var/lib/glance/glance.sqlite
```

### 6-9 イメージの取得と登録

Glanceへインスタンス用仮想マシンイメージを登録します。ここでは、クラウド環境で主にテスト用途で利用されるLinuxディストリビューションCirrOSを登録します。

#### 6-9-1 環境変数の設定

Image serviceにAPIバージョン2.0でアクセスするため、スクリプトを修正して読み込み直します。

```
controller# cd
controller# echo "export OS_IMAGE_API_VERSION=2" | tee -a admin-openrc.sh demo-openrc.shcontroller# source admin-openrc.sh
```

#### 6-9-2 イメージの取得

CirrOSのWebサイトより仮想マシンイメージをダウンロードします。

```
controller# wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

<!-- BREAK -->

#### 6-9-3 イメージを登録

ダウンロードした仮想マシンイメージをGlanceに登録します。

```
controller# glance image-create --name "cirros-0.3.4-x86_64" --file cirros-0.3.4-x86_64-disk.img --disk-format qcow2 --container-format bare \
 --visibility public --progress
[=============================>] 100%
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
| container_format | bare                                 |
| created_at       | 2015-11-04T08:17:27Z                 |
| disk_format      | qcow2                                |
| id               | debb1779-fb3c-42a7-aa18-4f6d0c9446f7 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | cirros-0.3.4-x86_64                  |
| owner            | e7a1bb2fb5a1488fbdd761136d0d9daa     |
| protected        | False                                |
| size             | 13287936                             |
| status           | active                               |
| tags             | []                                   |
| updated_at       | 2015-11-04T08:17:27Z                 |
| virtual_size     | None                                 |
| visibility       | public                               |
+------------------+--------------------------------------+
```

#### 6-9-4 イメージの登録を確認

仮想マシンイメージが正しく登録されたか確認します。

```
controller# openstack image list
+--------------------------------------+---------------------+
| ID                                   | Name                |
+--------------------------------------+---------------------+
| debb1779-fb3c-42a7-aa18-4f6d0c9446f7 | cirros-0.3.4-x86_64 |
+--------------------------------------+---------------------+
```

<!-- BREAK -->


## 7. Novaのインストールと設定（コントローラーノード）

### 7-1 データベースの作成

MariaDBにデータベースnovaを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
IDENTIFIED BY 'password';
EOF
Enter password:           ← MariaDBのrootパスワードpasswordを入力
```

### 7-2 データベースの確認

ユーザーnovaでログインしデータベースの閲覧が可能であることを確認します。

```
sql# mysql -u nova -p
Enter password: ← MariaDBのnovaパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| nova               |
+--------------------+
2 rows in set (0.00 sec)

```

<!-- BREAK -->

### 7-3 認証情報の作成

以下コマンドで認証情報を作成します。

+ 環境変数の設定

```
controller# source admin-openrc.sh
```

+ novaユーザーの作成

```
controller# openstack user create --domain default --password-prompt nova
User Password: password  #novaユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | cf0f0c462bcf47d9ae25402b3dca4ac3 |
| name      | nova                             |
+-----------+----------------------------------+
```

+ novaユーザーをadminロールに追加

```
controller# openstack role add --project service --user nova admin
```

+ novaサービスの作成
```
controller# openstack service create --name nova \
  --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 34ab683e8d39484fb4474fc07bf9608c |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

<!-- BREAK -->

+ ComputeサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2/%\(tenant_id\)s
```

### 7-4 パッケージのインストール

apt-getコマンドでNova関連のパッケージをインストールします。

```
controller# apt-get install -y nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy \
nova-scheduler python-novaclient
```

<!-- BREAK -->

### 7-5 Novaの設定を変更

nova.confに下記の設定を追記します。

```
controller# vi /etc/nova/nova.conf

[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
#lock_path=/var/lock/nova   ←コメントアウト
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=ec2,osapi_compute,metadata
rpc_backend = rabbit        ←追記
auth_strategy = keystone    ←追記

# controllerノードのIPアドレス:10.0.0.101
my_ip = 10.0.0.101                          ←追記

network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
enabled_apis=osapi_compute,metadata

[vnc]
vncserver_listen = 10.0.0.101               ←追記
vncserver_proxyclient_address = 10.0.0.101  ←追記

（次ページに続きます...）
```

<!-- BREAK -->

```
（前ページ/etc/nova/nova.confの続き）
...
(↓これ以下追記↓)
[database]
connection = mysql+pymysql://nova:password@sql/nova

[oslo_messaging_rabbit]rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = novapassword = password     ← novaユーザーのパスワード(6-2で設定したもの)

[glance]
host = controller

[oslo_concurrency]lock_path = /var/lib/nova/tmp
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 7-6 データベースにデータを作成

下記コマンドにてnovaデータベースのセットアップを行います。

```
controller# su -s /bin/sh -c "nova-manage db sync" nova
```

### 7-7 Novaサービスの再起動

設定を反映させるため、Novaのサービスを再起動します。

```
controller# service nova-api restart && service nova-cert restart && \
service nova-consoleauth restart && service nova-scheduler restart && \
service nova-conductor restart && service nova-novncproxy restart
```

### 7-8 使用しないデータベースファイル削除

データベースはMariaDBを使用するため、使用しないSQLiteファイルを削除します。

```
controller# rm /var/lib/nova/nova.sqlite
```

### 7-9 Glanceとの通信確認

NovaのコマンドラインインターフェースでGlanceと通信してGlanceと相互に通信できているかを確認します。

```
controller# nova image-list
+--------------------------------------+---------------------+--------+--------+
| ID                                   | Name                | Status | Server |
+--------------------------------------+---------------------+--------+--------+
| debb1779-fb3c-42a7-aa18-4f6d0c9446f7 | cirros-0.3.4-x86_64 | ACTIVE |        |
+--------------------------------------+---------------------+--------+--------+
```

※Glanceに登録したCirrOSイメージが表示できれば問題ありません。

<!-- BREAK -->


## 8. Nova-Computeのインストール・設定（コンピュートノード）

ここまでコントローラーノードの環境構築を行ってきましたが、ここでコンピュートノードに切り替えて設定を行います。

### 8-1 パッケージのインストール

```
compute# apt-get update
compute# apt-get install -y nova-compute sysfsutils
```

### 8-2 Novaの設定を変更

novaの設定ファイルを変更します。

```
compute# vi /etc/nova/nova.conf

[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
#lock_path=/var/lock/nova       ← コメントアウト
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=ec2,osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone

my_ip = 10.0.0.103  ← IPアドレスで指定

network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = 10.0.0.103  ← IPアドレスで指定
novncproxy_base_url = http://controller:6080/vnc_auto.html
vnc_keymap = ja                             ← 日本語キーボードの設定

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password

[keystone_authtoken]
auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = novapassword = password     ← novaユーザーのパスワード(6-2で設定したもの)

[glance]
host = controller

[oslo_concurrency]lock_path = /var/lib/nova/tmp
```


次のコマンドを実行して正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

nova-computeの設定ファイルを開き、KVMを利用するように設定変更します。「egrep -c '(vmx|svm)' /proc/cpuinfo」とコマンドを実行して、0と出たらqemu、0以上の数字が出たらkvmをvirt_typeパラメーターに設定する必要があります。

まず次のようにコマンドを実行し、KVMが動く環境であることを確認します。CPUがVMXもしくはSVM対応であるか、コア数がいくつかを出力しています。0と表示される場合は後述の設定でvirt_type = qemuを設定します。

```
# cat /proc/cpuinfo |egrep 'vmx|svm'|wc -l
4
```

VMXもしくはSVM対応CPUの場合はvirt_type = kvmと設定することにより、仮想化部分のパフォーマンスが向上します。

```
compute# vi /etc/nova/nova-compute.conf

[libvirt]...
virt_type = kvm
```

<!-- BREAK -->

### 8-3 Novaコンピュートサービスの再起動

設定を反映させるため、Nova-Computeのサービスを再起動します。

```
compute# service nova-compute restart
```

### 8-4 コントローラーノードとの疎通確認

疎通確認はコントローラーノード上にて、admin環境変数設定ファイルを読み込んで行います。

```
controller# source admin-openrc.sh
```

#### 8-4-1 ホストリストの確認

controllerノードとcomputeノードが相互に接続できているか確認します。もし、StateがXXXなサービスがあった場合は、該当のサービスをserviceコマンドで起動してください。

```
controller# openstack compute service list -c Binary -c Host -c State
+------------------+----------------+-------+
| Binary           | Host           | State |
+------------------+----------------+-------+
| nova-cert        | controller     | up    |
| nova-consoleauth | controller     | up    |
| nova-scheduler   | controller     | up    | ← Novaのステータスを確認
| nova-conductor   | controller     | up    |
| nova-compute     | compute        | up    |
+------------------+----------------+-------+
```
※一覧にcomputeが表示されていれば問題ありません。Stateがupでないサービスがある場合は-cオプションを外して確認します。


#### 8-4-2 ハイパーバイザの確認

controllerノードよりcomputeノードのハイパーバイザが取得可能か確認します。

```
controller# openstack hypervisor list
+----+---------------------+
| ID | Hypervisor Hostname |
+----+---------------------+
|  1 | compute             |
+----+---------------------+
```

※Hypervisor一覧にcomputeが表示されていれば問題ありません。

<!-- BREAK -->


## 9. Neutronのインストール・設定（コントローラーノード）

### 9-1 データベースを作成

MariaDBにデータベースneutronを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

### 9-2 データベースの確認

MariaDBにNeutronのデータベースが登録されたか確認します。

```
controller# mysql -h sql -u neutron -p
Enter password: ← MariaDBのneutronパスワードpasswordを入力
...

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| neutron            |
+--------------------+
2 rows in set (0.00 sec)

```

※ユーザーneutronでログイン可能でデータベースが閲覧可能なら問題ありません。

<!-- BREAK -->

### 9-3 認証情報の設定

以下コマンドで認証情報を設定します。

+ 環境変数の設定

```
controller# source admin-openrc.sh
```

+ neutronユーザーの作成

```
controller# openstack user create --domain default --password-prompt neutron
User Password: password  #neutronユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | 5c811d7bc6844c079a3608e1d431f38f |
| name      | neutron                          |
+-----------+----------------------------------+
```

+ neutronユーザーをadminロールに追加

```
controller# openstack role add --project service --user neutron admin
```

<!-- BREAK -->

+ neutronサービスの作成

```
controller# openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 4e6ef921426a4e5c8f7e296d90d07f3e |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

+ neutronサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  network public http://controller:9696
controller# openstack endpoint create --region RegionOne \
  network internal http://controller:9696
controller# openstack endpoint create --region RegionOne \
  network admin http://controller:9696
```

<!-- BREAK -->


### 9-4 パッケージのインストール

本例ではネットワークの構成は公式マニュアルの「[Networking Option 2: Self-service networks](http://docs.openstack.org/liberty/install-guide-ubuntu/neutron-controller-install-option2.html)」の方法で構築する例を示します。

```
controller# apt-get update
controller# apt-get install neutron-server neutron-plugin-ml2 \
 neutron-plugin-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
 neutron-metadata-agent python-neutronclient
```

### 9-5 Neutronコンポーネントの設定を変更

+ Neutronサーバーの設定

```
controller# vi /etc/neutron/neutron.conf 

[DEFAULT]...
core_plugin = ml2             ←確認service_plugins = router      ←追記allow_overlapping_ips = True  ←変更
rpc_backend = rabbit          ←アンコメント
auth_strategy = keystone      ←アンコメント
notify_nova_on_port_status_changes = True   ←アンコメントnotify_nova_on_port_data_changes = True     ←アンコメントnova_url = http://controller:8774/v2        ←変更

[database]
#connection = sqlite:////var/lib/neutron/neutron.sqlite  ← 既存設定をコメントアウト
connection = mysql+pymysql://neutron:password@sql/neutron  ← 追記

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = serviceusername = neutronpassword = password       ← neutronユーザーのパスワード(9-2で設定したもの)

（次ページに続きます...）
```

<!-- BREAK -->

```
（前ページ/etc/neutron/neutron.confの続き）

[nova]（以下末尾に追記）
...
auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultregion_name = RegionOneproject_name = serviceusername = novapassword = password     ← novaユーザーのパスワード(6-2で設定したもの)

[oslo_concurrency]
#lock_path = $state_path/lock     ← コメントアウト 
lock_path = /var/lib/neutron/tmp  ← 追記
[oslo_messaging_rabbit]（以下追記）...rabbit_host = controller
rabbit_userid = openstack
rabbit_password = password
```

[keystone_authtoken]セクションは追記した設定以外は取り除くかコメントアウトしてください。

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ ML2プラグインの設定

```
controller# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
...
type_drivers = flat,vlan,vxlan           ← 追記
tenant_network_types = vxlan             ← 追記
mechanism_drivers = linuxbridge,l2population   ← 追記
extension_drivers = port_security              ← 追記

[ml2_type_flat]
...
flat_networks = public                   ← 追記

[ml2_type_vxlan]
...
vni_ranges = 1:1000                      ← 追記

[securitygroup]
...                                                     
enable_ipset = True                      ← アンコメント
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/ml2_conf.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

+ Linuxブリッジエージェントの設定

PUBLIC_INTERFACE_NAMEをパブリックネットワークに接続している側のNICを指定します。

```
controller# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = public:PUBLIC_INTERFACE_NAME  ← 追記
```

OVERLAY_INTERFACE_IP_ADDRESSは、先にPUBLIC_INTERFACE_NAMEを置き換えたNIC側のIPアドレスを設定します。

```
[vxlan]
enable_vxlan = True                        ← アンコメント
local_ip = OVERLAY_INTERFACE_IP_ADDRESS    ← 追記
l2_population = True                       ← 追記
```

エージェントとセキュリティグループの設定を行います。

```
[agent]
...
prevent_arp_spoofing = True         ← 追記
...
[securitygroup]
...
enable_security_group = True        ← アンコメント
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver 
↑ 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/neutron/plugins/ml2/linuxbridge_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

+ Layer-3エージェントの設定

external_network_bridgeは単一のエージェントで複数の外部ネットワークを有効にするには値を指定する必要はないため、値を空白にします。

```
# vi /etc/neutron/l3_agent.ini

[DEFAULT]  (最終行に以下を追記)
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
external_network_bridge =
```

+ DHCPエージェントの設定

```
# vi /etc/neutron/dhcp_agent.ini 

[DEFAULT]  (最終行に以下を追記)
...
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```

+ dnsmasqの設定

一般的にデフォルトのイーサネットのMTUは1500に設定されています。通常のEthernet フレームにVXLANヘッダが加算されるため、VXLANを使う場合は少なくとも50バイト多い、1550バイト以上のMTUが設定されていないと通信が不安定になったり、通信が不可能になる場合があります。これらはジャンボフレームを設定することで約9000バイトまでのMTUをサポートできるようになり対応可能ですが、ジャンボフレーム非対応のネットワーク機器を使う場合や、ネットワーク機器の設定を変更できない場合はVXLANの50バイトのオーバーヘッドを考慮して1450バイト以内のMTUに設定する必要があります。これらの制約事項はOpenStack環境でも同様で、インスタンスを起動する際にMTU 1450を設定することで、この問題を回避可能です。この設定はインスタンス起動毎にUserDataを使って設定することも可能ですが、次のように設定しておくと仮想DHCPサーバーでMTUの自動設定を行うことができるので便利です。

+ DHCPエージェントにdnsmasqの設定を追記

```
# vi /etc/neutron/dhcp_agent.ini 

[DEFAULT]
...
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf  ← 追記
```

+ DHCPオプションの26番(MTU)を定義

```
# vi /etc/neutron/dnsmasq-neutron.conf

dhcp-option-force=26,1450
```

+ Metadataエージェントの設定

インスタンスのメタデータサービスを提供するMetadata agentを設定します。

```
# vi /etc/neutron/metadata_agent.ini

[DEFAULT]
#auth_url = http://localhost:5000/v2.0      ← コメントアウト
auth_region = RegionOne
#admin_tenant_name = %SERVICE_TENANT_NAME%  ← コメントアウト？
#admin_user = %SERVICE_USER%                ← コメントアウト？
#admin_password = %SERVICE_PASSWORD%        ← コメントアウト？
...
auth_uri = http://controller:5000           ← これ以下追記
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = neutron
password = password     ← neutronユーザーのパスワード(9-2で設定したもの)
nova_metadata_ip = controller  ← Metadataホストを指定
metadata_proxy_shared_secret = METADATA_SECRET
```

Metadata agentの`metadata_proxy_shared_secret`に指定する値と、次の手順でNovaに設定する`metadata_proxy_shared_secret`が同じになるように設定します。任意の値を設定すれば良いですが、思いつかない場合は次のように実行して生成した乱数を使うことも可能です。

```
controller# openssl rand -hex 10
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/neutron/metadata_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 9-6 Novaの設定を変更

Novaの設定ファイルにNeutronの設定を追記します。

```
controller# vi /etc/nova/nova.conf

[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = neutron
password = password       ← neutronユーザーのパスワード(9-2で設定したもの)

service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```

METADATA_SECRETはMetadata agentで指定した値に置き換えます。

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

### 9-7 データベースの作成

コマンドを実行して、エラーがでないで完了することを確認します。

```
controller# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
> --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

No handlers could be found for logger "neutron.quota"
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.  
...
INFO  [alembic.runtime.migration] Running upgrade kilo -> c40fbb377ad, Initial Liberty no-op script.
INFO  [alembic.runtime.migration] Running upgrade c40fbb377ad -> 4b47ea298795, add reject rule
  OK
```

<!-- BREAK -->

### 9-8 コントローラーノードのNeutronと関連サービスの再起動

設定を反映させるため、controllerノードの関連サービスを再起動します。

まずNova APIサービスを再起動します。

```
controller# service nova-api restart
```

次にNeutron関連サービスを再起動します。

```
controller# service neutron-server restart && service neutron-plugin-linuxbridge-agent restart && service neutron-dhcp-agent restart && service neutron-metadata-agent restart && service neutron-l3-agent restart
```

### 9-9 動作の確認

ログを確認して、エラーが出力されていないことを確認します。

```
controller# tailf /var/log/nova/nova-api.log
controller# tailf neutron-server.log
controller# tailf neutron-metadata-agent.log
controller# tailf neutron-plugin-linuxbridge-agent.log
```

### 9-10 使用しないデータベースファイル削除

```
controller# rm /var/lib/neutron/neutron.sqlite
```

<!-- BREAK -->

## 10. Neutronのインストール・設定（コンピュートノード）

次にコンピュートノードの設定を行います。

### 10-1 パッケージのインストール

```
compute# apt-get update
compute# apt-get install neutron-plugin-linuxbridge-agent
```

### 10-2 設定の変更

+ Neutronの設定

```
compute# vi /etc/neutron/neutron.conf

[DEFAULT]
...
rpc_backend = rabbit                  ← アンコメント
auth_strategy = keystone              ← アンコメント

[keystone_authtoken]（既存の設定はコメントアウトし、以下を追記）
...auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = neutronpassword = password       ← neutronユーザーのパスワード(9-2で設定したもの)

[database]
# This line MUST be changed to actually run the plugin.
# Example:
# connection = sqlite:////var/lib/neutron/neutron.sqlite   ← コメントアウト

[oslo_messaging_rabbit]
...
# fake_rabbit = false
rabbit_host = controller           ← 追記
rabbit_userid = openstack          ← 追記
rabbit_password = password         ← 追記
```

<!-- BREAK -->

本書の構成では、コンピュートノードのNeutron.confにはデータベースの指定は不要です。次のコマンドを実行して正しく設定を行ったか確認します。

```
compute# less /etc/neutron/neutron.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

+ Linuxブリッジエージェントの設定

PUBLIC_INTERFACE_NAMEにはパブリック側のネットワークに接続しているインターフェイスを指定します。OVERLAY_INTERFACE_IP_ADDRESSはパブリック側に接続しているNICに設定しているIPアドレスを指定します。

追記と書かれていない項目は設定があればアンコメントして設定を変更、なければ追記してください。

```
compute# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = public:PUBLIC_INTERFACE_NAME

[vxlan]
enable_vxlan = True
local_ip = OVERLAY_INTERFACE_IP_ADDRESS
l2_population = True

[agent]
...
prevent_arp_spoofing = True   ← 追記

[securitygroup]
...
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
↑ 追記
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
compute# less /etc/neutron/plugins/ml2/linuxbridge_agent.ini | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 10-3 コンピュートノードのネットワーク設定

デフォルトではComputeはレガシーなネットワークを利用します。Neutronを利用するように設定を変更します。

```
compute# vi /etc/nova/nova.conf

[neutron]
...
url = http://controller:9696
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
region_name = RegionOne
project_name = service
username = neutron
password = password       ← neutronユーザーのパスワード(9-2で設定したもの)
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
compute# less /etc/nova/nova.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 10-4 コンピュートノードのNeutronと関連サービスを再起動

ネットワーク設定を反映させるため、コンピュートノードのNeutronと関連のサービスを再起動します。

```
compute# service nova-compute restart && service neutron-plugin-linuxbridge-agent restart
```

### 10-5 ログの確認

```
compute# tailf /var/log/nova/nova-compute.log
compute# tailf /var/log/neutron/neutron-plugin-linuxbridge-agent.log
```

### 10-6 Neutronサービスの動作を確認

`neutron agent-list`コマンドを実行してNeutronエージェントが正しく認識されており、稼働していることを確認します。

```
controller# source admin-openrc.sh
controller# neutron agent-list -c host -c alive -c binary
+------------+-------+---------------------------+
| host       | alive | binary                    |
+------------+-------+---------------------------+
| controller | :-)   | neutron-linuxbridge-agent |
| compute    | :-)   | neutron-linuxbridge-agent |
| controller | :-)   | neutron-dhcp-agent        |
| controller | :-)   | neutron-metadata-agent    |
| controller | :-)   | neutron-l3-agent          |
+------------+-------+---------------------------+
```

 ※コントローラーとコンピュートで追加され、neutron-linuxbridge-agentが正常に稼働していることが確認できれば問題ありません。念のためログも確認してください。

<!-- BREAK -->

`neutron ext-list`コマンドを実行して、Neutron Serverが読み込んでいる拡張機能の一覧を出力し、必要なモジュールが読み込まれていることを確認します。

```
controller# neutron ext-list
+-----------------------+-----------------------------------------------+
| alias                 | name                                          |
+-----------------------+-----------------------------------------------+
| dns-integration       | DNS Integration                               |
| ext-gw-mode           | Neutron L3 Configurable external gateway mode |
| binding               | Port Binding                                  |
| agent                 | agent                                         |
| subnet_allocation     | Subnet Allocation                             |
| l3_agent_scheduler    | L3 Agent Scheduler                            |
| external-net          | Neutron external network                      |
| flavors               | Neutron Service Flavors                       |
| net-mtu               | Network MTU                                   |
| quotas                | Quota management support                      |
| l3-ha                 | HA Router extension                           |
| provider              | Provider Network                              |
| multi-provider        | Multi Provider Network                        |
| extraroute            | Neutron Extra Route                           |
| router                | Neutron L3 Router                             |
| extra_dhcp_opt        | Neutron Extra DHCP opts                       |
| security-group        | security-group                                |
| dhcp_agent_scheduler  | DHCP Agent Scheduler                          |
| rbac-policies         | RBAC Policies                                 |
| port-security         | Port Security                                 |
| allowed-address-pairs | Allowed Address Pairs                         |
| dvr                   | Distributed Virtual Router                    |
+-----------------------+-----------------------------------------------+
```

<!-- BREAK -->


## 11. 仮想ネットワーク設定（コントローラーノード）

### 11-1 外部接続ネットワークの設定

#### 11-1-1 admin環境変数読み込み

外部接続用ネットワーク作成するためにadmin環境変数を読み込みます。

```
controller# source admin-openrc.sh
```

#### 11-1-2 外部ネットワーク作成

ext-netという名前で外部用ネットワークを作成します。

```
controller(admin)# neutron net-create ext-net --router:external \
 --provider:physical_network public --provider:network_type flat 
Created a new network:
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 78983ca2-77e3-4cdf-9747-ae1b34addeb7 |
| mtu                       | 0                                    |
| name                      | ext-net                              |
| port_security_enabled     | True                                 |
| provider:network_type     | flat                                 |
| provider:physical_network | public                               |
| provider:segmentation_id  |                                      |
| router:external           | True                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | e7a1bb2fb5a1488fbdd761136d0d9daa     |
+---------------------------+--------------------------------------+
```

<!-- BREAK -->

#### 11-1-3 外部ネットワーク用サブネット作成

ext-subnetという名前で外部ネットワーク用サブネットを作成します。

```
controller(admin)# neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=10.0.0.200,end=10.0.0.250 \
  --disable-dhcp --gateway 10.0.0.1 10.0.0.0/24
Created a new subnet:
+-------------------+----------------------------------------------------+
| Field             | Value                                              |
+-------------------+----------------------------------------------------+
| allocation_pools  | {"start": "10.0.0.200", "end": "10.0.0.250"}       |
| cidr              | 10.0.0.0/24                                        |
| dns_nameservers   |                                                    |
| enable_dhcp       | False                                              |
| gateway_ip        | 10.0.0.1                                           |
| host_routes       |                                                    |
| id                | ae350158-4b24-4731-8242-e3069785d322               |
| ip_version        | 4                                                  |
| ipv6_address_mode |                                                    |
| ipv6_ra_mode      |                                                    |
| name              | ext-subnet                                         |
| network_id        | 78983ca2-77e3-4cdf-9747-ae1b34addeb7               |
| subnetpool_id     |                                                    |
| tenant_id         | e7a1bb2fb5a1488fbdd761136d0d9daa                   |
+-------------------+----------------------------------------------------+
```

### 11-2 インスタンス用ネットワーク設定

#### 11-2-1 demo環境変数読み込み

インスタンス用ネットワーク作成するためにdemo環境変数読み込みます。

```
controller# source demo-openrc.sh
```

<!-- BREAK -->

#### 11-2-2 インスタンス用ネットワーク作成

demo-netという名前でインスタンス用ネットワークを作成します。

```
controller(demo)# neutron net-create demo-net
Created a new network:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| id                    | 5b6a8b87-fd03-46b5-a968-72bac5091b7c |
| mtu                   | 0                                    |
| name                  | demo-net                             |
| port_security_enabled | True                                 |
| router:external       | False                                |
| shared                | False                                |
| status                | ACTIVE                               |
| subnets               |                                      |
| tenant_id             | b93bc87be9ed41219e8f9c8b5b74dea2     |
+-----------------------+--------------------------------------+
```

#### 11-2-3 インスタンス用ネットワークサブネット作成

demo-subnetという名前でインスタンス用ネットワークサブネットを作成します。

```
controller(demo)# neutron subnet-create demo-net 192.168.0.0/24 \--name demo-subnet --gateway 192.168.0.1 --dns-nameserver 8.8.8.8
Created a new subnet:
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "192.168.0.2", "end": "192.168.0.254"} |
| cidr              | 192.168.0.0/24                                   |
| dns_nameservers   | 8.8.8.8                                          |
| enable_dhcp       | True                                             |
| gateway_ip        | 192.168.0.1                                      |
| host_routes       |                                                  |
| id                | ed0190db-1f51-40e0-babe-3c71fa541f40             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | demo-subnet                                      |
| network_id        | 5b6a8b87-fd03-46b5-a968-72bac5091b7c             |
| subnetpool_id     |                                                  |
| tenant_id         | b93bc87be9ed41219e8f9c8b5b74dea2                 |
+-------------------+--------------------------------------------------+
```

<!-- BREAK -->

### 11-3 仮想ネットワークルーター設定

仮想ネットワークルーターを作成して外部接続用ネットワークとインスタンス用ネットワークをルーターに接続し、双方でデータのやり取りを行えるようにします。


#### 11-3-1 demo-routerを作成

仮想ネットワークルータを作成します。

```
controller(demo)# neutron router-create demo-router
Created a new router:
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| external_gateway_info |                                      |
| id                    | 006df4cc-6fb2-40e7-af58-dc68b6755165 |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | b93bc87be9ed41219e8f9c8b5b74dea2     |
+-----------------------+--------------------------------------+
```

#### 11-3-2 demo-routerにsubnetを追加

仮想ネットワークルーターにインスタンス用ネットワークを接続します。

```
controller(demo)# neutron router-interface-add demo-router demo-subnet
Added interface 7337070d-455f-406d-8ebd-24dba7deea3a to router demo-router.
```

#### 11-3-3 demo-routerにgatewayを追加

仮想ネットワークルーターに外部ネットワークを接続します。

```
controller(demo)# neutron router-gateway-set demo-router ext-net
Set gateway for router demo-router
```

<!-- BREAK -->

### 11-4 ネットワークの確認

```
controller(admin)# source admin-openrc.sh
controller(admin)# neutron port-list -c id -c fixed_ips --max-width 20
(ネットワークポート一覧の表示)
+--------------------------+--------------------------+
| id                       | fixed_ips                |
+--------------------------+--------------------------+
| 7337070d-455f-406d-8ebd- | {"subnet_id": "ed0190db- |
| 24dba7deea3a             | 1f51-40e0-babe-          |
|                          | 3c71fa541f40",           |
|                          | "ip_address":            |
|                          | "192.168.0.1"}           |
| b14a20ba-5fe9-4953-8a36- | {"subnet_id": "ed0190db- |
| d18cdfbc64b5             | 1f51-40e0-babe-          |
|                          | 3c71fa541f40",           |
|                          | "ip_address":            |
|                          | "192.168.0.2"}           |
+--------------------------+--------------------------+
controller(admin)# neutron port-show 9477c16b-e5c2-430b-b4b4-37d415fe9602 -F device_owner --max-width 24
(IPアドレス10.0.0.200を持つデバイスを確認)
+--------------+------------------------+
| Field        | Value                  |
+--------------+------------------------+
| device_owner | network:router_gateway |
+--------------+------------------------+
# ping -c 3 10.0.0.200|grep "packet loss"
3 packets transmitted, 3 received, 0% packet loss, time 1999ms
(ルーターゲートウェイ宛に各ノードからpingコマンドの実行)
```

※応答が返ってくれば問題ありません。


### 11-5 インスタンスの起動確認

OpenStackの最低限の構成ができあがったので、ここでOpenStack環境がうまく動作しているか確認しましょう。
まずはコマンドを使ってインスタンスを起動するために必要な情報を集める所から始めます。環境設定ファイルを読み込んで、各コマンドを実行し、情報を集めてください。

```
controller# source demo-openrc.sh
controller# openstack image list
(起動イメージ一覧を表示)
+--------------------------------------+---------------------+
| ID                                   | Name                |
+--------------------------------------+---------------------+
| debb1779-fb3c-42a7-aa18-4f6d0c9446f7 | cirros-0.3.4-x86_64 |
+--------------------------------------+---------------------+

controller# openstack network list -c ID -c Name
(ネットワーク一覧を表示)
+--------------------------------------+----------+
| ID                                   | Name     |
+--------------------------------------+----------+
| 78983ca2-77e3-4cdf-9747-ae1b34addeb7 | ext-net  |
| 5b6a8b87-fd03-46b5-a968-72bac5091b7c | demo-net |
+--------------------------------------+----------+

controller# openstack security group list -c ID -c Name
(セキュリティグループ一覧を表示)
+--------------------------------------+---------+
| ID                                   | Name    |
+--------------------------------------+---------+
| 978dc272-58b0-4a7d-b232-30771e9fa7c2 | default |
+--------------------------------------+---------+

controller# # openstack flavor list -c Name -c Disk
(フレーバー一覧を表示)
+-----------+------+
| Name      | Disk |
+-----------+------+
| m1.tiny   |    1 |
| m1.small  |   20 |
| m1.medium |   40 |
| m1.large  |   80 |
| m1.xlarge |  160 |
+-----------+------+
```

nova bootコマンドを使って、インスタンスを起動します。正常に起動したらnova deleteコマンドでインスタンスを削除してください。

```
controller# nova boot --flavor m1.tiny --image "cirros-0.3.4-x86_64" --nic net-id=858c0fe5-ea00-4426-aa4e-f2a2484b2471 --security-group a66e2962-312f-45a4-bfd3-f86ec69c5582 vm1
(インスタンスを起動)

controller# watch nova list
(インスタンス一覧を表示)
+--------------------------------------+------+--------+------------+-------------+----------------------+
| ID                                   | Name | Status | Task State | Power State | Networks             |
+--------------------------------------+------+--------+------------+-------------+----------------------+
| 5eddf2a7-0287-46ef-b656-18ff51f1c605 | vm1  | ACTIVE | -          | Running     | demo-net=192.168.0.6 |
+--------------------------------------+------+--------+------------+-------------+----------------------+

# grep "ERROR\|WARNING" /var/log/rabbitmq/*.log
# grep "ERROR\|WARNING" /var/log/neutron/*
# grep "ERROR\|WARNING" /var/log/nova/*
(各ノードの関連サービスでエラーが出ていないことを確認)

controller# nova delete vm1
Request to delete server vm1 has been accepted.
(起動したインスタンスを削除)
```

<!-- BREAK -->
<!--11/13ここまで構築完了-->
<!--11/13ここまで編集完了-->

## 12. Cinderインストール（controllerノード）

### 12-1 データベース作成

MariaDBのデータベースにCinderのデータベースを作成します。

```
sql# mysql -u root -p << EOF
CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'password';
EOF
Enter password: ← MariaDBのrootパスワードpasswordを入力
```

#### 12-2 データベースの確認

MariaDBにCinderのデータベースが登録されたか確認します。

```
sql# mysql -u cinder -p
Enter password: ← MariaDBのcinderパスワードpasswordを入力
...
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| cinder             |
+--------------------+
2 rows in set (0.00 sec)
```

※ユーザーcinderでログイン可能でデータベースの閲覧が可能なら問題ありません。

<!-- BREAK -->


### 12-3 認証情報の設定

以下コマンドで認証情報を設定します。

+ 環境変数の設定

```
controller# source admin-openrc.sh
```

+ cinderユーザーの作成

```
controller# openstack user create --password-prompt cinder
User Password: password  #cinderユーザーのパスワードを設定(本例はpasswordを設定)
Repeat User Password: password
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | default                          |
| enabled   | True                             |
| id        | c5832de55fd6406f8faa6df5cb2c3bca |
| name      | cinder                           |
+-----------+----------------------------------+
```

+ cinderユーザーをadminロールに追加

```
controller# openstack role add --project service --user cinder admin
```

<!-- BREAK -->

+ cinderサービスの作成

```
controller# openstack service create --name cinder \--description "OpenStack Block Storage" volume
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | b40b3624f21d40f786347cb706741fc1 |
| name        | cinder                           |
| type        | volume                           |
+-------------+----------------------------------+

controller# openstack service create --name cinderv2 \--description "OpenStack Block Storage" volumev2
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Block Storage          |
| enabled     | True                             |
| id          | ed76fc5250de43bd8cf428fbf1e9b9c6 |
| name        | cinderv2                         |
| type        | volumev2                         |
+-------------+----------------------------------+
```

<!-- BREAK -->

+ Block StorageサービスのAPIエンドポイントを作成

```
controller# openstack endpoint create --region RegionOne \
  volume public http://controller:8776/v1/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  volume internal http://controller:8776/v1/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  volume admin http://controller:8776/v1/%\(tenant_id\)s
```

```  
controller# openstack endpoint create --region RegionOne \
  volumev2 public http://controller:8776/v2/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  volumev2 internal http://controller:8776/v2/%\(tenant_id\)s
controller# openstack endpoint create --region RegionOne \
  volumev2 admin http://controller:8776/v2/%\(tenant_id\)s
```

<!-- BREAK -->


### 12-4 パッケージインストール

本書ではBlock StorageコントローラーとBlock Storageボリュームコンポーネントを一台のマシンで構築するため、両方の役割をインストールします。

```
controller# apt-get update
controller# apt-get install -y lvm2 cinder-api cinder-scheduler cinder-volume python-mysqldb python-cinderclient 
```


### 12-5 Cinderの設定を変更

```
controller# vi /etc/cinder/cinder.conf

[DEFAULT]
...
auth_strategy = keystone      ← 確認
#lock_path = /var/lock/cinder ← コメントアウト

↓↓ 以下追記 ↓↓

rpc_backend = rabbit

my_ip = 10.0.0.101   #controllerノード
enabled_backends = lvm
glance_host = controller

[oslo_messaging_rabbit]rabbit_host = controllerrabbit_userid = openstackrabbit_password = password

[oslo_concurrency]lock_path = /var/lib/cinder/tmp

[database]
connection = mysql+pymysql://cinder:password@sql/cinder

[keystone_authtoken]auth_uri = http://controller:5000auth_url = http://controller:35357auth_plugin = passwordproject_domain_id = defaultuser_domain_id = defaultproject_name = serviceusername = cinderpassword = password       ← cinderユーザーのパスワード(12-2で設定したもの)

[lvm]volume_driver = cinder.volume.drivers.lvm.LVMVolumeDrivervolume_group = cinder-volumesiscsi_protocol = iscsiiscsi_helper = tgtadm
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/cinder/cinder.conf | grep -v "^\s*$" | grep -v "^\s*#"
```

<!-- BREAK -->

### 12-6 データベースに表を作成

```
controller# su -s /bin/sh -c "cinder-manage db sync" cinder
```

### 12-7 Cinderサービスの再起動

設定を反映させるために、Cinderのサービスを再起動します。

```
controller# service cinder-scheduler restart && service cinder-api restart
```

### 12-8 使用しないデータベースファイルを削除

```
controller# rm /var/lib/cinder/cinder.sqlite
```

<!-- BREAK -->

### 12-9 イメージ格納用ボリュームの作成

イメージ格納用ボリュームを設定するために物理ボリュームの設定、ボリュームの作成を行います。

#### 12-9-1 物理ボリュームを追加

本例ではcontrollerノードにハードディスクを追加して、そのボリュームをCinder用ボリュームとして使います。controllerノードを一旦シャットダウンしてからハードディスクを増設し、再起動してください。新しい増設したディスクはdmesgコマンドなどを使って確認できます。

```
controller# # dmesg |grep sd|grep "logical blocks"
[    1.361779] sd 2:0:0:0: [sda] 62914560 512-byte logical blocks: (32.2 GB/30.0 GiB)  ← システムディスク
[    1.362105] sd 2:0:1:0: [sdb] 33554432 512-byte logical blocks: (17.1 GB/16.0 GiB)  ← 追加ディスク
```

仮想マシンにハードディスクを増設した場合は/dev/vdbなどのようにデバイス名が異なる場合があります。

#### 12-9-2 物理ボリュームを設定

以下コマンドで物理ボリュームを作成します。

+ LVM物理ボリュームの作成

```
controller# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created
```

+ LVMボリュームグループの作成

```
controller# vgcreate cinder-volumes /dev/sdb
  Volume group "cinder-volumes" successfully created
```

+ /etc/lvm/lvm.confにデバイスを指定

```
devices {
...
filter = [ "a/sdb/", "r/.*/"]
```


#### 12-9-3 Cinder-Volumeサービスの再起動

Cinderストレージの設定を反映させるために、Cinder-Volumeのサービスを再起動します。

```
controller# service cinder-volume restart && service tgt restart
```

#### 12-9-4 admin環境変数設定ファイルを読み込み

adminのみ実行可能なコマンドを実行するために、admin環境変数を読み込みます。

```
controller# source admin-openrc.sh
```

<!-- BREAK -->

#### 12-9-5 Cinderサービスの確認

以下コマンドでCinderサービスの一覧を表示し、正常に動作していることを確認します。


```
controller# cinder service-list
+------------------+--------------------+------+---------+-------+----------------------------+-----------------+
|      Binary      |        Host        | Zone |  Status | State |         Updated_at         | Disabled Reason |
+------------------+--------------------+------+---------+-------+----------------------------+-----------------+
| cinder-scheduler |   lib-controller   | nova | enabled |   up  | 2015-11-13T09:25:51.000000 |        -        |
|  cinder-volume   | lib-controller@lvm | nova | enabled |   up  | 2015-11-13T09:25:51.000000 |        -        |
+------------------+--------------------+------+---------+-------+----------------------------+-----------------+
```

#### 12-9-6 Cinderボリュームの作成を試行

以下コマンドでCinderボリュームを作成し、正常にCinderが動作していることを確認します。

```
controller# openstack volume create --size 1 volume
(1GBのストレージを作成)
controller# openstack volume list -c "Display Name" -c "Size"
+--------------+------+
| Display Name | Size |
+--------------+------+
| volume       |    1 |
+--------------+------+
```

<!-- BREAK -->


## 13. Dashboardインストール・確認（コントローラーノード）

クライアントマシンからブラウザーでOpenStack環境を操作可能なWebインターフェイスをインストールします。

### 13-1 パッケージインストール

controllerノードにDashboardをインストールします。

```
controller# apt-get update
controller# apt-get install -y openstack-dashboard
```

### 13-2 Dashboardの設定を変更

インストールしたDashboardの設定を変更します。

```
controller# vi /etc/openstack-dashboard/local_settings.py 

...
OPENSTACK_HOST = "controller"    ← 変更
ALLOWED_HOSTS = '*'              ← 確認

CACHES = {                       ← 確認'default': {'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache','LOCATION': '127.0.0.1:11211',   }}

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"  ← 変更
```

次のコマンドを実行して正しく設定を行ったか確認します。

```
controller# less /etc/openstack-dashboard/local_settings.py  | grep -v "^\s*$" | grep -v "^\s*#"
```

念のため、リダイレクトするように設定しておきます（数字は待ち時間）。

```
controller# vi /var/www/html/index.html
...
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <meta http-equiv="refresh" content="3; url=/horizon" />    ← 追記
```

変更した変更を反映させるため、Apacheとセッションストレージサービスを再起動します。

```
controller# service apache2 reload
```

<!-- BREAK -->


### 13-3 Dashboardにアクセス

controllerノードとネットワーク的に接続されているマシンからブラウザで以下URLに接続してOpenStackのログイン画面が表示されるか確認します。

※ブラウザで接続するマシンは予めDNSもしくは/etc/hostsにcontrollerノードのIPを記述しておく等controllerノードの名前解決を行っておく必要があります。

```
http://controller/horizon/
```

※上記URLにアクセスしてログイン画面が表示され、ユーザーadminとdemoでログイン（パスワード:password）でログインできれば問題ありません。


<!-- BREAK -->

### 13-4 セキュリティグループの設定

OpenStackの上で動かすインスタンスのファイアウォール設定は、セキュリティグループで行います。ログイン後、次の手順でセキュリティグループを設定できます。

1.demoユーザーでログイン<br>
2.「プロジェクト→コンピュート→アクセスとセキュリティ」を選択<br>
3.「ルールの管理」ボタンをクリック<br>
4.「ルールの追加」で許可するルールを定義<br>
5.「追加」ボタンをクリック<br>

セキュリティーグループは複数作成できます。作成したセキュリティーグループをインスタンスを起動する際に選択することで、セキュリティグループで定義したポートを解放したり、拒否したり、接続できるクライアントを制限することができます。

### 13-5 キーペアの作成

OpenStackではインスタンスへのアクセスはデフォルトで公開鍵認証方式で行います。次の手順でキーペアを作成できます。

1.demoユーザーでログイン<br>
2.「プロジェクト→コンピュート→アクセスとセキュリティ」をクリック<br>
3.「キーペア」タブをクリック<br>
4.「キーペアの作成」ボタンをクリック<br>
5.キーペア名を入力<br>
6.「キーペアの作成」ボタンをクリック<br>
7.キーペア（拡張子:pem）ファイルをダウンロード<br>

インスタンスにSSH接続する際は、-iオプションでpemファイルを指定します。

```
client$ ssh -i mykey.pem cloud-user@instance-floating-ip  
```

<!-- BREAK -->

### 13-6 インスタンスの起動

前の手順でGlanceにCirrOSイメージを登録していますので、早速構築したOpenStack環境上でインスタンスを起動してみましょう。

1.demoユーザーでログイン<br>
2.「プロジェクト→コンピュート→イメージ」をクリック<br>
3.イメージ一覧から起動するOSイメージを選び、「インスタンスの起動」ボタンをクリック<br>
4.「インスタンスの起動」詳細タブで起動するインスタンス名、フレーバー、インスタンス数を設定<br>
5.アクセスとセキュリティタブで割り当てるキーペア、セキュリティーグループを設定<br>
6.ネットワークタブで割り当てるネットワークを設定<br>
7.作成後タブで必要に応じてユーザーデータの入力（オプション）<br>
8.高度な設定タブでパーティションなどの構成を設定（オプション）<br>
9.右下の「起動」ボタンをクリック<br>

### 13-7 Floating IPの設定

起動したインスタンスにFloating IPアドレスを設定することで、Dashboardのコンソール以外からインスタンスにアクセスできるようになります。インスタンスにFloating IPを割り当てるには次の手順で行います。

1.demoユーザーでログイン<br>
2.「プロジェクト→コンピュート→インスタンス」をクリック<br>
3.インスタンスの一覧から割り当てるインスタンスをクリック<br>
4.アクションメニューから「Floating IPの割り当て」をクリック<br>
5.「Floating IP割り当ての管理」画面のIPアドレスで「+」ボタンをクリック<br>
6.右下の「IPの確保」ボタンをクリック<br>
7.割り当てるIPアドレスとインスタンスを選択して右下の「割り当て」ボタンをクリック<br>

<!-- BREAK -->

### 13-8 インスタンスへのアクセス

Floating IPを割り当てて、かつセキュリティグループの設定を適切に行っていれば、リモートアクセスできるようになります。セキュリティーグループでSSHを許可した場合、端末からSSH接続が可能になります（下記は実行例）。

```
client$ ssh -i mykey.pem cloud-user@instance-floating-ip  
```

その他、適切なポートを開放してインスタンスへのPingを許可したり、インスタンスでWebサーバーを起動して外部PCからアクセスしてみましょう。

#Part.2 監視環境 構築編
<br>
準備中。。

<!-- BREAK -->

