# 100LOG_FORK → 씬리더용 서사연속성 모듈 재설계 계획서

> 이 저장소는 기존 100LOG의 문제의식과 연속성 카테고리를 참고하여, 씬리더(Scene Reader) 계열 확장에 맞는 **서사연속성 판독 모듈**로 재설계하기 위한 계획 저장소입니다.
>
> 원본 100LOG의 구조를 그대로 복제하거나 단순 개조하는 것이 아니라, 제작자의 허락을 받은 범위 안에서 “어떤 종류의 정보를 연속성 관리 대상으로 볼 것인가”를 참고하고, 실제 판독·저장·주입·조립 방식은 씬리더 방식으로 새로 설계합니다.

---

## 1. 기본 방향

씬리더와 100LOG류 확장은 바라보는 방향이 다릅니다.

| 구분 | 역할 |
|---|---|
| 씬리더 | 다음 턴, 다음 장면, 미래 전개를 판독하는 확장 |
| 100LOG류 | 최근 과거 사건, 약속, 비밀, 정정, 인물별 지식 차이를 기억하는 확장 |
| 본 프로젝트 | 과거 사건을 현재 압력으로 해석하고, 다음 턴 조건으로 변환하는 서사연속성 모듈 |

즉, 본 프로젝트의 목표는 단순한 기억 확장이 아닙니다.

```text
과거 사건 → 현재 장면 압력 → 다음 턴 제약
```

이 흐름을 씬리더 내부 판독 체계에 맞게 다루는 것이 목적입니다.

---

## 2. 왜 별도 재설계가 필요한가

기존 100LOG는 최근 RP의 중요한 사실을 기억하고, 답변 공개 전 검수하는 독립형 확장입니다. 이 구조는 매우 유용하지만, 씬리더와 바로 합치기에는 다음 차이가 있습니다.

| 항목 | 100LOG식 구조 | 씬리더용 구조 |
|---|---|---|
| 작동 방식 | 최근 100개 visible 메시지를 롤링 관리 | 사용자가 켠 경우에만 판독 모듈로 작동 |
| 목적 | 과거 사실 규칙 저장 및 답변 검수 | 다음 턴 연속성 제약 추출 |
| 기본 동작 | 확장 사용 시 자동 기억/검수 | 씬리더 탭에서 ON/OFF |
| 주입 위치 | 독립 컨텍스트 블록 | 기본값: 월드인포 후, 필요 시 프리셋 위치 지정 |
| 생성 가로채기 | generate interceptor 기반 공개 전 검수 | 초기 버전에서는 사용하지 않음 |
| 저장 성격 | 기억 저장소 | 씬리더용 판독 결과 캐시 + 선택적 저장 |
| 통합 목표 | 독립 기억 확장 | 씬리더의 서사연속성 판독 레이어 |

따라서 이 프로젝트는 100LOG를 그대로 씬리더에 붙이는 작업이 아니라, 100LOG가 다루는 연속성 카테고리를 참고하여 씬리더용으로 다시 설계하는 작업입니다.

---

## 3. 본 프로젝트에서 참고할 100LOG의 핵심 범위

100LOG에서 직접적으로 참고할 부분은 “코드 구조 전체”가 아니라 다음과 같은 연속성 카테고리와 판단 기준입니다.

### 참고할 카테고리

- 미해결 약속·계획
- 일정, 날짜, 장소가 걸린 약속
- 비밀, 은폐, 거짓말
- 오해, 착각, 잘못 믿는 사실
- 인물별 지식 차이
- 사용자의 정정사항
- 해결되지 않은 갈등·질문·압박
- 관계 변화
- 최근 사건의 인과관계
- 요약 또는 하이드 후에도 잠시 유지해야 하는 중요 조건

### 참고할 판단 원칙

- 이름이 언급되었다고 해서 그 인물이 정보를 아는 것은 아님
- 약속은 시간이 지났다고 자동 완료되지 않음
- 실제 장면에서 이행되거나 명시적으로 취소되어야 완료/취소 처리됨
- 비밀, 거짓말, 오해, 정정사항은 일반 사건보다 우선순위가 높음
- 최근 장면의 복장·자세·위치 같은 live scene state는 별도 모듈이 담당하고, 서사연속성에는 과하게 저장하지 않음
- 단순 요약이 아니라 다음 턴에서 깨지면 안 되는 조건만 추출함

---

## 4. 씬리더용으로 새로 정의할 역할

본 모듈은 “기억 확장”이 아니라 “서사연속성 판독 모듈”입니다.

```text
서사연속성 판독은 최근 RP 원문에서 다음 턴의 장면·인물·세계 반응을 깨뜨릴 수 있는 미해결 연속성 조건을 추출한다.

서사연속성 판독은 장기 메모리, 로어북, 요약기, 채팅 로그 저장소가 아니다.

서사연속성 판독은 씬리더가 다음 답변을 조립할 때 참고할 수 있도록 현재 유효한 서사 상태와 다음 턴 제약을 제공한다.
```

---

## 5. 씬리더 내 위치

씬리더는 기존에 다음과 같은 미래 지향 판독을 담당합니다.

- 장면 흐름 판독
- 캐릭터 반응 판독
- 세계관/NPC/적대성 판독
- NSFW 판독
- 출력 비율/길이/속도 조절

여기에 새 모듈을 추가합니다.

```text
SCENE_READER_JEV_PIPELINE
 ├─ scene_flow
 ├─ character_reaction
 ├─ world_direction
 ├─ npc_presence
 ├─ nsfw_check
 └─ story_continuity    ← 추가 예정
```

`story_continuity`는 기본 OFF이며, 사용자가 씬리더허브의 별도 탭에서 켠 경우에만 작동합니다.

---

## 6. 주입 위치 설계

### 기본 주입 위치

기본 주입 위치는 **월드인포 / 로어북 뒤**로 합니다.

```text
World Info / Lorebook
↓
<STORY_CONTINUITY>
↓
Scene Reader 판독 블록
↓
응답 지시 / 출력 형식
```

이 위치를 기본으로 잡는 이유는 다음과 같습니다.

1. 로어북은 고정 설정을 제공함
2. 서사연속성은 그 설정 위에서 최근 RP 때문에 생긴 유효 상태를 얹음
3. 씬리더 판독은 이 상태를 바탕으로 다음 턴을 판단함

즉 순서는 다음과 같습니다.

```text
고정 설정 → 최근 서사 상태 → 다음 턴 판독
```

### Depth 주입을 기본으로 쓰지 않는 이유

서사연속성을 너무 깊은 위치에 넣으면 다음 문제가 생길 수 있습니다.

- 과거 사건이 현재 장면보다 과하게 우선됨
- 이미 해결 가능한 감정도 계속 고정됨
- 장면 전환이 둔해짐
- 모델이 기억 규칙을 대사처럼 반복할 수 있음
- 씬리더의 미래 판독보다 기억이 더 강하게 먹을 수 있음

따라서 기본값은 월드인포 후로 하고, depth 또는 다른 프리셋 위치는 고급 옵션으로만 제공합니다.

### 카드인젝터식 위치 지정

고급 사용자를 위해 카드인젝터 방식의 프리셋 위치 지정도 지원할 계획입니다.

예정 옵션:

- 월드인포 후
- 캐릭터 정의 후
- 작가노트 전
- 작가노트 후
- 메인 프롬프트 전
- 마지막 유저 메시지 전
- 사용자 지정 프리셋 위치

기본은 월드인포 후이며, 사용자가 필요할 때만 위치를 바꿀 수 있게 합니다.

---

## 7. 삽입 방식: append가 아니라 replace

서사연속성 블록은 매번 새로 덧붙이는 방식이 아니라, 기존 블록을 찾아 교체하는 방식으로 운용합니다.

```text
기존 <STORY_CONTINUITY>...</STORY_CONTINUITY> 블록이 있으면 교체
없으면 지정된 위치에 삽입
```

이 방식을 쓰는 이유는 다음과 같습니다.

- 이전 판독 결과가 찌꺼기로 남는 것을 방지
- 중복 주입 방지
- 확장 업데이트 후 구버전 블록을 안전하게 청소 가능
- 저장된 기록과 실제 프롬프트 주입 상태를 분리 가능

### 청소 버튼

다음 기능을 가진 청소 버튼을 둘 계획입니다.

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

## 8. 판독 카테고리 설계

v1에서 사용할 기본 카테고리는 다음과 같습니다.

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

## 9. 판독 결과 구조

기억을 단순히 저장하는 구조가 아니라, 과거 사건을 현재 압력과 다음 턴 조건으로 변환합니다.

예시:

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
  "source": "auto"
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
| `current_pressure` | 현재 장면에 남아 있는 압력 |
| `next_turn_constraint` | 다음 답변에서 지켜야 할 조건 |
| `knowledge` | 인물별 지식 상태 |
| `evidence` | 원문 근거 |
| `expires_when` | 언제 만료/완료 처리할지 |
| `source` | auto, manual, pinned 등 |

---

## 10. 인물별 지식 상태

v1에서는 다음 상태를 우선 지원합니다.

| 상태 | 의미 |
|---|---|
| `known` | 해당 정보를 알고 있음 |
| `unknown` | 해당 정보를 아직 모름이 지지됨 |
| `partial` | 일부만 알고 있거나 맥락을 불완전하게 앎 |
| `suspects` | 확정 지식은 아니지만 의심함 |
| `believes_false` | 사실과 다른 내용을 믿고 있음 |
| `misunderstands` | 정보를 접했지만 잘못 해석함 |

단, v1 구현 부담을 줄이기 위해 초기에는 `known / unknown / partial`을 우선 적용하고, 이후 확장할 수 있습니다.

판정 원칙:

- 이름이 언급되었다는 이유만으로 `known` 처리하지 않음
- 현장에 없었다는 이유만으로 `unknown` 처리하지 않음
- 통신, 감시, 보고, 기록, 목격 등 정보 흐름이 있으면 장면 밖 인물도 `known` 가능
- 비밀로 유지되었거나 전달되지 않았다는 근거가 있을 때만 `unknown` 처리
- 일부 조항만 알면 `partial` 처리

---

## 11. 읽기 범위

100LOG처럼 무조건 최근 100개를 기본으로 읽지 않습니다.

씬리더용 기본값은 더 가볍고 다음 턴 중심이어야 합니다.

예정 옵션:

| 모드 | 범위 |
|---|---|
| 가볍게 | 최근 20개 visible 메시지 |
| 균형 | 최근 40개 visible 메시지 |
| 넓게 | 최근 80개 visible 메시지 |
| 수동 재판독 | 최대 100개 visible 메시지 |

기본값은 최근 20개 또는 40개를 검토합니다.

범위가 너무 넓으면 다음 문제가 생길 수 있습니다.

- 비용과 속도 증가
- 중요하지 않은 잡기억 증가
- 씬리더가 요약기처럼 변함
- 장기 로어/메모리와 역할이 겹침

---

## 12. 저장 범위

v1 기본 저장 범위는 **현재 채팅 단위**입니다.

이유:

- 같은 캐릭터 카드로 여러 세계선/IF 채팅을 굴릴 수 있음
- 캐릭터 UUID 단위 공유는 다른 채팅의 연속성이 섞일 위험이 있음
- 배포용으로는 현재 채팅 저장이 가장 예측 가능함

향후 옵션:

- 현재 채팅만
- 캐릭터 카드 단위 공유
- 페르소나 + 캐릭터 조합 단위 공유
- 수동 내보내기/가져오기

---

## 13. 주입 조립 방식

저장된 기록 전체를 그대로 주입하지 않습니다.

주입 대상은 다음 기준으로 선별합니다.

1. `active` 또는 `pending` 상태
2. `priority`가 일정 기준 이상
3. `scope`가 `next_turn` 또는 `current_scene`
4. 현재 등장/언급 인물과 관련 있음
5. 기존 씬리더 판독 결과와 중복되면 압축
6. 사용자가 고정한 기록은 우선 포함

### 주입 우선순위

1. `user_correction`
2. `knowledge_boundary`
3. `concealed_truth`
4. `pending_commitment`
5. `unresolved_tension`
6. `misunderstanding`
7. `causal_link`
8. `relationship_shift`
9. `recent_event`
10. `state_change`

### 주입 블록 예시

```text
<STORY_CONTINUITY>
Recent story continuity constraints.
Use these as active story-state pressure between lore and the next scene.
They are not permanent lore, not dialogue, and not a summary.

- [knowledge_boundary] Dominic does not know Lucas's exact confession to Vivienne.
  Effect: Dominic should not directly reference the confession unless he learns it in-scene.

- [unresolved_tension] Lucas confessed love to Vivienne, and Vivienne has not answered yet.
  Effect: Do not skip past Vivienne's reaction or treat the confession as emotionally settled.

- [concealed_truth] Vivienne's Omega status remains dangerous if exposed to Wade.
  Effect: Family-facing scenes should preserve concealment pressure.
</STORY_CONTINUITY>
```

---

## 14. UI 계획

씬리더허브 안에서는 별도 탭으로 관리합니다.

예정 탭명:

```text
서사연속성
```

또는 단순하게:

```text
기억판독
```

### 기본 UI

```text
[서사연속성 / 기억판독]

□ 사용

판독 범위:
○ 최근 20개
○ 최근 40개
○ 최근 80개
○ 수동 100개

판독 강도:
○ 핵심만
○ 균형
○ 세세하게

수집 대상:
☑ 미해결 약속/계획
☑ 비밀/은폐/거짓말
☑ 인물별 지식 차이
☑ 사용자 정정사항
☑ 오해/착각
☑ 미해결 갈등
☑ 관계 변화
☑ 최근 인과관계

주입 위치:
● 월드인포 후
○ 작가노트 전
○ 작가노트 후
○ 사용자 지정 프리셋 위치

주입 방식:
● 다음 턴 관련만
○ 중요도 높은 것만
○ 활성 기록 전체
○ 주입 안 함

버튼:
[지금 판독]
[최근 기록 재판독]
[선택 기록 고정]
[완료 처리]
[삭제]
[주입 찌꺼기 청소]
```

기본값:

```text
사용: OFF
판독 범위: 최근 20개 또는 40개
판독 강도: 균형
주입 위치: 월드인포 후
주입 방식: 다음 턴 관련만
삽입 방식: 기존 블록 교체
```

---

## 15. 외부 확장 → 씬리더허브 병합 계획

처음부터 씬리더허브 코어에 바로 넣지 않고, 외부 확장으로 먼저 검증합니다.

### Phase 1. 외부 확장 제작

가칭:

```text
Scene_Continuity_Reasoner
```

기능:

- 최근 visible 메시지 읽기
- 서사연속성 판독 실행
- JSON 결과 저장
- 수동 실행 버튼
- 주입 블록 미리보기
- 씬리더허브가 읽을 수 있는 window API 제공

예정 API:

```js
window.SceneReaderMemoryJudge = {
  getRecords(),
  getInjectionBlock(),
  runJudge(),
  clearRecords(),
  cleanInjectedBlocks()
};
```

### Phase 2. 씬리더허브 연동

씬리더허브는 외부 확장이 감지되면 결과를 읽고, 감지되지 않으면 조용히 무시합니다.

```js
const memoryBlock = window.SceneReaderMemoryJudge?.getInjectionBlock?.();
```

### Phase 3. 실사용 검증

검증 항목:

- 실제 RP에서 다음 턴 품질이 좋아지는지
- 과거 사건에 과하게 묶이지 않는지
- 주입 위치가 적절한지
- 월드인포 후 삽입이 가장 안정적인지
- 프리셋 위치 변경이 필요한 사용자군이 있는지
- 저장된 기록과 주입 블록이 중복되지 않는지
- 청소 버튼이 안전하게 작동하는지

### Phase 4. UI 병합

외부 확장이 충분히 안정화되면 씬리더허브 안에 탭으로 병합합니다.

```text
씬리더허브
 ├─ 기본
 ├─ 세계관
 ├─ 인물판정
 ├─ NSFW
 ├─ 서사연속성
 └─ 고급
```

### Phase 5. 코어 통합

최종적으로는 씬리더 제브판독 파이프라인의 선택 모듈로 편입합니다.

```text
story_continuity: OFF by default
```

---

## 16. 원본 100LOG와의 관계

이 프로젝트는 원본 100LOG를 대체하거나 그대로 복제하려는 목적이 아닙니다.

원본 100LOG의 강점:

- 최근 RP 연속성 규칙 추출
- 인물별 지식 차이 관리
- 미해결 약속/계획 관리
- 사용자 정정사항과 비밀 관리
- 답변 공개 전 검수

본 프로젝트의 방향:

- 원본의 카테고리 감각과 문제의식을 참고
- 씬리더용 미래 판독 구조에 맞게 재해석
- 과거 기억을 현재 압력과 다음 턴 제약으로 변환
- 생성 가로채기보다는 주입 조립 중심으로 시작
- 월드인포 후 주입을 기본값으로 사용
- 카드인젝터식 위치 지정은 고급 옵션으로 지원

---

## 17. 우선 구현하지 않을 것

초기 버전에서는 다음 기능을 바로 넣지 않습니다.

- 생성 가로채기 기반 공개 전 검수
- 모든 답변 강제 재작성
- 캐릭터 UUID 단위 전체 채팅 공유 저장
- 깊이 주입 기본값
- 장기 로어북 자동 생성
- 전체 채팅 요약기
- 모든 과거 사건 무제한 저장

이 기능들은 추후 필요성이 확인되면 선택 옵션으로 검토합니다.

---

## 18. 핵심 원칙

```text
서사연속성은 장면을 고정하지 않는다.
서사연속성은 장면이 자연스럽게 다음 상태로 넘어가도록 현재 압력을 보존한다.
```

나쁜 예:

```text
Vivienne must stay shocked and cannot move on.
```

좋은 예:

```text
Vivienne's response should account for the unresolved confession; she may answer, avoid, deflect, freeze, or escalate, but the confession should not be treated as already settled.
```

즉, 본 모듈은 캐릭터의 선택지를 줄이는 것이 아니라, 과거 사건이 납작하게 사라지지 않도록 다음 턴의 조건을 보존합니다.

---

## 19. 최종 목표

최종 목표는 다음과 같습니다.

```text
씬리더 = 미래 전개 판독
100LOG류 = 과거 사건 기억
서사연속성 모듈 = 과거 사건을 현재 압력으로 해석하고 다음 전개 조건으로 변환
```

이를 통해 로어북이나 일반 기억확장과는 다른, RP 전개용 스토리 관리 레이어를 만드는 것이 목표입니다.

```text
고정 설정을 담당하는 로어북
과거 정보를 저장하는 기억확장
다음 전개를 판독하는 씬리더

그 사이에서
과거-현재-미래를 이어주는 서사연속성 엔진
```
