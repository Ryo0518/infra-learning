# Infra Learning

LinuC、Linux、CCNA、ハンズオン、トラブルシューティングの学習記録。

## Logs

### ⭐︎UTM + Ubuntu キー入力トラブル

#### 発生事象
Macで外部モニター接続後、UTM上のUbuntuを操作中に異常発生。

viのコマンドモードで `q` を入力すると、Ubuntuデスクトップ左側サイドバーのアイコンに数字が表示された。

#### 環境
- Mac
- UTM
- Ubuntu
- 外部モニター接続

#### 原因
Command（⌘）キーが押しっぱなし状態として認識されていた可能性。

#### 解決方法
- Command（⌘）キーを数回押して離した

#### 学び
Linuxやviではなく、ホストOS側の入力状態も疑う。


### ⭐︎UTM Ubuntu SSH接続トラブル

#### 症状
Mac:AからSSH接続可能だがMac:Bからタイムアウト

#### 原因
UTMネットワークが「共有ネットワーク(NAT)」になっていた

#### 対応
UTM
→仮想マシン停止※サスペンド状態ではなく、停止中にすることで設定画面に進める
→設定
→ネットワーク
→Shared Network → Bridgedに変更

#### 確認

Ubuntu:
ip a

Mac:B:
ssh ryo@192.168.179.16
メモ：ネットワーク設定変更前のIPアドレスは192.178.64.3だった

#### 学び
- NATではホストの背後に隠れる
共有ネットワーク(NAT)時の現象を図にすると以下のとおり。
　　Mac:B ------ WiFiルータ ------- Mac:A
     　　                             ┗ UTM Ubuntu
UTM UbuntuがMac:Aの後ろに隠れており、Mac:Bからみても「Ubuntu無くない？」となっていた。

- BridgedではLAN上の独立マシンとして振る舞う
ブリッジモードにすることによって、UTM Ubuntuを独立した１台としてLAN上に参加させることができる。
これにより、UbuntuもWifiルータから直接IPをもらうことになる。
(NAT時もMac:Aとは異なる、UTM内部用のIPを持っていることは押さえておきたいポイント。しかし、Mac:Bや外部のサーバから見たらMac:AのIPアドレスしか見えない仕組みになっている。それは通信時にMac:AがUbuntuの代理人になっているからである)
そして、Mac:BがUTM Ubuntuを直接確認し、ssh接続することが可能になる。

- UTMの設定を行う際はサスペンド状態ではなく、完全停止させる必要がある
サスペンド状態では設定変更がロックされることがあるため

#### 切り分け手順
1. Mac:BからSSH接続するとタイムアウト
2. UTMネットワーク設定確認
3. Shared Network(NAT)を確認
4. Bridgedへ変更
5. ip a でIP確認
6. SSH再試行
7. 接続成功


### ⭐︎UFWによるHTTP接続の遮断トラブル

#### 症状
Mac:BのみHTTP接続を許可したはずなのにMac:Aのブラウザからもnginxの初期画面が表示されてしまう

#### 対応
- ポートの確認
sudo ss -tulpn | grep ':80'
確認結果
　0.0.0.0:80 LISTEN
- UFW設定の確認
sudo ufw status verbose
確認結果
　80/tcp  ALLOW IN 192.168.179.29
HTTP接続はMac:Bからのみになるように正しく設定されている。
そのほかにfrom Anywhereになっている 80/tcpは無し

#### 原因
ブラウザキャッシュの可能性がある
プライベートモードにして、Mac:AからubuntuWebサーバにアクセスしようとしたところ想定通り通信が失敗したため、解決。

#### 学んだこと
- ブラウザ表示だけでは通信成功とは限らない

#### SSH/Webサーバ構築メモ
Mac:A
└ UTM
   └ Ubuntu Server
      ├ SSH Server
      └ nginx Web Server

Mac:B
└ SSH/Web接続クライアント

#### UFW設定
Mac:BのみSSH/HTTP許可
sudo ufw allow from 192.168.179.29 to any port 2222 proto tcp
sudo ufw allow from 192.168.179.29 to any port 80 proto tcp
状態確認コマンド
sudo ufw status verbose


### ⭐︎自宅DNSサーバ構築ログ

#### ゴール
MacのブラウザでIP直打ちではなく、名前でUbuntu Webサーバへアクセスできるようにする。
例：
http://192.168.179.16
では無く、
http://web.lab.home
のようにアクセスすること

#### 対応
- bind9インストール
sudo apt update
sudo apt install bind9
※メモ※
Ubuntuのシステムクロックが１日ほどずれていたため、インストールが失敗した。システムクロックを合わせたのちインストール成功

- BIND設定
【ゾーン定義】
ファイル: /etc/bind/named.conf.local
設定：
zone "lab.home" {
     type master;
     file "/etc/bind/db.lab.home";
  };

【ゾーンファイル】
ファイル: /etc/bind/db.lab.home
設定:
$TTL 604800
@   IN  SOA lab.home. root.lab.home. (
        2
        604800
        86400
        2419200
        604800 )

@   IN  NS  lab.home.
@   IN  A   192.168.179.16

web IN  A   192.168.179.16

-------------------------------------------------------------
※メモ※
DNSサーバ(BIND)は、「どのドメインを管理するの？」を知る必要がある。
そのための設定が【ゾーン定義】、
「そのドメインの具体的な中身は？」を書くのが【ゾーンファイル】。

全体の流れ
Mac
 ↓
「web.lab.home のIP教えて」
 ↓
Ubuntu DNS(named)
 ↓
named.conf.local を見る
 ↓
「lab.home は db.lab.home だな」
 ↓
db.lab.home を読む
 ↓
web IN A 192.168.179.16
 ↓
「192.168.179.16です」
 ↓
MacがWebサーバへ接続

①ゾーン定義忘れるとどうなるか？
例えば、
zone "lab.home" {
    file "/etc/bind/db.lab.home";
};
を書き忘れる。
すると、BINDはlab.homeの問い合わせがあった時に
「lab.home？知らないドメインです」
状態になり、digコマンドの結果「status: NXDOMAIN」などになる。

②ゾーンファイルの設定をミスすると？
例えば、
web IN A 192.168.179.16
を書き忘れる。
すると、
lab.home自体は管理している。
でも、webが存在しない
状態になる。
-------------------------------------------------------------

- 動作確認
BIND設定チェック
sudo named-checkconf

ゾーンファイルチェック
sudo named-checkzone lab.home /etc/bind/db.lab.home

DNSサーバ再起動
sudo systemctl restart named

DNS待受確認
sudo ss -lntup | grep 53

DNS問い合わせ確認
dig @192.168.179.16 web.lab.home

#### トラブルログ
- 症状
digコマンドて問い合わせた結果、timed-outしてしまう。
connection timed out; no servers could be reached

- 原因
UFWで53番ポート開けていなかった

- 解決
sudo ufw allow 53/tcp
sudo ufw allow 53/udp

#### 学んだこと
- DNSは「名前→IP」の翻訳をおこなう
- UFWで53番ポートを開けておく必要がある
- ゾーン定義とゾーンファイルを設定しておく必要がある。




