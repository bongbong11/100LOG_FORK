# 100LOG_FORK — 씬리더용 서사연속성 판독 확장 설계서

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
- 관련 기록 선별을 위해 임베딩을 보조 랭커로 쓰는 아이디어

### 2.2 씬리더식으로 바꿀 것

- 읽기 범위는 100개 고정이 아니라 씬리더 설정과 초기 판독 범위 기준으로 운용
- 모델 운용은 `확장 연결모델 후보 추출 → JEV 검증/정제 → 씬판독 delta 갱신` 구조로 변경
- 생성 가로채기, 숨은 초안, 자동 재작성은 기본 설계에서 제외
- 저장 범위는 현재 채팅 기준을 기본값으로 사용
- 주입 위치는 월드인포 후를 기본값으로 사용
- CardInjector식 프리셋 위치 지정 옵션 제공
- 주입 블록은 `<STORY_CONTINUITY>`로 통일
- append가 아니라 replace 방식으로 주입 찌꺼기 방지
- 실제 주입문은 저장 기록 전체가 아니라, 현재 턴에 필요한 짧은 제약 목록만 사용

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
   - 활성 기록 중 현재 씬에 관련 높은 것만 선별
   - 필수 규칙 + 규칙 기반 점수 + 선택적 임베딩 보조 점수를 혼합
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

### 5.4 JEV 사용 위치

JEV는 매턴 주입 후보 선별에 기본적으로 사용하지 않습니다. 주입 선별까지 매번 JEV로 검수하면 정확도는 올라갈 수 있지만, 속도와 비용이 크게 늘고 기존 100LOG처럼 재검수·재작성 구조로 흐를 수 있습니다.

JEV 권장 사용 위치:

- 초기 베이스라인 후보 검증
- 위험한 상태 변경 검증
- 수동 정밀 재판독

위험한 상태 변경 예:

- `knowledge_boundary` 변경
- `concealed_truth`가 노출 상태로 바뀜
- `pending_commitment`를 resolved/cancelled 처리함
- `user_correction`을 덮어씀
- priority 5 기록을 종료하려 함

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

중요한 점은 단순 사건 요약이 아니라, 다음 턴에서 깨지면 안 되는 조건만 추출한다는 것입니다.

---

## 8. 저장 데이터 구조

저장은 자세히 하되, 실제 주입은 짧게 합니다.

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
  "retrieval_text": "Lucas Vivienne love confession unanswered unresolved tension Dominic unknown",
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
| `current_pressure` | 현재 장면에 남아 있는 압력 또는 유효 상태 |
| `next_turn_constraint` | 다음 답변에서 깨지면 안 되는 제약 |
| `knowledge` | 인물별 지식 상태 |
| `evidence` | 저장 검증용 원문 근거. 기본 주입에는 넣지 않음 |
| `retrieval_text` | 규칙 기반/임베딩 선별용 짧은 검색 텍스트 |

---

## 9. 주입 선별 원칙

### 9.1 저장 전체 주입 금지

저장된 서사연속성 기록 전체를 매번 프롬프트에 보내지 않습니다.

기본 원칙:

```text
저장소: 자세하게 보관
실제 주입: 현재 턴에 필요한 기록만 짧게 선별
```

금지:

- 저장 기록 전체 주입
- 전체 JSON 주입
- evidence quote 기본 주입
- 전체 knowledge map 기본 주입
- 긴 줄글 요약 주입

허용:

- 현재 턴 관련 기록만 선별
- 짧은 bullet 제약 목록
- 필요할 때만 knowledge boundary를 문장화
- next_turn_constraint 중심 주입

### 9.2 주입량은 100LOG_FORK 단독 설정

주입량은 씬리더의 출력량, 판독량, 답변 길이 설정과 공유하지 않습니다.

권장값:

```text
짧게: 최대 3개
기본: 최대 5개
중요 장면: 최대 8개
수동 전체 확인: 최대 12개
```

기본값은 `최대 5개`입니다.

### 9.3 필수 포함 후보

임베딩 점수와 상관없이 후보 풀에 반드시 올리는 항목입니다.

- pinned 된 기록
- `user_correction`
- 현재 등장/언급 인물과 직접 관련된 `knowledge_boundary`
- 현재 등장/언급 인물과 직접 관련된 `concealed_truth`
- 현재 메시지의 시간·장소·행동·참여자 단서와 겹치는 `pending_commitment`
- 최근 1~2턴 안에 add/update된 기록
- priority 5 기록

이 항목들은 임베딩 유사도가 낮아도 빠지면 안 됩니다.

### 9.4 규칙 기반 랭킹

기본 선별은 규칙 기반으로 작동합니다.

예시 점수 요소:

```text
type_weight
+ priority_weight
+ target_overlap
+ recent_delta_bonus
+ pinned_bonus
+ status_weight
```

우선순위가 높은 type:

```text
user_correction
knowledge_boundary
concealed_truth
pending_commitment
unresolved_tension
misunderstanding
causal_link
relationship_shift
recent_event
```

### 9.5 임베딩은 보조 랭커로만 사용

임베딩은 관련 후보를 넓히는 데 유용하지만, 단독 결정권을 주지 않습니다.

이유:

```text
임베딩은 비슷한 것을 잘 찾지만, 반드시 필요한 것을 보장하지 않는다.
```

예를 들어 `Dominic does not know Lucas confessed love to Vivienne` 기록은 현재 장면에 Dominic이 등장하기만 해도 중요할 수 있습니다. 하지만 현재 메시지에 love/confession 키워드가 없다면 임베딩 점수가 낮게 나올 수 있습니다. 이런 기록은 `knowledge_boundary + target_overlap` 규칙으로 반드시 후보에 올라야 합니다.

반대로 임베딩은 표현이 달라도 의미가 가까운 기록을 찾는 데 유용합니다.

예:

```text
저장 기록:
Vivienne's Omega status remains concealed from Wade.

현재 장면:
Her father summoned the doctor and asked why the suppressant records were missing.
```

이런 경우 임베딩은 concealed truth 후보를 보강할 수 있습니다.

### 9.6 혼합형 선별 방식

권장 선별 흐름:

```text
1. 필수 포함 후보를 먼저 모은다.
2. 나머지 active 기록 중 규칙 기반 점수 상위 후보를 모은다.
3. 임베딩이 사용 가능하면 현재 씬 query_text와 기록 retrieval_text의 유사도를 계산한다.
4. 임베딩 top 후보를 후보 풀에 보강한다.
5. 최종 점수 = 규칙 기반 점수 + 임베딩 보조 점수.
6. 상위 3~5개만 짧은 STORY_CONTINUITY bullet로 주입한다.
```

추천 모드:

```text
[주입 후보 선별 방식]
● 안정형: 규칙 기반
○ 혼합형: 규칙 기반 + 임베딩 보조
○ 실험형: 임베딩 우선
```

기본값은 `안정형`입니다.
권장 테스트값은 `혼합형`입니다.
`실험형`은 기본 배포값으로 쓰지 않습니다.

임베딩 실패 시에는 조용히 규칙 기반 선별로 fallback합니다.

---

## 10. 실제 주입문 형식

실제 모델에 들어가는 내용은 모델이 이해하기 쉽되 최대한 짧아야 합니다.

권장 형식:

```text
<STORY_CONTINUITY>
Use only for the next reply.
- [unresolved_tension] Lucas confessed love; Vivienne has not answered. Do not treat it as resolved.
- [knowledge_boundary] Dominic did not hear the confession. Do not let him reference exact wording.
</STORY_CONTINUITY>
```

비권장 형식:

```text
<STORY_CONTINUITY>
아래는 최근 20턴에서 추출된 서사연속성 기록입니다. 첫 번째 기록은 루카스가 비비안에게 사랑을 고백했다는 사실이며...
...
</STORY_CONTINUITY>
```

비권장 이유:

- 토큰 낭비
- 모델이 요약문을 대사처럼 반복할 가능성
- 핵심 제약이 흐려짐
- 씬리더 주입 블록과 중복 가능

---

## 11. 주입 위치

### 기본 위치

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

### depth 주입 기본 제외

서사연속성을 너무 깊은 위치에 넣으면 다음 문제가 생길 수 있습니다.

- 과거 사건이 현재 장면보다 과하게 우선됨
- 이미 해결 가능한 감정도 계속 고정됨
- 장면 전환이 둔해짐
- 모델이 기억 규칙을 대사처럼 반복할 수 있음
- 씬리더의 미래 판독보다 기억이 더 강하게 먹을 수 있음

따라서 depth 또는 다른 프리셋 위치는 고급 옵션으로만 제공합니다.

### CardInjector식 위치 지정

고급 사용자를 위해 카드인젝터 방식의 프리셋 위치 지정도 지원합니다.

예정 옵션:

- 월드인포 후
- 캐릭터 정의 후
- 작가노트 전
- 작가노트 후
- 메인 프롬프트 전
- 마지막 유저 메시지 전
- 사용자 지정 프리셋 위치

---

## 12. 삽입 방식: append 금지, replace 기본

서사연속성 블록은 매번 새로 덧붙이는 방식이 아니라, 기존 블록을 찾아 교체합니다.

```text
기존 <STORY_CONTINUITY>...</STORY_CONTINUITY> 블록이 있으면 교체
없으면 지정된 위치에 삽입
```

이 방식을 쓰는 이유:

- 이전 판독 결과가 찌꺼기로 남는 것을 방지
- 중복 주입 방지
- 확장 업데이트 후 구버전 블록을 안전하게 청소 가능
- 저장된 기록과 실제 프롬프트 주입 상태를 분리 가능

### 청소 버튼

다음 기능을 가진 청소 버튼을 둡니다.

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

## 13. 씬리더허브와의 연동 원칙

초기 버전에서는 씬리더허브 UI에 아무 탭도 추가하지 않습니다.

```text
v1:
100LOG_FORK 독립 UI + 독립 설정 + 독립 저장 + 독립 주입
씬리더허브 UI 변경 없음
```

다만 이후 연동을 위해 window API를 제공합니다.

예정 API:

```js
window.SceneReaderStoryContinuity = {
  isAvailable(),
  isEnabled(),
  getStatus(),
  getRecords(),
  getInjectionBlock(),
  runBaseline(),
  runDelta(),
  clearInjection()
};
```

씬리더허브는 설치 여부가 아니라 `isEnabled()`를 기준으로 판단합니다.

```js
const external = window.SceneReaderStoryContinuity;
const useFork = Boolean(external?.isAvailable?.() && external?.isEnabled?.());

if (useFork) {
  const storyContinuityBlock = external.getInjectionBlock();
} else {
  const storyContinuityBlock = sceneReaderDefaultStoryContinuity();
}
```

동작 원칙:

```text
100LOG_FORK 미설치
→ 씬판독기 기본 서사연속성 사용

100LOG_FORK 설치됨 + 사용 OFF
→ 씬판독기 기본 서사연속성 사용

100LOG_FORK 설치됨 + 사용 ON
→ 씬판독기 기본 서사연속성을 100LOG_FORK 결과로 대체
```

즉, 두 서사연속성 블록을 동시에 주입하지 않습니다.

---

## 14. 씬리더허브 인물판독/임베딩에도 적용할 원칙

씬리더허브가 인물판독에 임베딩을 사용하는 경우에도 같은 주의가 필요합니다.

임베딩은 후보를 넓히는 도구이지, 최종 포함/제외를 혼자 결정하는 도구가 아닙니다.

허브 인물판독에도 다음 원칙을 적용하는 것이 좋습니다.

```text
1. 현재 발화자, 직접 등장 인물, 최근 언급 인물, 수동 고정 인물 기록은 임베딩 점수와 무관하게 후보 풀에 포함한다.
2. 임베딩은 표현이 달라도 의미가 가까운 보조 후보를 찾는 데 쓴다.
3. 최종 주입은 임베딩 유사도 + 인물 등장 여부 + 관계 중요도 + 최근 delta + priority를 혼합한다.
4. 임베딩 실패 시 기본 규칙 기반 선별로 fallback한다.
5. 임베딩 점수가 낮다는 이유만으로 지식 경계, 금기, 사용자 정정사항, 현재 발화자의 핵심 반응 기준을 제외하지 않는다.
```

이 원칙은 100LOG_FORK뿐 아니라 씬리더허브의 인물판독 정확도에도 도움이 됩니다.

---

## 15. 개발 단계

### Phase 1. 독립 확장 제작

- 100LOG_FORK 독립 UI 제작
- 씬리더허브 UI 수정 없음
- 초기 베이스라인 판독
- 기록 저장/수정/삭제/고정
- STORY_CONTINUITY 주입/청소
- 규칙 기반 주입 선별
- 선택적 혼합형 임베딩 선별

### Phase 2. 씬리더허브 선택 연동

- window API 제공
- 100LOG_FORK 사용 ON일 때만 씬리더 기본 서사연속성 대체
- 사용 OFF일 때는 씬리더 기본 서사연속성 유지

### Phase 3. UI 병합 검토

- 충분히 안정화된 뒤 씬리더허브에 탭 추가 검토
- 탭 내부에 `기존 공달 100LOG 포크 기능` 표기 유지
- 외부 확장 UI를 거의 그대로 옮길 수 있도록 v1 UI 구조를 설계

### Phase 4. 완전 통합 여부 결정

- 저장소 마이그레이션
- 주입 설정 이전
- 외부 확장 독립 유지 또는 허브 내장 여부 결정

---

## 16. 핵심 원칙 요약

```text
100LOG_FORK는 저장된 기억을 전부 주입하는 확장이 아니다.
100LOG_FORK는 현재 씬에 필요한 서사연속성 제약만 짧게 주입한다.
임베딩은 단독 결정자가 아니라 보조 랭커다.
필수 규칙은 임베딩 점수와 상관없이 후보에 포함한다.
JEV는 매턴 주입 선별이 아니라 초기 검증과 위험한 상태 변경 검증에 쓴다.
생성 가로채기와 자동 재작성은 기본 설계에서 제외한다.
주입은 월드인포 후 기본, append가 아니라 replace 방식으로 한다.
씬리더허브와 연동할 때는 설치 여부가 아니라 100LOG_FORK 사용 ON/OFF를 기준으로 한다.
```
