

# 소개

이 글에서는 벡터검색을 위한 데이터 파이프라인에 대해 설명합니다.

vertor embedding 과정에서 [max_seq 문제와 해결방법](#벡터-검색에서의-제약사항)에 대해 다룹니다.

# 아키텍쳐
![vector_pipeilne.png](images%2Fvector_pipeilne.png)


**작업 요약**

이번 서비스에선 줄글로 된 아티클을 키워드의 배열로 추출하는 과정을 거친다.
해당 파이프 라이닝을 간략하게 요약하면 아래와 같다.

```
1. 배치 업데이트를 위해 문서의 원본 정보를 읽어옵니다.
2. OpenAI를 통해 텍스트 필드를 키워드 위주로 압축하고 동의어를 추가합니다.
3. Elasticsearch의 Ingest 노드에서 수신된 텍스트를 내장된 e5 모델로 벡터화합니다.
4. 벡터화된 데이터를 데이터 노드에 저장합니다.
```

**실행 주기**
30초


**문서 처리 TPM**

총 처리 문서 :  xx 개

각 문서 종류 :  xx 개

(openai 의 분당 처리 토큰수를 고려 )

---



### 벡터 검색에서의 제약사항

Sentence transformer 모델은 입력받을 수 있는 최대 토큰 갯수가 정해져 있습니다.

Sentence transformer 모델에서는 입력받은 각 토큰을 선형적으로 처리하지 않습니다. 입력받은 토큰 간의 관계를 기반으로 각 토큰의 가중치를 계산하게 되는데, 이 과정에서 모델별로 정해놓은 N*N 배열에 토큰을 적제하는 과정이 필요합니다.

많은 경우(거의 모든 경우), 해당 N*N 배열은 가변이 아닙니다. 보통 512 ~ 1024 의 값을 가지고 있으며 BigBird, LongFormer 등의 제한된 모델에서만 4000 토큰 이상의 길이를 가진 텍스트를 처리할 수 있습니다.

따라서, 길이가 긴 글을 받아서 벡터화 하려면 전처리가 필요합니다.

이 서비스에서는 open ai 를 통해서 문서를 키워드 단위로 추출합니다.


---


### openai api 호출을 통한 전체 길이 줄이기.

이번 프로젝트에서 사용을 해봤는데, 굉장히 만족스러운 부분입니다. vector 검색과 llm 키워드 추출의 조합은 적은 노력으로 어색하지 않은 퀄리티의 검색서비스를 양산할 수 있게 만든다.

openai api 를 사용하는 목적은 다음과 같습니다.

1. text의 길이 축소
2. 대표적인 동의어가 있다면 추가하기.

아래 프롬프트 예시를 통해 text_field 의 문장에서 대표 단어 키워드를 뽑아낸 후, 기술과 관련한 고유명사인 경우 동의어가 있다면 추가하는 예시입니다.

```java
  private static final String TEMPLATE_EXAMPLE_FOR_ES_PIPELINE =
            """
You will receive an input string called Content which contains various sentences. Your task is to extract keywords from each sentence. Keywords are the main words or phrases that represent the essential information in the sentence. Ensure the following:

0. Keywords in any sentence refer to technical proper nouns, or words/verbs that represent the main idea of the sentence.
1. Extract keywords from each sentence. Do not exceed 2 keywords per sentence.
2. If a keyword is a technical proper noun, extract it in both English and Korean.
3. If a keyword is a technical proper noun and has clear synonyms or abbreviations, include those in the keyword list as well. For example, if "k8s" is a keyword, also add "kubernetes" to the keyword list.
4. If the content includes section titles or headings, extract all the words in the title or heading as keywords, and include synonyms for these as well.
5. If the content includes code blocks, example codes, or example commands, extract the main class names, functions, or variables as keywords.
6. Focus on extracting meaningful and significant words that contribute to the main ideas or concepts. Avoid common or less informative words (e.g., "the", "is", "and").
7. The output should be an array of the extracted keywords.

Here is an example for clarity:

[INPUT EXAMPLE]
``
시작하기 전에, 쿠버네티스 클러스터가 시작되고 실행 중인지 확인한다. 위의 디플로이먼트를 생성하려면 다음 단계를 따른다.

다음 명령어를 실행해서 디플로이먼트를 생성한다.

kubectl apply -f https://k8s.io/examples/controllers/nginx-deployment.yaml
``

[OUTPUT EXAMPLE]

[
  "쿠버네티스", "k8s", "kubernetes", 클러스터", "디플로이먼트", "Deployment",
  "생성", "명령어","kubectl", "apply"
]

Make sure to follow the guidelines strictly and ensure that all relevant keywords, including technical synonyms where applicable, are captured accurately.
                    
---

[CONTENT]         
&{CONTENT&}

                    """;
```

- 실행 예시 | 추가로 인덱싱될 컨텍스트  

```json
input_text : "앞 장에서 Elasticsearch의 다양한 검색 방법에 대해 살펴보았습니다. 풀 텍스트 검색을 하기 위해서는 데이터를 검색에 맞게 가공하는 작업을 필요로 하는데 Elasticsearch는 데이터를 저장하는 과정에서 이 작업을 처리합니다. 이번 장에서는 Elasticsearch가 검색을 위해 텍스트 데이터를 어떻게 처리하고 데이터를 색인 할 때 Elasticsearch에서 어떤 과정이 이루어지는지에 대해 살펴보겠습니다."
```

```json
result :["Elasticsearch", "검색", "풀 텍스트", "데이터", "가공", "작업", "저장", "색인", "텍스트 데이터", "처리", "과정"]
```