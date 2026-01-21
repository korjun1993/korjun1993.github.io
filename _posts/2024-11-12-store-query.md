---
layout: post
title: 검색 속도 개선하기
categories: Elasticsearch
description: 실무 경험 - 검색 쿼리 개선
keywords: Elasticsearch
---

## 목표

닉네임을 가게 검색 API를 개선한다.

## 닉네임 규칙

- 최대 100 글자 (DB 테이블 확인 결과)

## 개선 배경

- 유저 검색 응답속도가 느림
- search profile 결과, 특정 키워드로 검색할 경우, 100~500ms 소요
- esrally 테스트 결과, P90 latency 400ms 초과
- 쿼리가 느린 이유는 유저 검색 쿼리의 match_prefix, 정규표현식이 포함되어 있기 때문
- match_prefix, 정규표현식은 성능이 좋지 않음

## 목표

- 가게검색 쿼리의 응답속도를 100ms 이내로 개선

## 기존 방법

### 인덱스 정보

```
"nick_name": {
  "type": "text",
  "fields": {
    "keyword": {
      "type": "keyword"
    }
  },
  "analyzer": "korean"
}
```

```
"analysis": {
  "analyzer": {
    "korean": {
      "filter": [
        "lowercase"
      ],
      "type": "custom",
      "tokenizer": "my_tokenizer"
    }
  },
  "tokenizer": {
    "my_tokenizer": {
      "token_chars": [
        "letter",
        "digit"
      ],
      "type": "ngram",
      "min_gram": "2",
      "max_gram": "2"
    }
  }
}
```

### 검색 쿼리

```
GET user/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "match_phrase_prefix": {
            "nick_name": {
              "query": "초짜싱글맘",
              "boost": 1.5
            }
          }
        },
        {
          "match": {
            "nick_name": {
              "query": "초짜싱글맘",
              "operator": "AND"
            }
          }
        },
        {
          "regexp": {
            "nick_name.keyword": {
              "value": ".*초짜싱글맘.*"
            }
          }
        }
      ]
    }
  }
}
```

## 개선 방법

### 색인 정보

- 닉네임을 토큰으로 분리할 때, ngram(min:1, max:100)을 활용한다.

```
"analysis": {
  "analyzer": {
    "korean": {
      "filter": [
        "lowercase"
      ],
      "type": "custom",
      "tokenizer": "my_tokenizer"
    }
  },
  "tokenizer": {
    "my_tokenizer": {
      "token_chars": [
        "letter",
        "digit"
      ],
      "type": "ngram",
      "min_gram": "1",
      "max_gram": "100" // 닉네임 최대 글자수: 100
    }
  }
}
```

ngram 토크나이저에 의해 닉네임 "초짜싱글맘"은 아래와 같은 토큰으로 분리된다.

|     |     |     |
|-----|-----|-----|
| 초   |     |     |
| 짜   |     |     |
| 싱   |     |     |
| 글   |     |     |
| 맘   |     |     |
| 초   | 짜   |     |
| 짜   | 싱   |     |
| 싱   | 글   |     |
| 글   | 맘   |     |
| 초   | 짜   | 싱   |
| 짜   | 싱   | 글   |
| 싱   | 글   | 맘   |
| ... | ... | ... |

### 검색 쿼리

```
GET user/_search
{
  "query": {
    "match": {
      "nick_name": {
        "analyzer": "standard",
        "query": "초짜싱글맘",
        "operator": "and"
      }
    }
  }
}
```

하기 단계에 의해 닉네임 “초짜 싱글맘” 문서가 검색 결과로 반환된다.

1. 검색 분석기 standard analyzer에 의해 검색어 “초짜 싱글맘”은 tokens=[“초짜”, “싱글맘”]으로 분리된다.
2. 역색인표에서 토큰 [“초짜”, “싱글맘”]이 모두 포함된 문서를 찾는다.

## N-gram 적용에 의한 샤드 용량 변화

N-gram 적용 전/후 샤드의 용량은 다음과 같다.

- 적용 전
    - Primaries: 1.6GB
    - Total: 6.2GB
- 적용 후
    - Primaries: 2.3GB
    - Total: 9GB

## 개선된 검색 쿼리의 응답속도

- “중고” 검색: 321ms → 226ms
- “나라” 검색: 317ms → 224ms
- “12” 검색: 125ms → 30ms
- “신발” 검색: 88ms → 4.7ms

<img src="/images/posts/2024-11-12-store-query-1.png" width="691" alt="exp2_node_cpu"/>