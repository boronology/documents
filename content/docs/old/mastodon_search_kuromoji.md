# mastodonの検索にkuromojiを導入して日本語検索に対応する

## 目的
mastodonでも日本語検索に対応したい。Dockerで運用している環境である。

## 作業

### 1. elasticsearchにkuromojiプラグインを入れる
面倒なので[plusminusio/elasticsearch](https://hub.docker.com/r/plusminusio/elasticsearch)を使用することにした。[GitHub](https://github.com/mstdn-plusminus-io/elasticsearch)のソースを見ての通り、 **analysis-icu** と **analysis-kuromoji** が導入されている。もちろん同様のイメージを自分でビルドするようにしてもよい。

どちらにしても *docker-compose.yml* を編集し

```yml
  es:
    restart: always
    #image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.8.10
    image: plusminusio/elasticsearch:7.9.3
```

使用するイメージを変更しておく。

### 2. mastodonのソースコードを編集
kuromojiを使うようにソースを編集する。
*app/chewy/statuses_index.rb* の該当部分を

```rb
class StatusesIndex < Chewy::Index
  settings index: { refresh_interval: '15m' }, analysis: {
    filter: {
      english_stop: {
        type: 'stop',
        stopwords: '_english_',
      },
      english_stemmer: {
        type: 'stemmer',
        language: 'english',
      },
      english_possessive_stemmer: {
        type: 'stemmer',
        language: 'possessive_english',
      },
    },
    tokenizer: {
      kuromoji: {
        type: 'kuromoji_tokenizer',
        #mode: 'search',
      },
    },
    analyzer: {
      content: {
        tokenizer: 'kuromoji',
        type: 'custom',
        char_filter: %w(
          icu_normalizer
          html_strip
          kuromoji_iteration_mark
        ),
        filter: %w(
          english_possessive_stemmer
          lowercase
          asciifolding
          kuromoji_stemmer
          kuromoji_number
          kuromoji_baseform
          icu_normalizer
          cjk_width
          english_stop
          english_stemmer
        ),
      },
    },
  }
```
こんな感じに書き換える。実のところこれも[mstdn-plusminus-io/mastodon](https://github.com/mstdn-plusminus-io/mastodon)の引き写しである。


### 3. イメージ作成
*docker-compose.yml* を少し編集し、イメージ名を独自のもの、`build`を実行するようにしておく。もちろん`web`以外の`streaming`や`sidekiq`も同様にイメージ名を変更するが、`build`は1つを残してコメントアウトしておいてよい。

```yaml
  web:
    build: .
    image: boronology/mastodon:v3.3.0
    restart: always
```

できたらイメージをビルドする。

```shell
$ docker-compose build web
```

### 4. 動かす
```shell
$ docker-compose up -d
```

ここは普段と変わらず。

### 5. elasticsearchのインデクスを消す
ここからが本題。どうやらプラグインを入れる前後でインデクスの不整合がおきるらしく、いきなり`tootctl search deploy`するとエラーが出て止まる。

> [400] {"error":{"root_cause":[{"type":"illegal_argument_exception","reason":"The provided expression [statuses] matches an alias, specify the corresponding concrete indices instead."}],"type":"illegal_argument_exception","reason":"The provided expression [statuses] matches an alias, specify the corresponding concrete indices instead."},"status":400} (Elasticsearch::Transport::Transport::Errors::BadRequest)

そこで一度インデクスをすべて削除してしまうことにする。

```shell
# Elasticsearchのコンテナに入る
$ docker-compose exec es sh

# 存在するインデクスを表示
$ curl -XGET 'http://localhost:9200/_cat/indices?pretty=true'

# インデクスを削除
curl -X DELETE 'http://localhost:9200/statuses_xxxxxx?pretty=true'
```

これで既存のインデクスが削除できるので、あらためて再生成を行う。

```sh
docker-compose run --rm web bin/tootctl search deploy
```

このときメモリをケチっているとメモリ不足で止まる。 *docker-compose.yml*のesで`"ES_JAVA_OPTS=-Xms256m -Xmx256m"`くらいは用意しておいたほうがいいと思われる。`"ES_JAVA_OPTS=-Xms128m -Xmx128m"`ではエラーが起きた。

## まとめ
elasticsearchの設定を変えたらインデクスを再生成しましょう。
