# ISUCON14 問題

## 当日に公開したマニュアルおよびアプリケーションについての説明

- [ISUCON14 当日マニュアル](./docs/manual.md)
- [ISUCON14 アプリケーションマニュアル](./docs/ISURIDE.md)

## ディレクトリ構成

```
.
+- bench          # ベンチマーカー
+- browsercheck   # ブラウザチェック用スクリプト
+- development    # 開発環境用のdocker compose
+- docs           # ドキュメント類
+- envcheck       # 競技環境の確認用プログラム
+- frontend       # 問題アプリケーションのフロントエンド
+- provisioning   # Ansible および Packer
+- webapp         # リファレンス実装
```

## TLS証明書について

ISUCON14で使用したTLS証明書は`provisioning/ansible/roles/nginx/files/etc/nginx/tls`以下にあります。

本証明書は有効期限が切れている可能性があります。定期的な更新については予定しておりません。

## ISUCON14で使用した競技環境

- 競技者 VM 3台
  - InstanceType: c5.large (2vCPU, 4GiB Mem)
  - VolumeType: gp3 20GB
- ベンチマーカー VM 1台
  - ECS Fargate (8vCPU, 8GB Mem)

## AWS上での過去問環境の構築

AMIの作成方法、Go版のビルド、EC2への接続方法、PR用CIの設定は[provisioningのREADME](./provisioning/README.md)を参照してください。PR用CIは[`.github/workflows/isucon-ci.yml`](./.github/workflows/isucon-ci.yml)にあります。

## Ansibleでの環境構築

Ubuntu 24.04への競技者・ベンチマーカー環境の構築手順は[provisioningのREADME](./provisioning/README.md#ansibleでの直接構築)を参照してください。

## docker compose での環境構築（Go/Perl言語のみ）

作問時に利用した docker compose で環境を構築することもできます。ただし、スペックやTLS証明書の有無など競技環境とは異なります。

[Task](https://taskfile.dev/)を使用するので、事前にインストールしておいてください。

### アプリケーションの起動
```
$ task up
$ task go:run
```

### 負荷走行の実行

同一サーバー内でアプリケーションを起動している場合は、以下のコマンドで負荷走行を実行できます。
```
$ cd bench
$ task run-local
```

異なるホストに向けて負荷走行を行う場合は、以下のようなコマンドで負荷走行を実行できます。
```
$ cd bench
$ go run . run --target http://{{ 対象のIPアドレス }}:{{ 対象のポート番号 }} --payment-url http://{{ 対象のホストから見たベンチマーカーのIPアドレス }}:{{ 決済サーバーのポート番号 デフォルト:12345 }} -t 60
```

ベンチマーカーは負荷走行の実行中に決済サーバーとしても動作します。`--payment-url` は問題アプリケーションへ決済サーバーのURLを通知するためのオプションです。

ベンチマーカーを競技インスタンス上で動作させる場合は、競技インスタンス上で動作しているモックサーバーのポートと被ってしまうため `--payment-bind-port` オプションを指定してポートを変更してください。

静的ファイルのチェックに失敗する場合は `--skip-static-sanity-check` オプションを追加して実行することで、チェックをスキップできます。
（ただし、静的ファイル取得のリクエストもスキップされるため本番での負荷とは厳密には一致しなくなることに注意してください。）

## Links

- [ISUCON14 まとめ](https://isucon.net/archives/58818382.html)
