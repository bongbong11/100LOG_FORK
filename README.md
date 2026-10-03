# 100LOG_FORK — 씬리더용 서사연속성 모듈 설계서

> 이 저장소는 원본 100LOG의 문제의식과 정보 처리 방식을 참고하여, 씬리더(Scene Reader Hub)용 **서사연속성 판독 모듈**을 새로 설계하기 위한 작업 저장소입니다.
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

이 표기는 사용자가 기능의 계열과 목적을 혼동하지 않도록 유지합니다.

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
   - 활성 기록 중 관련 높은 것만 선별
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

우선순위 기본값:

```text
1. user_correction
2. knowledge_boundary
3. concealed_truth
4. pending_commitment
5. unresolved_tension
6. misunderstanding
7. causal_link
8. relationship_shift
9. state_change
10. recent_event
```

---

## 8. 기록 스키마

서사연속성 기록은 단순 기억이 아니라 다음 턴 제약을 포함합니다.

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

### knowledge 상태값

v1 권장값:

```text
known        알고 있음
unknown      아직 모름
partial      일부만 알고 있음
suspects     의심함
false_belief 잘못 믿고 있음
```

최소 구현은 다음 셋으로 시작할 수 있습니다.

```text
known
unknown
partial
```

---

## 9. 저장 구조

기본 저장 범위는 현재 채팅입니다.

```text
기본:
현재 채팅 단위 저장

고급 옵션:
같은 캐릭터 카드 공유
페르소나+캐릭터 조합 공유
직접 선택한 채팅 포함
```

저장 구조 예시:

```json
{
  "schema": "scene_reader_story_continuity.v1",
  "enabled": true,
  "chat_id": "current_chat",
  "initialized": true,
  "baseline": {
    "created_at": 0,
    "source_scope": "current_chat_recent_20_turns",
    "verified_by_jev": true
  },
  "cursor": {
    "last_visible_message_id": 147,
    "last_signature": "147:hash"
  },
  "records": [],
  "journal": []
}
```

---

## 10. 주입 위치 및 조립

### 10.1 기본 주입 위치

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

이 위치를 쓰는 이유:

```text
고정 설정 → 최근 서사 상태 → 다음 턴 판독
```

### 10.2 depth 주입은 기본값이 아님

depth 주입은 너무 강하게 읽힐 수 있으므로 기본값으로 쓰지 않습니다.

위험:

- 과거 사건이 현재 장면보다 과하게 우선됨
- 이미 움직일 수 있는 감정선이 고정됨
- 장면 전환이 둔해짐
- 모델이 기록을 대사처럼 반복함
- 씬리더의 미래 판독보다 과거 기록이 강하게 작동함

### 10.3 CardInjector식 위치 지정

고급 사용자를 위해 주입 위치를 바꿀 수 있게 합니다.

예정 옵션:

- 월드인포 후
- 캐릭터 정의 후
- 작가노트 전
- 작가노트 후
- 메인 프롬프트 전
- 마지막 유저 메시지 전
- 사용자 지정 프리셋 위치

### 10.4 replace 방식

append가 아니라 replace를 기본으로 합니다.

```text
기존 <STORY_CONTINUITY>...</STORY_CONTINUITY> 블록이 있으면 교체
없으면 지정 위치에 삽입
```

이 방식으로 중복 주입과 찌꺼기를 방지합니다.

---

## 11. STORY_CONTINUITY 블록 예시

```text
<STORY_CONTINUITY>
Recent story-state constraints. Use these after lore and before deciding the next scene.
They are not permanent lore, dialogue, or a full summary.

- [unresolved_tension] Lucas confessed love to Vivienne; she has not answered.
  Constraint: Do not treat the confession as resolved, casual, or repaired unless Vivienne responds in-scene.

- [knowledge_boundary] Dominic does not know the exact private confession.
  Constraint: Dominic may react only to information he has actually witnessed, inferred, or received.

- [concealed_truth] Vivienne's Omega status remains dangerous if exposed to Wade.
  Constraint: Preserve concealment pressure in family-facing scenes unless exposure is explicitly chosen.
</STORY_CONTINUITY>
```

주입 시 전체 기록을 모두 넣지 않습니다.

```text
기본: 최대 5개
정밀: 최대 8개
수동 전체: 최대 12개
```

---

## 12. UI 계획

씬리더허브 안에 새 탭을 둘 수 있습니다. 탭 이름은 추후 씬리더 쪽에서 결정합니다.

후보:

- 서사연속성
- 기억판독
- 스토리관리

탭 내부에는 고정으로 다음 문구를 표시합니다.

```text
기존 공달 100LOG 포크 기능
```

기본 UI:

```text
□ 100로그 포크 / 서사연속성 사용

초기 판독:
[초기 베이스라인 판독 시작]
범위: 최근 20턴 / 최근 30턴 / 현재 채팅 전체 / 직접 선택

증분 판독:
□ 씬판독 시 변화분 자동 반영
최대 delta 수: 3 / 5

주입:
□ STORY_CONTINUITY 주입
위치: 월드인포 후 / 사용자 지정
최대 주입 개수: 5 / 8 / 12

관리:
[수동 재판독]
[선택 기록 고정]
[완료 처리]
[삭제]
[주입 찌꺼기 청소]
[저장 기록 초기화]
```

기본값:

```text
사용: OFF
초기 판독 범위: 최근 20턴
증분 판독: ON, 단 기능 사용 시에만
주입 위치: 월드인포 후
삽입 방식: replace
최대 주입 개수: 5
자동 재작성: OFF / 미구현
```

---

## 13. 청소 기능

### 13.1 주입 찌꺼기 청소

```text
[주입 찌꺼기 청소]
```

삭제 대상:

- `<STORY_CONTINUITY>...</STORY_CONTINUITY>`
- 구버전 `<MEMORY_CONTINUITY>...</MEMORY_CONTINUITY>`
- 구버전 `<SCENE_MEMORY>...</SCENE_MEMORY>`
- 본 확장이 만든 기타 주입 블록

주의:

- 저장 기록은 삭제하지 않음
- 프롬프트에 남은 삽입물만 제거

### 13.2 저장 기록 초기화

```text
[저장 기록 초기화]
```

삭제 대상:

- records
- journal
- baseline
- cursor

주의:

- 사용자가 직접 고정한 기록은 삭제 전 확인
- 주입 블록은 별도 청소 버튼으로 제거

---

## 14. 구현 단계

### Phase 1. 설계 고정

- 원본 100LOG 링크 및 참고 범위 명시
- 100LOG_FORK 명칭 고정
- 씬리더용 스키마 확정
- 초기 판독 / 증분 판독 흐름 확정
- 주입 위치와 replace 방식 확정

### Phase 2. 외부 확장 v0

- 단독 확장으로 제작
- 현재 채팅 visible 메시지 읽기
- 초기 베이스라인 판독 버튼
- records 저장
- STORY_CONTINUITY 블록 미리보기
- window API 제공

예정 API:

```js
window.SceneReaderStoryContinuity = {
  runBaselineScan,
  runDeltaScan,
  getRecords,
  getInjectionBlock,
  clearInjectionBlock,
  clearRecords
};
```

### Phase 3. 씬리더허브 연동

- 씬리더허브가 외부 확장 존재 여부 감지
- 씬리더허브 내부 탭에 `기존 공달 100LOG 포크 기능` 표기
- STORY_CONTINUITY 블록을 월드인포 후 위치에 조립
- CardInjector식 위치 지정 옵션 추가

### Phase 4. 증분 처리 강화

- cursor 저장
- message signature 저장
- journal rollback
- 리롤/수정/삭제 감지
- 요약·하이드 시 carryover 처리

### Phase 5. 통합 검토

- 외부 확장으로 충분히 테스트
- 씬리더허브 UI에 병합 여부 검토
- 자동 재작성은 기본 제외 상태 유지
- 필요 시 고급 충돌 경고 모드만 별도 검토

---

## 15. 최종 설계 요약

```text
읽기 추적과 정보 처리 원리:
100LOG 참고

읽는 기준, 모델 운용, 저장 범위, 주입 위치, 프롬프트 조립:
씬리더 방식으로 재설계

기본 운용:
초기 베이스라인 판독 + 매턴 증분 delta 갱신

주입:
월드인포 후 <STORY_CONTINUITY> replace 삽입

기본 제외:
생성 가로채기, 숨은 초안, 자동 재작성, 답변 공개 전 검수

표기:
외부 확장명은 100LOG_FORK / 100로그 포크
씬리더 내부 UI에는 기존 공달 100LOG 포크 기능 문구 고정
```
