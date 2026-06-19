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
