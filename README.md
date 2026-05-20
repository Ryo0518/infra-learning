# Infra Learning

LinuC、Linux、CCNA、ハンズオン、トラブルシューティングの学習記録。

## Logs

### UTM + Ubuntu キー入力トラブル

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
