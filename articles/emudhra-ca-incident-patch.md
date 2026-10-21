---
title: "2026年10月のeMudhra中間CA失効と、Python・Linux環境で行った信頼チェーン更新手順"
emoji: "🔒"
type: "tech"
topics: ["セキュリティ", "linux", "python", "ubuntu", "ssl"]
published: true
---

2026年10月20日、認証局（CA）であるeMudhraの中間CA証明書（8件）の秘密鍵が侵害されたインシデント（`emudhra.we-have-a-problem.com`）が公表されました。

ChromeやFirefoxなどの主要ブラウザは、OneCRLやCRLSetsなどの失効情報配信機能によってすでにブラウザ側で対象の中間証明書をブロックしています。一方、常時稼働しているデータ収集クローラーやLinuxサーバー（Ubuntuなど）、Pythonスクリプトといった「ブラウザ外のHTTPクライアント」は、OSのCA証明書パッケージや言語ごとのルート証明書バンドルを自前で更新しない限り、漏洩した中間CAを信頼し続けるリスクが残ります。

本稿では、運用中のUbuntuサーバー（Azure VM）およびローカルのPython環境において、対象の中間CAが信頼ストアに含まれているかの確認、パッケージの更新、および動作検証を行った実際の手順を記録します。

---

## 1. インシデントの概要とサーバー環境への影響

公表された内容によると、eMudhraが発行した8つの中間CA証明書において、秘密鍵の生成アルゴリズムに不備があり、公開証明書から秘密鍵を復元可能な状態になっていました。これにより、第三者が任意のドメインになりすました不正な証明書を発行できる状態となっていました。

* **ブラウザ環境**: Chrome、Firefox、Safari、Edgeの各チームにより、ブラウザの証明書検証層で即座に対象証明書が無効化されました。
* **サーバー・クローラー環境**: Linuxの `ca-certificates` や Pythonの `certifi` など、独自の証明書ストアを参照している環境では、個別に対策済みバージョンへ更新する必要があります。

---

## 2. サーバー（Ubuntu 22.04）での確認と更新

### 現在のパッケージバージョンの確認

まず、サーバーにインストールされている `ca-certificates` のバージョンを確認します。

```bash
dpkg -l | grep ca-certificates
```

実行結果の例：
```text
ii  ca-certificates   20260601~22.04.1   all   Common CA certificates
```

Ubuntu公式のセキュリティ情報（Bug #2171507）によると、対策済みバージョンは `20260816ubuntu0.22.04.1` 以降となっています。上記の環境では対策前の古いバージョンが適用されていました。

### パッケージの更新

`apt` 経由で `ca-certificates` のみを安全に更新します。

```bash
sudo apt-get update -qq
sudo apt-get install --only-upgrade -y ca-certificates
```

更新後の確認：
```bash
dpkg -l | grep ca-certificates
```

実行結果：
```text
ii  ca-certificates   20260816ubuntu0.22.04.1   all   Common CA certificates
```

証明書の更新フックが走り、対象の中間CAが信頼リストから正常に除去されました。

---

## 3. Python（certifi）環境の確認と更新

Pythonの `requests` や `urllib3` などのHTTP通信ライブラリは、システムの証明書ではなく Pythonパッケージ `certifi` にバンドルされた Mozilla のルート証明書リストを使用することが多いです。

### certifi のバージョン確認

```bash
python3 -c "import certifi; print('certifi version:', certifi.__version__)"
```

実行結果の例：
```text
certifi version: 2025.11.12
```

対策済みのバージョンは `2026.10.19` 以降であるため、こちらも更新が必要でした。

### pip による certifi のアップデート

```bash
pip install -U certifi
```

実行結果：
```text
Collecting certifi
  Downloading certifi-2026.10.19-py3-none-any.whl (136 kB)
Installing collected packages: certifi
  Attempting uninstall: certifi
    Successfully uninstalled certifi-2025.11.12
Successfully installed certifi-2026.10.19
```

`certifi.where()` が指す証明書バンドルが `2026.10.19` に更新され、Python環境からの通信においても問題のある中間CAチェーンが信頼対象から外れました。

---

## 4. 動作確認

更新後、サーバー上で稼働しているクローラーやバッチ処理の外部通信テストを実施しました。

1. **自社サービスへのHTTPS通信**:
   自社ドメイン（Cloudflare / Google Trust Services 経由）へのアクセスが `HTTP 200` で正常に検証・完了することを確認。
2. **外部APIへのHTTPS通信**:
   GitHub APIや連携先各社サーバーへの通信が正常に行われ、SSL/TLSハンドシェイクに問題がないことを確認。

---

## まとめ

Webブラウザは自動失効の仕組みが整っていますが、バッチ処理やスクレイピング、API連携を行うサーバー環境では、証明書バンドルの更新が手動または定期的なパッケージ更新に依存します。

今回のように中間CAの秘密鍵漏洩が公表された際は、以下の2点を確認・更新することが確実です。

1. OSの証明書ストア（`ca-certificates`）
2. 言語固有の証明書バンドル（Pythonの `certifi`、Node.js、Go等）
