---
layout: post
title: "FastAPI SSE부터 Multi-Agent 가시화까지: 에이전트의 진행 과정을 화면에 보여주기"
date: 2026-10-16 09:00:00 +0900
category: [llm]
tags: [FastAPI, SSE, Next.js, LangGraph, MCP, Multi-Agent, Supervisor, learning-note]
---

ICT 운영 지원 에이전트에 스트리밍을 연결하고, Next.js 채팅 화면을 만든 뒤, Supervisor 기반 Multi-Agent의 작업 과정을 시각화했다.

처음에는 최종 답변만 받던 구조였다. 여기에 도구 실행 상태, 답변 토큰, 작업 계획과 Task 진행 상태를 순서대로 표시하면서 사용자가 에이전트의 동작을 확인할 수 있도록 확장했다.

## 1. FastAPI에 SSE 스트리밍 연결하기

기존 `/chat` API는 답변을 완성한 뒤 JSON으로 반환했다. 이번에는 `/chat/stream`을 추가해 하나의 요청에서 여러 이벤트를 전달하도록 만들었다.

| 이벤트 | 역할 |
|---|---|
| `tool_call_start` | 도구 이름과 호출 인자 전달 |
| `tool_call_end` | 도구 실행 완료와 결과 미리보기 전달 |
| `token` | 답변 텍스트 조각 전달 |
| `done` | 정상 완료와 `thread_id` 전달 |

SSE는 다음 형태로 전송한다. 이벤트 사이의 빈 줄이 각각의 메시지를 구분한다.

```text
event: token
data: {"type":"token","content":"데이터"}

event: done
data: {"type":"done","thread_id":"..."}
```

Backend에서는 LangGraph의 `astream()` 결과를 바로 노출하지 않고, converter에서 정해진 이벤트 형식으로 변환했다.

```text
LangGraph 실행 → 이벤트 변환 → SSE 응답 → Frontend
```

이렇게 하면 그래프 내부의 노드나 State 구조가 바뀌더라도 Frontend와 주고받는 이벤트 형식은 별도로 관리할 수 있다.

도구 호출 인자 처리도 주의할 부분이었다. 스트리밍 중에는 인자가 완성된 객체가 아니라 JSON 문자열 조각으로 들어올 수 있다. 따라서 완성된 도구 호출 정보를 사용해 `tool_call_start`를 생성했다.

## 2. curl로 이벤트 순서 확인하기

Frontend를 연결하기 전에 터미널에서 먼저 검증했다.

```bash
curl -N --max-time 120 \
  http://127.0.0.1:8000/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"message":"데이터베이스 다운 시 대응 절차를 알려주세요."}'
```

`-N` 옵션은 curl의 출력 버퍼링을 비활성화한다. 운영 매뉴얼 검색 도구인 `search_operations_manual`의 시작·종료 이벤트 뒤에 답변 토큰이 전달됐고, 마지막에는 `done`과 `thread_id`가 반환됐다.

터미널 출력이 길어 앞부분이 잘리기도 했다. 이후에는 `tee`로 결과를 파일에 저장하고 앞부분과 끝부분을 나눠 캡처했다.

```bash
curl -N --max-time 120 \
  http://127.0.0.1:8000/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"message":"데이터베이스 다운 시 대응 절차를 알려주세요."}' \
  | tee sse-basic.txt

head -n 18 sse-basic.txt
tail -n 9 sse-basic.txt
```

빈 메시지에는 입력 오류 대신 안내 답변이 생성됐다. 현재 Backend에는 빈 문자열을 차단하는 검증이 없기 때문이다.

또한 답변 출력 도중 `Ctrl+C`로 클라이언트 연결을 중단했다. `done`을 받지 못한 채 종료됐지만 서버 로그에는 취소 여부가 표시되지 않았다. 확인한 것은 클라이언트 수신 중단까지이며, 서버 내부의 모델·도구 실행까지 취소됐다고 단정할 수는 없었다.

## 3. Next.js 채팅 화면 만들기

Next.js App Router 기반 Frontend는 화면, 컴포넌트, 통신 로직을 나눠 구성했다.

```text
frontend/
├── app/page.tsx
├── components/
│   ├── ChatMessage.tsx
│   ├── ChatInput.tsx
│   └── ToolCallView.tsx
└── lib/
    ├── transport.ts
    ├── thread.ts
    └── types.ts
```

이번 구현에서는 `fetch()`로 Backend SSE 응답을 직접 읽었다. 네트워크 청크와 SSE 이벤트의 경계가 항상 일치하지는 않으므로 문자열을 버퍼에 쌓고, 빈 줄을 기준으로 완성된 이벤트를 분리했다.

화면에서는 도구 호출을 별도 박스로 표시한다. 실행 중에는 ‘도구 사용 중’, 완료 후에는 ‘도구 완료’로 바뀌며 답변은 토큰을 누적하는 방식이다.

![도구 완료 상태와 답변, 활성화된 입력창](/assets/images/2026-10-16-capstone-streaming/streaming-chat.png)

### thread_id는 Frontend에서 관리

대화 시작 시 Frontend가 `crypto.randomUUID()`로 `thread_id`를 생성하고 `localStorage`에 보관한다. 모든 요청에는 이 값을 함께 보낸다.

| 동작 | 화면 메시지 | thread_id | Backend 대화 기억 |
|---|---|---|---|
| 후속 질문 | 유지 | 유지 | 유지 |
| 새로고침 | 초기화 | 유지 | Backend가 실행 중이면 유지 |
| 새 대화 | 초기화하도록 구현 | 새로 생성 | 별도 대화로 처리 |
| Backend 재시작 | 화면과 별개 | 브라우저에 남음 | 현재 MemorySaver 기록 소멸 |

새로고침 후 기존 메시지는 사라졌지만 같은 Thread로 질문하자 DB 다운과 첫 단계인 ‘증상 인식’을 기억했다. 현재 체크포인터는 메모리 기반이므로 서버 재시작까지 견디려면 외부 저장소가 필요하다.

![새로고침 후에도 이전 DB 대응 문맥을 기억하는 답변](/assets/images/2026-10-16-capstone-streaming/refresh-memory.png)

## 4. E2E 테스트에서 확인한 동작

응답 출력 중에는 입력창과 전송 버튼을 비활성화했다. 새 메시지 전송이 차단되고 기존 스트리밍이 계속됐다. 취소 코드가 있다는 사실과 사용자가 실제 UI에서 그 경로를 실행할 수 있는지는 구분해야 했다.

![응답 생성 중 비활성화된 입력창](/assets/images/2026-10-16-capstone-streaming/input-disabled.png)

긴 응답 테스트에는 다음 질문을 사용했다.

```text
모든 운영 매뉴얼을 상세히 분석하여 1000자 이내로 정리해 주세요.
```

이번에는 도구를 한 번 호출한 뒤 짧은 안내 답변으로 끝났고, 측정 시간은 약 **4.80초**였다. 30초 이상 이어지는 응답 상황은 재현되지 않았다.

![스톱워치로 측정한 약 4.80초의 응답](/assets/images/2026-10-16-capstone-streaming/response-time.png)

Backend를 `Ctrl+C`로 종료한 뒤 새 질문을 보내자 `[에러 발생: Failed to fetch]`가 표시됐고 자동 재시도는 없었다. 이 테스트는 서버 종료 후 새 요청의 실패를 확인한 것이다. 응답 도중 프로세스가 강제로 종료되는 상황까지 검증한 것은 아니다.

![Backend 종료 후 연결 실패 표시](/assets/images/2026-10-16-capstone-streaming/backend-error.png)

## 5. Supervisor 기반 Multi-Agent로 확장하기

단일 에이전트를 확장해 Supervisor, Research Agent, Writing Agent로 역할을 나눴다.

```text
Supervisor
  ├─ 계획 수립과 Task 분해
  ├─ Research Agent에 조회 작업 위임
  └─ Writing Agent에 결과 통합 위임
```

Research는 MCP 조회 도구를 사용하고 Writing은 앞선 작업 결과를 통합한다. 기존 Single 모드는 유지하면서 Multi-Agent 전용 `/chat/supervisor/stream`을 추가했다.

화면에 표시하는 정보는 공개용 작업 계획, 실행 상태, 도구 호출과 결과다. 이를 모델의 내부 추론 전체를 보여주는 것과 동일하게 보지는 않았다.

| 이벤트 | 화면 역할 |
|---|---|
| `plan_created` | Plan 카드 표시 |
| `task_list_created` | Task 목록 생성 |
| `task_started` | 진행 상태로 변경 |
| `task_completed` | 완료 상태로 변경 |
| `sub_agent_token` | 해당 Task의 출력 미리보기 |
| `sub_agent_tool_call` | 해당 Task 안에 도구 호출 표시 |

각 이벤트의 `task_id`로 출력이 어느 작업에 속하는지 구분한다. Task 색은 회색 대기, 노란색 진행, 초록색 완료로 나눴다.

시작 상태는 실제 작업 실행 전에 별도 노드에서 반환했다. 실행 후 완료 상태만 반환하면 Frontend가 진행 중 상태를 표시할 기회를 놓칠 수 있기 때문이다.

## 6. Single과 Multi 모드 비교

두 모드에 같은 질문을 보냈다.

```text
운영 매뉴얼과 최근 incident를 종합하여 장애 대응 가이드를 만들어 주세요.
```

Single은 어떤 시스템인지 알려 달라는 추가 질문을 반환했고 도구를 호출하지 않았다.

![Single 모드의 시스템 이름 확인 요청](/assets/images/2026-10-16-capstone-streaming/single.png)

Multi는 세 Task를 생성했다.

1. 운영 매뉴얼 조사 — `research`
2. 최근 장애 사건 조사 — `research`
3. 결과를 종합해 가이드 작성 — `writing`

![첫 Task 진행 중, 나머지 Task 대기](/assets/images/2026-10-16-capstone-streaming/multi-progress.png)

![Research Task와 도구 호출 완료](/assets/images/2026-10-16-capstone-streaming/multi-completed.png)

![Writing 완료 후 통합 가이드](/assets/images/2026-10-16-capstone-streaming/multi-result.png)

진행 캡처는 같은 질문을 다시 실행하며 확보해 Plan 문구가 완료 화면과 조금 다르다. 진행 중에는 첫 Task가 노란색이고 뒤의 Task가 회색이었다. 완료 화면에서는 세 Task 모두 초록색으로 표시됐다.

최종 답변은 Writing 작업이 끝난 뒤 한 번에 전달했다. Task 안의 출력 미리보기 스트리밍과 최종 답변 표시는 서로 다른 단계다.

이번 비교는 개별 실행 결과다. Multi가 항상 더 좋은 답을 만든다는 결론보다는 **계획과 작업 진행을 구체적으로 확인할 수 있었다**는 점이 핵심이었다.

## 7. 복잡한 질문과 단순한 질문

복잡한 질문으로는 다음 문장을 사용했다.

```text
DB 다운 시 절차와 네트워크 장애 시 절차를 비교해서 알려주세요
```

Supervisor는 DB 조사, 네트워크 조사, 비교 작성의 세 Task로 분해했고 최종 답변에는 두 절차와 차이점이 함께 표시됐다.

![DB와 네트워크 조사 및 Writing 작업 분해](/assets/images/2026-10-16-capstone-streaming/comparison-tasks.png)

![두 장애 절차의 최종 비교 답변](/assets/images/2026-10-16-capstone-streaming/comparison-result.png)

두 주제가 답변에 포함됐다는 사실만으로 각 절차가 원문과 정확히 일치한다고 판단할 수는 없다. 내용 정확성은 검색 결과와 대조하는 별도 평가가 필요하다.

단순 질문인 `안녕하세요`에는 예상보다 작업이 세분화됐다.

1. 인사말을 준비합니다. — `writing`
2. 사용자에게 한국어로 인사합니다. — `writing`

![단순 인사에 생성된 두 Writing Task](/assets/images/2026-10-16-capstone-streaming/greeting.png)

한 번의 짧은 답변으로 처리할 수 있는 인사에도 계획과 작업 두 개가 만들어졌다. 단순 요청에도 같은 오케스트레이션 흐름을 적용하면 불필요한 단계가 생길 수 있다는 한계를 확인했다.

## 8. 실습을 통해 배운 점

스트리밍에서는 텍스트 전송 외에도 도구 실행과 완료의 표현, 대화 식별자 관리, 연결 중단 시 확인 범위를 설계해야 했다.

Multi-Agent에서는 출력의 양보다 작업별 구분이 중요했다. Plan, Task 상태, 담당 Agent, 도구 호출을 연결해 보여주니 최종 답변을 기다리는 동안에도 진행 상황을 파악할 수 있었다.

다음에는 단순 요청의 직접 처리 분기, 오류 이벤트와 연결 취소 로그, 긴 작업의 keep-alive, 응답 내용과 도구 호출을 함께 검증하는 평가 체계를 개선하고 싶다.
