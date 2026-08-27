---
layout: post
title: "LLM은 문장을 어떻게 학습 데이터로 바꿀까: 토큰화부터 슬라이딩 윈도우까지"
date: 2026-08-26 14:00:00 +0900
category: [ai, llm]
tags: [GPT, tokenization, BPE, PyTorch, learning-note]
---

LLM 공부를 시작하면서 가장 먼저 든 궁금증은 단순했다. 우리가 입력하는 문장을 컴퓨터는 도대체 어떤 형태로 받아들이는 걸까? 모델 구조나 학습 공식을 보기 전까지는 문장이 그대로 신경망 안으로 들어가는 것처럼 막연하게 생각했다. 하지만 이번 실습을 따라가 보니, 실제로는 텍스트를 잘게 나누고 숫자로 바꾼 뒤, 다시 문제와 정답 형태로 만드는 준비 과정이 먼저 필요했다.

이번 글에서는 강의의 `01_Embedding.ipynb`를 따라 실습하며 배운 내용을 정리한다. 범위는 다음 네 단계다.

```text
텍스트
  → 토큰화
  → Token ID 변환
  → 특수 토큰과 BPE
  → 슬라이딩 윈도우로 입력·정답 생성
```

## 1. 문장을 토큰으로 나누기

첫 단계는 토큰화(Tokenization)다. 토큰화란 긴 문자열을 모델이 처리할 수 있는 작은 단위로 나누는 과정이다.

처음에는 공백만 기준으로 문장을 나누면 충분할 것으로 생각했다.

```python
import re

text = "Hello, world. This, is a test."
result = re.split(r'(\s)', text)
print(result)
```

하지만 결과를 확인해 보니 `Hello,`와 `world.`처럼 문장부호가 단어에 붙어 있었다. 이렇게 처리하면 `Hello`와 `Hello,`가 서로 다른 토큰이 될 수 있다. 그래서 쉼표와 마침표도 분리 기준에 추가했다.

```python
result = re.split(r'([,.]|\s)', text)
result = [item for item in result if item.strip()]

print(result)
# ['Hello', ',', 'world', '.', 'This', ',', 'is', 'a', 'test', '.']
```

이 부분을 실습하면서 토큰이 반드시 단어 하나와 같지는 않다는 점을 배웠다. 문장부호도 하나의 토큰이 될 수 있고, 뒤에서 살펴본 BPE에서는 단어의 일부도 토큰이 될 수 있었다.

실제 강의에서는 쉼표와 마침표뿐 아니라 여러 특수기호도 분리했다.

```python
preprocessed = re.split(r'([,.:;?_!"()\']|--|\s)', raw_text)
preprocessed = [item.strip() for item in preprocessed if item.strip()]
```

정규표현식이 처음에는 복잡해 보였지만, 역할은 명확했다. 문장에서 단어와 문장부호를 분리하고, 그 과정에서 생긴 공백이나 빈 문자열을 제거하는 코드다.

## 2. 토큰에 번호표 붙이기

문자열 형태의 토큰은 아직 모델이 계산할 수 없다. 각 토큰에 고유한 정수 번호를 붙여야 한다. 이 번호가 Token ID다.

먼저 텍스트에 등장한 토큰의 중복을 제거하고 정렬한다.

```python
all_words = sorted(set(preprocessed))
vocab_size = len(all_words)
```

그다음 `enumerate()`를 사용해 각 토큰에 0부터 번호를 부여한다.

```python
vocab = {
    token: integer
    for integer, token in enumerate(all_words)
}
```

예를 들어 어휘사전은 다음과 같은 형태가 된다.

```python
{
    ',': 0,
    '.': 1,
    'Hello': 2,
    'world': 3
}
```

여기서 한 번 짚고 넘어갈 점이 있다. Token ID는 계산 결과가 아니라 단순한 번호표다. `world`의 ID가 3이고 `Hello`의 ID가 2라고 해서 `world`가 더 크거나 중요하다는 뜻은 아니다.

강의에서는 이 변환을 `SimpleTokenizerV1` 클래스로 묶었다.

```python
class SimpleTokenizerV1:
    def __init__(self, vocab):
        self.str_to_int = vocab
        self.int_to_str = {i: s for s, i in vocab.items()}

    def encode(self, text):
        tokens = re.split(r'([,.:;?_!"()\']|--|\s)', text)
        tokens = [item.strip() for item in tokens if item.strip()]
        return [self.str_to_int[token] for token in tokens]

    def decode(self, ids):
        text = " ".join(self.int_to_str[i] for i in ids)
        return re.sub(r'\s+([,.?!"()\'])', r'\1', text)
```

`encode()`는 텍스트를 Token ID로 바꾸고, `decode()`는 Token ID를 다시 텍스트로 복원한다. 직접 구현해 보니 토크나이저가 거창한 블랙박스라기보다, 두 방향의 변환 규칙을 가진 도구라는 것이 눈에 들어왔다.

## 3. 사전에 없는 단어는 어떻게 처리할까

`SimpleTokenizerV1`에는 문제가 있었다. 학습용 텍스트에 없던 단어를 입력하면 사전에서 ID를 찾을 수 없어 오류가 발생했다.

```text
KeyError: 'Hello'
```

이를 해결하기 위해 두 개의 특수 토큰을 추가했다.

```python
all_tokens.extend(["<|endoftext|>", "<|unk|>"])
```

- `<|unk|>`는 어휘사전에 없는 토큰을 대신한다.
- `<|endoftext|>`는 한 문서가 끝나고 다른 문서가 시작되는 경계를 표시한다.

두 번째 토크나이저에서는 사전에 없는 단어를 다음과 같이 처리했다.

```python
tokens = [
    token if token in self.str_to_int else "<|unk|>"
    for token in tokens
]
```

이 방식으로 오류는 막을 수 있지만 정보가 사라지는 문제가 남는다. `pineapple`과 `smartphone`이 모두 사전에 없다면 둘 다 똑같은 `<|unk|>`로 변환되기 때문이다.

## 4. BPE는 모르는 단어도 조각내서 읽는다

GPT-2는 Byte Pair Encoding(BPE) 방식의 토크나이저를 사용한다. 이번 실습에서는 `tiktoken`으로 GPT-2 토크나이저를 불러왔다.

```python
import tiktoken

tokenizer = tiktoken.get_encoding("gpt2")
```

BPE의 핵심은 모르는 단어를 통째로 `<|unk|>`로 바꾸지 않는다는 것이다. 어휘사전에 있는 더 작은 부분 단어나 문자 조각으로 분해한다.

```text
someunknownPlace
  → some + unknown + Place
```

실제 분할 결과는 토크나이저가 가진 어휘사전에 따라 달라지지만, 핵심 원리는 같다. 처음 보는 단어도 이미 알고 있는 작은 조각의 조합으로 표현한다. 이 부분에서 LLM이 신조어나 긴 합성어를 어느 정도 처리할 수 있는 이유를 이해할 수 있었다.

특수 토큰을 포함한 인코딩은 다음처럼 수행했다.

```python
token_ids = tokenizer.encode(
    text,
    allowed_special={"<|endoftext|>"}
)

restored_text = tokenizer.decode(token_ids)
```

## 5. GPT의 문제와 정답 만들기

텍스트를 Token ID로 바꾼 뒤에는 GPT가 학습할 문제와 정답을 만들어야 한다. GPT의 기본 학습 목표는 앞의 토큰을 보고 다음 토큰을 예측하는 것이다.

강의에서는 문맥 길이를 4로 설정하고 입력과 정답을 다음처럼 만들었다.

```python
context_size = 4

x = enc_sample[:context_size]
y = enc_sample[1:context_size + 1]
```

구조를 단순하게 표시하면 다음과 같다.

```text
전체: [A, B, C, D, E]
입력: [A, B, C, D]
정답:    [B, C, D, E]
```

정답은 입력보다 정확히 한 칸 오른쪽으로 이동한다. 따라서 하나의 데이터 묶음에서 다음 네 가지 관계를 학습할 수 있다.

```text
A       → B
A B     → C
A B C   → D
A B C D → E
```

예를 들어 문장이 `나는 오늘 학교에 갔다`라면 다음과 같은 문제를 푸는 셈이다.

```text
나는              → 오늘
나는 오늘         → 학교에
나는 오늘 학교에  → 갔다
```

이 부분을 통해 별도의 사람이 정답을 붙이지 않아도 원문을 한 칸 이동하는 것만으로 학습용 정답을 만들 수 있다는 점을 배웠다.

## 6. 긴 문장을 자르는 슬라이딩 윈도우

실제 텍스트는 매우 길지만 모델은 한 번에 정해진 길이만 처리할 수 있다. 그래서 일정한 크기의 창을 텍스트 위에서 이동시키며 여러 학습 샘플을 만든다. 이것이 슬라이딩 윈도우 방식이다.

```text
전체 토큰: [A B C D E F G H I]

첫 번째 입력: [A B C D]
두 번째 입력:   [B C D E]
세 번째 입력:     [C D E F]
```

강의의 `GPTDatasetV1`은 이 작업을 자동으로 수행한다.

```python
for i in range(0, len(token_ids) - max_length, stride):
    input_chunk = token_ids[i:i + max_length]
    target_chunk = token_ids[i + 1:i + max_length + 1]

    self.input_ids.append(torch.tensor(input_chunk))
    self.target_ids.append(torch.tensor(target_chunk))
```

여기서 `max_length`는 한 샘플에 들어가는 토큰 수이고, `stride`는 창을 몇 칸씩 이동할지를 뜻한다.

### stride가 1인 경우

```text
[A B C D]
  [B C D E]
    [C D E F]
```

학습 샘플을 많이 만들 수 있지만 내용이 상당히 겹친다.

### stride가 max_length와 같은 경우

```text
[A B C D]
        [E F G H]
```

샘플 간 중복은 줄지만 만들어지는 데이터 수가 적어진다. 작은 데이터에서 지나친 중복은 모델이 내용을 암기하는 과대적합으로 이어질 수 있기 때문에, 데이터의 크기와 학습 목적에 맞게 조절해야 한다.

## 7. 여러 샘플을 배치로 묶기

마지막으로 PyTorch의 `DataLoader`를 이용해 여러 샘플을 한 번에 묶는다.

```python
dataloader = create_dataloader_v1(
    raw_text,
    batch_size=8,
    max_length=4,
    stride=4,
    shuffle=False
)
```

이 설정은 다음 의미를 가진다.

- 한 번에 8개의 학습 샘플을 처리한다.
- 각 샘플은 4개의 토큰으로 구성된다.
- 슬라이딩 윈도우는 4칸씩 이동한다.
- 확인을 위해 데이터 순서를 섞지 않는다.

입력 텐서의 크기가 `[8, 4]`라면 4개의 토큰으로 이루어진 샘플 8개가 한 배치에 들어 있다는 뜻이다. 정답 텐서도 같은 크기를 가지지만 각 행의 내용은 입력보다 한 토큰 앞서 있다.

## 마무리

이번 실습 전에는 LLM 학습을 신경망 안에서 벌어지는 복잡한 계산으로만 생각했다. 그런데 그 전에 텍스트를 어떤 단위로 나누고, 모르는 단어를 어떻게 처리하며, 긴 문장에서 문제와 정답을 어떻게 뽑는지가 모델 학습의 출발점이었다.

특히 기억에 남은 내용은 세 가지다.

1. Token ID는 의미의 크기가 아니라 토큰을 구별하는 번호표다.
2. BPE는 처음 보는 단어를 작은 조각으로 나누어 정보 손실을 줄인다.
3. GPT의 정답은 입력 토큰을 한 칸 오른쪽으로 이동해 만들 수 있다.

여기까지는 모델에 넣을 데이터를 준비한 과정이다. 다음 단계에서는 Token ID를 학습 가능한 벡터로 바꾸는 Token Embedding과, 같은 단어라도 문장 속 위치를 구별하게 해주는 Position Embedding을 살펴볼 예정이다.
