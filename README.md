# さくらVPSを開発サーバーにする

さくらVPSを開発サーバーとして活用します


## 概要

さくらVPSをDockerホストにして、ローカル環境からデバッグを可能にします  
開発端末はMac/Windowsどちらも可能です  


## 必須

- リモートサーバー
  - 今回はさくらVPSを使用します
- Docker Tool Box
  - 開発端末(Mac/Windows)にインストールされている必要があります
- PhpStorm
  - PhpStormでしか試していませんが、おそらくNetBeans/Eclipseでも可能かと思います
    - NetBeansでも動作確認できました。


## おおまかな流れ

- リモートサーバーにUbuntuをインストール
- リモートサーバーにDockerをインストール
  - Dockerホストになります
- Dockerホスト上にコンテナを起動します
  - コンテナではXDebugを有効にします
- PhpStormの設定を行います


## セットアップ

### さくらVPSにUbuntuをインストール

- [Ubuntuインストールガイド](https://help.sakura.ad.jp/app/answers/detail/a_id/2403/~/カスタムosインストールガイド---ubuntu-12.04%2F14.04) を参照してください
- 途中でユーザー作成があるので、適宜作成してください
  - 例として「sockets」としました

### セキュリティ設定

インストール時に作成したユーザー/パスワードでsshログインしてください。  

#### sshd_configを変更します

```
sudo vi /etc/ssh/sshd_config
```
してから、

```
- Port 22
+ Port 2220
```

```
- PermitRootLogin yes
+ PermitRootLogin no
```

```
- PasswordAuthentication yes
+ PasswordAuthentication no
```
と修正  
※ 「-」が修正前、「+」が修正後

```
sudo systemctl restart sshd
```
で反映します。


#### ログインユーザー用ssh公開鍵を登録

```
mkdir -p ~/.ssh
echo 'ssh-rsa AAAAxxxx....' > ~/.ssh/authorized_keys
chmod 0600 ~/.ssh/authorized_keys
chmod 0700 ~/.ssh
```

#### sudoでパスワードなし実行を許可

ほんとはやりたくありませんが、docker-machine実行時にsudo実行します。  
なので、仕方なくsudoをパスワードなし実行させます。

```
sudo visudo
```
してから
```
sockets ALL=NOPASSWD: ALL
```
を追記  
※「sockets」はログインユーザー名です


#### 開発端末側での確認
ここらへんで、開発端末側(Mac/Windows)で別ターミナルウィンドウを起動して、以下でログインできることを確認すると良いです。  
※現在作業中のターミナルを抜けてしまうと、ログインできなくなった時にインストールし直しなので、からなず別ターミナルウィンドウで確認してください。

```
ssh -o 'StrictHostKeyChecking no' -i 秘密鍵のパス -p 2220 sockets@さくらVPSのIP
```

ログインできればOKです。  
できない場合は、設定を見直してください  



### 開発端末でdocker-machine実行

- Docker Tool Boxはインストール済みであること

以下のコマンドを開発端末で実行するとリモートサーバーをDockerホストにできます

```
docker-machine --debug create -d generic --generic-ip-address さくらVPSのIP --generic-ssh-user sockets --generic-ssh-key 秘密鍵のパス --generic-ssh-port 2220 sakura-vps
```
- --debug : デバッグ表示あり
- -d : ドライバ
  - awsやvirtualboxなどが指定できます。今回はgeneric
- --generic-ip-address : リモートサーバーのIPアドレス
- --generic-ssh-user : sshログインするユーザー名
  - Ubuntuインストール時に作成したユーザーを指定しています
- --generic-ssh-key : sshで使用する秘密鍵のパス
- --generic-ssh-port : sshログインポート
- sakura-vps : なんでも良いです。docker-machine名になります。

#### なぜかConnection Timeout

実行すると、
```
```
と表示され、60後くらいにTimeoutします。  
が、問題なくdocker-machine作成できていますので、無視してください。  
※回避方法がわからりませんでした...

#### Dockerホスト作成確認

開発端末で
```
docker-machine ls
```
すると
```
NAME             ACTIVE   DRIVER       STATE     URL                          SWARM
sakura-vps                generic      Running   tcp://さくらVPSのIP:2376
```
と表示されます。


### コンテナ作成

適当にnginx/php/sshdが入ったコンテナを作成します。  

#### Dockerファイルに必要なこと

- ユーザーを作成
  - wheelグループに追加すること
  - 公開鍵を登録すること
- コンテナにもセキュリティ設定すること
  - /etc/ssh/sshd_config を修正

例えば、以下のように。
```
# ユーザーの作成
RUN useradd sockets
RUN usermod -aG wheel sockets
RUN mkdir -p /home/sockets/.ssh
RUN echo "ssh-rsa AAAAxxx..." > /home/sockets/.ssh/authorized_keys
RUN chown sockets:sockets /home/sockets/.ssh
RUN chown sockets:sockets /home/sockets/.ssh/authorized_keys
RUN chmod 0600 /home/sockets/.ssh/authorized_keys
RUN chmod 0700 /home/sockets/.ssh
RUN chmod 0755 /home/sockets

# セキュリティ設定
RUN sed -ri 's/^#PermitEmptyPasswords no/PermitEmptyPasswords no/' /etc/ssh/sshd_config
RUN sed -ri 's/^#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
RUN sed -ri 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
RUN sed -ri 's/^#GatewayPorts no/GatewayPorts yes/' /etc/ssh/sshd_config
```

また、コンテナに含まれるphp.iniには以下を追記すること
```

[xdebug]
xdebug.remote_enable=1
xdebug.remote_autostart=1
xdebug.remote_host = 172.17.0.1
xdebug.remote_port=49190
xdebug.profiler_enable=1
xdebug.profiler_output_dir="/tmp"
xdebug.max_nesting_level=1000
xdebug.idekey = "PHPSTORM"
```
- xdebug.remote_host : コンテナからみたIP。正確にはdocker runしてからでもOK
- xdebug.remote_port : nginxでも9000を使用しているので、なんとなく変更


#### docker runする

docker実行に必要なファイル一式をリモートサーバーにアップロードします。

開発端末で以下を実行すると良いでしょう。
```
scp ./* -i 秘密鍵のパス -o 'StrictHostKeyChecking no' -p 2220 sockets@さくらVPSのIP:/home/sockets/
```
※カレントディレクトリにDockerファイルなどがある想定

リモートサーバー上で以下を実行します。
```
docker build -t minimum:latest -f Dockerfile .
```
でイメージファイルの作成してから、
```
docker run -id --name minimum-server -p 12220:22 -p 18080:80 -p 49190:49190 minimum
```
でコンテナ起動します。  
ポートフォワードはssh/http/xdebugを指定しています。
- -p 12220:22 : コンテナのsshポートへ外部からアクセスするには12220を指定する
- -p 18080:80 : コンテナのhttpポートへ外部からアクセスするには18080を指定する
- -p 49190:49190 : コンテナのxdebuポートへ外部からアクセスするには49190を指定する
  - php.iniで指定したxdebug.remote_portと同じ値にします


### 開発端末のPhpStorm設定

#### ソースコードのDeploy設定

- PhpStorm -> Preferences -> Build, Execution, Deployment -> Deployment
![](http://133.242.185.114:18080/assets/deployment_connection.png)
  - 「+」ボタンクリック
![](http://133.242.185.114:18080/assets/deployment_dialog.png)
  - 「Name」は適当に。例えば「Sakura-VPS Docker」
  - 「Connection」タブ
    - 「Type」は「SFTP」
    - 「SFTP host」は「さくらVPSのIPアドレス」
    - 「Port」は「12220」
    - 「Root Path」はnginxで設定したDocument Root
      - 例えば、ログインユーザーのhomeディレクトリ
    - 「User name」はログインユーザー名
      - 例えば、「sockets」
    - 「Auth type」は「Key pair」
    - 「Private key file」は秘密鍵のパス
    - 「Web server root URL」は「http://さくらのIPアドレス:18080」
    - 設定し終えたら「Test SFTP connection...」してみる
![](http://133.242.185.114:18080/assets/deployment_mapings.png)
  - 「Mappings」タブ
    - 「Local path」は開発端末のプロジェクトディレクトリ
    - 「Deployment path on 'Sakura-VPS Docker'」は「/」
    - 「Web path on server 'Sakura-VPS Docker'」は「/」
  - 「Apply」をクリック

- PhpStorm -> Preferences -> Build, Execution, Deployment -> Deployment -> Option
![](http://133.242.185.114:18080/assets/deployment_options.png)
  - 「Upload changed files automatically to the default server」は「Always」
  - 「Upload external changes」にチェックをする


#### リモートサーバーのPHP指定

- PhpStorm -> Preferences -> Languages & Framework -> PHP
![](http://133.242.185.114:18080/assets/php.png)
  - 「PHP language level」はコンテナにインストールしたPHPのバージョン
  - 「Interpreter」は「...」ボタンをクリック
    - 別ダイアログが開くので、「+」ボタンをクリック
![](http://133.242.185.114:18080/assets/php_interpreter_dialog.png)
      - 「Name」は適当に。例えば「Sakura-VPS PHP5.6」
      - 「Deployment configuration」を選択
      - 「Deployment configuration」は先ほど設定した「Sakura-VPS Docker」を選択
      - おそらく「PHP executable」は設定されているはずなので、「更新」ボタンをクリックし、「i」ボタンをクリック
        - php.iniの内容が表示されます
    - 「Apply」をクリック
  - 「Interpreter」は設定されているはずなので、「i」ボタンをクリックしてphp.iniの内容を確認
  - 「Apply」をクリック


#### Xdebug設定

- PhpStorm -> Preferences -> Languages & Framework -> PHP -> Debug
![](http://133.242.185.114:18080/assets/php_debug.png)
  - 「Debug port」は「49190」

- Run -> Start Listening for PHP Debug Connection を選択
![](http://133.242.185.114:18080/assets/run_menu.png)


#### サーバー設定

- PhpStorm -> Preferences -> Languages & Framework -> PHP -> Servers
![](http://133.242.185.114:18080/assets/php_servers.png)
  - 「Name」は適当に。例えば「Sakura-VPS Server」
  - 「Host」はさくらVPSのIP
  - 「Port」は「80」
    - Windowsはなぜか「18080」
  - 「Debugger」は「Xdebug」
  - 「Absolute path on server」はコンテナないのプロジェクト設置ディレクトリを指定
    - ここを設定しないとブレークポイントでブレークしても、ソースで表示されない



### ポートフォワードする

開発端末からリモートフォワードします。

#### Mac
```
ssh -vvv -f -N -R 49190:192.168.0.5:49190 -p 12220 -i 秘密鍵のパス sockets@さくらVPSのIP
```

- 192.168.0.5 はMacのリモートログインIPアドレス


#### Windows

Puttyを利用すると楽です。

![](http://133.242.185.114:18080/assets/putty.png)

- WindowsはipconfigしたIPv4アドレス
- Puttyのターミナルが起動中はフォワードされます


## デバッグ方法

### アクセスしたいURLを設定

- Run -> Edit Configurations...
![](http://133.242.185.114:18080/assets/menu_edit_configurations.png)

    - 「+」ボタンをクリック
    - 「PHP HTTP Request」を選択
    - 「Server」はサーバー設定したものを選択。例えば「Sakura-VPS Server」
    - 「URL」はプロジェクトURLを指定。POST/GETしたいURLを相対パスで指定してください
    - 「Apply」をクリック

### デバッグ実行

デバッグ実行ボタンを押すと、ブレークします
