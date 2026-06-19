# Input Guardrail 수정 결과 보고서 (2026-06-19)

대상: `manufacturing_agent_v2.ipynb` (원본 `manufacturing_agent.ipynb` 복사본)

## 변경 요약 (T1/T2/T3 · ~~T4~~ 철회 · T5)

- **T1 (1층 정규식)**: 셀 16 `INJECTION_PATTERNS` 한/영 변형 보강, 셀 29 `input_gate`에서 빈 값·인젝션 즉시 차단. 왜: 비용 0·오프라인 안전망.
- **T2 (2층 LLM)**: 셀 6 `call_llm_json`, 셀 29 2층 판정(gibberish/out_of_scope). 왜: 정규식이 못 잡는 모호 케이스. 실패/StubLLM이면 정규식-only 폴백. (※ T5에서 `no_control_authority` 제거됨)
- **T3 (리포트 재설계)**: 셀 8 `GateReport`에 block/reason/layer/message, 셀 29 flags→details 강등, 셀 32 `final_answer_node`가 message 출력. 왜: 리포트가 판단을 *구동*하지 않고 *설명*.
- ~~**T4 (멀티턴 재예측 over-block 수정)**~~ — **철회(T5로 대체)**. 시도: 셀 29에 `_REPREDICT_PATTERNS`+재예측 휴리스틱(옵션2, 직전 예측 state 활용)으로 턴2 over-block 우회. 철회 사유: 260619 피드백 — "맥락을 가져와 4b를 거르는 것은 입력 단 책임 과중". → 코드·마커·테스트 전부 제거됨.
- **T5 (제어·승인 4b 통과 전환 + T4 철회)**: 셀 29 2층 차단 화이트리스트에서 `no_control_authority` 제거(`("gibberish","out_of_scope")`), `_GUARDRAIL_SYS`가 제어·승인 명령을 `ok`로 분류, `REASON_MESSAGES`·enum에서 `no_control_authority` 제거, T4 전부 revert. 왜: 가드레일을 **stateless(raw input only)** 로 유지, **4b는 PASS** — 위험 부분집합만 하류 SafetyGate가 차단.

## 변경 지점 목록

| 마커 | 셀(경로 주석) | 심볼 | 변경 내용 |
| --- | --- | --- | --- |
| T2 | 셀 6 (설정) | call_llm_json | 구조화 출력 래퍼 신설 |
| T3 | 셀 8 contracts/routing.py | GateReport | block/reason/layer/message 추가 |
| T1 | 셀 16 context/context_policy.py | INJECTION_PATTERNS | 한/영 변형 5개 보강 |
| T1/T3 | 셀 29 gates/input_gate.py | input_gate, REASON_MESSAGES | 1층 차단+리포트 기록+flags 강등 |
| T2 | 셀 29 gates/input_gate.py | input_gate | 2층 LLM 판정 블록 |
| T3 | 셀 32 nodes/final_answer_node.py | final_answer_node | 차단 message 출력 분기 |
| T1 | 셀 47 (실행) | run_turn 턴3 | 기대 동작 주석 |
| ~~T4~~ | ~~셀 29 / 끝 테스트 셀~~ | ~~`_REPREDICT_PATTERNS`·repredict·테스트 14~16~~ | **철회·전부 제거**(T5) |
| T5 | 셀 8 contracts/routing.py | GateReport | reason 주석에서 `no_control_authority` 제거 |
| T5 | 셀 29 gates/input_gate.py | input_gate, REASON_MESSAGES, `_GUARDRAIL_SYS` | 2층 화이트리스트에서 `no_control_authority` 제거(4b PASS) + 프롬프트를 제어·승인=ok로 + REASON_MESSAGES 정리 + T4 revert |
| T5 | 끝 테스트 셀 | 테스트(시뮬레이션) | `call_llm_json` 임시 교체로 "no_control_authority여도 PASS / out_of_scope·gibberish는 차단" 결정론 검증 |

## 라우팅 검증 (T3 Step 1)

셀 37 `route_after_input` 확인 결과:

```python
def route_after_input(state) -> str:
    rep = _last_report(state, "input_gate")
    return "context_manager" if rep and rep["status"] == "PASS" else "final_answer"
```

- `status == "PASS"` 기준으로만 분기 — `input_flags` 참조 없음. **코드 변경 불필요 (확인만).**
- `supervisor`의 `flags` 사용은 intent 분류용이며 Supervisor 담당자 소관(후속 참조).

## 마커 현황 (`GUARDRAIL-260619`)

| 종류 | 수 |
| --- | --- |
| 여는 마커 (open) | 15 |
| 닫는 마커 (END) | 9 |

태스크별: T1 5/3 · T2 2/2 · T3 4/4 · **T4 0/0(철회·제거)** · **T5 4/0**.
여는 15개 중 9개는 `END` 쌍이 있고, 나머지 6개는 **단일 주석 라인**(`END` 불필요):
- T1: 셀 47 `T1 (데모)` 주석, 끝 테스트 셀 `T1/T2/T3/T5 DoD 검증` 헤더 (2개)
- T5: 셀 29 REASON_MESSAGES·`_GUARDRAIL_SYS`·화이트리스트 주석 3개 + 테스트 셀 `T5 통과 전환 검증` 헤더 (4개)

T4 마커는 0건(완전 제거). T1/T2/T3/T5 모두 마커 존재 확인. 구조상 불균형 없음.

## 테스트 결과 (오프라인 StubLLM, 13케이스)

실행 명령: `jupyter nbconvert --to notebook --execute --ExecutePreprocessor.timeout=300 --output _exec_v2.ipynb manufacturing_agent_v2.ipynb`

```
 # block                 reason  layer  결과
 1  True                  empty  regex  PASS
 2  True              injection  regex  PASS
 3  True              injection  regex  PASS
 4  True              injection  regex  PASS
 5  True              injection  regex  PASS
 6 False                     ok   none  PASS
 7 False                     ok   none  PASS
 8 False                     ok   none  PASS
 9 False                     ok   none  PASS
10 False                     ok   none  PASS
11 False                     ok   none  PASS
12 False                     ok   none  PASS
13 False                     ok   none  PASS
final_answer passthrough: PASS

✅ 가드레일 오프라인 DoD 13/13 + final_answer passthrough 통과
```

**폴백 회귀 요약:**
- 케이스 1~5: block=True, layer=regex (1층 정규식 차단) — 인젝션·빈값 정상 차단
- 케이스 6~9: block=False, layer=none (2층 skip, StubLLM 환경) — 1층 통과분 정상 통과
- 케이스 10~13: block=False, layer=none — 경계선 자문/실제 제조 질문 모두 통과
- `final_answer passthrough`: PASS — `final_answer_node`가 `GateReport.message` 그대로 출력 확인

## ~~테스트 결과 (T4 재예측 휴리스틱)~~ — 철회됨 (T5로 대체)

> ⚠️ T4(재예측 휴리스틱)와 그 테스트(14~16)는 **철회**되어 노트북에서 제거됨. 아래는 *시도했던 기록*이며 현재 코드에는 없다. 프로브로 확인한 사실(체크포인터가 직전 state를 `input_gate`에 전달)은 여전히 참이나, **T5에서는 그 맥락을 의도적으로 사용하지 않는다**(stateless).

## 테스트 결과 (T5 제어·승인 4b 통과 전환, 오프라인·결정론적)

> 실제 over-block은 라이브 LLM 2층에서만 재현되므로, `call_llm_json`을 임시 교체(시뮬레이션)해 "2층이 어떤 verdict를 내든 가드레일이 어떻게 처리하나"를 결정론적으로 검증.

```
✅ 가드레일 오프라인 DoD 13/13 + final_answer passthrough 통과

[T5] 제어·승인(4b) 통과 전환
  PASS  4b 제어명령 통과(설비 정지시켜)     (2층이 no_control_authority 반환해도 → block=False, ok, none)
  PASS  4b 승인요청 통과                    (block=False, ok, none)
  PASS  out_of_scope 여전히 차단            (2층 out_of_scope → block=True, llm)
  PASS  gibberish 여전히 차단               (2층 gibberish → block=True, llm)
✅ T5 제어·승인(4b) 통과 전환 4/4 통과
```

- **핵심 검증**: 2층 LLM이 `{"block":true,"reason":"no_control_authority"}`를 반환해도 화이트리스트(`gibberish`/`out_of_scope`)에 없으므로 **차단되지 않고 PASS**. 반대로 `out_of_scope`/`gibberish`는 여전히 차단(회귀 안전).
- **회귀 불변**: 기존 13케이스 + final_answer passthrough 그대로 통과(1~5 차단 / 6~13 통과).
- **T4 잔재 0건**: `_REPREDICT_PATTERNS`·`repredict`·`repredict_followup`·T4 마커 모두 제거 확인.

## 알려진 한계 / 후속

- **실제 LLM 2층 미검증**: API 키 없음 → StubLLM 회귀 + verdict 시뮬레이션으로만 DoD 고정. 실제 키 환경에서 gibberish/out_of_scope 케이스(6·7) 라이브 차단, 4b(8·9) 라이브 통과 수동 검증 필요.
- **4b는 가드레일 책임 아님(설계상 수용)**: `"설비 정지시켜"` 같은 순수 제어 명령은 가드레일 통과 후 일반 응답으로 처리됨. 위험 부분집합(`"재가동해"` 등)만 하류 **SafetyGate**(`FORBIDDEN_PATTERNS`)가 차단. 소프트 리다이렉트 안내가 필요하면 하류/Supervisor 소관(범위 밖).
- **리뷰 보류(Minor)**: ① 셀 29 2층 주석에 철회된 T4 역사 prose 잔존(가독성) ② `_GUARDRAIL_SYS`가 `input_gate` 함수 내 정의(pre-existing, 매 호출 재생성). 둘 다 동작 무관 — 최종 리뷰에서 triage.
- **over-block 의심**: 경계선 자문(케이스 10/11) — 실제 LLM 프롬프트 튜닝 여지 있음.
- **Supervisor 담당자에게 (dead-code 주의)**: 셀 37 `supervisor`의 `intent="general"` 분기 및 `route_after_supervisor`의 `general→evidence` 직행 경로는, 서비스 범위 밖 질문이 입력 단에서 차단되므로 사실상 죽은 코드(dead code). 정리 필요 — 이 작업 범위 밖.

---

## T6 (질문 없는 구조화 피처 입력) — 추가

### 변경 요약

질문 텍스트 없이 **설비 피처(`state["input_features"]`, dict)만** 들어오는 입력(카드형 대시보드에서 질문 비우고 공정데이터만 전송, 또는 다른 화면/시스템이 센서 dict를 던지는 경우)을 기존엔 `empty`로 **오차단**했다. T6는 이를 **features-only 통과**(2층 LLM skip)로 바꾸되, 인젝션 텍스트+피처면 인젝션이 우기고(우회 불가), 체크포인터가 피처를 턴 간 영속하는 **stale 리스크는 `run_turn` 리셋으로 방어**한다.

- **State (셀: ManufacturingState)**: `input_features: Optional[dict]` 채널 신설. 왜: LangGraph가 채널로 받아 `input_gate`에 전달하려면 TypedDict 선언 필수.
- **input_gate (셀 29)**: `has_features = bool(features)` → 빈 텍스트+피처면 `features_only=True`로 통과(`empty` 차단 대신), 2층 가드를 `if not block and not features_only:`로(빈 입력이 라이브 LLM에서 gibberish로 오차단되는 것 방지), `details["features_only"]` 기록, `blocked_by_raw_input`을 `(텍스트 없음 AND 피처 없음)`으로 정확화. 왜: 피처만 와도 진단은 본업이므로 통과시키고, 인젝션 우선·2층 비용 0 유지.
- **run_turn (셀 47 실행)**: `input_features=None` 파라미터 + `state_in`에 **매 턴 항상 기록**(None 포함). 왜: 체크포인터(thread_id=session_id)가 직전 턴 피처를 영속 → 호출자 미관리 시 빈 턴이 stale 피처로 오통과. `gate_reports: []` 리셋과 동일한 "매 턴 리셋" 패턴으로 구조적 차단.

### 변경 지점 목록 (마커 `T6`, 9건 — 전부 단일 라인)

| 셀(경로 주석) | 심볼 | 변경 내용 |
| --- | --- | --- |
| ManufacturingState | `input_features` | `Optional[dict]` 채널 선언 |
| 셀 29 gates/input_gate.py | `input_gate` (features 추출) | `features=state.get("input_features")`, `has_features=bool(features)` |
| 셀 29 gates/input_gate.py | `input_gate` (`blocked_by_raw_input`) | `(not msg.strip()) and not has_features` 로 정확화 |
| 셀 29 gates/input_gate.py | `input_gate` (`features_only` 초기화) | `features_only = False` |
| 셀 29 gates/input_gate.py | `input_gate` (빈 텍스트 분기) | 피처 있으면 `features_only=True` 통과, 없으면 `empty` 차단 |
| 셀 29 gates/input_gate.py | `input_gate` (2층 가드) | `if not block and not features_only:` (features-only는 2층 skip) |
| 셀 29 gates/input_gate.py | `input_gate` (details) | `_details["features_only"]=features_only` 기록 |
| 셀 47 (실행) | `run_turn` | `input_features=None` 파라미터 + state_in 매 턴 기록 |
| 끝 테스트 셀 (guardrail-t6-test-260619) | `_ig_t6`/`T6_CASES`/케이스21 | T6 DoD 검증 셀 헤더 |

### 마커 현황 갱신 (`GUARDRAIL-260619`)

- 전체 여는 마커: 기존 15 + **T6 9 = 24** (T6는 전부 단일 라인 주석이라 `END` 쌍 불필요).
- 태스크별: T1 5/3 · T2 2/2 · T3 4/4 · T4 0/0(철회) · T5 4/0 · **T6 9/0**.

### 테스트 결과 (오프라인 StubLLM)

검증: `.env`를 잠시 옮겨 키 없이(StubLLM) `jupyter nbconvert --to notebook --execute` 실행 후 `.env` 복원.

```
[T6] 질문 없는 구조화 피처 입력
  PASS  block=False reason=ok     layer=none features_only=True    # 17 빈+피처 → features-only 통과
  PASS  block=True  reason=empty  layer=regex features_only=False  # 18a 빈+{} → empty
  PASS  block=True  reason=empty  layer=regex features_only=False  # 18b 빈+미지정 → empty
  PASS  block=False reason=ok     layer=none features_only=False   # 19 텍스트+피처 → 텍스트 경로
  PASS  block=True  reason=injection layer=regex features_only=False  # 20 인젝션+피처 → 우회 불가
[T6] stale 방어(턴2 empty 차단): PASS  block=True reason=empty     # 21 통합: 턴2 빈입력이 직전 피처 미상속
✅ T6 질문 없는 구조화 피처 입력 6/6 통과
```

- **회귀 불변**: 기존 13 offline + final_answer passthrough + T5 4/4 모두 PASS(동작 변화 0 — 기존 테스트는 `input_features` 없이 호출 → `has_features=False` 경로).
- **케이스 18 커버리지**: `{}`(18a)·키 미존재(18b)·값 `None`(케이스 21 통합, `run_turn(input_features=None)`) 세 형태 모두 검증됨.

### 독립 리뷰 (서브에이전트 2종): Approved

- **Spec 적합성**: 검증 항목 10개(케이스 17~21·2층 가드·details·stale·회귀·마커·enum) 전부 충족. Critical/Important 0.
- **코드 품질**: stale 방어 실효성을 LangGraph로 **실증** — 키 생략 시 직전 피처 생존(버그), `None` 명시 기록 시 채널 덮어써짐(방어 성공) 확인. 케이스 21이 실동작을 검증함. Critical 0.

### 알려진 한계 / 후속

- **통과는 되나 현 데모는 피처를 진단에 *주입하지 않음***(리뷰 반영): `context_manager`/`prediction_agent`는 `user_message` 텍스트에서만 값을 추출하고 `input_features`는 읽지 않는다. 따라서 features-only 입력은 가드레일을 통과하지만 빈 텍스트라 누락값 응답(`[예측 불가]…`)이 나갈 수 있다. **이는 스펙상 의도된 범위 분리**(피처를 진단에 쓰는 배선 = 외부 담당자 소관)이며 버그가 아니다. T6 책임은 "통과/차단 판정"까지.
- **타입 변동 시 코드 수정 필요**: `input_features`는 외부 소유라 dict 외(Pydantic/dataclass/JSON 문자열/list)로 바뀔 수 있다. 현재 `has_features = bool(features)`는 비어있지 않은 dict를 전제하므로, 타입이 바뀌면 `input_gate`의 `has_features` 한 줄(+`ManufacturingState`/`run_turn` 타입 힌트)을 새 타입에 맞게 동기화해야 한다(예: Pydantic은 모든 필드 None이어도 truthy → 오통과). 외부 담당자 변경과 동기 진행.
- **피처 값 자체 미검사**: features-only는 2층을 건너뛰므로 피처 값 이상치/악성 문자열은 guardrail이 보지 않음. 값 검증은 하류(`prediction_gate`/예측) 책임.
- **라이브 LLM 무관**: T6는 전적으로 결정론적(피처 dict 존재 여부)이라 StubLLM/오프라인에서 완전 검증된다(T2/T5의 라이브-검증 잔여와 달리 미해결 항목 없음).
