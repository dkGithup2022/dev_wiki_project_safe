
# 소개


검색 디자인에 따른 A/B 테스트를 진행하고 있습니다. 운영 중 click rate가 더 높은 방향으로 지속적으로 조정해나갈 계획입니다.

검색 옵션은 크게 두 가지로 나뉩니다.
    1.  BM25 기반 검색
    2.  벡터 검색 기반입니다

. BM25 검색에서는 인기 지표를 반영하는 sorting script와 analyzer 튜닝이 적용되어 있습니다.
벡터 검색에서는 LLM을 활용하여 검색 컨텍스트를 e5 model 의 e5 모델 안으로 줄여서 넣고 있습니다

정확한 코드와 config 는 아래의 문서를 각각 참고해주세요.

## Search : BM25

BM25 기반 검색의 디자인에 필요한 내용에 대해 다룹니다. 

tokenizing 을 위한 analyzer spec 과 sorting script 의 기반 내용이 포함되어 있습니다.

바로가기 : [BM25디자인](search_bm25.md)

## Search : vector  

vector search 를 위한 모델과 piplining 에 대해 ㄷ룹니다.

e5 모델의 max_seq 제약을 통과하기 위해 openai api 를 사용한 내용이 있습니다. 

바로가기 :[vector search](search_vector.md)

## AB테스트와 웹로깅

