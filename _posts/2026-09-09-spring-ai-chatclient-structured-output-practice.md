---
layout: post
title: "Spring AI 실습: ChatClient부터 PromptTemplate과 Structured Output까지"
date: 2026-09-09 17:00:00 +0900
category: [backend-cloud]
tags: [Spring-AI, ChatClient, ChatOptions, PromptTemplate, SystemMessage, BeanOutputConverter, Structured-Output, Java, learning-note]
---

오늘은 Spring AI 예제 프로젝트를 실행하고, ChatClient의 동기·스트리밍 호출부터 동적 프롬프트와 구조화 출력까지 단계적으로 실습했다. 단순히 ChatGPT의 답변을 화면에 출력하는 데서 끝나지 않고, 응답을 Java 객체와 컬렉션으로 변환하는 과정까지 확인했다.

## 1. 실습 프로젝트 준비

실습 코드는 GitHub 저장소에서 내려받았다.

```bash
git clone https://github.com/himang10/spring-ai.git
```

저장소는 직접 코드를 작성하는 `01.training-code`, 막힐 때 참고하는 `02.answer-code`, RAG 실습용 `pgvector`, 로컬 임베딩 모델용 `embedding-model` 등으로 구성되어 있었다.

![Spring AI 실습 코드 구성]({{ '/assets/images/spring-ai-practice-2026-09-09/01-source-structure.png' | relative_url }})

이번 실습에서는 Java 21과 Maven을 사용했다. OpenAI API 키는 코드나 설정 파일에 직접 입력하지 않고 터미널 환경변수로 전달했다.

```bash
read -s "OPENAI_API_KEY?API 키를 입력하세요: "; export OPENAI_API_KEY; echo
mvn spring-boot:run
```

## 2. ChatClient로 동기·스트리밍 호출

첫 번째 과제는 비어 있는 `ChatController`의 두 메서드를 구현하는 것이었다. 동기 호출은 응답이 완성된 후 문자열을 한 번에 반환한다.

```java
@GetMapping("/ai")
public String chat(@RequestParam String userInput) {
    return this.chatClient.prompt()
            .user(userInput)
            .call()
            .content();
}
```

스트리밍 호출은 생성되는 내용을 `Flux<String>`으로 차례대로 반환한다.

```java
@GetMapping(value = "/ai/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<String> chatStream(@RequestParam String userInput) {
    return this.chatClient.prompt()
            .user(userInput)
            .stream()
            .content();
}
```

두 방식의 핵심 차이는 `call()`과 `stream()`이다.

- `call()`: 완성된 응답을 기다렸다가 한 번에 반환한다.
- `stream()`: 생성되는 내용을 조각 단위로 바로 반환한다.

처음에는 기존 메서드 내부를 채우지 않고 같은 이름의 메서드를 새로 추가해 `method is already defined` 컴파일 오류가 발생했다. 중복 메서드를 제거하고 `@GetMapping`이 붙은 기존 메서드 내부에 코드를 작성해 해결했다. 스트리밍 메서드에서 실수로 `call()`을 사용했을 때는 `String cannot be converted to Flux<String>` 오류가 발생했고, 이를 `stream()`으로 바꾸어 반환 타입을 맞췄다.

![ChatClient 첫 실행 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/02-chat-options-slide.png' | relative_url }})

## 3. ChatOptions로 요청 설정 변경

다음으로 모델과 응답 생성 옵션을 코드에서 지정했다.

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

- `model`: 사용할 AI 모델
- `temperature`: 응답의 무작위성과 표현 다양성
- `topP`: 다음 토큰 후보를 선택하는 확률 범위
- `maxTokens`: 생성할 수 있는 최대 토큰 수

설정을 적용한 뒤 브라우저에서 실제 답변이 반환되는 것을 확인했다.

![ChatOptions를 적용한 ChatClient 응답]({{ '/assets/images/spring-ai-practice-2026-09-09/03-chat-result.png' | relative_url }})

같은 질문을 반복해도 표현이 조금씩 달라질 수 있었다. `temperature`는 답변의 사실성을 높이는 옵션이라기보다 생성 결과의 변동성을 조절하는 값이라는 점이 중요했다.

## 4. PromptTemplate로 동적 System Message 구성

사용자 질문만 전달하던 방식에서 한 단계 더 나아가 AI의 역할, 말투, 답변 규칙을 System Message로 지정했다. 또한 `{aiName}`과 `{terms}`를 실행 시점에 바꿀 수 있도록 `PromptTemplate`을 사용했다.

![동적 프롬프트 템플릿 실습 자료]({{ '/assets/images/spring-ai-practice-2026-09-09/04-prompt-template-slide.png' | relative_url }})

```java
private static final String PROMPT_TEMPLATE = """
        너는 SKALA Spring AI 교육 과정의 친절하고 열정적인 보조교사 {aiName}입니다.
        - 수강생의 질문에 따뜻하고 격려하는 어조로 답변해 주세요.
        - {terms}는 초보자도 이해하기 쉽게 비유를 들어서 설명해 주세요.
        - 답변 끝에는 항상 실습을 응원하는 따뜻한 한마디를 덧붙여 주세요.
        """;

private final PromptTemplate systemPrompt =
        new PromptTemplate(PROMPT_TEMPLATE);
```

템플릿 변수는 `Map`으로 치환하고 `SystemMessage`로 만들었다.

```java
SystemMessage systemMessage = new SystemMessage(
        systemPrompt.render(Map.of(
                "aiName", "코디",
                "terms", "Spring AI와 ChatClient"
        ))
);
```

생성된 시스템 메시지는 ChatClient 요청에 추가했다.

```java
return this.chatClient.prompt()
        .messages(systemMessage)
        .user(userInput)
        .call()
        .content();
```

사용자는 단순히 "너를 소개시켜줘"라고 질문했지만, 모델은 템플릿에 따라 자신을 SKALA 교육 보조교사 `코디`라고 소개하고 마지막에 응원 문구를 덧붙였다.

![PromptTemplate와 System Message 적용 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/05-prompt-template-result.png' | relative_url }})

이를 통해 System Message가 사용자 질문보다 앞서 AI의 역할과 응답 방식을 결정한다는 것을 확인했다.

## 5. BeanOutputConverter로 단일 객체 받기

일반 Chat 응답은 읽기에는 편하지만 프로그램에서 다시 분석하기 어렵다. Structured Output은 AI 응답을 정해진 Java 타입으로 바로 변환한다.

먼저 배우 이름과 영화 목록을 담을 `record`를 정의했다.

```java
private static record ActorsFilms(
        String actor,
        List<String> films
) {}
```

그리고 `.content()` 대신 `.entity(ActorsFilms.class)`를 사용했다.

```java
@GetMapping("/bean")
public ActorsFilms getSingleActorFilms(@RequestParam String userInput) {
    return chatClient.prompt()
            .user(userInput)
            .call()
            .entity(ActorsFilms.class);
}
```

![단일 ActorsFilms 객체 변환 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/06-bean-result.png' | relative_url }})

AI의 응답이 다음 구조를 가진 Java 객체로 변환되었다.

```json
{
  "actor": "톰 행크스",
  "films": [
    "포레스트 검프",
    "Saving Private Ryan",
    "캐스트 어웨이",
    "빅",
    "더 그린 마일"
  ]
}
```

## 6. List<Bean>으로 여러 객체 받기

배우 여러 명의 데이터를 받을 때는 `List<ActorsFilms>`가 필요하다. Java의 제네릭 타입 정보를 전달하기 위해 `ParameterizedTypeReference`를 사용했다.

```java
@GetMapping("/list-bean")
public List<ActorsFilms> getMultipleActorsFilms(
        @RequestParam String userInput
) {
    return chatClient.prompt()
            .user(userInput)
            .call()
            .entity(
                    new ParameterizedTypeReference<List<ActorsFilms>>() {}
            );
}
```

![List ActorsFilms 변환 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/07-list-bean-result.png' | relative_url }})

단일 객체는 `ActorsFilms.class`로 충분하지만 `List<ActorsFilms>`처럼 제네릭이 포함되면 런타임에 내부 타입 정보가 사라질 수 있다. `ParameterizedTypeReference`는 변환기에 전체 타입 정보를 알려주는 역할을 한다.

## 7. Map과 단순 List로 변환

필드 구조가 고정되지 않은 데이터는 `Map<String, Object>`로 받았다.

```java
@GetMapping("/map")
public Map<String, Object> getMapResult(@RequestParam String userInput) {
    return chatClient.prompt()
            .user(userInput)
            .call()
            .entity(
                    new ParameterizedTypeReference<Map<String, Object>>() {}
            );
}
```

![과일 이름과 맛을 Map으로 변환한 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/08-map-result.png' | relative_url }})

이번 결과에서는 최상위 `fruits` 키 안에 과일별 `name`과 `taste`가 담긴 배열이 생성되었다. `Map`은 Java 타입에서 필드 구조를 고정하지 않으므로 질문과 모델 응답에 따라 내부 모양이 달라질 수 있다.

단순 문자열 목록은 `List<String>`으로 변환했다.

```java
@GetMapping("/list")
public List<String> getListResult(@RequestParam String userInput) {
    return chatClient.prompt()
            .user(userInput)
            .call()
            .entity(
                    new ParameterizedTypeReference<List<String>>() {}
            );
}
```

![아이스크림 맛을 문자열 List로 변환한 결과]({{ '/assets/images/spring-ai-practice-2026-09-09/09-list-result.png' | relative_url }})

## 8. 자유 응답과 구조화 출력 비교

실습에서 사용한 네 가지 출력 형식을 정리하면 다음과 같다.

| 출력 방식 | Java 반환형 | 특징 |
|---|---|---|
| 일반 Chat | `String` | 설명이 풍부하지만 후처리가 필요함 |
| Bean | `ActorsFilms` | 필드가 고정된 객체 하나 |
| List Bean | `List<ActorsFilms>` | 같은 구조의 객체 여러 개 |
| Map | `Map<String, Object>` | 유연한 키-값 구조 |
| List | `List<String>` | 단순 값의 목록 |

구조화 출력은 데이터의 형태를 안정적으로 만드는 기능이지, 답변의 사실을 검증하는 기능은 아니다. 실제 일반 Chat 결과에서는 톰 행크스의 영화 목록에 그가 출연하지 않은 작품이 포함되기도 했다. 따라서 실제 서비스에서는 출력 형식 변환과 별개로 검색 결과, 데이터베이스 또는 검증 로직을 이용해 내용의 정확성을 확인해야 한다.

## 실습 중 해결한 문제

### 같은 메서드를 두 번 작성한 오류

기존 메서드 내부를 구현해야 했지만 같은 이름과 매개변수의 메서드를 새로 추가하면서 다음 오류가 발생했다.

```text
method chat(java.lang.String) is already defined
method chatStream(java.lang.String) is already defined
```

중복으로 추가한 메서드를 지우고 `@GetMapping`이 붙은 기존 메서드의 본문만 채워 해결했다.

### 스트리밍 반환 타입 불일치

`Flux<String>`을 반환하는 `chatStream()`에서 `call()`을 사용해 다음 오류가 발생했다.

```text
String cannot be converted to Flux<String>
```

`call()`을 `stream()`으로 교체해 해결했다.

### 새 엔드포인트 호출 시 404

`/ai/bean`을 추가한 직후 브라우저에서 호출했을 때 404가 발생했다. 서버가 변경 이전 코드로 계속 실행 중이었기 때문이다.

```bash
mvn compile
mvn spring-boot:run
```

코드를 컴파일하고 Spring Boot 서버를 재시작한 뒤 새 엔드포인트가 정상 등록되었다.

## 오늘 배운 핵심

- `ChatClient`의 Fluent API로 AI 요청을 구성할 수 있다.
- `call()`은 완성된 응답을, `stream()`은 생성 중인 응답을 반환한다.
- `ChatOptions`로 모델, 응답 다양성, 후보 범위와 최대 길이를 설정할 수 있다.
- `PromptTemplate`은 실행 시점에 변수를 치환하는 동적 프롬프트를 만든다.
- `SystemMessage`는 AI의 역할과 답변 규칙을 지정한다.
- `.entity(Class)`는 AI 응답을 단일 Java 객체로 변환한다.
- 제네릭 컬렉션은 `ParameterizedTypeReference`로 전체 타입을 전달한다.
- `Bean`, `List<Bean>`, `Map`, `List`를 목적에 맞게 선택해야 한다.
- 구조화 출력은 데이터 형태를 보장하지만 사실의 정확성까지 보장하지 않는다.

이번 실습을 통해 Spring AI가 단순한 챗봇 연결 도구를 넘어, 생성형 AI의 응답을 Java 애플리케이션에서 다루기 좋은 데이터 구조로 연결해 주는 계층이라는 점을 확인했다.
