# Mac で Ansible で LAMP

## ゴール
Mac 上に Ansible + Vagrant で LAMP 開発環境をつくります。

- CentOS 6
- Apache 2.2
- MySQL 5 系
- PHP 最新
- Git

プロジェクトに入った人が１日目にコマンドをばーんと打ち込んでお昼を食べにいけば席に戻るまでに出来上がってる、みたいのが理想です。また、必要に応じて何度も作り直せるとなお良いです。

## やらないこと
- production環境の作り方（あくまで開発環境）
- Javaなど他言語の開発環境の作り方

## 手順
Virtualboxをインストールします。（brew caskから入れたほうがよい？）

Vagrantをインストールします。（brew caskから入れたほうがよい？）
https://www.vagrantup.com/downloads.html

Homebrew をインストールします。Xcode のインストールを要求されるかもしれません、その場合 App Store 等からダウンロードしてインストールしてください。
```
$ ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
Ansible の動作には Python が必要なのでインストールします。
```
$ brew install python
```
Ansibleをインストールします。
```
$ brew install ansible
```
作業用ディレクトリを作成します（どこでもOK）。
```
$ cd
$ mkdir vagrant-ansible
$ cd vagrant-ansible
```
Vagrantを初期化します。
```
$ vagrant init hansode/centos-6.6-x86_64
```
init のあとは使いたいVMのタイプを指定します。指定できるものは https://atlas.hashicorp.com/search で検索できます。ここでは CentOS 6.6 を指定しました。

ファイル Vagrantfile が生成されますので編集します。

AnsibleはVMへSSH接続することでサーバ設定を行うため、ローカルからのSSH接続ができるように設定を変更します。
```
config.vm.network "private_network", ip: "192.168.33.10"
```
仮想マシンをダウンロード＆起動します。時間がかかります（最初だけ）
```
$ vagrant up
```
ちなみに仮想マシンを作りなおすときはこうです。
```
$ vagrant destroy
$ vagrant up
```
仮想マシンにSSHログインできることを確認します。
```
$ vagrant ssh
```
VMへSSH接続するための情報を書き出します。
```
$ vagrant ssh-config --host 192.168.33.10 >> ~/.ssh/config
```
SSHできるかテストします。
```
$ ssh 192.168.33.10
```
Ansibleに管理先サーバを教えるための hosts ファイルを作成します。
```
$ echo 192.168.33.10 > hosts
```
Ansibleから管理先サーバへ ping を打ちます。
```
$ ansible -i hosts all -m ping
```
こうなれば成功です。これでAnsibleからサーバを操作できるようになりました。
```
192.168.33.10 | success >> {
    "changed": false, 
    "ping": "pong"
}
```
こんなふうに任意のコマンドを実行させることもできます。
```
$ ansible -i hosts all -a 'uname -a'
192.168.33.10 | success | rc=0 >>
2.6.32-504.el6.x86_64
```
インベントリファイル（先ほどの hosts ファイル）を編集し、管理先サーバに名前をつけます。複数指定できますが、今回は開発環境用ですので１つで十分でしょう。
```
[dev-servers]
192.168.33.10
```
続いて LAMP 環境を構築するための Playbook を作成します。Playbook は Ansible にさせたいことを記述するファイルで、これの作成が本作業の肝になります。

※ 内容は playbook.yml を見てください

できあがった Playbook の文法チェックをします。
```
$ ansible-playbook -i hosts playbook.yml --syntax-check
```
テスト実行もできます。
```
$ ansible-playbook -i hosts playbook.yml --check
```
いよいよ Playbook を実行します。
```
$ ansible-playbook -i hosts playbook.yml
```
```
PLAY [dev-servers] ************************************************************

GATHERING FACTS ***************************************************************
ok: [192.168.33.10]

TASK: [iptables stopped] ******************************************************
ok: [192.168.33.10]

TASK: [yum.repos.d/CentOS-Base.repo is fixed] *********************************
changed: [192.168.33.10]

TASK: [install Apache] ********************************************************
changed: [192.168.33.10] => (item=httpd)

TASK: [install MySQL] *********************************************************
changed: [192.168.33.10] => (item=mysql,mysql-server,mysql-devel)

TASK: [install PHP] ***********************************************************
changed: [192.168.33.10] => (item=php,php-devel,php-mbstring,php-mysql)

TASK: [other] *****************************************************************
ok: [192.168.33.10] => (item=git)

NOTIFIED: [restart apache] ****************************************************
changed: [192.168.33.10]

NOTIFIED: [mysql setup] *******************************************************
changed: [192.168.33.10]

NOTIFIED: [mysql set password] ************************************************
changed: [192.168.33.10]

PLAY RECAP ********************************************************************
192.168.33.10              : ok=10   changed=7    unreachable=0    failed=0
```

Webサーバが動いているか http://192.168.33.10/ にアクセスして確認します。

MySQL に接続できるか見てみます。パスワードは playbook.yml で設定したものです。
```
$ vagrant ssh
# mysql -uroot -p
```

PHP の動作確認をします。
```
vagrant ssh
sudo vi /var/www/html/test.php
```
```
<?php phpinfo(); ?>
```
```
sudo chown apache:apache /var/www/html/test.php
```
http://192.168.33.10/test.php にアクセスし、PHP情報が表示されればOKです。

## TODO
- できた環境に git から本物っぽい PHP プロジェクトを取り込んで動かし、改修するところまでやってみたい
- ユースケースに応じて、playbook.yml のパターンをいくつか考えてみたい
- 社内で Ansible 経験のある人にみせてツッコミもらうのもいいかも
- せっかくだから動作確認の手順も自動化したいですね!! serverspecとか使えばできるのかな

## 作成者
Naoko Yamazaki <yamazaki.n@opst.co.jp>

## 参考サイト
- Ansible公式 http://docs.ansible.com/intro_getting_started.html
- Ansibleチュートリアル http://yteraoka.github.io/ansible-tutorial/
- YAMLの文法 http://magazine.rubyist.net/?0009-YAML
