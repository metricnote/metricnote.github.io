---
layout: post
title: "Spring AI 실습: ChatClient부터 Structured Output, pgvector와 RAG까지"
date: 2026-09-10 11:00:00 +0900
category: [backend-cloud]
tags: [Spring-AI, ChatClient, ChatOptions, PromptTemplate, SystemMessage, Structured-Output, Advisor, PostgreSQL, pgvector, Embedding, Ollama, BAAI, bge-m3, RAG, ETL, RewriteQueryTransformer, CompressionQueryTransformer, Java, Docker, learning-note]
---

하루 동안 Spring AI 예제 프로젝트를 실행하며 ChatClient의 기본 호출부터 동적 프롬프트, 구조화 출력, Advisor, 임베딩과 RAG까지 순서대로 실습했다. OpenAI 모델의 응답을 Java 객체로 변환하고, Docker의 PostgreSQL·pgvector에 문서를 저장했으며, 로컬 BAAI/bge-m3 임베딩 모델과 PDF RAG-ETL도 연결했다. 마지막에는 헌법 전문을 기반으로 RewriteQueryTransformer와 CompressionQueryTransformer를 적용했다.

## 1. 실습 프로젝트 준비

실습 코드는 GitHub 저장소에서 내려받았다.

```bash
git clone https://github.com/himang10/spring-ai.git
```

저장소는 직접 작성하는 `01.training-code`, 정답 참고용 `02.answer-code`, RAG용 `pgvector`, 로컬 임베딩용 `embedding-model` 등으로 구성돼 있었다.

![Spring AI 실습 코드 구성]({{ '/assets/images/spring-ai-practice-2026-09-09/01-source-structure.png' | relative_url }})

Java 21과 Maven을 사용했고 OpenAI API 키는 소스나 설정 파일에 넣지 않고 터미널 환경변수로 전달했다.

```bash
read -s "OPENAI_API_KEY?API 키를 입력하세요: "; export OPENAI_API_KEY; echo
mvn spring-boot:run
```

## 2. ChatClient 동기·스트리밍 호출

동기 호출은 완성된 응답을 문자열로 한 번에 반환한다.

```java
@GetMapping("/ai")
public String chat(@RequestParam String userInput) {
    return this.chatClient.prompt()
            .user(userInput)
            .call()
            .content();
}
```

스트리밍 호출은 생성되는 내용을 `Flux<String>`으로 순서대로 반환한다.

```java
@GetMapping(value = "/ai/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String userInput) {
    return this.chatClient.prompt()
            .user(userInput)
            .stream()
            .content();
}
```

기존 메서드를 채우지 않고 같은 이름의 메서드를 새로 작성해 `method is already defined` 오류가 발생했고, 스트리밍 메서드에서 `call()`을 사용해 `String cannot be converted to Flux<String>` 오류도 경험했다. 기존 메서드 본문만 구현하고 `stream()`을 사용해 해결했다.

## 3. ChatOptions로 모델 응답 설정

![ChatOptions 추가 실습 자료]({{ '/assets/images/spring-ai-practice-2026-09-09/02-chat-options-slide.png' | relative_url }})

```java
ChatOptions.Builder<?> chatOptions = ChatOptions.builder()
        .model("gpt-4o-mini")
        .temperature(0.7)
        .topP(0.9)
        .maxTokens(500);

return this.chatClient.prompt()
        .options(chatOptions)
        .user(userInput)
        .call()
        .content();
```

`temperature`는 사실성을 보장하는 옵션이 아니라 표현의 변동성을 조절하고, `topP`는 다음 토큰 후보의 확률 범위를 제한한다.

![ChatOptions를 적용한 ChatClient 응답]({{ '/assets/images/spring-ai-practice-2026-09-09/03-chat-result.png' | relative_url }})

## 4. PromptTemplate와 System Message

AI의 역할과 말투를 동적으로 구성하기 위해 `{aiName}`과 `{terms}` 변수가 포함된 `PromptTemplate`을 만들었다.

![동적 PromptTemplate 실습 자료]({{ '/assets/images/spring-ai-practice-2026-09-09/04-prompt-template-slide.png' | relative_url }})

```java
private static final String PROMPT_TEMPLATE = """
        너는 SKALA Spring AI 교육 과정의 친절하고 열정적인 보조교사 {aiName}입니다.
        - 수강생의 질문에 따뜻하고 격려하는 어조로 답변해 주세요.
        - {terms}는 초보자도 이해하기 쉽게 비유를 들어서 설명해 주세요.
        - 답변 끝에는 항상 실습을 응원하는 따뜻한 한마디를 덧붙여 주세요.
        """;

private final PromptTemplate systemPrompt = new PromptTemplate(PROMPT_TEMPLATE);
```

변수를 렌더링해 `SystemMessage`로 전달하자 단순한 자기소개 요청에도 지정한 보조교사 이름, 말투와 응원 문구가 반영됐다.

![PromptTemplate와 System Message 적용 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/05-prompt-template-result.png' | relative_url }})

## 5. BeanOutputConverter로 단일 객체 받기

일반 문자열 대신 배우와 영화 목록을 가진 Java 객체로 응답을 변환했다.

```java
private static record ActorsFilms(String actor, List<String> films) {}

@GetMapping("/bean")
public ActorsFilms getSingleActorFilms(@RequestParam String userInput) {
    return chatClient.prompt()
            .user(userInput)
            .call()
            .entity(ActorsFilms.class);
}
```

![단일 ActorsFilms 객체 변환 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/06-bean-result.png' | relative_url }})

## 6. List<Bean>으로 여러 객체 받기

`List<ActorsFilms>`처럼 제네릭을 포함한 반환형은 `ParameterizedTypeReference`로 전체 타입 정보를 전달했다.

```java
return chatClient.prompt()
        .user(userInput)
        .call()
        .entity(new ParameterizedTypeReference<List<ActorsFilms>>() {});
```

![여러 배우를 List Bean으로 변환한 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/07-list-bean-result.png' | relative_url }})

## 7. Map과 단순 List 출력

필드가 유동적인 과일 이름과 맛 데이터는 `Map<String, Object>`로 받았다.

![과일 이름과 맛을 Map으로 변환한 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/08-map-result.png' | relative_url }})

단순한 아이스크림 맛 목록은 `List<String>`으로 변환했다.

```java
.entity(new ParameterizedTypeReference<List<String>>() {})
```

![아이스크림 맛을 문자열 List로 변환한 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/09-list-result.png' | relative_url }})

구조화 출력은 응답의 데이터 형태를 안정적으로 만들지만 내용의 사실성까지 보장하지는 않는다. 실제 서비스에서는 검색 결과나 데이터베이스를 이용한 별도 검증이 필요하다.

## 8. SimpleLoggerAdvisor로 AI 요청과 응답 관찰

Advisor는 ChatClient 요청 전후에 공통 로직을 실행하는 인터셉터와 비슷하다. 로깅, 안전성 검사, 메모리, 검색 문서 추가, 성능 측정처럼 여러 요청에 반복되는 기능을 본래 비즈니스 코드와 분리할 수 있다.

![SimpleLoggerAdvisor 적용 실습 자료]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/01-advisor-slide.png' | relative_url }})

ChatClient 요청에 `SimpleLoggerAdvisor`를 등록했다.

```java
return this.chatClient.prompt()
        .options(chatOptions)
        .system(systemMessage)
        .user(userInput)
        .advisors(new SimpleLoggerAdvisor(Ordered.LOWEST_PRECEDENCE))
        .call()
        .content();
```

상세 로그를 보기 위해 `application.yml`에도 DEBUG 레벨을 설정했다.

```yaml
logging:
  level:
    '[org.springframework.ai]': INFO
    '[org.springframework.ai.chat.client.advisor.SimpleLoggerAdvisor]': DEBUG
```

실행 로그에서는 System Message, User Message, 모델 옵션, 생성 결과와 토큰 사용량을 확인할 수 있었다. 함께 등록된 사용자 정의 Advisor의 실행 순서도 드러났다.

```text
[전처리] AdvisorA
[전처리] AdvisorB
[전처리] AdvisorC
SimpleLoggerAdvisor 요청 로그
OpenAI 호출
SimpleLoggerAdvisor 응답 로그
[후처리] AdvisorC
[후처리] AdvisorB
[후처리] AdvisorA
```

전처리는 등록 순서대로 실행되고 후처리는 반대 순서로 돌아왔다. 한 요청에서는 입력 120토큰과 출력 216토큰, 총 336토큰을 사용했고 약 3.22초가 걸렸다. 설정해 둔 100토큰 기준을 넘자 `TokenLatencyProfilerAdvisor`의 비용 경고도 정상적으로 발생했다.

## 9. Docker로 PostgreSQL 18과 pgvector 실행

RAG 검색에는 문서의 임베딩 벡터를 저장하고 유사도를 계산할 공간이 필요하다. 저장소에 포함된 실행 스크립트로 PostgreSQL 18과 pgvector 확장이 들어 있는 컨테이너를 실행했다.

![pgvector 컨테이너 설치 실습 자료]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/02-pgvector-install-slide.png' | relative_url }})

```bash
cd spring-ai/pgvector
./pgvector-run.sh
```

상태는 `docker ps`로 확인했다.

```text
IMAGE: pgvector/pgvector:pg18
STATUS: healthy
PORT: 5432:5432
NAME: pgvector
```

![Docker Desktop에서 확인한 pgvector 실행 상태]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/03-docker-pgvector-running-public.png' | relative_url }})

## 10. 같은 5432 포트를 바라본 두 PostgreSQL 문제

처음 DBeaver에서 `CREATE EXTENSION vector`를 실행했을 때 다음 오류가 발생했다.

```text
extension "vector" is not available
Could not open extension control file
"/opt/homebrew/share/postgresql@17/extension/vector.control"
```

![Homebrew PostgreSQL에 잘못 연결된 DBeaver 오류]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/04-dbeaver-wrong-postgres.png' | relative_url }})

Docker 컨테이너 문제처럼 보였지만 오류 경로의 `/opt/homebrew`가 중요한 단서였다. 맥북에는 과거 Homebrew로 설치한 PostgreSQL 17이 로그인 서비스로 실행되고 있었고, Docker의 PostgreSQL 18도 같은 5432 포트를 사용하고 있었다.

```text
Homebrew PostgreSQL 17 → 127.0.0.1:5432
Docker PostgreSQL 18   → 0.0.0.0:5432
```

그 결과 DBeaver와 Spring 애플리케이션이 pgvector가 없는 Homebrew PostgreSQL에 연결됐다. 실습 중에는 로컬 서비스를 중지해 Docker pgvector가 5432를 사용하도록 정리했다.

```bash
brew services stop postgresql@17
```

두 데이터베이스를 동시에 사용해야 한다면 한쪽을 5433처럼 다른 포트로 매핑하는 것이 안전하다. DBeaver는 데이터베이스 서버가 아니라 접속 도구이므로 DBeaver 연결을 삭제하는 것만으로 실제 포트 충돌이 해결되지는 않는다.

## 11. 문장과 메타데이터를 pgvector에 저장

`EmbeddingService`에서 헌법 제1조부터 제5조까지를 Spring AI `Document`로 만들었다. 각 문서에는 원문뿐 아니라 카테고리, 문서 유형과 조문 번호를 메타데이터로 넣었다.

![헌법 문장을 Document로 저장하는 실습 자료]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/05-embedding-code-slide.png' | relative_url }})

```java
save(
        "제1조 ①대한민국은 민주공화국이다. " +
        "②대한민국의 주권은 국민에게 있고, 모든 권력은 국민으로부터 나온다.",
        Map.of(
                "category", "constitution",
                "type", "law",
                "article", 1
        )
)
```

실행 시 문서는 다음 과정을 거쳤다.

```text
헌법 문장
→ OpenAI text-embedding-3-small
→ 임베딩 벡터
→ PostgreSQL vector_store 저장
```

검색에는 `SearchRequest`를 사용했다.

```java
SearchRequest request = SearchRequest.builder()
        .query(query)
        .topK(5)
        .similarityThreshold(0.4)
        .build();

return vectorStore.similaritySearch(request);
```

`대한민국은 민주국가인가?`라는 질문에 헌법 제1조가 가장 높은 유사도로 반환됐다. 문자열이 정확히 일치하지 않아도 의미가 가까운 문서를 찾는다는 것을 확인했다.

![OpenAI 임베딩을 이용한 헌법 의미 검색]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/06-openai-embedding-search-public.png' | relative_url }})

DBeaver에서도 `vector_store`에 헌법 1~5조의 ID, 본문, 메타데이터가 저장된 것을 확인했다.

![DBeaver에서 확인한 vector_store 데이터]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/07-dbeaver-vector-store-public.png' | relative_url }})

## 12. OpenAI 임베딩을 로컬 BAAI/bge-m3로 교체

다음 단계에서는 외부 API 임베딩 대신 Docker에서 실행되는 오픈소스 BAAI/bge-m3 모델을 사용했다.

![BAAI bge-m3 모델 전환 실습 자료]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/08-baai-model-slide.png' | relative_url }})

저장소의 모델 컨테이너를 실행하고 Ollama API에서 모델 목록을 확인했다.

```bash
cd embedding-model/baai-ollama
./run-ollama.sh
curl http://localhost:11436/api/tags
```

실행된 모델 정보는 다음과 같았다.

```text
model: bge-m3
family: bert
parameters: 567M
context length: 8192
embedding length: 1024
capability: embedding
```

이때 두 컨테이너가 함께 실행된다.

```text
ollama-baai-embedding → 11436:11434 → 문장을 1024차원 벡터로 변환
pgvector              → 5432:5432   → 벡터 저장과 유사도 검색
```

Spring Boot는 `baai` 프로필로 실행했다.

```bash
mvn spring-boot:run -Dspring-boot.run.profiles=baai
```

프로필 설정에서는 임베딩 모델을 Ollama로 바꾸고 로컬 컨테이너 주소와 모델 이름을 지정했다.

```yaml
spring:
  config:
    activate:
      on-profile: baai

  ai:
    model:
      embedding: ollama

    ollama:
      base-url: http://localhost:11436
      embedding:
        model: bge-m3
```

로그의 `The following 1 profile is active: "baai"`를 통해 설정 적용을 확인했다. 같은 질문을 검색했을 때 헌법 제1조의 유사도가 약 0.6263으로 나타났다.

![로컬 bge-m3 임베딩 검색 결과]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/09-baai-search-result-public.png' | relative_url }})

OpenAI 모델에서 약 0.5402였던 점수가 bge-m3에서는 약 0.6263으로 달라졌다. 이는 어느 모델이 무조건 더 정확하다는 뜻이 아니라 모델마다 문장을 벡터로 표현하는 공간과 점수 분포가 다르다는 의미다. 따라서 유사도 임계값도 모델과 데이터에 맞춰 다시 평가해야 한다.

## 13. PDF를 처리하는 PdfEtlService 구현

마지막으로 TXT 문서만 처리하던 RAG-ETL에 PDF 처리를 추가했다. 대상 파일은 `대한민국형법(20250318).pdf`였다.

![PDF RAG-ETL 확장 실습 자료]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/10-pdf-etl-slide.png' | relative_url }})

기존 `TxtEtlService`와 동일한 Extract-Transform-Load 구조를 유지하되, Extract 단계에서 `TextReader`를 `PagePdfDocumentReader`로 교체했다.

```java
public PdfEtlService extract(String fileName) {
    PdfDocumentReaderConfig config = PdfDocumentReaderConfig.builder()
            .withPagesPerDocument(1)
            .build();

    this.documents = new PagePdfDocumentReader(
            resourceLoader.getResource(DOCUMENTS_PATH + fileName),
            config
    ).get();

    log.info(
            "[PDF ETL] Extract 완료 - fileName: {}, document 수: {}",
            fileName,
            documents.size()
    );

    return this;
}
```

한 페이지를 하나의 `Document`로 읽은 뒤 `TokenTextSplitter`로 더 작은 청크로 나눴다.

```java
DocumentTransformer transformer = TokenTextSplitter.builder()
        .withChunkSize(200)
        .withMinChunkSizeChars(100)
        .withMinChunkLengthToEmbed(5)
        .withMaxNumChunks(10000)
        .withKeepSeparator(true)
        .build();
```

PDF ETL API도 Controller에 추가했다.

```java
@PostMapping("/pdf")
public Map<String, Object> executePdfETL(@RequestParam String fileName) {
    List<Document> loaded = pdfEtlService.extract(fileName)
            .transform()
            .load();

    return Map.of(
            "fileName", fileName,
            "chunkCount", loaded.size()
    );
}
```

구현 중 `private final PdfEtlService pdfEtlService;`를 실수로 import 구역에 작성해 Java가 이를 unnamed class로 해석하는 컴파일 오류가 발생했다.

```text
unnamed classes are a preview feature and are disabled by default
class, interface, enum, or record expected
```

preview 기능이 필요한 문제가 아니라 필드 선언 위치가 잘못된 문제였다. 해당 필드를 `RagEtlController` 클래스 내부로 옮겨 해결했다.

## 14. 대한민국 형법 PDF 419개 청크 적재

UI에서 PDF 파일을 선택해 ETL을 실행했다. Transform 결과 PDF는 총 419개 청크로 나뉘었다.

```text
[PDF ETL] Transform 완료 - 청크 수: 419
[PDF ETL] Load 완료 - VectorDB 저장 건수: 419
```

Transform 완료 시각은 11:35:43, Load 완료 시각은 11:36:29였다. 로컬 bge-m3가 419개 청크를 임베딩하고 pgvector에 저장하는 데 약 47초가 걸렸다.

![대한민국 형법 PDF 419개 청크 저장 결과]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/11-pdf-etl-loaded-public.png' | relative_url }})

## 15. 형법 PDF 의미 검색

처음에는 질문을 `제5조 외국인의 국외법은?`으로 잘못 입력해 헌법 문서가 검색됐다. `국외법`을 정확한 용어인 `국외범`으로 고쳐 검색하자 형법 제4조와 제5조가 포함된 청크가 유사도 약 0.7060으로 반환됐다.

![형법 제5조 외국인의 국외범 검색 결과]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/12-pdf-vector-search-public.png' | relative_url }})

더 자연스러운 질문도 시도했다.

```text
외국인이 대한민국 영역 밖에서 범죄를 저지르면 대한민국 형법은 언제 적용되나요?
```

이 질문에는 제6조 청크가 가장 높은 점수를 받았고 제4조·제5조 청크가 뒤를 이었다. 벡터 검색은 조문 번호를 정확히 조회하는 키워드 검색이 아니라 질문 전체의 의미와 문서 청크의 의미를 비교한다. 질문 표현과 청크 경계에 따라 순위가 달라질 수 있다.

또한 한 검색 결과에 제4조와 제5조가 함께 포함됐다. 이는 PDF를 페이지 단위로 추출한 후 200토큰 전후로 다시 나누면서 인접 조문이 같은 청크에 담겼기 때문이다.

## 16. 헌법 전문 TXT를 Vector DB에 추가

RAG 답변에 사용할 지식 범위를 넓히기 위해 `대한민국헌법(19880225).txt`를 ETL했다. Extract, Transform, Load를 거친 결과 총 108개 청크가 pgvector에 저장됐다.

![대한민국 헌법 TXT 108개 청크 저장 결과]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/13-constitution-txt-etl.png' | relative_url }})

적재 후 다음 문장으로 의미 검색을 실행했다.

```text
국회의원은 국가이익을 우선하여 양심에 따라 직무를 행해야 하는가?
```

검색 결과의 첫 번째 청크에는 헌법 제45조와 제46조가 함께 들어 있었고 유사도는 약 0.6670이었다. 특히 제46조의 청렴 의무, 국가이익 우선 의무와 지위 남용 금지 조항이 검색됐다.

![헌법 제46조가 포함된 벡터 검색 결과]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/14-constitution-article46-search-public.png' | relative_url }})

이 결과로 데이터 적재와 벡터 검색 자체는 정상이라는 것을 확인했다. 화면 설명에는 최대 3개라고 적혀 있었지만 실제 서비스의 `topK(5)` 설정에 따라 5개의 결과가 표시됐다.

## 17. RewriteQueryTransformer로 검색 질문 정제

`RetrievalAugmentationAdvisor`에 `RewriteQueryTransformer`와 `VectorStoreDocumentRetriever`를 연결했다.

```java
RetrievalAugmentationAdvisor retrievalAdvisor =
        RetrievalAugmentationAdvisor.builder()
                .queryTransformers(
                        RewriteQueryTransformer.builder()
                                .chatClientBuilder(transformerBuilder)
                                .build()
                )
                .documentRetriever(
                        VectorStoreDocumentRetriever.builder()
                                .topK(5)
                                .similarityThreshold(0.3)
                                .vectorStore(vectorStore)
                                .build()
                )
                .build();
```

처음 사용한 구어체 질문은 검색 의도가 모호해 `모르겠습니다`라는 답변이 나왔다. 질문에 판단 기준과 원하는 근거를 명시하자 제46조가 정상적으로 검색됐다.

```text
국회의원이 개인의 이익만 챙기는 행위는 헌법상 적절한가?
관련 헌법 조항을 근거로 설명해줘.
```

![RewriteQueryTransformer가 헌법 제46조를 근거로 생성한 답변]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/15-rewrite-query-success-public.png' | relative_url }})

최종 답변은 국회의원이 청렴 의무를 지고 국가이익을 우선해 양심에 따라 직무를 수행해야 한다는 제46조의 내용을 근거로 제시했다. 질문 재작성은 단순한 문장 교정이 아니라, 벡터 검색에 적합하도록 질문의 핵심 의도를 명확하게 만드는 전처리 단계라는 것을 확인했다.

## 18. CompressionQueryTransformer와 대화 문맥 실험

다음으로 `CompressionQueryTransformer`와 `MessageChatMemoryAdvisor`를 함께 등록했다. 목적은 앞선 대화를 이용해 `대통령은?`과 같은 모호한 후속 질문을 독립적인 검색 질문으로 바꾸는 것이다.

```java
.queryTransformers(
        CompressionQueryTransformer.builder()
                .chatClientBuilder(transformerBuilder)
                .build(),
        RewriteQueryTransformer.builder()
                .chatClientBuilder(transformerBuilder)
                .build()
)
```

```java
this.chatClient = ChatClient.builder(chatModel)
        .defaultAdvisors(
                MessageChatMemoryAdvisor.builder(chatMemory).build(),
                retrievalAdvisor,
                new SimpleLoggerAdvisor()
        )
        .build();
```

먼저 국회의원의 개인 이익 추구가 적절한지 질문했고 제46조를 근거로 한 답변을 받았다.

![CompressionQueryTransformer 실습의 첫 번째 질문]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/16-compression-first-question-public.png' | relative_url }})

같은 대화 ID에서 `대통령은?`이라고 후속 질문했다. 세션 ID는 유지됐지만 답변은 대통령의 일반적인 헌법상 지위와 직무를 설명하는 수준에 머물렀다.

![CompressionQueryTransformer의 모호한 후속 질문 결과]({{ '/assets/images/spring-ai-rag-practice-2026-09-10/17-compression-followup-result-public.png' | relative_url }})

앞선 개인 이익 문맥을 어느 정도 연상할 수 있는 답변이었지만, 기대했던 `대통령도 개인의 이익만 추구하는 것이 직무에 부합하는가?`라는 독립 질문으로 변환됐는지는 화면만으로 확정하기 어려웠다. 이번에는 여기까지 구현하고, 다음 실습에서 Transformer 요청 로그와 Chat Memory 전달 순서를 다시 확인하기로 했다.

## 실습 중 해결한 문제

### API 키 환경변수 소실

새 터미널을 열자 `OPENAI_API_KEY` 환경변수가 사라져 401 인증 오류가 발생했다. 키를 파일에 저장하지 않고 실행할 터미널 세션에 다시 입력해 해결했다.

```bash
read -s "OPENAI_API_KEY?API 키를 입력하세요: "; export OPENAI_API_KEY; echo
```

### Homebrew PostgreSQL과 Docker pgvector의 포트 충돌

두 서버가 모두 5432를 사용하면서 애플리케이션과 DBeaver가 pgvector가 없는 로컬 PostgreSQL 17에 연결됐다. Homebrew 서비스를 중지하고 Docker PostgreSQL 18에 연결해 해결했다.

### DBeaver 인증 실패

DBeaver 연결에 저장된 비밀번호가 컨테이너 설정과 달라 `password authentication failed for user "postgres"` 오류가 발생했다. 호스트, 포트, 데이터베이스, 사용자와 비밀번호를 컨테이너 설정에 맞추고 새 SQL Editor를 열어 해결했다.

### 필드 선언 위치 오류

`PdfEtlService` 필드를 import 사이에 넣으면서 preview 기능 관련 메시지가 발생했다. Java 컴파일러의 첫 메시지만 보고 preview 설정을 켜는 대신 소스 구조와 줄 번호를 확인해 잘못된 필드 위치를 찾았다.

## 오늘 배운 핵심

- Advisor를 사용하면 AI 요청 전후의 공통 관심사를 분리할 수 있다.
- SimpleLoggerAdvisor로 실제 프롬프트, 응답과 토큰 사용량을 관찰할 수 있다.
- 전처리 Advisor는 등록 순서대로, 후처리는 반대 순서로 실행된다.
- pgvector는 PostgreSQL에서 임베딩 벡터 저장과 유사도 검색을 제공한다.
- DBeaver는 DB 서버가 아니라 DB 서버에 접속하는 클라이언트다.
- 같은 포트를 사용하는 로컬 PostgreSQL과 Docker PostgreSQL을 구분해야 한다.
- Spring AI의 `VectorStore`는 문서 저장 시 임베딩 생성을 함께 처리한다.
- `similarityThreshold`로 관련성이 낮은 검색 결과를 제외할 수 있다.
- BAAI/bge-m3를 Ollama로 실행하면 임베딩을 로컬에서 생성할 수 있다.
- 임베딩 모델이 달라지면 벡터 차원과 유사도 점수 분포도 달라진다.
- RAG-ETL은 문서를 Extract하고, 검색 가능한 청크로 Transform한 뒤 Vector DB에 Load한다.
- `PagePdfDocumentReader`와 `TokenTextSplitter`로 PDF를 검색 가능한 단위로 만들 수 있다.
- 구조화된 벡터 데이터라도 검색 품질은 질문 표현, 임계값과 청크 크기의 영향을 받는다.
- RewriteQueryTransformer는 구어체 질문에서 검색 의도를 추출해 Vector DB 검색에 적합하게 다듬는다.
- CompressionQueryTransformer는 대화 이력을 이용해 모호한 후속 질문을 독립 질문으로 만드는 역할을 한다.

## 추후 추가할 실습

오늘은 RAG 전처리 모듈 중 RewriteQueryTransformer까지 정상 동작을 확인하고, CompressionQueryTransformer의 기본 구성과 후속 질문 호출까지 진행했다. 다음 학습에서 아래 내용을 이어서 추가할 예정이다.

- CompressionQueryTransformer의 변환 로그와 Chat Memory 전달 과정 재검증
- MultiQueryExpander로 하나의 질문을 여러 검색 질문으로 확장
- 검색 문서 후처리와 ContextualQueryAugmenter를 포함한 Full RAG
- Chat Memory 저장 방식과 대화별 메모리 관리
- Spring AI Tool Calling 실습
- Docker 기반 MCP Server와 Spring AI MCP Client 연결
- MCP Client에서 컨테이너 목록, 로그와 설정 조회

이번 실습으로 일반적인 LLM 호출을 넘어, 애플리케이션이 자체 문서를 적재하고 검색한 뒤 답변의 근거로 활용하는 RAG 흐름까지 연결했다. 검색이 기대와 다르게 동작할 때는 데이터 적재 여부, 검색 결과, 질문 변환과 대화 메모리를 각각 나눠 확인해야 한다는 점도 배웠다.
