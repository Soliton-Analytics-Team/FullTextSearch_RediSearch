[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/FullTextSearch_RediSearch/blob/main/FullTextSearch_RediSearch.ipynb)

# Colabで全文検索（その７：RediSearch編）

各種全文検索ツールをColabで動かしてみるシリーズです。今回は[Redis](https://redis.io/)の拡張モジュール[RediSearch](https://oss.redis.com/redisearch/)です。

処理時間の計測はストレージのキャッシュとの兼ね合いがあるので、2回測ります。2回目は全てがメモリに載った状態での性能評価になります。ただ1回目もデータを投入した直後なので、メモリに載ってしまっている可能性があります。

## 準備

まずは検索対象のテキストを日本語wikiから取得して、Google Driveに保存します。（※ Google Driveに約１GBの空き容量が必要です。以前のデータが残っている場合は取得せず再利用します。）

Google Driveのマウント

```python
from google.colab import drive
drive.mount('/content/drive')
```

    Mounted at /content/drive


jawikiの取得とjson形式に変換。90分ほど時間がかかります。他の全文検索シリーズでも同じデータを使うので、他の記事も試す場合は wiki.json.bz2 を捨てずに残しておくことをおすすめします。

```shell
%%time
%cd /content/
import os
if not os.path.exists('/content/drive/MyDrive/wiki.json.bz2'):
    !wget https://dumps.wikimedia.org/jawiki/latest/jawiki-latest-pages-articles.xml.bz2
    !pip install wikiextractor
    !python -m wikiextractor.WikiExtractor --no-templates --processes 4 --json -b 10G -o - jawiki-latest-pages-articles.xml.bz2 | bzip2 -c > /content/drive/MyDrive/wiki.json.bz2
```

    /content
    CPU times: user 3.41 ms, sys: 50 µs, total: 3.46 ms
    Wall time: 8.8 ms


json形式に変換されたデータを確認

```python
import json
import bz2

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    for n, line in enumerate(fin):
        data = json.loads(line)
        print(data['title'].strip(), data['text'].replace('\n', '')[:40], sep='\t')
        if n == 5:
            break
```

    アンパサンド	アンパサンド（&amp;, ）は、並立助詞「…と…」を意味する記号である。ラテン
    言語	言語（げんご）は、広辞苑や大辞泉には次のように解説されている。『日本大百科事典』
    日本語	 日本語（にほんご、にっぽんご）は、日本国内や、かつての日本領だった国、そして日
    地理学	地理学（ちりがく、、、伊：geografia、）は、。地域や空間、場所、自然環境
    EU (曖昧さ回避)	EU
    国の一覧	国の一覧（くにのいちらん）は、世界の独立国の一覧。対象.国際法上国家と言えるか否


## Redisのインストール

Redisを取得して、インストールします。

```shell
!sudo yes | add-apt-repository ppa:redislabs/redis
!sudo apt-get update
!sudo apt-get install redis
```

     Redis is an open source (BSD licensed), in-memory data structure store, used as a database, cache and message broker.
    .
    .
    .
    Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
    Processing triggers for systemd (237-3ubuntu10.53) ...


インストールされたRedisのバージョンを確認します。

```shell
!redis-cli --version
!redis-server --version
```

    redis-cli 7.0.0
    Redis server v=7.0.0 sha=00000000:0 malloc=jemalloc-5.2.1 bits=64 build=698b5df4772f6164

モジュールを導入可能にする設定を追加します。

```shell
!echo "enable-module-command yes" >> /etc/redis/redis.conf
```

Redisを起動します。

```shell
!service redis-server start
```

    Starting redis-server: redis-server.


## RediSearchのビルド

Redisの拡張モジュールであるRediSearchを取得してビルドします。


```shell
%cd /content
!git clone --recursive https://github.com/RediSearch/RediSearch.git
%cd /content/RediSearch
!make setup
!make build
```

    /content
    Cloning into 'RediSearch'...
    .
    .
    .
    [100%] Built target rstest


redisearch.soの場所を確認して、RediSearchモジュールをロードします。

```shell
!find /content/RediSearch -name redisearch.so
```

    /content/RediSearch/bin/linux-x64-release/search/redisearch.so

```shell
!redis-cli module load /content/RediSearch/bin/linux-x64-release/search/redisearch.so
```

    OK


## データのインポート

Pythonのredisearchモジュールをインストールします。pipのエラーが出力される場合がありますが、モジュールはインストールされるのでここでは気にせず進めます。

```shell
!pip install redisearch
```

    Collecting redisearch
      Downloading redisearch-2.1.1-py3-none-any.whl (26 kB)
    Collecting hiredis<3.0.0,>=2.0.0
      .
      .
      .
    Successfully installed hiredis-2.0.0 redis-3.5.3 redisearch-2.1.1 rejson-0.5.6 six-1.16.0

50万件のデータをインポートします。RediSearchには日本語を分割する機能がないので、一文字づつスペースで区切った形でインポートします。


```python
from redisearch import Client, TextField, NumericField, Query
client = Client('myIndex')
client.create_index([TextField('body')])
```

    'OK'


```python
%%time

import json
import bz2
from tqdm.notebook import tqdm

limit = 500000
with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    n = 0
    for line in tqdm(fin, total=limit*1.5):
        data = json.loads(line)
        title = data['title'].strip()
        body = data['text'].replace('\n', '')
        if len(title) > 0 and len(body) > 0:
            client.add_document("doc_{}".format(n), title=title, body=' '.join(body))
            n += 1
        if n == limit:
            break

```

      0%|          | 0/750000.0 [00:00<?, ?it/s]

    CPU times: user 8min 45s, sys: 40.9 s, total: 9min 26s
    Wall time: 16min 40s


## 検索

1回目の検索。検索文字列は"日 本 語"のようにスペースで区切って指定します。in_order()は"日","本","語"を順番通りに並ばせる指定、slop(0)はこれらの文字の間になにも入らない指定、paging(0, 20000)は2万件以下を一気に検索する指定です。


```python
%%time
query = Query("日 本 語").in_order().slop(0).paging(0, 20000)
res = client.search(query)
print('total:', res.total, 'time:', '{:.2f} sec'.format(res.duration / 1000))
```

    total: 17007 time: 1.25 sec
    CPU times: user 770 ms, sys: 216 ms, total: 986 ms
    Wall time: 1.41 s

検索結果数も他と合っているので、正しく検索できているようです。

2回目の検索。

```python
%%time
query = Query("日 本 語").in_order().slop(0).paging(0, 20000)
res = client.search(query)
print('total:', res.total, 'time:', '{:.2f} sec'.format(res.duration / 1000))
```

    total: 17007 time: 1.17 sec
    CPU times: user 655 ms, sys: 209 ms, total: 864 ms
    Wall time: 1.27 s


検索結果を確認します。


```python
res.docs[1]
```

    Document {'id': 'doc_226353', 'payload': None, 'body': 'エ イ ジ 、 エ ー ジ 日 本 語 . 人 名 . 日 本 語 の 男 性 名 で 用 い ら れ る 。', 'title': 'エイジ'}

検索対象をbodyフィールドのみにして検索。

```python
%%time
query = Query('@body:"日 本 語"').in_order().slop(0).paging(0, 20000)
res = client.search(query)
print('total:', res.total, 'time:', '{:.2f} sec'.format(res.duration / 1000))
```

    total: 17007 time: 1.15 sec
    CPU times: user 856 ms, sys: 152 ms, total: 1.01 s
    Wall time: 1.41 s


あまり変わりません。

次に、検索結果のコンテンツを取得せず、idだけを取得する検索。


```python
%%time
query = Query('@body:"日 本 語"').in_order().slop(0).no_content().paging(0, 20000)
res = client.search(query)
print('total:', res.total, 'time:', '{:.2f} sec'.format(res.duration / 1000))
```

    total: 17007 time: 0.23 sec
    CPU times: user 92.3 ms, sys: 46.1 ms, total: 138 ms
    Wall time: 325 ms

非常に速い。

検索結果を確認します。idのみが取れているのが確認できます。

```python
res.docs[1]
```

    Document {'id': 'doc_226353', 'payload': None}

## Redisの停止

```shell
!service redis-server stop
```

    Stopping redis-server: redis-server.

