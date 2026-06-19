# Input Guardrail 수정 결과 보고서 (2026-06-19)

대상: `manufacturing_agent_v2.ipynb` (원본 `manufacturing_agent.ipynb` 복사본)

## 변경 요약 (T1/T2/T3)

- **T1 (1층 정규식)**: 셀 16 `INJECTION_PATTERNS` 한/영 변형 보강, 셀 29 `input_gate`에서 빈 값·인젝션 즉시 차단. 왜: 비용 0·오프라인 안전망.
- **T2 (2층 LLM)**: 셀 6 `call_llm_json`, 셀 29 2층 판정(gibberish/out_of_scope/no_control_authority). 왜: 정규식이 못 잡는 모호 케이스. 실패/StubLLM이면 정규식-only 폴백.
- **T3 (리포트 재설계)**: 셀 8 `GateReport`에 block/reason/layer/message, 셀 29 flags→details 강등, 셀 32 `final_answer_node`가 message 출력. 왜: 리포트가 판단을 *구동*하지 않고 *설명*.

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
| 여는 마커 (open) | 11 |
| 닫는 마커 (END) | 9 |

여는 마커 11개 중 9개는 `END` 쌍이 있음. 쌍이 없는 2개:
- **셀 47** `T1 (데모)`: 단일 주석 라인 — `END` 불필요(범위 없음).
- **셀 54** `T1/T2/T3 DoD 검증`: 테스트 셀 전체가 범위 — 셀 끝이 닫힘으로 간주.

T1/T2/T3 세 태스크 모두 마커 존재 확인. 구조상 불균형 없음.

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

## 알려진 한계 / 후속

- **실제 LLM 2층 미검증**: API 키 없음 → StubLLM 회귀로만 DoD 고정. 실제 키 환경에서 gibberish/out_of_scope/no_control_authority 케이스(6~9) 수동 검증 필요.
- **over-block 의심**: 경계선 자문(케이스 10/11 — "토크 60으로 올리면 위험해?", "지금 멈춰야 할까?") — 실제 LLM 프롬프트 튜닝 여지 있음.
- **Supervisor 담당자에게 (dead-code 주의)**: 셀 37 `supervisor`의 `intent="general"` 분기(`flags.possible_manufacturing_query`/`possible_safety_query` 모두 False 조건) 및 `route_after_supervisor`의 `general→evidence` 직행 경로는, 서비스 범위 밖 질문이 이제 입력 단(input_gate)에서 먼저 차단되므로 사실상 죽은 코드(dead code)가 됨. 정리 필요 — 이 작업 범위 밖.
