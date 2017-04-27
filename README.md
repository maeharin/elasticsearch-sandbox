# elasticaserch sandbox

elasticsearchの練習をするためのリポジトリ
サンプルデータとして、movielens(https://grouplens.org/datasets/movielens/)のデータを使っています

## dependencies

- virtualbox
- docker
- docker-machine >= 0.6.0
- docker-compose >= 1.6.0

## how to start

```
# see: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
$ docker-machine ssh
$ sudo sysctl -w vm.max_map_count=262144
```

```
$ docker-compose up
```

- elasticsearch: http://{docker-ip}:9200
- kibana: http://{docker-ip}:5601
  - kibanaのdev toolsでクエリを簡単に試せる

## embulkでデータを投入

embulkインストール方法：https://github.com/embulk/embulk#linux--mac--bsd

```
$ embulk run embulk/movies_config.yml
$ embulk run embulk/ratings_config.yml
```

## 主なapi

- Single document APIs
    - Index API
    - Get API
    - Delete API
    - Update API
- Multi-document APIs
    - Multi Get API
    - Bulk API
    - Delete By Query API
    - Update By Query API
    - Reindex API
- Search API
    - Search
    - URI Search
    - Request Body Search
    - Search Template
    - Multi Search Template
    - Search Shards API
    - Suggesters
    - Multi Search API
    - Count API
    - Validate API
    - Explain API
    - Profile API
    - Percolator
    - Field stats API
- Aggregations
- Indices APIs
- cat APIs
- Cluster APIs

### /_search

| 種類             | json要素名 |内容 |
|------------------|---|------|
| Query            | query|query dsl（詳細は後述）|
| From / Size      | from, size|取ってくる数の制限（offset, limit)|
| Sort             | sort|並び順 |
| Source filtering |_source|ES5から、fieldsがなくなって、_sourceを使うようになったぽい。https://www.elastic.co/guide/en/elasticsearch/reference/5.x/search-request-source-filtering.html|
| Fields           |_source, stored_fields| The stored_fields parameter is about fields that are explicitly marked as stored in the mapping, which is off by default and generally not recommended. Use source filtering instead to select subsets of the original source document to be returned|
| Script Fields    |script_fields||
| Doc value Fields |docvalue_fields||
| Post filter      |post_filter||
| Highlighting     |highlight||
| Rescoring        |rescore||
| Search Type      |||
| Scroll           |/_search/scroll||
| Preference       |/_search?preference=||
| Explain          |explain||
| Version          |version||
| Index Boost      |indices_boost||
| min_score        |min_score||
| Named Queries    |_name||
| Inner hits       |inner_hits||
| Field Collapsing |collapse||
| Search After     |search_after||

例

```json
GET movielines/movies/_search
{ 
  "query": {
    "match": {
        "title": "rock star"
    }
  },
  "_source": ["title", "genres"],
  "sort": [
    {
      "title.keyword": {
        "order": "desc"
      }
    }
  ],
  "size": 2,
  "from": 0
}
```

## _count

```json
GET movielines/movies/_count
{
  "query": {"match": {"title": "star"}}
}
```

## query dslの概要

- Query clausesはLeaf query clausesとCompound query clausesにわけられる
    - Leaf query clausesはそれ自身で完結して使えるもの
        - 例：match, term, range
    - Compound query clausesは他のleaf clausesを内包するもの
        - 例：bool, constant_score
- 各Query clausesはqueryコンテキストとして使われるのか、filterコンテキストで使われるのかで挙動が変わる
    - queryコンテキストで使われる場合
        - `_score`に影響を及ぼす
        - マッチするかどうか（boolean）ではなく、どれくらいマッチするか（度合い）を知りたいときに使う（主に並べ替えのため）
    - filterコンテキストで使われる場合
        - `_score`に影響を及ぼさない
        - マッチするかどうか（boolean）だけ返す
        - 計算が早いので、query使う必要無いときは、極力filterで書く
        - 例：statusがpublished、作成日が2013-2014
        - filterコンテキストを使う主なケースは
            - boolクエリ内のfilter, must_not
            - constant_scoreクエリ内のfilter
            - filter aggregation
        - 以下はboolクエリの例、titleやcontentのキーワードマッチ度はソート順に使いたいのでmustに。statusやpublish_dateはただのフィルタリングに使って並び順には使わないので、filterに

```json
GET _search
{
  "query": { 
    "bool": { 
      "must": [ # queryコンテキスト（_scoreに影響）
        { "match": { "title":   "Search"        }}, 
        { "match": { "content": "Elasticsearch" }}  
      ],
      "filter": [ # filterコンテキスト（_scoreに影響なし）
        { "term":  { "status": "published" }}, 
        { "range": { "publish_date": { "gte": "2015-01-01" }}} 
      ]
    }
  }
}
```

## query

様々な`query`がある。ざっと分類すると

- full textクエリ: matchなど
- term levelクエリ: termなど
- compoundクエリ: boolなど
- Joiningクエリ: nestedなど
- Geoクエリ: geo_shapeなど
- Specializedクエリ: more_like_thisなど
- Spanクエリ: span_termなど

### full textクエリ

- 検索実行前に、クエリをアナライズする
- 例：タイトル、本文、emailなど

| full textクエリ     | 内容                                                                                                                                                    |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| match               | The standard query for performing full text queries, including fuzzy matching and phrase or proximity queries.                                          |
| match_phrase        | Like the match query but used for matching exact phrases or word proximity matches.                                                                     |
| match_phrase_prefix | The poor man’s search-as-you-type. Like the match_phrase query, but does a wildcard search on the final word.                                          |
| multi_match         | The multi-field version of the match query.                                                                                                             |
| common_terms        | A more specialized query which gives more preference to uncommon words.                                                                                 |
| query_string        | Lucene query string syntax, allowing you to specify AND, OR, NOT conditions and multi-field search within a single query string. For expert users only. |
| simple_query_string | A simpler, more robust version of the query_string syntax suitable for exposing directly to users.                                                      |

#### match

フリーワード検索の主力

```json
GET movielines/movies/_search
{ 
  "query": {
    "match": {
      "title": "school rock"
    }
  }
}

GET movielines/movies/_search
{ 
  "query": {
    "match": {
      "title": {
        "query": "school rock",
        "operator": "and"
      }
    }
  }
}
```

#### match_phrase

fooとbaaの間に未知の単語をslop個許容したフレーズを検索

```json
GET movielines/movies/_search
{ 
  "query": {
    "match_phrase": {
      "title": {
        "query": "school 2003",
       "slop": 2
      }
    }
  }
}
```

#### multi_match

複数のフィールドに検索をかける

```json
GET movielines/movies/_search
{ 
  "query": {
    "multi_match": {
      "query": "musical",
      "fields": ["title", "genres"]
    }
  }
}
```


### term levelクエリ

- operate on the exact terms that are stored in the inverted index
- usually used for structured data like numbers, dates, and enums Alternatively, they allow you to craft low-level queries, foregoing the analysis process

| term levelクエリ | 内容                                                                                                                                          |
|------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| term             | contain the exact term specified in the field specified.                                                                                      |
| terms            | contain any of the exact terms specified in the field specified.                                                                              |
| range            | contains values (dates, numbers, or strings) in the range specified.                                                                          |
| exists           | contains any non-null value.                                                                                                                  |
| prefix           | contains terms which begin with the exact prefix specified.                                                                                   |
| wildcard         | contains terms which match the pattern specified, where the pattern supports single character wildcards (?) and multi-character wildcards (*) |
| regexp           | contains terms which match the regular expression specified.                                                                                  |
| fuzzy            | contains terms which are fuzzily similar to the specified term. Fuzziness is measured as a Levenshtein edit distance of 1 or 2.               |
| type             | the specified type.                                                                                                                           |
| ids              | the specified type and IDs.                                                                                                                   |

#### term

アナライズされない検索。状態とかに

```json
GET movielines/movies/_search
{ 
  "query": {
    "term": {
      "title": "school of rock"
    }
  }
}
```

#### terms

#### range


### Compoundクエリ

| compoundクエリ       | 内容                                                                                                                                                                                                                                                                                            |
|----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| constant_score query | A query which wraps another query, but executes it in filter context. All matching documents are given the same constant ` _score.`                                                                                                                                                             |
| bool query           | The default query for combining multiple leaf or compound query clauses, as must, should, must_not, or filter clauses. The must and should clauses have their scores combined — the more matching clauses, the better — while the must_not and filter clauses are executed in filter context. |
| dis_max query        | A query which accepts multiple queries, and returns any documents which match any of the query clauses. While the bool query combines the scores from all matching queries, the dis_max query uses the score of the single best- matching query clause.                                         |
| function_score query | Modify the scores returned by the main query with functions to take into account factors like popularity, recency, distance, or custom algorithms implemented with scripting.                                                                                                                   |
| boosting query       | Return documents which match a positive query, but reduce the score of documents which also match a negative query.                                                                                                                                                                             |
| indices query        | Execute one query for the specified indices, and another for other indices.                                                                                                                                                                                                                     |
#### bool

filterと組み合わせる時の主力

- Elasticsearch 2.0からandクエリとorクエリは全部非推奨になり、その代わりにboolクエリの方が推奨されます。Boolクエリは複数のクエリを組み合わせる（つまりAND、OR、NOTで結合）のに使います。
- must, filter, should, must_notが使える

```json
GET movielines/movies/_search 
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "star"}}
      ],
      "filter": [
        {"range": {"movieId": {"gte": 900, "lte": 1500}}}
      ]
    }
  }
}
```

### Joiningクエリ

### Geoクエリ

### Specializedクエリ

### Spanクエリ




### スコア

#### function_score

```json
GET movielines/movies/_search
{ 
  "query": {
    "function_score": {
      "query": {
        "match": {
          "title": "rock"
        }
      }, 
      "script_score": {
        "script": {
          "lang": "painless", 
          "params": {
            "ranking": {
              "373": 1000000000,
              "6863": 2000000000
            }
          },
          "inline": "String movieId = doc['movieId'].value.toString(); params.ranking.getOrDefault(movieId, 0) + _score"
        }
      }
    }
  }
}
```
