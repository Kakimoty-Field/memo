<!-- code_chunk_output -->

- [初めに](#初めに)
- [実行環境準備](#実行環境準備)
  - [tsung セットアップ](#tsung-セットアップ)
- [負荷テストしてみる](#負荷テストしてみる)
  - [非分散負荷 (Client / Server)](#非分散負荷-client-server)
  - [負荷の設定 (load)](#負荷の設定-load)
    - [第１フェーズ](#第1フェーズ)
      - [arrivalphase](#arrivalphase)
      - [users](#users)
    - [第２フェーズ](#第2フェーズ)
      - [arrivalphase](#arrivalphase-1)
      - [users](#users-1)
  - [HTTP アクセス](#http-アクセス)
    - [オプション(ユーザエージェント)](#オプションユーザエージェント)
    - [HTTP セッション](#http-セッション)
  - [postgreSQL アクセス](#postgresql-アクセス)
- [tsung でのレポート](#tsung-でのレポート)

<!-- /code_chunk_output -->


# 初めに

このドキュメントは tsung を使って、自身が作った環境の負荷テストをするためのメモです

**このドキュメントは、筆者が試したテストを随時追記していきたいと思っています。**

# 実行環境準備

- [tsung](http://tsung.erlang-projects.org/)

- [Amazon EC2](https://aws.amazon.com/jp/ec2/) (Amazon Linux 2)

## tsung セットアップ
tsung 用に新しく EC2 インスタンスを作成した場合、tsung は **epel** に含まれているということなので Amazon Liunx で **epel** を使えるようにします。

```
sudo amazon-linux-extras install epel
```

必要であれば、yum のパッケージをアップデートしておきます。

```
sudo yum update
```

**tsung** をインストールします

```
sudo yum install tsung
```

# 負荷テストしてみる

tsung の詳細は [公式ドキュメント](http://tsung.erlang-projects.org/user_manual/) を参照します。

**負荷をかけるコマンド**

```
tsung -l [ログフォルダ] -f [設定ファイル] start 
```
ログフォルダは、存在しない場合は自動で作成されます。

## 非分散負荷 (Client / Server)
基本的な設定は以下の通りです。

[公式ドキュメント 6.2 Clients and server](http://tsung.erlang-projects.org/user_manual/conf-client-server.html)

```xml
<clients>
  <client host="localhost" use_controller_vm="true" />
</clients>

<servers>
  <server host="[負荷を掛けたいサーバ] port="ポート番号" type="tcp"></server>
</servers>
```
type には `tcp`, `ssl`, `udp` が指定できます。例えば負荷を掛けたいサーバが https で待ち受けている場合は、`ssl` を指定します。

## 負荷の設定 (load)
基本的な設定は以下の通りです。

[公式ドキュメント 6.4 Defining the load progression](http://tsung.erlang-projects.org/user_manual/conf-load.html)

```xml
  <load>
   <!-- 第１フェーズ -->
   <arrivalphase phase="1" duration="5" unit="minute">
     <users interarrival="2"  unit="second"></users>
   </arrivalphase>

   <!-- 第２フェーズ -->
   <arrivalphase phase="2" duration="1" unit="minute">
     <users maxnumber="100000" arrivalrate="1000"  unit="second"></users>
   </arrivalphase>
  </load>
```
負荷テストはフェーズとして定義できます。 <br>
フェーズは `phase`属性で識別します。

### 第１フェーズ
最初のフェーズでは、`5分間の間`、`２秒に一回` 新しい接続を追加します。

#### arrivalphase

- duration : 5
- unit : minute

#### users

- interarrival : 2
- unit : second

### 第２フェーズ
２つ目ののフェーズでは、`1分間の間`、`1秒に1,000接続`、`最大100,000接続` します。

#### arrivalphase

- duration : 1
- unit : minute

#### users
- arrivalrate : 1000
- unit : second
- maxnumber : 100000


## HTTP アクセス
### オプション(ユーザエージェント)
HTTP の接続テストでは仮想接続元のユーザエージェントを設定できます

[※公式ドキュメント 6.5.13 HTTP options](http://tsung.erlang-projects.org/user_manual/conf-options.html#http-options)

```xml
  <options>
   <option type="ts_http" name="user_agent">
    <user_agent probability="80">Mozilla/5.0 (X11; U; Linux i686; en-US; rv:1.7.8) Gecko/20050513 Galeon/1.3.21</user_agent>
    <user_agent probability="20">Mozilla/5.0 (Windows; U; Windows NT 5.2; fr-FR; rv:1.7.8) Gecko/20050511 Firefox/1.0.4</user_agent>
   </option>
  </options>
```
**probability** で、使用するユーザエージェントの割合を設定することができます。合計で100%になるように設定します。

### HTTP セッション
HTTP 接続テストの実際のパターンを設定します。

[※公式ドキュメント 6.6.2 HTTP](http://tsung.erlang-projects.org/user_manual/conf-sessions.html#http)

```xml
 <sessions>
  <session name="http-example" probability="100" type="ts_http">
    <request> <http url="/" method="GET" version="1.1"></http> </request>
    <request> <http url="/images/img1.gif" method="GET" version="1.1" if_modified_since="Fri, 14 Nov 2003 02:43:31 GMT"></http> </request>
    <thinktime value="20" random="true"></thinktime>
    <request> <http url="/index.en.html" method="GET" version="1.1" ></http> </request>
  </session>
 </sessions>
```
上記設定の接続処理は以下の流れになります。

- まず、`/` にアクセスしてホームページを取得します
- `/images/img1.gif` を取得します
- 平均 20 秒、待ちます
- `/index.en.html` を取得します

## postgreSQL アクセス

postgreSQL 接続テストの実際のパターンを設定します。

[※公式ドキュメント 6.6.4 PostgreSQL](http://tsung.erlang-projects.org/user_manual/conf-sessions.html#postgresql)

```xml
 <sessions>
  <session name="pgsql-example" probability="100" type="ts_pgsql">
   <!-- 接続設定 -->
   <transaction name="connection">
     <request>
      <pgsql type="connect" database="[データベース名]]" username="[ユーザ名]" />
     </request>
   </transaction>

   <!-- 認証 -->
   <request><pgsql type="authenticate" password="[パスワード]]" /> </request>
   <thinktime value="10" />

   <!-- クエリ実行 -->
   <request><pgsql type="sql"> SELECT count(*) from test01; </pgsql></request>
   <thinktime value="20" />

   <!-- 切断 -->
   <request><pgsql type="close">  </pgsql></request>
  </session>
 </sessions>
```
指定したデータベースに接続し、クエリを実行して、セッションを切断します。

AWS の コンソール側で、DB接続(カウント)を確認した例です。
![](https://raw.githubusercontent.com/Kakimoty-Field/memo/main/tsung/img/300.png)

# tsung でのレポート
※ 後日