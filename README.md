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

## 1. 목표

씬리더와 100LOG류 확장은 바라보는 방향이 다릅니다.

| 구분 | 역할 |
|---|---|
| 씬리더 | 다음 턴, 다음 장면, 미래 전개를 판독하는 확장 |
| 100LOG류 | 최근 과거 사건, 약속, 비밀, 정정, 인물별 지식 차이를 기억하는 확장 |
| 100LOG_FORK | 과거 사건을 현재 압력으로 해석하고, 다음 턴 조건으로 변환하는 씬리더 연동 모듈 |

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

## 2. 참고할 것과 바꿀 것

### 2.1 100LOG에서 참고할 것

- 최근 RP에서 어떤 정보를 연속성 관리 대상으로 삼는지
- 약속, 계획, 비밀, 은폐, 거짓말, 오해, 정정, 인물별 지식 차이의 카테고리
- visible 메시지만 읽는 방식
- cursor 기반 증분 처리
- message signature / journal 기반 수정·리롤 대응
- 기존 기록과 새 메시지를 분리해 비교하는 방식
- 완료·취소·갱신 판정 원칙
- 요약·하이드 후 중요한 미해결 조건만 유지하는 문제의식

### 2.2 씬리더식으로 바꿀 것

- 읽기 범위는 100개 고정이 아니라 씬리더 설정과 초기 판독 범위 기준으로 운용
- 모델 운용은 `확장 연결모델 후보 추출 → JEV 검증/정제 → 씬판독 delta 갱신` 구조로 변경
- 생성 가로채기, 숨은 초안, 자동 재작성은 기본 설계에서 제외
- 저장 범위는 현재 채팅 기준을 기본값으로 사용
- 주입 위치는 월드인포 후를 기본값으로 사용
- CardInjector식 프리셋 위치 지정 옵션 제공
- 주입 블록은 `<STORY_CONTINUITY>`로 통일
- append가 아니라 replace 방식으로 주입 찌꺼기 방지
- 저장된 기록 전체를 매번 주입하지 않고, 현재 턴에 필요한 기록만 짧게 선별 주입

---

## 3. 전체 운용 흐름

```text
1. 사용자가 100LOG_FORK / 서사연속성 기능 ON

2. 초기 베이스라인 판독
   - 현재 채팅의 선택 범위를 넓게 읽음
   - 확장 연결모델이 서사연속성 후보 추출
   - JEV가 후보를 원문 근거 기준으로 검증/정제
   - 승인된 기록만 저장

3. 평소 RP 진행

4. 씬판독 실행 시 증분 판독
   - 씬판독기가 매턴 원래 읽는 범위만 읽음
   - 새 변화분만 continuity_delta로 추출
   - 기존 저장 기록을 add/update/resolve/cancel/supersede

5. 프롬프트 조립
   - 저장된 기록 전체가 아니라, 현재 턴에 필요한 활성 기록만 선별
   - 선별된 기록을 짧은 제약 목록으로 압축
   - <STORY_CONTINUITY> 블록 생성
   - 기본 위치인 월드인포 후에 replace 삽입

6. 수정/리롤 감지
   - message signature 확인
   - 해당 지점 이후 journal rollback
   - 필요한 구간만 재판독
```

---

## 4. 읽기 기준

### 4.1 읽기 대상

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

### 4.2 초기 베이스라인 판독

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

이유:

- 최근 흐름을 충분히 잡음
- 모델/JEV 비용을 과하게 쓰지 않음
- 다른 세계선이나 테스트 채팅이 섞일 위험을 줄임

`같은 캐릭터의 최근 열린 채팅 포함`은 고급 옵션으로만 제공합니다. 같은 캐릭터 카드라도 다른 세계선이나 IF 루트일 수 있기 때문입니다.

### 4.3 이후 증분 판독

초기화 이후에는 전체를 다시 읽지 않습니다.

```text
씬판독기가 원래 읽는 최근 범위
+
직전 cursor 이후 새 visible 메시지
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

## 5. 모델 운용

### 5.1 초기 1차 판독: 확장 연결모델

초기 베이스라인 판독에서는 확장에 연결된 모델이 넓은 범위를 읽고 후보를 뽑습니다.

역할:

```text
최근 RP에서 다음 턴 또는 최근 장면 흐름을 깨뜨릴 수 있는 서사연속성 후보를 찾는다.
단순 요약하지 않는다.
각 후보는 과거 사건, 현재 압력, 다음 턴 제약, 인물별 지식 경계, 원문 근거를 포함한다.
```

### 5.2 초기 2차 판독: JEV 검증

JEV는 후보를 새로 창작하는 모델이 아니라, 후보의 근거와 상태를 검증하는 역할로 사용합니다.

검증 기준:

- 원문 근거가 있는가
- 해석이 과하지 않은가
- 이미 해결된 일을 active로 잡지 않았는가
- 인물별 지식 상태가 실제 정보 흐름에 맞는가
- 다음 턴 제약으로 저장할 가치가 있는가

### 5.3 매턴 증분 판독: 씬판독 결과에 포함

씬판독기가 매턴 읽는 입력에 가벼운 `story_continuity_delta` 섹션을 포함합니다.

역할:

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

## 6. 제외 기능

### 6.1 생성 가로채기 제외

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

### 6.2 자동 재작성 제외

본 모듈은 답변 생성 후 틀린 답변을 폐기하고 다시 쓰게 하는 구조가 아닙니다.

기본 방식:

```text
서사연속성 저장
→ 현재 턴에 필요한 기록만 선별
→ 짧은 STORY_CONTINUITY 조립
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

## 7. 판독 카테고리

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

중요한 점은 단순 사건 요약이 아니라, 다음 턴에서 깨지면 안 되는 조건만 추출한다는 것입니다.

---

## 8. 저장 구조

저장은 상세하게 하되, 주입은 짧게 합니다.

저장 기록 예시:

```json
{
  "id": "srmem_001",
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
  "source": "auto",
  "created_at": 0,
  "updated_at": 0
}
```

핵심 필드:

| field | 의미 |
|---|---|
| `type` | 판독 카테고리 |
| `status` | active, pending, resolved, cancelled, superseded 등 |
| `priority` | 주입 우선순위 |
| `scope` | next_turn, current_scene, recent_arc, carryover 등 |
| `target` | 관련 인물 또는 대상 |
| `past_event` | 과거에 발생한 근거 사건 |
| `current_pressure` | 현재 장면에 남은 압력 또는 유효 상태 |
| `next_turn_constraint` | 다음 턴에서 깨지면 안 되는 조건 |
| `knowledge` | 인물별 지식 상태 |
| `evidence` | 원문 근거 |
| `expires_when` | 만료 조건 |
| `source` | baseline, delta, manual 등 |

---

## 9. 주입 원칙: 저장 전체가 아니라 현재 턴 관련 기록만

100LOG_FORK는 저장소에 많은 기록을 보관할 수 있지만, 실제 프롬프트에 매번 저장된 전체를 넣지 않습니다.

주입 원칙:

```text
저장은 충분히 자세하게 한다.
주입은 현재 턴에 필요한 것만 짧게 한다.
```

즉, 씬판독기처럼 그때 씬에 필요한 내용만 선별합니다.

### 9.1 선별 기준

주입 후보는 다음 기준으로 고릅니다.

1. `status`가 `active` 또는 `pending`인 기록
2. 현재 씬판독 범위에 등장하거나 언급된 인물과 관련 있는 기록
3. `priority`가 높은 기록
4. `scope`가 `next_turn` 또는 `current_scene`인 기록
5. 지식 차이, 비밀, 미해결 약속, 미해결 갈등처럼 다음 턴 오류를 직접 막는 기록
6. 최근 delta에서 새로 추가·갱신된 기록

낮은 우선순위의 과거 기록, 현재 장면과 무관한 기록, 이미 해결된 기록은 저장소에 남아 있어도 주입하지 않습니다.

### 9.2 주입량 설정

주입량은 씬리더 설정과 공유하지 않고, 100LOG_FORK 자체 설정으로 관리합니다.

권장 기본값:

```text
주입 최대 개수: 5개
중요 장면 최대 개수: 8개
수동 전체 점검 모드: 최대 12개
```

권장 UI:

```text
주입량
● 짧게: 최대 3개
○ 기본: 최대 5개
○ 자세히: 최대 8개
○ 수동 점검: 최대 12개
```

`짧게`와 `기본`을 배포 기본값으로 권장합니다.

### 9.3 주입문 형식

실제 주입문은 긴 줄글 설명이 아니라, 모델이 알아듣기 쉬운 짧은 제약 목록입니다.

나쁜 예:

```text
Lucas confessed love to Vivienne in the previous scene, and this was an important emotional moment because Vivienne had never received such a direct confession before. Dominic was not present at the time and therefore should not know about the exact wording of the confession unless he learns it later in the scene...
```

좋은 예:

```text
<STORY_CONTINUITY>
Recent story constraints. Use only for the next reply.
- [unresolved_tension] Lucas confessed love; Vivienne has not answered. Do not treat it as resolved.
- [knowledge_boundary] Dominic did not hear the confession. Do not let him reference its exact wording.
- [concealed_truth] Vivienne's Omega status remains dangerous if exposed to Wade. Preserve concealment pressure.
</STORY_CONTINUITY>
```

### 9.4 주입문 압축 규칙

- 한 기록은 가능하면 한 줄로 압축
- `past_event`, `current_pressure`, `next_turn_constraint`를 모두 줄글로 넣지 않음
- 실제 주입에는 `next_turn_constraint` 중심으로 넣음
- 필요할 때만 아주 짧은 원인 정보를 앞에 붙임
- evidence quote는 기본 주입에 넣지 않음
- JSON 전체를 그대로 주입하지 않음
- 저장된 모든 knowledge map을 그대로 넣지 않고, 현재 턴에 필요한 지식 경계만 문장화

예시:

```text
저장 기록:
Lucas confessed love to Vivienne; Vivienne has not answered; Dominic unknown; evidence quote exists.

주입문:
- [unresolved_tension] Lucas confessed love; Vivienne has not answered. Do not treat it as resolved.
- [knowledge_boundary] Dominic did not hear the confession. Do not let him know exact wording.
```

---

## 10. 주입 위치 설계

### 10.1 기본 위치

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
2. STORY_CONTINUITY는 그 설정 위에서 최근 RP 때문에 생긴 유효 상태를 얹음
3. 씬리더 판독은 이 상태를 바탕으로 다음 턴을 판단함

즉 순서는 다음과 같습니다.

```text
고정 설정 → 최근 서사 상태 → 다음 턴 판독
```

### 10.2 Depth 주입을 기본으로 쓰지 않는 이유

서사연속성을 너무 깊은 위치에 넣으면 다음 문제가 생길 수 있습니다.

- 과거 사건이 현재 장면보다 과하게 우선됨
- 이미 해결 가능한 감정도 계속 고정됨
- 장면 전환이 둔해짐
- 모델이 기억 규칙을 대사처럼 반복할 수 있음
- 씬리더의 미래 판독보다 기억이 더 강하게 먹을 수 있음

따라서 기본값은 월드인포 후로 하고, depth 또는 다른 프리셋 위치는 고급 옵션으로만 제공합니다.

### 10.3 CardInjector식 위치 지정

고급 사용자를 위해 CardInjector 방식의 프리셋 위치 지정도 지원합니다.

예정 옵션:

- 월드인포 후
- 캐릭터 정의 후
- 작가노트 전
- 작가노트 후
- 메인 프롬프트 전
- 마지막 유저 메시지 전
- 사용자 지정 프리셋 위치

---

## 11. 삽입 방식: append가 아니라 replace

서사연속성 블록은 매번 새로 덧붙이는 방식이 아니라, 기존 블록을 찾아 교체하는 방식으로 운용합니다.

```text
기존 <STORY_CONTINUITY>...</STORY_CONTINUITY> 블록이 있으면 교체
없으면 지정된 위치에 삽입
```

이유:

- 이전 판독 결과가 찌꺼기로 남는 것을 방지
- 중복 주입 방지
- 확장 업데이트 후 구버전 블록을 안전하게 청소 가능
- 저장된 기록과 실제 프롬프트 주입 상태를 분리 가능

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

## 12. 씬리더 연동 방식

초기에는 100LOG_FORK 자체 UI로만 운용합니다. 씬리더 UI에는 아무 탭도 추가하지 않습니다.

다만 씬리더와 연동할 수 있도록 window API를 제공합니다.

예상 API:

```js
window.SceneReaderStoryContinuity = {
  isAvailable() {},
  isEnabled() {},
  getStatus() {},
  getRecords() {},
  getInjectionBlock(context) {},
  runBaseline(options) {},
  runDelta(context) {},
  clearInjection() {},
};
```

씬리더는 설치 여부만으로 100LOG_FORK를 사용하지 않습니다. 반드시 100LOG_FORK 내부 사용 상태를 확인합니다.

```js
const external = window.SceneReaderStoryContinuity;
const useFork = Boolean(external?.isAvailable?.() && external?.isEnabled?.());

if (useFork) {
  const storyContinuityBlock = external.getInjectionBlock(sceneContext);
} else {
  const storyContinuityBlock = sceneReaderDefaultStoryContinuity(sceneContext);
}
```

동작 기준:

```text
100LOG_FORK 미설치
→ 씬판독기 기본 서사연속성 사용

100LOG_FORK 설치됨 + 사용 OFF
→ 씬판독기 기본 서사연속성 사용

100LOG_FORK 설치됨 + 사용 ON
→ 씬판독기 기본 서사연속성을 100LOG_FORK 결과로 대체
```

즉, 대체 조건은 설치 여부가 아니라 **사용 ON/OFF 상태**입니다.

---

## 13. UI 설계

초기 버전에서는 100LOG_FORK 자체 UI에서 모든 기능을 제공합니다.

권장 UI:

```text
[100로그 포크]

□ 사용

초기 판독
- 범위 선택
- 연결모델 선택
- JEV 키/검증 상태
- [초기 베이스라인 판독]

저장 기록
- 활성 기록
- 미해결 약속/계획
- 인물별 지식 경계
- 지난 기록
- 수동 추가/수정/삭제/고정

주입
- 주입 사용 ON/OFF
- 주입량: 짧게 / 기본 / 자세히 / 수동 점검
- 주입 위치: 월드인포 후 / 사용자 지정
- [주입 청소]

연동
- 씬판독기 연동 상태
- 현재 100LOG_FORK 사용 여부
- getInjectionBlock 미리보기

진단
- 마지막 작업
- 마지막 오류
- 오류 복사
```

나중에 씬리더에 병합할 때는 이 UI 구조를 거의 그대로 탭으로 옮길 수 있게 만듭니다.

---

## 14. 개발 단계

### Phase 1. 독립 확장

- 씬판독기 UI 수정 없음
- 100LOG_FORK 자체 UI 제작
- 초기 베이스라인 판독
- 저장 기록 관리
- 짧은 STORY_CONTINUITY 주입
- 주입 위치 설정
- 주입 청소
- 생성 가로채기 없음
- 자동 재작성 없음

### Phase 2. 선택 연동

- window API 제공
- 씬판독기가 100LOG_FORK 사용 ON 상태일 때만 결과 사용
- 100LOG_FORK가 사용 OFF면 씬판독기 기본 서사연속성 유지

### Phase 3. 씬판독기 탭 병합 검토

- 충분히 안정화된 뒤 씬판독기 내부에 탭 추가
- 탭 이름은 씬판독기 쪽에서 별도 결정
- 탭 내부에는 `기존 공달 100LOG 포크 기능` 문구 고정 표시
- 외부 확장 UI를 거의 그대로 이식 가능하도록 컴포넌트화

### Phase 4. 완전 통합 여부 결정

- 외부 확장을 계속 독립 유지할지
- 씬판독기 안으로 완전히 병합할지
- 둘 다 유지할지 결정

---

## 15. 핵심 원칙 요약

```text
100LOG_FORK는 저장된 모든 기억을 매번 주입하지 않는다.
100LOG_FORK는 현재 턴에 필요한 서사연속성 제약만 짧게 주입한다.
주입량은 씬리더와 공유하지 않고 독립 설정으로 관리한다.
실제 주입문은 긴 줄글이 아니라 짧은 제약 목록이다.
증거문, JSON 원문, 전체 knowledge map은 기본 주입에 넣지 않는다.
설치 여부가 아니라 사용 ON 상태일 때만 씬리더 기본 서사연속성을 대체한다.
생성 가로채기와 자동 재작성은 초기 버전에서 제외한다.
```
