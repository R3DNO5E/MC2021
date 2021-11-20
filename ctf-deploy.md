# Linuxシステムプログラミング入門 CTF問題サーバのデプロイ方法
ダウンロードしたファイルを展開してください．
```
tar xvf q4.tar.xz
```
Dockerをインストールしたのち，コンテナをビルドします．
```
sudo docker build -t ctf-q4:0.01 .
```
コンテナを実行します．
```
sudo docker run -d --privileged -p 13020:22 ctf-q4:0.01
```
localhostのポート13020にsshすればログインできます．
