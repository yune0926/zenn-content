---
title: "Kaggle宣言したのにパイプライン練習を始めた話——AIと一緒に週末で環境構築してみた"
emoji: "🔧"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["aws", "snowflake", "python", "初心者向け", "データエンジニアリング"]
published: true
---

## 1. Kaggleじゃなくなった理由

前回の記事で「Kaggleやっていきます！」と宣言しておきながら、
今回はデータパイプラインの練習をした話です。

直近の業務では AWS や Snowflake を使う場面が増えそうで、
「初めて触ります」状態で年下の子たちに、手取り足取り教えてもらうのも申し訳ないと思いました。

長期的には、Kaggle はデータ分析の力をつけていくこともやっていきたいですが、
直近の業務で **「S3って何ですか？」「Snowflakeどうやって使うんですか？」**
という状態で入るのは、さすがにまずい。

ということで、今回は業務ですぐ使えそうな基礎技術を先に練習することにしました。
Kaggle はちゃんと続けてます。ゆっくりですが。

---

## 2. 練習した内容

土曜日の合計で5〜6時間くらい使って、以下を動かしました。

| STEP | 内容 |
|------|------|
| 1 | AWS S3・Snowflake・PostgreSQL（NAS）への接続確認 |
| 2 | S3基本操作（ファイルのアップロード・ダウンロード・一覧取得） |
| 3 | CSV → S3 → Pandas → PostgreSQL というパイプライン |
| 4 | S3 の CSV を Snowflake に直接ロードする（COPY INTO） |

正直、「これくらい一瞬でしょ」と思っていましたが、
環境構築だけで土曜のほとんどが終わりました。（笑）

---

## 3. 環境構成

### 全体のイメージ

```
[ローカル PC（Mac mini）]
     |
     | DevContainer（Docker）
     |   Python 3.11
     |   boto3 / snowflake-connector / psycopg2
     |   Claude Code
     |
     ├── AWS S3（クラウド）
     ├── Snowflake（クラウド）
     └── PostgreSQL（NAS）
```

NAS に PostgreSQL を入れているのは、以前から趣味で使っていたからです。
「データソースとして活用できるな」と思って今回も使いました。

### DevContainer を使った理由

**「環境をコンテナにまとめると、PC が変わっても同じ環境で再現できる」**

というのが主な理由です。

DevContainer とは、VSCode の拡張機能で Docker コンテナを開発環境として使う仕組みです。
難しく聞こえますが、VSCode で「コンテナに入って作業する」という感覚です。

Python のバージョン管理で毎回消耗していたのが、
DevContainer に移行してからは「コンテナ立ち上げれば動く」になりました。

### AWS・Snowflake を使った理由

業務で使う可能性が高いから、というのがシンプルな答えです。

どちらも個人アカウントで無料枠・低コストで使えます。
AWS は Billing Alert と Budget 上限を設定して、**月 $0〜$5 の範囲に収めています。**
Snowflake も触る月だけ数百円で済む設定にしています。

---

## 4. Claude Code にどう頼ったか

今回の練習で Claude Code（AI のコーディングアシスタント）をかなり使いました。

### 実際の使い方

- **コードのたたき台を作ってもらう**
  「boto3 で S3 にファイルをアップロードするコードを書いて。.env から認証情報を読む形で」
  という指示だけで、動くコードが出てきます。

- **エラーをそのまま貼る**
  エラーメッセージをコピーして貼るだけで、原因と修正案を出してくれます。

- **「なぜそうするのか」を聞く**
  コードが動いても、意味がわからなければ「この `s3_client.upload_file` って何をやってるの？」
  と聞けば、かみ砕いて説明してもらえます。

### やったこと・やらなかったこと

**やったこと：** コードを自分の手で打ち直して、一行ずつ動かして確認する。
エラーが出たら、まず自分で読む。わからなければ Claude に聞く。

**やらなかったこと：** コードをコピーして貼り付けるだけで終わる、という進め方。

これは Kaggle の反省からで、「丸投げしたコードは自分の力にならない」というのを
前回身をもって実感したので。

---

## 5. 実際に動かしたパイプライン

### CSV → S3 → Pandas → PostgreSQL

```python
import boto3
import pandas as pd
import psycopg2
from dotenv import load_dotenv
import os

load_dotenv()

# S3 にアップロード
s3 = boto3.client('s3', ...)
s3.upload_file('data/sample.csv', BUCKET_NAME, 'sample.csv')

# S3 から Pandas で読み込む
obj = s3.get_object(Bucket=BUCKET_NAME, Key='sample.csv')
df = pd.read_csv(obj['Body'])

# PostgreSQL に INSERT
conn = psycopg2.connect(...)
df.to_sql('practice_table', conn, ...)
```

（省略しています。ちゃんと動くコードは .env 管理で認証情報を外出しにしています）

### S3 → Snowflake（COPY INTO）

これが今回一番「おお」となった部分です。

S3 上の CSV を Snowflake のテーブルに直接ロードできます。

```sql
COPY INTO practice_table
FROM @my_s3_stage/sample.csv
FILE_FORMAT = (TYPE = CSV FIELD_OPTIONALLY_ENCLOSED_BY = '"' SKIP_HEADER = 1);
```

S3 と Snowflake の間で直接データが流れる——
「これが業務でやってるやつか」と実感できました。

---

## 6. やってみてわかったこと

### 詰まるのはコードじゃなくて設定

IAM の権限設定・Snowflake のステージ設定・.env の書き方……
コード自体は Claude Code がサポートしてくれるので書けますが、
AWS と Snowflake の「設定」の部分で思った以上に時間を使いました。

でも、この詰まりを体験できたのが収穫でした。
業務で初めて触ったときより、圧倒的に早く動けると思います。

### 「わかる」と「動かせる」は別

接続確認ができただけで「できた」と思っていたら、
パイプラインを組もうとしたとき「あれどうやるんだっけ」となりました。
手を動かしながら実装して、ようやく「動かせる」になっていく感覚を改めて感じました。

---

## 7. 次にやりたいこと

**MCP サーバの構築を試してみたい**と思っています。

MCP（Model Context Protocol）とは、Claude Code のような AI ツールが
外部サービス（GitHub・AWS など）のドキュメントや情報を参照できるようにする仕組みです。

「Claude Code が AWS の公式ドキュメントを直接見ながらコードを書いてくれる」
という状態が作れると、さらに学習効率が上がりそうです。

---

## 最後に

Kaggle の宣言からパイプラインの練習に寄り道しましたが、
「業務で初めてです」をなくすために動いたのは、今にして思えば正解だったと思っています。

手を動かして、詰まって、調べて、動いた。
その繰り返しが、確実に体に積み上がっているのを感じます。

次回は Kaggle に戻るか、MCP サーバを試すか——
気分で決めます。（笑）

読んでくださった方、ありがとうございました。
