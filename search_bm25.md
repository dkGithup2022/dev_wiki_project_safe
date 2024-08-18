
# 소개 

bm25 기반 검색에 적용된 개념과 파이프라인 설계에 대해 다룹니다 .

analyzer spec, sorting script 예시와 설명을 포함합니다.

## 목차 

1. [Analyzer 스펙](#analyzer-스펙)
2. [Sorting score](#소팅-스코어-디자인)
3. [pipeline rule](#파이프라인)

#  1. Analyzer 스펙

```json
{
  "settings": {
    "index": {
      "analysis": {
        "tokenizer": {
          "nori_configured_tokenizer": {
            "type": "nori_tokenizer",
            "decompound_mode": "mixed",
            "discard_punctuation": "false",
            "user_dictionary": "synonym/nori_dictionary.txt"
          },
          "standard": {
            "type": "standard"
          }
        },
        "filter": {
          "nori_stop_filter": {
            "type": "nori_part_of_speech",
            "stoptags": [
              "E",
              "IC",
              "J",
              "MAG",
              "MAJ",
              "MM",
              "SP",
              "SSC",
              "SSO",
              "SC",
              "SE",
              "SF",
              "XPN",
              "XSA",
              "XSN",
              "XSV",
              "VCP",
              "UNA",
              "NA",
              "VSV",
              "E",
              "NNB",
              "VX",
              "MAG"
            ]
          },
          "english_stemmer": {
            "type": "stemmer",
            "language": "english"
          },
          "lowercase": {
            "type": "lowercase"
          },
          "synonym_filter": {
            "type": "synonym",
            "synonyms_path": "synonym/synonym.txt"
          }
        },
        "analyzer": {
          "test_analyzer": {
            "type": "custom",
            "tokenizer": "nori_configured_tokenizer",
            "filter": [
              "nori_stop_filter",
              "trim",
              "english_stemmer",
              "synonym_filter"
            ]
          }
        }
      }
    }
  }
}
```


**synonym dict**
- openai 에서 기술관련 키워드 300개를 뽑고 그에 해당하는 동의어를 2~3개를 포함시켜 만든 동의어 사전입니다.
- 이후 유저 검색어를 수집해서 추가하는 관리가 필요합니다.

예시
```java
ELASTICSEARCH 엘라스틱서치
DYNAMO 다이나모 다이다모디비
...
```


**nori analyzer**
- 한국어 형태소 분석기로 nori analyzer 를 사용합니다
- 의미 단위의 명사, 동사 정도만 남기고 다른 품사의 단어는 제거합니다.



## 소팅 스코어 디자인

BM25 스코어와 인기지표(조회수,좋아요,댓글수), 오래된 정도 를 검색 옵션에 반영한 painless script 입니다.

간략한 식은 아래와 같습니다.
`sorting score = decay_by_date( 정규화(bm25 score) + 정규화(인기지표 누계) )`


복잡한 식은 아래와 같습니다.

**sort scoring painless script**
```java
double max_views = %f;  
double max_comments = %f;  
double max_likes = %f;  
double max_bm25 = %f;  
double bm25_weight = %f;  
  
double views = doc['popularity.views'].size() > 0 ? doc['popularity.views'].value : 0;  
double comments = doc['popularity.comments'].size() > 0 ? doc['popularity.comments'].value : 0;  
double likes = doc['popularity.likes'].size() > 0 ? doc['popularity.likes'].value : 0;  
double bm25_score = _score;  
  
// 최대값으로 보정  
views = Math.min(views, max_views);  
comments = Math.min(comments, max_comments);  
likes = Math.min(likes, max_likes);  
bm25_score = Math.min(bm25_score, max_bm25);  
  
// 정규화된 인기 지표 계산 (min-max, min = 0)double normalized_views = views / max_views;  
double normalized_comments = comments / max_comments;  
double normalized_likes = likes / max_likes;  
  
double normalized_popularity_score = (normalized_views + normalized_comments + normalized_likes ) ;  
double log_normalized_popularity_score = Math.log1p(normalized_popularity_score);  // 로그 처리  
  
double normalized_bm25_score = bm25_score / max_bm25;  
double log_normalized_bm25_score = Math.log1p(normalized_bm25_score);  
  
double final_score = log_normalized_popularity_score + log_normalized_bm25_score * bm25_weight;  
return final_score;
```

##  이후 남은 것들

1. 카테고리도 옵션으로 두기 .
2. min-max 노말라이즈 말고 다른 정규화 규칙으로 교체



# 파이프라인

![service_backend.png](images%2Fservice_backend.png)

**파이프라인의 목적**

사실 파이프라인 자체는 굉장히 간단하다.

RDMS 에 커밋이 완료된 데이터 중, 검색에 노출시킬 데이터를 짧은 주기로 읽어와 ES Document type 으로 인덱싱하는 것이 전부다.

다음 장에서 설명할 Vector 검색을 위한 파이프라인에서는 pipelining 에서 외부 api 를 호출하던가 ingest pipeline 의 벡터화 모델을 사용한다.
해당 내용에 대해 보고 싶다면 [링크](search_vector.md) 를 참고해주세요 .

**수집대상**
- Tech article
- Translation article
- Question article
- Answer article

**데이터 수집 주기**
- 30초


**데이터 장애관리**
- 쿼리, 인덱싱 각각 3회까지 시도
- 3회 재처리 실패시 로깅


**데이터 전처리**
- 각 아티클을 Document data type 으로 변경
- Question 글의 경우 포함하는 Answer 내용을 검색 컨텐츠에 이어붙임
- 각 아티클의 category 목록을 비트배열로 저장 .