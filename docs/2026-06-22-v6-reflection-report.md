# 내 작업(T1~T6)의 v6 반영 분석 보고서

> 작성일: 2026-06-22
> 비교 대상: 내 작업 `0616/manufacturing_agent_v2.ipynb` ↔ 팀장 재작성본 `0622/manufacturing_agent_v6.ipynb`
> 근거 문서: `0616/docs/2026-06-19-input-guardrail-changes.md`(내 작업기록), `0622/2026-06-19-v5-input-guardrail-handoff.md`(보관용), `0622/NOTEBOOK_GUIDE.md`, `0622/README.md`

---

## 0. 핵심 전제 — v6는 부분 수정이 아니라 전면 재작성

`manufacturing_agent_v6.ipynb`는 내 `v2`를 이어받아 고친 것이 아니라 아키텍처를 통째로 재설계한 버전이다. 따라서 내 작업은 "코드가 그대로 이식"되었다기보다 **개념/설계 의도가 v6 구조로 흡수되었거나, 일부는 의도적으로 폐기**되었다.

| 변화 | v2 (내 작업) | v6 (팀장) |
|---|---|---|
| 게이트 구조 | `input_gate` + 하류 `SafetyGate` 분리 | **단일 `intake_gate`**(입력+위험실행 안전 통합) + 뒤단 `output_safety_gate` 신설 |
| LLM 전제 | StubLLM 오프라인 폴백 지원 | **실제 LLM 전제, 키 없으면 명시 실패**(StubLLM 0건) |
| 세션 식별 | `session_id` 중심 | `thread_id`(checkpointer) + `user_id`(Store) 분리 |
| 컨텍스트 | 이전 feature 자동 병합 경향 | `ContextMode`로 명시 재사용만 허용 |

전체 그래프:
```
intake_gate → context_manager → supervisor_planner → orchestrator_dispatcher
  → prediction/sql/evidence agent → worker gates → (supervisor_replanner)
  → final_answer → output_safety_gate → memory_writer → END
```

---

## 1. T1~T6 반영 요약표

| 태스크 | 반영 상태 | v6에서의 모습 |
|---|---|---|
| **T1** 1층 정규식 | 🟢 반영(흡수) | `INJECTION_PATTERNS`·`detect_injection` 존재, `intake_gate`가 빈값·인젝션 `layer="regex"` 즉시 차단 |
| **T2** 2층 LLM | 🟡 부분 반영(변형) | LLM 판정(`gibberish`/`out_of_scope`)은 유지, **StubLLM 폴백 제거** → 파싱 실패 시 `HUMAN_HANDOFF` 차단 |
| **T3** 리포트 재설계 | 🟢 반영(흡수) | `GateReport`에 `block/block_reason/layer/message/flags/diagnostics` 존재, flags는 diagnostics로 강등 |
| **T4** 재예측 휴리스틱 | ⚪ 해당 없음 | 내가 이미 철회, v6에도 흔적 0건(정상) |
| **T5** 제어·승인 4b 통과 | 🔴 설계 자체 교체 | `no_control_authority` 제거(0건)는 일치하나, "4b PASS+하류차단" 철학이 "intake 직접차단+output 이중화"로 반전 |
| **T6** features-only 통과 | 🟢 반영(흡수·강화) | `has_fields=bool(input_features)` 통과, `run_turn`이 매턴 기록(stale 방어), **진단 주입까지 배선** |

---

## 2. 태스크별 상세

### 🟢 T1 — 반영됨 (개념·코드 흡수)
- `INJECTION_PATTERNS`(한/영 변형 다수), `detect_injection()`이 셀 15에 존재.
- `intake_gate`에서 `not has_text and not has_fields → empty`, `flags.is_injection → injection`을 **`layer="regex"`로 즉시 차단**. 1층 정규식 안전망 그대로.

### 🟡 T2 — 부분 반영 (LLM 2층 유지, 폴백 폐기)
- 2층 LLM 개념은 `_llm_intake()`로 유지. `input_reason` enum이 `none|empty|injection|gibberish|out_of_scope`로 내 enum과 동일.
- **단, StubLLM 오프라인 폴백은 v6에 0건.** 내 설계는 "LLM 실패/StubLLM이면 정규식-only 통과"였으나, v6는 실제 LLM 전제라 파싱 실패 시 `HUMAN_HANDOFF`(차단)로 떨어진다. → 폴백 방향이 "통과"에서 "차단"으로 반전.

### 🟢 T3 — 반영됨 (리포트가 판단을 설명만)
- `GateReport`에 `block`, `block_reason`, `layer`, `message`, `flags`, `diagnostics` 모두 존재.
- `intake_gate`가 PASS/BLOCK과 `message`(안내문)를 리포트에 기록 → 라우팅은 `status`로만, 리포트는 설명용. T3 의도 유지.

### ⚪ T4 — 해당 없음
- T5에서 이미 철회·제거, v6에도 `_REPREDICT_PATTERNS` 등 흔적 0건. 정상.

### 🔴 T5 — 설계 자체가 교체됨 (가장 큰 차이) → §3에서 상세
- 일치: `no_control_authority` v6에 0건.
- 반전: 위험 실행 차단을 하류로 위임하던 철학이, intake 직접 차단 + output_safety_gate 이중화로 뒤집힘.

### 🟢 T6 — 반영됨 (흡수 + 강화)
- `intake_gate`: `has_fields=bool(state.get("input_features"))`, `not has_text → 통과(layer="pass", 2층 skip)`. features-only 통과 = T6 그대로(`features_only` 플래그명만 `has_fields`로).
- 인젝션+피처: 텍스트 있으면 injection 정규식이 잡음(우회 불가) — 동일.
- stale 방어: `run_turn(input_features=None)`이 매턴 `"input_features": input_features or None`을 항상 기록 → "매턴 리셋" 방어 이식.
- **추가 강화**: 내 T6 한계("통과는 되나 진단 주입 안 됨")를 v6 `run_turn`이 `effective_msg`로 피처를 텍스트 합성해 진단까지 배선함으로써 해소.

---

## 3. T5 설계 교체 심층 분석

### 3.1 내 T5가 결정했던 것
- 2층 차단 화이트리스트를 `("gibberish","out_of_scope")`로 한정, `no_control_authority` 전부 제거.
- 가드레일을 **stateless(raw input only)** 로 유지, **제어·승인(4b)은 일단 PASS**.
- 위험 부분집합("재가동해" 등)은 **하류 SafetyGate(`FORBIDDEN_PATTERNS`)가 차단**.
- 근거: 260619 피드백 — "맥락을 가져와 4b를 거르는 것은 입력 단 책임 과중".

```
입력 단(가드레일): 제어·승인 명령 → 통과
        ↓
하류 SafetyGate: 위험 부분집합만 차단
```

### 3.2 v6가 대신 한 것 — 정반대 방향
**(A) 입력 단 `intake_gate`가 위험 실행을 직접 분류·차단** — `safety_action` 4단계:

| safety_action | 동작 | 예시 |
|---|---|---|
| `ALLOW` | 통과 | 일반 진단·조회·자문 |
| `ANSWER_SAFELY` | 통과하되 제약 | "재가동해도 되나?"(자문) |
| `BLOCK_DANGEROUS_EXECUTION` | **입력 단 차단** | "점검 없이 재가동해"(실행 지시) |
| `HUMAN_HANDOFF` | 책임자 확인 요구 차단 | 정비·교체·LOTO 요청 |

+ deterministic backstop: `_is_forbidden_action(msg)`가 LLM의 `ALLOW`를 뒤집어 `BLOCK_DANGEROUS_EXECUTION`(layer="hybrid").

**(B) 출력 단 `output_safety_gate` 신설**(내 설계엔 없던 계층):
- `OUTPUT_FORBIDDEN_PATTERNS`(정규식) + `_llm_output_safety`(LLM judge)로 final answer의 "점검 없는 재가동/안전장치 우회/경고 무시" 표현을 차단·대체.
- LLM이 통과시켜도 정규식이 재차 잡는 post-LLM backstop.

### 3.3 같은 입력의 행동 차이

| 입력 | 내 T5(v2) | v6 |
|---|---|---|
| "정지시켜야 하나?"(자문) | 가드레일 PASS | `ANSWER_SAFELY` 통과+제약 |
| "점검 없이 재가동해"(실행) | PASS → 하류 SafetyGate 책임 | `intake_gate` **즉시 차단** |
| 위험 표현이 답변에 섞임 | 방어 계층 없음(범위 밖) | `output_safety_gate`가 차단·대체 |

### 3.4 왜 바뀌었나
`2026-06-19-v5-input-guardrail-handoff.md`(보관용)가 명시:
- 고민 지점: "입력 단에서 일찍 차단 vs 하류 safety 계층에서 위험 부분집합만 차단" — 내 T5와 동일한 질문.
- v6 결론: "input/safety를 `intake_gate`로 통합, 위험 실행은 intake에서 차단하거나 안전 자문으로 제한, final_answer 이후 `output_safety_gate`가 재검사."
- 명시 금지: "v5의 `input_gate`/`safety_gate`/`SafetyAgent` 구조를 되살리지 않는다."

→ 내 T5가 의존하던 **하류 SafetyGate 구조 자체가 폐기**되어, "통과-위임" 모델이 성립 불가가 됨.

### 3.5 일치 vs 반전 정리

| 항목 | 상태 |
|---|---|
| `no_control_authority` 제거 | ✅ 일치(결과물 동일) |
| 가드레일 stateless 유지 | ✅ 부분 일치(intake도 raw+직전요약만) |
| 제어·승인(4b) 무조건 PASS | ❌ 반전(자문만 통과, 실행은 차단) |
| 위험 차단을 하류에 위임 | ❌ 반전(하류 폐기, 입력+출력 이중화) |

핵심: `no_control_authority`를 없앤 **결과**는 같지만, 그 **이유와 책임 구조**는 정반대.

---

## 4. T5 교체의 장단점

### ✅ 장점 (v6 방향)
1. **방어 심층화**: 입력·출력 두 지점 × 각 LLM·정규식 이중. 한 계층이 뚫려도 다른 계층이 잡음.
2. **위험 명령 조기 차단**: "점검 없이 재가동해"가 파이프라인 진입 전 멈춤(토큰·지연·사고표면↓).
3. **책임 경계 명문화**: intake=입력 판정, output=최종 표현 검증으로 분리.
4. **출구 보장**: 사용자가 보는 final answer를 직접 검사 → 중간 worker가 만든 위험 표현도 차단.
5. **자문/실행 세분화**: `ANSWER_SAFELY`로 자문은 안전 가이드와 함께 살림.

### ⚠️ 단점 (치른 비용)
1. **입력 단 LLM 의존↑ + 오프라인 검증 불가**: StubLLM 폴백 제거로 키 없으면 동작 안 함. 결정론적 오프라인 검증 자산 상실.
2. **over-block 위험이 입력 단으로 이동**: 260619 피드백이 우려한 "입력 단 책임 과중"이 재활성화. 경계선 자문 오차단 가능.
3. **비용·지연↑**: 정상 입력도 intake LLM + output LLM 추가 호출.
4. **유지보수 표면↑**: `FORBIDDEN_PATTERNS`(intake)와 `OUTPUT_FORBIDDEN_PATTERNS`(output) 두 벌 동기화 필요.
5. **정규식 한계 잔존**: 한국어 표현 변형 취약성은 양쪽 모두 동일.
6. **`intake_gate` 책임 과중**: 입력+인젝션+위험실행+handoff를 한 함수가 판정 → 테스트·디버깅 복잡도↑.

### 균형 평가

| 관점 | 우세 |
|---|---|
| 안전 보장 강도 | v6 |
| 오프라인/결정론 검증 | 내 T5 |
| over-block 회피 | 내 T5 |
| 비용/지연 | 내 T5 |
| 책임 경계 명확성 | v6 |
| 유지보수 단순성 | 내 T5 |

**종합**: 위험 실행이 인명·설비 사고로 직결되는 제조 도메인에서는 v6의 이중화가 합리적 방향. 단, 새로 떠안은 리스크(입력 단 over-block, LLM 부재 시 동작, 두 정규식 동기화)는 명시적 관리 필요.

---

## 5. 후속 권고

1. **over-block 튜닝**: `ANSWER_SAFELY` vs `BLOCK_DANGEROUS_EXECUTION` 경계를 실제 LLM으로 검증(경계선 자문이 안 막히는지).
2. **LLM 부재 동작 명세 고정**: 키 없을 때 fail-closed가 의도인지 명문화.
3. **정규식 커버리지 동기화 점검**: `FORBIDDEN_PATTERNS` ↔ `OUTPUT_FORBIDDEN_PATTERNS` 일치.
4. **회귀 검증 의존처 확인**: 내 T5의 오프라인 결정론 검증이 사라졌으므로 `scripts/run_manufacturing_scenarios.py`의 `S02_dangerous_execution`, `S03_safe_advisory`, `S10_output_safety_direct`, `S14_injection_inside_maintenance_request`가 실제로 도는지 확인.

---

## 부록 — 코드 레벨 확인 근거 (v6)

- `StubLLM` 출현 0건, `features_only` 0건, `no_control_authority` 0건 (grep 확인).
- `INJECTION_PATTERNS`·`detect_injection`: 셀 15.
- `intake_gate`·`FORBIDDEN_PATTERNS`·`_is_forbidden_action`·`safety_action` 4단계·`_llm_intake`: 셀 29.
- `output_safety_gate`·`OUTPUT_FORBIDDEN_PATTERNS`·`_llm_output_safety`: 셀 30.
- `run_turn(... input_features=None)` 매턴 `"input_features": input_features or None` 기록 + `effective_msg` 피처 합성: `run_turn` 정의 셀.
