## Get Started Elasticsearch 7.x 비디오 예제에 사용된 코드

```
#--------------------------------------------------
# 1. CRUD : REST API 를 사용해서 데이터 색인, 조회 및 삭제
#--------------------------------------------------

# 1-1 도큐먼트 색인
curl -XPUT "http://localhost:9200/my_index/_doc/1" -H 'Content-Type: application/json' -d'
{
  "message":"안녕하세요 Elasticsearch"
}'

# 1-2 도큐먼트 조회
curl -XGET "http://localhost:9200/my_index/_doc/1"

# 1-3 도큐먼트 색인 : 기존 도큐먼트에 업데이트
curl -XPUT "http://localhost:9200/my_index/_doc/1" -H 'Content-Type: application/json' -d'
{
  "message":"안녕하세요 Elastic Stack"
}'

# 1-4 도큐먼트 색인 : ID 자동 생성
curl -XPOST "http://localhost:9200/my_index/_doc" -H 'Content-Type: application/json' -d'
{
  "message":"안녕하세요 Kibana"
}'

# 1-5 도큐먼트 삭제
curl -XDELETE "http://localhost:9200/my_index/_doc/1"

# 1-6 인덱스 삭제
curl -XDELETE "http://localhost:9200/my_index"

#--------------------------------------------------
# 2. Bulk 색인
# 다량의 도큐먼트를 한꺼번에 색인 할 때는 반드시 bulk API를 사용
#--------------------------------------------------
#이렇게 해야 훨씬 빠르다고 함

POST my_index/_bulk
{"index":{"_id":1}}
{"message":"The quick brown fox"}
{"index":{"_id":2}}
{"message":"The quick brown fox jumps over the lazy dog"}
{"index":{"_id":3}}
{"message":"The quick brown fox jumps over the quick dog"}
{"index":{"_id":4}}
{"message":"Brown fox brown dog"}
{"index":{"_id":5}}
{"message":"Lazy jumping dog"}

#입력후에 확인해보면 _type이 다 _doc인걸 확인할 수 있음

#--------------------------------------------------
# 3. 풀텍스트 검색 (_search)
#--------------------------------------------------

# 3-1 인덱스의 전체 도큐먼트 검색 : match_all
#message필드에 있는 모든 document 다 검색
GET my_index/_search
{
  "query":{
    "match_all":{ }
  }
}

# 3-2 match 쿼리 : dog 검색
# message필드에 dog가 들어간 모든 document 검색
GET my_index/_search
{
  "query": {
    "match": {
      "message": "dog"
    }
  }
}

# 3-3 match 쿼리 : quick 또는 dog 검색 (or)
# message필드에 dog이 있거나 quick도 있는 경우
GET my_index/_search
{
  "query": {
    "match": {
      "message": "quick dog"
    }
  }
}

# 3-4 match 쿼리 : quick 과 dog 검색 (and)
# message에 quick과 dog이 있는 경우만 검색
# "The quick brown fox jumps over the quick dog", "The quick brown fox jumps over the lazy dog"
# 모두 찾아짐
GET my_index/_search
{
  "query": {
    "match": {
      "message": {
        "query": "quick dog",
        "operator": "and"
      }
    }
  }
}

# 3-5 match_phrase 쿼리 : "lazy dog" 구문 검색
# and operation이 아니라 "lazy dog" 이 한 구문을 검색하고 싶은 경우
GET my_index/_search
{
  "query": {
    "match_phrase": {
      "message": "lazy dog"
    }
  }
}


#--------------------------------------------------
# 4. 복합 쿼리 - bool 쿼리를 이용한 서브쿼리 조합
# - must : 쿼리가 참인 도큐먼트들을 검색
# - must_not : 쿼리가 거짓인 도큐먼트들을 검색
# - should : 검색 결과 중 이 쿼리에 해당하는 도큐먼트의 점수를 높임
# - filter : 쿼리가 참인 도큐먼트를 검색하지만 스코어를 계산하지 않음. must 보다 검색 속도가 빠르고 캐싱됨.
#--------------------------------------------------

# 4-1 "quick" 와 "lazy dog" 가 포함된 모든 문서 검색
# 여러개의 쿼리를 넣을 수 있는거고, 
# quick도 들어가고 "lazy dog"도 들어간 document를 찾고 싶은 경우
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": { "message": "quick" }
        },
        {
          "match_phrase": { "message": "lazy dog" }
        }
      ]
    }
  }
}

# 4-2 "fox" 를 포함하는 모든 도큐먼트 중 "lazy" 가 포함된 결과에 가중치 부여
# score : 가중치 (더 적합한 document를 평가하는 지표인데),
# should를 써서 "lazy"가 들어간 document에는 더 높은 점수를 부여할 수 있음
# lazy가 들어간 document가 '_score'값이 더 높은걸 확인하자.
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "message": "fox"
          }
        }
      ],
      "should": [
        {
          "match": {
            "message": "lazy"
          }
        }
      ]
    }
  }
}

# 4-3 "fox" 와 "quick" 을 포함하는 쿼리의 must & filter 스코어 비교
#아래 커맨드까지가 한 세트임
#"the quick brown fox"점수가 0.48
GET my_index/_search
{
  "query": {
    "match": {
      "message": "fox"
    }
  }
}

#"the quick brown fox"점수가 1.25로 높아짐
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "message": "fox"
          }
        },
        {
          "match": {
            "message": "quick"
          }
        }
      ]
    }
  }
}
#"the quick brown fox"점수가 0.4868
# filter를 걸으면 점수가 동일한 이유는, quick이 들어간 놈들만 걸러서 받기 때문
# 위에는 점수는 높이되, 다른 것들도 받아옴
GET my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "message": "fox"
          }
        }
      ],
      "filter": [
        {
          "match": {
            "message": "quick"
          }
        }
      ]
    }
  }
}

#숫자나 날짜와 관련해서 쿼리를 할 때

#--------------------------------------------------
# 5. range 쿼리
# - gte (Greater-than or equal to) : 이상 (같거나 큼)
#	- gt (Greater-than) : 초과 (큼)
#	- lte (Less-than or equal to) : 이하 (같거나 작음)
#	- lt (Less-than) : 미만 (작음)
#--------------------------------------------------

#timestamp, int 값들에 대해서 elastic search는 아주 훌륭하게 알아서 잘 판단해줌.

#테스트를 위해서 bulk로 데이터들 넣음
POST phones/_bulk
{"index":{"_id":1}}
{"model":"Samsung GalaxyS 5","price":475,"date":"2014-02-24"}
{"index":{"_id":2}}
{"model":"Samsung GalaxyS 6","price":795,"date":"2015-03-15"}
{"index":{"_id":3}}
{"model":"Samsung GalaxyS 7","price":859,"date":"2016-02-21"}
{"index":{"_id":4}}
{"model":"Samsung GalaxyS 8","price":959,"date":"2017-03-29"}
{"index":{"_id":5}}
{"model":"Samsung GalaxyS 9","price":1059,"date":"2018-02-25"}

#bulk로 넣는건 아래 커맨드를 여러번 실행하는 것과 동일한 결과를 가져오지만, 
#성능 측면에서 더 효율적임
PUT phones/_doc/6
{
  "message":"안녕하세요 Elasticsearch"
}

#지금 document의 필드명들이 다른데 어떻게 되는지 한번 확인해보자
GET phones/_search
{
  "query":{
    "match_all":{ }
  }
}

#자 이제 형식에 맞춰서 다시 수정
PUT phones/_doc/6
{"model":"Samsung GalaxyS 10","price":2000,"date":"2021-07-10"}


# 5-1 price 필드 값이 700 이상, 900 미만인 데이터를 검색
GET phones/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 700,
        "lt": 900
      }
    }
  }
}

# 5-2 date필드의 날짜가 2016년 1월 1일 이후인 도큐먼트들을 검색
GET phones/_search
{
  "query": {
    "range": {
      "date": {
        "gt": "2016-01-01"
      }
    }
  }
}

# 5-2 date필드의 날짜가 오늘 (2021년 7월 21일) 부터 2년 전 이후인 도큐먼트들을 검색
GET phones/_search
{
  "query": {
    "range": {
      "date": {
        "gt": "now-2y"
      }
    }
  }
}


#--------------------------------------------------
# 6. 텍스트 분석 - Analysis (_analyze API)
#--------------------------------------------------
# 얘네는 조금 lowlevel에서 엘라스틱 서치가 어떤 analyzer를 사용해서, 문서들을 색인하는지 보기 위해
# 어떤식으로 처리되는지 확인하기 위한 섹션


# 6-1 Tokenizer 을 통해 문장을 검색어 텀(term)으로 쪼갬
GET my_index/_analyze
{
  "tokenizer": "standard",
  "text": "Brown fox brown dog"
}

# 6-2 Filter(토큰필터) 를 통해 쪼개진 텀들을 가공
# 6-2-1. lowercase - 소문자로 변경
GET my_index/_analyze
{
  "tokenizer": "standard",
  "filter": [
    "lowercase"
  ],
  "text": "Brown fox brown dog"
}



# 6-2-2. unique - 중복 텀 제거
# brown이 여러개 있지만 하나만 남음
GET my_index/_analyze
{
  "tokenizer": "standard",
  "filter": [
    "lowercase",
    "unique"
  ],
  "text": "Brown brown brown fox brown dog"
}

#loser case 빼면 Brown도 들어가는것 확인할 수 있음
GET my_index/_analyze
{
  "tokenizer": "standard",
  "filter": [
    "unique"
  ],
  "text": "Brown fox brown dog"
}

# 6-3 (Tokenizer + Filter) 대신 Analyzer 사용
# tokenizer + filter(lowercase token filter)를 합친게 analyzer임. 
#필드명이  tokenizer, filter에서 analyzer로 바뀐 것 확인
GET my_index/_analyze
{
  "analyzer": "standard",
  "text": "Brown fox brown dog"
}




#--------------------------------------------------
# 7. 분석 과정 이해하기
#--------------------------------------------------

# 7-1 복합적인 문장 분석 - T:standard, F:lowercase
# 아래처럼 standard를 쓰게 되면 
# 1) prefix, suffix에 해당하는 기호는 모두 제거
# 2) 문자열 중간에 있는 기호들은 유지
# 3) whitespace (띄어쓰기 한 칸)기준으로 모두 토크나이징
GET my_index/_analyze
{
  "tokenizer": "standard",
  "filter": [
    "lowercase"
  ],
  "text": "THE quick.brown_FOx jumped! $19.95 @ 3.0"
}

# letter analyzer를 사용하면
# 알파벳이 아닌 모든 특수기호, 숫자들을 모두 분리하는 구분자로 인식
# 19.95 이런게 완전 다 사라짐
# 7-2 복합적인 문장 분석 - T:letter, F:lowercase
GET my_index/_analyze
{
  "tokenizer": "letter",
  "filter": [
    "lowercase"
  ],
  "text": "THE quick.brown_FOx jumped! $19.95 @ 3.0"
}

# 7-3 Email, URL 분석 - T:standard
# Email, Url은 못잡음
GET my_index/_analyze
{
  "tokenizer": "standard",
  "text": "elastic@example.com website: https://www.elastic.co"
}


# 7-4 Email, URL 분석 - T:uax_url_email
#Email Url 잡아줌
GET my_index/_analyze
{
  "tokenizer": "uax_url_email",
  "text": "elastic@example.com website: https://www.elastic.co"
}

# 8-5 한글 형태소 분석기 nori 설치
# $ bin/elasticsearch-plugin install analysis-nori
# nori를 설치해서 한글 분석도 가능

# 8-6 nori_tokenizer 를 이용한 한글 분석
GET _analyze
{
  "tokenizer": "standard",
  "text": ["동해물과 백두산이"]
}

#nori 설치해야 실행 가능
GET _analyze
{
  "tokenizer": "nori_tokenizer",
  "text": ["동해물과 백두산이"]
}


#--------------------------------------------------
# 8. 인덱스 생성
# - settings : analyzer, 샤드 수, 리프레시 주기 등을 설정
# - mappings : 각 필드별 데이터 명세를 정의
#--------------------------------------------------

#이제는 별도로 index를 생성해보자
# 8-1 사용자 정의 analyzer

#먼저 내 analyzer를 적어두고, 밑에서 매핑해서 사용
#인덱스는 여러가지 속성을 가지고 있음
#settings, mappings로 구분됨
#settings : 인덱스에서 사용할 사용자 정의 analyzer, refresh주기, shard 설정
#mappings : 각 필드 별로 필드가 text필드인지, 숫자 필드인지, 그리고 text 필드일때는 어떤 anaylzer를 사용할지
#중복 생성하면 에러 발생
#인덱스는 대부분 변경 ㅂ루가능
#refesh time
#replica shad갯수
#primary shad갯수변경은 안됨
#analyzer를 바꾸는것 등은 안됨
PUT my_index_2
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "tokenizer": "letter",
          "filter": [
            "lowercase",
            "stop"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "analyzer": "my_analyzer"
      }
    }
  }
}


PUT my_index_3
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_analyzer": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "stop"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "analyzer": "my_analyzer"
      }
    }
  }
}

# 8-2 사용자 정의 analyzer 필드에 데이터 색인
PUT my_index_2/_doc/1
{
  "message": "THE quick.brown_FOx jumped! $19.95 @ 3.0"
}

# 8-2-2 사용자 정의 analyzer 필드에 데이터 색인
PUT my_index_3/_doc/1
{
  "message": "THE quick.brown_FOx jumped! $19.95 @ 3.0"
}

# 8-3 데이터 검색
#letter analyzer를 쓰고 있으니 brown도 분리돼서 저장될 것이기 때문엔 찾아짐
GET my_index_2/_search
{
  "query": {
    "match": {
      "message": "brown"
    }
  }
}
#standard analyzer 쓰고 있어서 안찾아짐
GET my_index_3/_search
{
  "query": {
    "match": {
      "message": "brown"
    }
  }
}
GET my_index_3/_search
{
  "query": {
    "match": {
      "message": "quick"
    }
  }
}
GET my_index_3/_search
{
  "query": {
    "match": {
      "message": "quick.brown_FOx"
    }
  }
}

#the의 경우에는 자동으로 삭제되는 analyzer이기 때문에
GET my_index_2/_search
{
  "query": {
    "match": {
      "message": "the"
    }
  }
}

#standard analyzer는 stop필터를 썼기 때문에 사라짐
#standard analyzer만 썼으면 남았을 것임
GET my_index_3/_search
{
  "query": {
    "match": {
      "message": "the"
    }
  }
}



#--------------------------------------------------
# 9. 애그리게이션 - 집계 (Aggregation)
#집계를 하는 기능 -> 크게 두 종류가 있음
# - metrics : min, max, sum, avg 등의 계산
# - bucket : 특정 기준으로 도큐먼트들을 그룹화
#--------------------------------------------------

PUT my_stations/_bulk
{"index": {"_id": "1"}}
{"date": "2019-06-01", "line": "1호선", "station": "종각", "passangers": 2314}
{"index": {"_id": "2"}}
{"date": "2019-06-01", "line": "2호선", "station": "강남", "passangers": 5412}
{"index": {"_id": "3"}}
{"date": "2019-07-10", "line": "2호선", "station": "강남", "passangers": 6221}
{"index": {"_id": "4"}}
{"date": "2019-07-15", "line": "2호선", "station": "강남", "passangers": 6478}
{"index": {"_id": "5"}}
{"date": "2019-08-07", "line": "2호선", "station": "강남", "passangers": 5821}
{"index": {"_id": "6"}}
{"date": "2019-08-18", "line": "2호선", "station": "강남", "passangers": 5724}
{"index": {"_id": "7"}}
{"date": "2019-09-02", "line": "2호선", "station": "신촌", "passangers": 3912}
{"index": {"_id": "8"}}
{"date": "2019-09-11", "line": "3호선", "station": "양재", "passangers": 4121}
{"index": {"_id": "9"}}
{"date": "2019-09-20", "line": "3호선", "station": "홍제", "passangers": 1021}
{"index": {"_id": "10"}}
{"date": "2019-10-01", "line": "3호선", "station": "불광", "passangers": 971}

# 9-1 전체 passangers 필드값의 합계를 가져오는 metrics aggregation
#query가 아니라 aggs, aggregation라고적어주어야 함
#all_passengers -> 합계값에 대한 키를정할 수 있음
GET my_stations/_search
{
  "size": 0, 
  "aggs": {
    "all_passangers": {
      "sum": {
        "field": "passangers"
      }
    }
  }
}


# 9-2 "station": "강남" 인 도큐먼트의 passangers 필드값의 합계를 가져오는 metrics aggregation
#search에서도 aggregation 사용 가능함.
GET my_stations/_search
{
  "query": {
    "match": {
      "station": "강남"
    }
  }, 
  "size": 0, 
  "aggs": {
    "gangnam_passangers": {
      "sum": {
        "field": "passangers"
      }
    }
  }
}

# 9-3 date_histogram으로 date 필드를 1개월 간격으로 구분하는 bucket aggregation
#bucket aggregation은 보통 count만 됨
GET my_stations/_search
{
  "size": 0,
  "aggs": {
    "date_his": {
      "date_histogram": {
        "field": "date",
        "interval": "month"
      }
    }
  }
}

# 9-4 stations.keyword 필드 별로 passangers 필드의 평균값을 계산하는 bucket & metrics aggregation
# bucket aggregation안에 각 bucket별로 metric aggregation을 사용하거나 
# 또다른 bucket도 만들 수 있음

# terms aggregation
# station.keyword라는 필드
# 역 이름별로 버킷을 만들고 terms = keyword타입. 
# station.keyword = ES는 text필드를 저장할 때 keyword타입도 같이 저장됨
# keyword 타입은 검색을 위해서 anaylsis과정을 거치지 않고 통으로 저장함
# keyword필드에 저장된 값들안 terms aggregation으로 사용이 가능함.

# station필드의 값들로 나눈다는 소리
GET my_stations/_search
{
  "size": 0,
  "aggs": {
    "stations": {
      "terms": {
        "field": "station.keyword"
      }
      , "aggs": {
        "avg_psg_per_st": {
          "avg": {
            "field": "passangers"
          }
        }
      }
    }
  }
}

#--------------------------------------------------
# 10. Geo - 위치정보
# - geo_point : { "lat": 41.12, "lon": -71.34 } 같은 형식으로 입력
#--------------------------------------------------

# 10-1 geo_point 타입의 location 필드 선언
PUT my_geo
{
  "mappings": {
    "properties": {
      "location": {
        "type": "geo_point"
      }
    }
  }
}

# 10-2 예제 데이터 입력
PUT my_geo/_bulk
{"index": {"_id": "1"}}
{"station": "강남", "location": {"lon": 127.027926, "lat":37.497175 }, "line": "2호선"}
{"index": {"_id": "2"}}
{"station": "종로3가", "location": {"lon":126.991806, "lat":37.571607}, "line": "3호선"}
{"index": {"_id": "3"}}
{"station": "여의도", "location": {"lon":126.924191, "lat":37.521624}, "line": "5호선"}
{"index": {"_id": "4"}}
{"station": "서울역", "location": {"lon":126.972559, "lat":37.554648}, "line": "1호선"}

# 10-3 geo_bounding_box : 두 점을 기준으로 하는 네모 안에 있는 도큐먼트들을 가져옴
GET my_geo/_search
{
  "query": {
    "geo_bounding_box": {
      "location": {
        "bottom_right": {
          "lat": 37.4899,
          "lon": 127.0388
        },
        "top_left": {
          "lat": 37.5779,
          "lon": 126.9617
        }
      }
    }
  }
}

# 10-4 geo_distance : 한 점을 기준으로 반경 안에 있는 도큐먼트들을 가져옴
GET my_geo/_search
{
  "query": {
    "geo_distance": {
      "distance": "5km",
      "location": {
        "lat": 37.5358,
        "lon": 126.9559
      }
    }
  }
}
