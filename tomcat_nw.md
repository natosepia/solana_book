**検討中のアーキテクチャ**

ユーザーから依頼された要件に基づき、HTTPdサーバー(A)とTomcatサーバー(B)の連携方法を考えています。AJPプロトコルを使用してTomcatへリクエストをプロキシするため、`mod_proxy_ajp`を使います。静的ファイル（例: `/var/data/xxx.csv`）は、Tomcat経由で提供可能ですが、NFS経由でHTTPdにマウントする方法もあります。さらに、Tomcatでファイルを直接提供する方法と、HTTPd経由で提供する方法の提案が必要です。

**アーキテクチャと設定提案**

HTTPdサーバー(A)は、mod\_proxy\_ajpを使用してTomcatサーバー(B)にアプリケーションのリクエストをプロキシします。静的ファイルはTomcatで提供できますが、NFS共有を使ってファイルをHTTPdに直接提供する方が効率的です。設定例として、httpd.confで`ProxyPass`と`ProxyPassReverse`を使い、Tomcatの`server.xml`でもAJPコネクタが必要です。セキュリティのため、`secret`と`ajpSecret`を適切に設定することが重要です。


**アーキテクチャの概要**

Server AでHTTPdがmod\_proxy\_ajpを使用してTomcatのアプリケーションパスをプロキシします。静的ファイルはTomcatで提供可能で、`/var/data`ディレクトリをエイリアスで設定するか、HTTPdがそのURIをTomcatへプロキシします。設定例として、`ProxyPass`や`ProxyPassReverse`をHTTPd.confに追加します。Tomcat 10では、`secret`と`address`の設定が必要で、適切なセキュリティ設定が求められます。

**Tomcat 10 AJP設定の確認**

Tomcat 10では、AJPのデフォルトで`secretRequired`が`true`に変更されました。これにより、AJP接続には秘密鍵が要求されます。この変更に関する詳細は公式のドキュメントで確認できます（リンクは提供します）。また、パラメータ名や設定項目（例えば`async`など）も最新の情報を確認する必要があります。HTTPdのmod\_proxy\_ajpについても、`secret`パラメータの設定方法を調べる必要があります。


**TomcatとHTTPdの設定**

Tomcatの`server.xml`でAJPコネクタを設定し、`secret`や`address`の設定が必要です。HTTPd側では、`ProxyPass`や`ProxyPassReverse`を使ってTomcatへのリクエストを転送します。静的ファイルについては、NFSを使ってHTTPdが直接提供するか、Tomcatのデフォルトサーブレットを使用します。セキュリティを考慮して、AJPは内部ネットワークに制限し、`ajpSecret`の設定が重要です。HTTPSやキャッシュ、リバースプロキシの利点も検討します。


思考時間: 57 秒


### 1. アーキテクチャは妥当か？

| レイヤ  | サーバ                                   | 主な役割                                    | 通信プロトコル             | コメント                                           |
| ---- | ------------------------------------- | --------------------------------------- | ------------------- | ---------------------------------------------- |
| フロント | **Server A (Apache HTTP Server 2.4)** | SSL終端、リバースプロキシ、アクセス制御（WAF／Rate‑limit 等） | **AJP 1.3** 内部LANのみ | HTTPd が外部公開ポートを担当。TLS 設定や ModSecurity などをここで行う |
| アプリ  | **Server B (Apache Tomcat 10)**       | Java WebApp 実行、CSV ファイル生成               | AJP, localhost HTTP | `AJP/1.3` コネクタを 8009/tcp で待受                   |

この 2 層構成は **Tomcat を DMZ 外に置きつつ静的/動的両方を単一 URL に統合**できる古典的パターンで、
(1) TLS 解除が 1 か所に集約できる
(2) WebApp のメンテナンス時に HTTPd 側でメンテナンスページを返せる
といった運用上の利点があります。

**注意点**

* **AJP は暗号化されていない**ため、Ghostcat (CVE‑2020‑1938) 以降 **`secretRequired="true"`** がデフォルトです。内部 NW に限定し、必ずシークレットを設定してください。([tomcat.apache.org][1], [Stack Overflow][2])
* Apache 2.4.42 以降であれば `ProxyPass … secret=` で秘密値を送れます。([アパッチ HTTP サーバープロジェクト][3], [アパッチ HTTP サーバープロジェクト][4])

---

### 2. 「Tomcat が吐き出した /var/data/xxx.csv を Web で公開」する 2 つの方法

| 方式                                    | 概要                                                                                          | メリット                                | デメリット                          |
| ------------------------------------- | ------------------------------------------------------------------------------------------- | ----------------------------------- | ------------------------------ |
| **(A) Tomcat に `/csv` コンテキストを切る**     | Tomcat 側に `<Context docBase="/var/data" path="/csv" />` を追加し、HTTPd から `/csv/**` を AJP 経由で転送 | シンプル。ファイル生成後すぐ配信可                   | 大容量ファイルを Tomcat が送るためヒープを食いやすい |
| **(B) 共有ストレージ/rsync で Server A へコピー** | NFS 共有、rsync、GlusterFS などで `/var/data` を HTTPd へ複製。HTTPd は `Alias /csv/ "/srv/data/"` で直接配信 | 静的ファイルは HTTPd が高速・低メモリで処理。Range も効く | 共有領域準備 or バッチ転送が必要             |

要件③で「**Web サーバ (A) に渡す必要あり**」と明言されているなら、(B) が本来の解釈です。
Tomcat 側に「生成→NFS 上に書き出し」、HTTPd 側に `Alias` で公開、が運用・性能とも安定します。

---

### 3. 主要設定例

#### 3.1 Tomcat 10 (server.xml 抜粋)

```xml
<Connector protocol="AJP/1.3"
           port="8009"
           address="0.0.0.0"
           secretRequired="true"
           secret="SuperSecret123!"
           packetSize="65536"
           redirectPort="8443" />   <!-- HTTPS へリダイレクトさせる場合 -->
```

* `secret` と HTTPd 側の `secret=` 値を合わせる
* `address` は内部サブネットの IP に限定推奨
* 大容量応答なら `packetSize` を 65536 に上げてフラグメント減少

#### 3.2 静的 CSV を Tomcat が直接配る場合 (方式 A)

`$CATALINA_BASE/conf/Catalina/localhost/csv.xml`

```xml
<Context docBase="/var/data" path="/csv" />
```

`docBase` は絶対パス指定可です。([tomcat.apache.org][5])

#### 3.3 Apache HTTPd 2.4 (Server A)

```apache
# 必要モジュール
LoadModule proxy_module        modules/mod_proxy.so
LoadModule proxy_ajp_module    modules/mod_proxy_ajp.so
LoadModule proxy_http_module   modules/mod_proxy_http.so

<VirtualHost *:443>
  ServerName  example.com
  SSLEngine   on
  # …SSL 証明書設定…

  # Tomcat アプリ (例: /app/*)
  <Proxy "ajp://serverB.internal:8009" secret="SuperSecret123!"/>
  ProxyPass        "/app/"  "ajp://serverB.internal:8009/app/"  secret=SuperSecret123!
  ProxyPassReverse "/app/"  "ajp://serverB.internal:8009/app/"

  # Tomcat が配る CSV (方式 A の場合)
  ProxyPass        "/csv/"  "ajp://serverB.internal:8009/csv/"  secret=SuperSecret123!
  ProxyPassReverse "/csv/"  "ajp://serverB.internal:8009/csv/"

  # 方式 B の場合: HTTPd 自身が配信
  #Alias  /csv/  "/srv/data/"
  #<Directory "/srv/data">
  #    Require all granted
  #</Directory>

  # タイムアウトやバッファ
  ProxyTimeout         60
  ProxyPassMatch ^/(.*\.csv)$ ajp://serverB.internal:8009/$1  secret=SuperSecret123!
  ProxyIOBufferSize   65536
</VirtualHost>
```

**ポイント**

* `ProxyPass` か `ProxyPassMatch` に `secret=` を必ず明示。
* `ProxyIOBufferSize` / `ProxyTimeout` は大きな CSV のダウンロード時間に合わせる。
* HTTP→HTTPS リダイレクトは `RewriteRule` か `Redirect permanent / https://example.com/` で別 VirtualHost を立てる。

---

### 4. セキュリティ／運用のチェックリスト

| 項目       | 推奨設定                                                               |
| -------- | ------------------------------------------------------------------ |
| AJP ポート  | ファイアウォールで **Server A のみ** に許可                                      |
| `secret` | 16 byte 以上のランダム文字列                                                 |
| ログ       | HTTPd: `mod_log_config`, `mod_security` / Tomcat: `AccessLogValve` |
| ファイル権限   | `/var/data/*.csv` は Tomcat の実行ユーザだけ書込可、HTTPd 読取のみ                  |
| 監視       | `mod_status`, `jkstatus` でバックエンドヘルス確認                              |

---

## まとめ

* **AJP 2 層構成**自体は要件に対して適切。
* 大容量 CSV をどう配るかで **方式 A (プロキシ) / B (共有ストレージ)** の選択肢がある。性能と保守性の観点では **方式 B が推奨**。
* **Tomcat 10 以降は `secretRequired` が true** なので Apache 2.4.42+ の `secret=` パラメータを忘れずに合わせる。

上記サンプルをベースに、ネットワーク ACL と TLS 証明書の整備を行えば本番適用できます。

[1]: https://tomcat.apache.org/tomcat-10.1-doc/config/ajp.html?utm_source=chatgpt.com "Apache Tomcat 10 Configuration Reference (10.1.40) - The AJP ..."
[2]: https://stackoverflow.com/questions/60501470/the-ajp-connector-is-configured-with-secretrequired-true-but-the-secret-attrib?utm_source=chatgpt.com "The AJP Connector is configured with secretRequired=\"true\" but the ..."
[3]: https://httpd.apache.org/docs/2.4/en/mod/mod_proxy_ajp.html?utm_source=chatgpt.com "mod_proxy_ajp - Apache HTTP Server Version 2.4"
[4]: https://httpd.apache.org/docs/2.4/mod/mod_proxy.html?utm_source=chatgpt.com "mod_proxy - Apache HTTP Server Version 2.4"
[5]: https://tomcat.apache.org/tomcat-5.5-doc/config/context.html?utm_source=chatgpt.com "The Context Container - Apache Tomcat Configuration Reference"
