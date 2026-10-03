# 100LOG_FORK — 씬리더용 서사연속성 모듈 설계서

> 이 저장소는 원본 100LOG의 문제의식과 정보 처리 방식을 참고하여, 씬리더(Scene Reader Hub)와 연동 가능한 **서사연속성 판독 확장**을 새로 설계하기 위한 작업 저장소입니다.
>
> 원본 코드를 단순 복제하거나 100LOG를 그대로 씬리더에 합치는 프로젝트가 아닙니다. 제작자의 허락을 받은 범위 안에서 원본의 좋은 정보 처리 원리와 연속성 카테고리를 참고하되, 읽기 기준·모델 운용·저장·주입·프롬프트 조립·UI는 씬리더 구조에 맞게 새로 작성합니다.

---

## 0. 원본 및 명칭

### 원본 100LOG

- 원본 저장소: https://github.com/foreverharibo-boop/100log.git
- 원본 성격: 최근 RP의 중요한 사실을 기억하고, 설정 오류가 있는 AI 답변을 사용자에게 보여주기 전에 검사하는 SillyTavern 확장
- 원본 핵심 흐름: 최근 visible 메시지 기반 기억 수집 → 규칙 저장 → 답변 공개 전 검수 → 필요 시 재작성

### 본 연동 확장의 고정 명칭

- 저장소명: `100LOG_FORK`
- 외부 연동 확장명: `100LOG_FORK`
- 사용자 표시명: `100로그 포크`
- 씬리더허브 내부 표기: `기존 공달 100LOG 포크 기능`

추후 씬리더허브에 새 탭을 추가할 때, 탭 이름은 씬리더 쪽에서 별도로 정할 수 있습니다. 예: `서사연속성`, `기억판독`, `스토리관리`.

다만 해당 탭 내부에는 이 기능의 출처와 계열을 명확히 하기 위해 다음 고정 문구를 표시합니다.

```text
기존 공달 100LOG 포크 기능
```

---

## 1. 핵심 목표

씬리더와 100LOG류 확장은 바라보는 방향이 다릅니다.

| 구분 | 역할 |
|---|---|
| 씬리더 | 다음 턴, 다음 장면, 미래 전개를 판독하는 확장 |
| 100LOG류 | 최근 과거 사건, 약속, 비밀, 정정, 인물별 지식 차이를 기억하는 확장 |
| 100LOG_FORK | 과거 사건을 현재 압력으로 해석하고, 다음 턴 조건으로 변환하는 씬리더 연동 확장 |

본 모듈은 단순 기억 저장소가 아닙니다.

```text
과거 사건 → 현재 유효 상태 / 장면 압력 → 다음 턴 제약
```

예시:

```text
과거 사건:
Lucas confessed love to Vivienne.

현재 압력:
Vivienne has not answered yet, so the confession is still unresolved.

다음 턴 제약:
Do not treat the confession as resolved, casual, or repaired unless Vivienne responds in-scene.
```

즉, 이 모듈은 로어북·요약기·장기 기억 확장이 아니라 **스토리 진행 상태를 씬리더가 사용할 수 있는 제약으로 변환하는 판독 레이어**입니다.

---

## 2. 개발 원칙

### 2.1 초기 버전은 독립 확장으로 제작

초기 버전에서는 씬판독기 UI에 아무것도 추가하지 않습니다.

```text
100LOG_FORK 독립 UI
100LOG_FORK 독립 설정
100LOG_FORK 독립 저장소
100LOG_FORK 독립 주입/청소 기능
```

씬판독기 쪽에는 초기 버전에서 새 탭, 버튼, 패널을 추가하지 않습니다.

이유:

- 문제 발생 시 원인을 100LOG_FORK와 씬판독기 사이에서 분리하기 위함
- 기존 씬판독기의 자동/수동 실행, NSFW 판독, generation cycle, swipe/edit/delete 흐름을 건드리지 않기 위함
- 기능이 충분히 안정화된 뒤 씬판독기 탭으로 그대로 이식하기 위함

### 2.2 UI는 나중에 씬판독기 탭으로 옮길 수 있게 설계

초기 UI는 독립 확장 안에 만들지만, 나중에 씬판독기 탭으로 그대로 옮길 수 있게 구성합니다.

권장 UI 구조:

```text
100LOG_FORK / 100로그 포크
├─ 사용 ON/OFF
├─ 초기 베이스라인 판독
├─ 판독 범위 선택
├─ 저장된 서사연속성 기록
├─ 기록 수정/삭제/고정
├─ STORY_CONTINUITY 주입 설정
├─ 주입 위치 선택
├─ 주입 청소
└─ 마지막 작업/오류
```

추후 씬판독기에 탭을 추가할 경우, 이 UI를 거의 그대로 탭 내부에 이식합니다.

### 2.3 연동 기준은 설치 여부가 아니라 사용 ON/OFF 상태

씬판독기와 연동할 때, 기준은 `100LOG_FORK가 설치되어 있는가`가 아닙니다.

기준은 반드시 다음이어야 합니다.

```text
100LOG_FORK가 설치되어 있고,
100LOG_FORK 내부의 사용 ON/OFF가 ON인가?
```

동작 규칙:

| 상태 | 씬판독기 동작 |
|---|---|
| 100LOG_FORK 미설치 | 씬판독기 기본 서사연속성 사용 |
| 100LOG_FORK 설치됨 + 사용 OFF | 씬판독기 기본 서사연속성 사용 |
| 100LOG_FORK 설치됨 + 사용 ON | 씬판독기 기본 서사연속성을 100LOG_FORK 결과로 대체 |

즉, 100LOG_FORK가 설치되어 있어도 사용자가 `사용 안 함`을 누르면 씬판독기는 기존 자체 서사연속성 기능을 그대로 사용합니다.

### 2.4 대체 방식

씬판독기 기본 서사연속성과 100LOG_FORK 서사연속성은 동시에 같은 역할로 작동하지 않습니다.

```text
나쁨:
씬판독기 기본 서사연속성 + 100LOG_FORK STORY_CONTINUITY 동시 주입

좋음:
100LOG_FORK 사용 ON이면 100LOG_FORK가 해당 영역을 대체
100LOG_FORK 사용 OFF이면 씬판독기 기본 기능 사용
```

이 규칙은 중복 판독, 중복 주입, 서로 다른 기록 충돌을 막기 위한 핵심 원칙입니다.

---

## 3. 100LOG에서 참고할 것과 바꿀 것

### 3.1 참고할 것

- 최근 RP에서 어떤 정보를 연속성 관리 대상으로 삼는지
- 약속, 계획, 비밀, 은폐, 거짓말, 오해, 정정, 인물별 지식 차이의 카테고리
- visible 메시지만 읽는 방식
- cursor 기반 증분 처리
- message signature / journal 기반 수정·리롤 대응
- 기존 기록과 새 메시지를 분리해 비교하는 방식
- 완료·취소·갱신 판정 원칙
- 요약·하이드 후 중요한 미해결 조건만 유지하는 문제의식

### 3.2 씬리더식으로 바꿀 것

- 최근 100개 고정 운용 대신 `초기 베이스라인 + 매턴 delta` 구조 사용
- 모델 운용은 `확장 연결모델 후보 추출 → JEV 검증/정제 → 씬판독 delta 갱신` 구조로 변경
- 생성 가로채기, 숨은 초안, 자동 재작성은 기본 설계에서 제외
- 저장 범위는 현재 채팅 기준을 기본값으로 사용
- 주입 위치는 월드인포 후를 기본값으로 사용
- CardInjector식 프리셋 위치 지정 옵션 제공
- 주입 블록은 `<STORY_CONTINUITY>`로 통일
- append가 아니라 replace 방식으로 주입 찌꺼기 방지

---

## 4. 전체 운용 흐름

```text
1. 사용자가 100LOG_FORK 사용 ON

2. 초기 베이스라인 판독
   - 현재 채팅의 선택 범위를 넓게 읽음
   - 확장 연결모델이 서사연속성 후보 추출
   - JEV가 후보를 원문 근거 기준으로 검증/정제
   - 승인된 기록만 저장

3. 평소 RP 진행

4. 증분 판독
   - 씬판독기가 매턴 원래 읽는 범위 또는 100LOG_FORK가 설정한 범위 안에서 변화분 확인
   - 새 변화분만 continuity_delta로 추출
   - 기존 저장 기록을 add/update/resolve/cancel/supersede

5. 프롬프트 조립
   - 활성 기록 중 관련 높은 것만 선별
   - <STORY_CONTINUITY> 블록 생성
   - 기본 위치인 월드인포 후에 replace 삽입

6. 수정/리롤 감지
   - message signature 확인
   - 해당 지점 이후 journal rollback
   - 필요한 구간만 재판독
```

---

## 5. 읽기 기준

### 5.1 읽기 대상

읽기 대상은 실제 RP 원문으로 취급할 수 있는 visible 메시지입니다.

포함:

- 사용자 메시지
- AI 답변
- 화면에 보이는 RP 본문

제외:

- system 메시지
- hidden 메시지
- 빈 메시지
- 확장 주입문
- 내부 프롬프트
- 명확히 분리된 비스토리 OOC 설정문

### 5.2 초기 베이스라인 판독

초기 판독은 자동으로 무조건 실행하지 않고, 사용자가 직접 실행합니다.

권장 UI:

```text
초기 판독 범위
● 현재 채팅 최근 20턴
○ 현재 채팅 최근 30턴
○ 현재 채팅 전체
○ 같은 캐릭터의 최근 열린 채팅 포함
○ 직접 선택한 채팅만 포함
```

기본값은 `현재 채팅 최근 20턴`입니다.

`같은 캐릭터의 최근 열린 채팅 포함`은 고급 옵션으로만 제공합니다. 같은 캐릭터 카드라도 다른 세계선이나 IF 루트일 수 있기 때문입니다.

### 5.3 이후 증분 판독

초기화 이후에는 전체를 다시 읽지 않습니다.

```text
직전 cursor 이후 새 visible 메시지
+
씬판독기가 원래 읽는 최근 범위
+
기존 활성 서사연속성 기록
```

모델은 전체 기억을 다시 작성하지 않고, 새 변화만 반환합니다.

```text
add       새 기록 추가
update    기존 기록 수정
resolve   미해결 기록 완료
cancel    계획/약속 취소
supersede 기존 기록을 새 상태로 대체
ignore    저장할 변화 없음
```

---

## 6. 모델 운용

### 6.1 초기 1차 판독: 확장 연결모델

초기 베이스라인 판독에서는 확장에 연결된 모델이 넓은 범위를 읽고 후보를 뽑습니다.

역할:

```text
최근 RP에서 다음 턴 또는 최근 장면 흐름을 깨뜨릴 수 있는 서사연속성 후보를 찾는다.
단순 요약하지 않는다.
각 후보는 과거 사건, 현재 압력, 다음 턴 제약, 인물별 지식 경계, 원문 근거를 포함한다.
```

### 6.2 초기 2차 판독: JEV 검증

JEV는 후보를 새로 창작하는 모델이 아니라, 후보의 근거와 상태를 검증하는 역할로 사용합니다.

검증 기준:

- 원문 근거가 있는가
- 해석이 과하지 않은가
- 이미 해결된 일을 active로 잡지 않았는가
- 인물별 지식 상태가 실제 정보 흐름에 맞는가
- 다음 턴 제약으로 저장할 가치가 있는가

### 6.3 매턴 증분 판독

매턴에는 무거운 재판독을 하지 않습니다.

```text
이번 새 메시지에서 기존 서사연속성 기록에 영향을 줄 변화가 있는지 찾는다.
전체 기억을 다시 쓰지 않는다.
새 add/update/resolve/cancel만 반환한다.
```

출력 제한:

```text
기본 최대 3개
중요 장면 최대 5개
없으면 빈 배열
```

---

## 7. 제외 기능

### 7.1 생성 가로채기 제외

초기 버전에서는 100LOG식 생성 가로채기를 구현하지 않습니다.

제외 대상:

- `generate_interceptor` 기반 생성 중단
- 숨은 초안 생성
- 답변 공개 전 전체 검수
- 충돌 감지 시 자동 재작성
- 재검수 후 게시

이유:

- 씬리더허브의 기존 판독/주입 구조와 충돌 가능
- 다른 확장과 generate interceptor 충돌 가능
- 답변 속도 저하
- JEV 비용 증가
- 씬리더의 역할이 사전 예방에서 사후 검수/재작성으로 흐려질 위험

### 7.2 자동 재작성 제외

본 모듈은 답변 생성 후 틀린 답변을 폐기하고 다시 쓰게 하는 구조가 아닙니다.

기본 방식:

```text
서사연속성 저장
→ STORY_CONTINUITY 조립
→ 생성 전 프롬프트에 주입
→ 처음부터 틀릴 가능성을 줄임
```

향후 필요하면 고급 옵션으로만 검토합니다.

```text
고급 검수 모드 후보
○ 사용 안 함
○ 충돌 경고만 표시
○ 충돌 기록만 남김
○ 자동 재작성 요청
```

기본값은 `사용 안 함`입니다.

---

## 8. 판독 카테고리

v1 기본 카테고리입니다.

| type | 의미 |
|---|---|
| `pending_commitment` | 아직 실행·취소되지 않은 약속, 계획, 일정 |
| `concealed_truth` | 비밀, 은폐, 거짓말, 숨겨진 사실 |
| `knowledge_boundary` | 누가 무엇을 알고/모르는지에 대한 경계 |
| `user_correction` | 사용자가 직접 정정한 설정·사건·용어 |
| `misunderstanding` | 오해, 착각, 잘못 믿고 있는 사실 |
| `unresolved_tension` | 아직 해소되지 않은 감정적/실무적 갈등 |
| `relationship_shift` | 최근 관계 상태의 의미 있는 변화 |
| `causal_link` | 다음 반응에 필요한 최근 원인-결과 연결 |
| `recent_event` | 다음 턴에 직접 영향이 있는 최근 사건 |
| `state_change` | 현재 장면 또는 관계의 유효 상태 변화 |

---

## 9. 기록 스키마 초안

```json
{
  "id": "srsc_001",
  "type": "unresolved_tension",
  "status": "active",
  "priority": 5,
  "scope": "next_turn",
  "target": ["Lucas", "Vivienne"],
  "past_event": "Lucas confessed love to Vivienne.",
  "current_pressure": "Vivienne has not answered yet, so the confession remains emotionally unresolved.",
  "next_turn_constraint": "Do not treat the confession as resolved, casual, or repaired unless Vivienne responds in-scene.",
  "knowledge": {
    "Lucas": "known",
    "Vivienne": "known",
    "Dominic": "unknown"
  },
  "evidence": [
    {
      "message_id": 147,
      "quote": "Love."
    }
  ],
  "expires_when": "resolved_in_scene",
  "source": "baseline",
  "created_at": 0,
  "updated_at": 0
}
```

중요 필드:

| field | 의미 |
|---|---|
| `past_event` | 과거에 실제로 발생한 근거 사건 |
| `current_pressure` | 현재 장면에 남아 있는 압력 또는 유효 상태 |
| `next_turn_constraint` | 다음 답변이 깨뜨리면 안 되는 조건 |
| `knowledge` | 인물별 지식 상태 |
| `evidence` | 원문 근거 |
| `expires_when` | 완료/취소/이월 조건 |

---

## 10. 지식 상태

v1 기본값:

```text
known
unknown
partial
```

확장 후보:

```text
suspects
misunderstands
believes_false
withheld_from
```

원칙:

- 이름이 언급되었다고 해서 `known` 처리하지 않음
- 장면에 없었다는 이유만으로 `unknown` 처리하지 않음
- 실제 전달, 목격, 기록 접근, 감시, 통신, 폭로 등 정보 흐름이 있어야 `known`
- 비밀 유지, 미전달, 은폐, 실패한 전달, 명시적 무지가 있어야 `unknown`
- 일부만 알면 `partial`

---

## 11. 완료 / 취소 / 갱신 원칙

- 시간이 지났다고 완료 처리하지 않음
- 언급이 없어졌다고 취소 처리하지 않음
- 비슷한 분위기라고 해결 처리하지 않음
- 실제 약속한 장소 도착, 행동 수행, 명시적 답변, 폭로, 합의가 있어야 완료 가능
- 명시적 취소, 거절, 파기, 철회가 있어야 취소 가능
- 새 정보가 기존 기록을 대체하면 supersede 처리
- 불명확하면 active 상태를 유지하고 재판독 대상으로 둠

---

## 12. 주입 설계

### 12.1 기본 주입 위치

기본 주입 위치는 **월드인포 / 로어북 뒤**입니다.

```text
World Info / Lorebook
↓
<STORY_CONTINUITY>
↓
Scene Reader 판독 블록
↓
응답 지시 / 출력 형식
```

이 위치를 기본으로 잡는 이유:

1. 로어북은 고정 설정을 제공함
2. 서사연속성은 그 설정 위에서 최근 RP 때문에 생긴 유효 상태를 얹음
3. 씬리더 판독은 이 상태를 바탕으로 다음 턴을 판단함

### 12.2 CardInjector식 위치 지정

고급 사용자를 위해 프리셋 위치 지정 옵션을 둡니다.

예정 옵션:

- 월드인포 후
- 캐릭터 정의 후
- 작가노트 전
- 작가노트 후
- 메인 프롬프트 전
- 마지막 유저 메시지 전
- 사용자 지정 프리셋 위치

기본은 월드인포 후입니다.

### 12.3 replace 방식

서사연속성 블록은 매번 새로 덧붙이지 않습니다.

```text
기존 <STORY_CONTINUITY>...</STORY_CONTINUITY> 블록이 있으면 교체
없으면 지정된 위치에 삽입
```

이 방식으로 중복 주입과 주입 찌꺼기를 방지합니다.

### 12.4 주입 청소

청소 버튼:

```text
[서사연속성 주입 청소]
```

청소 대상:

- `<STORY_CONTINUITY>...</STORY_CONTINUITY>`
- 구버전 `<MEMORY_CONTINUITY>...</MEMORY_CONTINUITY>`
- 구버전 `<SCENE_MEMORY>...</SCENE_MEMORY>`
- 기타 본 확장이 만든 주입 블록

청소 버튼은 주입 찌꺼기만 지우며, 저장된 판독 기록은 삭제하지 않습니다.

---

## 13. 주입 블록 예시

```text
<STORY_CONTINUITY>
Recent story-state constraints. Use these after lore and before deciding the next scene.
They are not permanent lore, dialogue, or a full summary.

- [unresolved_tension] Lucas confessed love to Vivienne; she has not answered.
  Constraint: Do not treat the confession as resolved or casual.

- [knowledge_boundary] Dominic does not know the exact private confession.
  Constraint: Dominic may react only to information he has actually witnessed or received.

- [concealed_truth] Vivienne's Omega status remains dangerous if exposed to Wade.
  Constraint: Preserve concealment pressure in family-facing scenes.
</STORY_CONTINUITY>
```

주입 원칙:

- 저장 기록 전체를 넣지 않음
- 활성 기록 중 현재 턴과 관련 높은 항목만 선별
- 기본 최대 5개
- 정밀 모드 최대 8개
- 수동 전체 주입 최대 12개

---

## 14. 씬판독기 연동 API 초안

초기 버전은 독립 UI로 운용하지만, 나중에 씬판독기가 선택적으로 읽을 수 있도록 window API를 제공합니다.

```js
window.SceneReaderStoryContinuity = {
  isAvailable() {},
  isEnabled() {},
  getStatus() {},
  getRecords() {},
  getInjectionBlock() {},
  runBaseline() {},
  runDelta() {},
  clearInjection() {}
};
```

씬판독기 쪽 연동 판단 예시:

```js
const external = window.SceneReaderStoryContinuity;
const useFork = Boolean(external?.isAvailable?.() && external?.isEnabled?.());

if (useFork) {
  // 100LOG_FORK가 켜져 있으므로 씬판독기 기본 서사연속성을 대체한다.
  const storyContinuityBlock = external.getInjectionBlock();
} else {
  // 미설치 또는 설치되어 있어도 사용 OFF이므로 씬판독기 기본 서사연속성을 사용한다.
  const storyContinuityBlock = sceneReaderDefaultStoryContinuity();
}
```

핵심은 `isEnabled()`입니다. 설치되어 있어도 사용자가 100LOG_FORK를 사용 안 함으로 둔 경우에는 대체하지 않습니다.

---

## 15. 저장 범위

기본값:

```text
현재 채팅 단위 저장
```

고급 옵션 후보:

```text
같은 캐릭터 카드 공유
페르소나 + 캐릭터 조합 공유
직접 선택한 채팅 포함
```

기본을 현재 채팅으로 두는 이유:

- 같은 캐릭터 카드라도 다른 세계선/IF/테스트 채팅이 있을 수 있음
- 잘못 섞이면 서사연속성 기록이 오염됨
- 초기 안정성을 우선해야 함

---

## 16. 구현 단계

### Phase 0. 설계 문서

- 원본 100LOG 참고 범위 정리
- 씬리더식 서사연속성 구조 정의
- 독립 UI / 추후 탭 이식 가능 구조 확정

### Phase 1. 100LOG_FORK 독립 확장

- 씬판독기 UI 수정 없음
- 자체 UI에서 초기 판독/저장/주입/청소 제공
- 생성 가로채기 없음
- 자동 재작성 없음
- 사용 ON/OFF 상태 제공

### Phase 2. 선택 연동

- window API 제공
- 씬판독기는 `isAvailable()`와 `isEnabled()`를 확인
- 100LOG_FORK 사용 ON이면 씬판독기 기본 서사연속성을 대체
- 100LOG_FORK 사용 OFF이면 씬판독기 기본 서사연속성 유지

### Phase 3. 씬판독기 UI 병합 검토

- 충분히 안정화된 뒤 탭 추가
- 탭 이름은 씬리더 쪽에서 별도 결정
- 탭 내부에는 `기존 공달 100LOG 포크 기능` 고정 표기
- 독립 확장 UI를 거의 그대로 이식

### Phase 4. 완전 통합 검토

- 저장소/주입/설정 이전 여부 검토
- 외부 확장 없이 씬판독기 내부 기능으로 포함할지 결정

---

## 17. 최종 원칙

```text
읽기 추적과 정보 정리 원리는 100LOG에서 참고한다.
모델 운용, 판독 목적, 저장 범위, 주입 위치, 프롬프트 조립, UI는 씬리더식으로 바꾼다.
초기에는 독립 확장으로 충분히 검증한다.
씬판독기와 연동할 때는 설치 여부가 아니라 사용 ON/OFF 상태를 기준으로 한다.
100LOG_FORK 사용 ON이면 씬판독기 기본 서사연속성을 대체한다.
100LOG_FORK 사용 OFF이면 씬판독기 기본 서사연속성을 그대로 사용한다.
```
