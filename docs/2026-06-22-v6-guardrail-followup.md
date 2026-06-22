# v6 가드레일 후속 작업 문서 (2026-06-22)

> 담당: 가드레일(입력/출력 안전)
> 대상(가드레일 수정본): `manufacturing_agent_v6_noh000.ipynb` (레포 루트) — G01/G02·정규식 패치가 들어있는 파일, 검증(nbconvert) 대상
> 병합 베이스(원본): `manufacturing_agent_v6.ipynb` (GitHub 원본, 변경 전 상태) — 팀장이 머지해 들어갈 대상
> 선행 분석: `2026-06-22-v6-reflection-report.md`(T1~T6 반영), `2026-06-19-input-guardrail-changes.md`(내 v2 작업기록)
>
> **파일명 분리(2026-06-22)**: 원본/수정본 구분을 위해 작업 노트북을 `manufacturing_agent_v6_noh000.ipynb`로 분리. 본 문서 아래 "1단계 검토 반영" 등 일부 서술은 분리 이전 시점 기록이라 `manufacturing_agent_v6.ipynb`로 표기되어 있으나, 이는 당시 경로 검토 이력이며 실제 코드 변경은 모두 `_noh000` 파일에 있다.
>
> **1단계(계획 검토) 반영(2026-06-22)**: ① 대상 노트북 경로는 `0622/` 폴더가 아니라 **레포 루트** `manufacturing_agent_v6.ipynb`임(0622 폴더 부재 확인) → 본 문서 전반의 `0622/` 표기를 레포 루트로 정정. ② Phase 1 계획을 코드와 대조 검증 완료 — `detect_injection`(셀15), `_is_forbidden_action`·`_normalize_intake_payload`·`_decision_from_intake`·`intake_gate`(셀29), `_contains_unsafe_execution_instruction`·`_normalize_output_safety_payload`(셀30), `sanitize_retrieved_doc`(셀21) **8개 전부 기재된 위치에 현존**, 시그니처 일치, empty/features-only/injection 분기는 `_llm_intake` 호출 이전 return으로 오프라인 직접호출 검증 가능 → **계획 유효**. ③ 코드 마커는 작업지시 규칙에 따라 `# [GUARDRAIL-260622-P1-START]`/`# [GUARDRAIL-260622-P1-END]` 형식 사용(문서의 `# guardrail-260622` 표기를 대체·우선; Phase별 `P1`/`P2` 접미사). Phase 1은 **완료**(G01 신규 셀 2개, 오프라인 43/43 통과). 변경 기록: `docs/2026-06-22-v6-guardrail-phase-changelog.md`.
>
> **Phase 2(계획 검토) 반영(2026-06-22, probe 근거)**: 현 4세트에 적대 코퍼스를 통과시켜 갭을 측정함 → ⓐ intake `_is_forbidden_action`이 "재기동", 중간 부사("점검 없이 **다시** 가동"), 조사+동의어("점검도 안 하고 그냥 **돌려**"), "인터록 해제", "경고무시+**강행**"을 **미포착**. ⓑ "안전장치 **끄고/무시하고** 가동"은 intake=True인데 output=**False**(intake↔output **비대칭** 실재). ⓒ 한글 injection("이전 규칙 무시·시스템 프롬프트 출력", "you are now")은 입력(셀15)=True인데 검색문서(셀21)=**False**(injection 비대칭). **갭 실재 확인 → 계획 유효**. 단 다음 3가지로 설계를 고정한다: (1) **parity는 정규식 세트 동일화가 아니라 "공유 canonical 코퍼스 양쪽 True"**(answer-style 명령문은 이미 양쪽 True로 확인됨; 세트 단일화는 G-4(c)대로 후속 보류). (2) **보강은 위험 맥락 앵커(점검 없이 / 안전장치+우회·해제 / 경고+무시)에서만 확장**하고 맨동사 단독 확장 금지 — 자문/진단 답변("재가동 전에 점검하세요", "재가동하면 위험합니다")은 보강 후에도 output **False** 유지(T03 FP 보호). (3) **Phase 2는 정규식 수정 = 동작 변경**이므로, 진짜 회귀(live T01~T22, 특히 T03 통과·T01/T02/T07/T12 차단)는 **키 필요 → 팀장 실행으로 위임**; 이번 환경에서는 오프라인 G02(parity+FP assert)와 정적 분석으로 검증. 마커는 `# [GUARDRAIL-260622-P2-START]`/`-P2-END]`(기존 셀15/21/29/30 수정 블록 + 신규 G02).
>
> **Phase 2(4단계 코드 리뷰) 반영(2026-06-22)**: FP 정량 probe에서 보강 부작용 1건을 self-catch — 셀30에 새로 넣은 **bare `강행`/`돌려` 종결어가 정당한 안전 경고 답변("안전장치 우회를 강행하면 사고가 납니다")을 output 오차단**(§3(2) FP 보호 원칙 위반). → 셀30을 **명령형 한정**(`강행해|강행하세요`, `다시\s*돌려|돌려라|돌리세요`)으로 교정. 재검증: G01 43/43·G02 36/36 유지, 해당 경고문 output=False로 교정, 비대칭 보강("끄고/무시…가동")은 그대로 유지. 잔여(저위험)는 §6 참조. Phase 2 코드/검증 **완료**, 변경 기록은 `docs/2026-06-22-v6-guardrail-phase-changelog.md`(5단계에서 누적).

---

## 1. 배경

팀장이 `manufacturing_agent_v2.ipynb`를 `v6`로 전면 재작성하면서 입력 가드레일이 단일 `intake_gate`로 통합되고, 뒤단에 `output_safety_gate`가 신설되었다. 내 작업(T1~T6) 중 T1/T3/T6는 v6에 흡수되고, T2는 변형(StubLLM 폴백 폐기), T5는 설계 교체되었다. 본 문서는 **가드레일 담당자 관점에서 v6에 대해 지금 점검·고도화해야 할 항목**과 그 실행 계획을 정리한다.

v6 안전 책임 분리:
- `intake_gate`(셀 29): 입력 가능성 + 프롬프트 인젝션 + 위험 실행 요청 1차 판정 (LLM + 정규식 backstop)
- `output_safety_gate`(셀 30): 최종 답변의 위험 실행 표현 deterministic + LLM 이중 검증
- `evidence_agent`/RAG(셀 21): 검색 문서 내 인젝션 sanitize

> 참고: 노트북은 레포 루트(`manufacturing_agent_v6.ipynb`)에 있고 `0622/` 폴더는 없다. 가드레일 관련 산출물은 노트북 외에 `docs/`의 md 문서들뿐이며 README가 참조하는 `scripts/run_manufacturing_scenarios.py`·`sql/`는 없다. 따라서 **노트북 내장 T01~T22 셀이 유일한 검증 인프라**다.

---

## 2. 식별된 갭 (코드 확인 근거)

### G-1. 오프라인 결정론 검증망 소실 〔심각도: 높음〕
- v6는 StubLLM 폴백을 제거(`StubLLM` 출현 0건). 안전 테스트 셀(T01·T02·T03·T07·T12)이 전부 `run_turn` → `_llm_intake`(LLM 호출)를 탄다.
- → **OpenAI 키 없이는 가드레일 회귀를 검증할 수 없고**, 키가 있어도 비결정·유료. 내 T5/T6의 오프라인 DoD 안전망이 통째로 사라졌다.
- 그러나 핵심 차단 로직은 순수함수다: `detect_injection`(셀15), `_is_forbidden_action`(셀29), `_contains_unsafe_execution_instruction`(셀30), `_normalize_intake_payload`/`_decision_from_intake`(셀29), `_normalize_output_safety_payload`(셀30), `sanitize_retrieved_doc`(셀21). **LLM 없이 검증 가능한데 테스트가 없다.**
- 추가로 `intake_gate`는 `_llm_intake` 호출 **전에** empty/features-only/injection 세 분기를 먼저 return → 노드 직접 호출로 결정론 검증 가능.

### G-2. 정규식 4세트 분산·비동기화 〔심각도: 높음〕
| 세트 | 위치 | 역할 |
|---|---|---|
| `INJECTION_PATTERNS` | 셀15 | intake 입력 인젝션 |
| `FORBIDDEN_PATTERNS` | 셀29 | intake 위험실행 backstop |
| `OUTPUT_FORBIDDEN_PATTERNS` | 셀30 | output 최종답변 검사 |
| `RETRIEVED_DOC_INJECTION_RE` | 셀21 | RAG 문서 인젝션 sanitize |

- **비대칭 위험**: 위험표현 정의가 intake(`FORBIDDEN_PATTERNS`)와 output(`OUTPUT_FORBIDDEN_PATTERNS`)에 따로 있어 한쪽만 잡으면 "입력 통과/출력 차단"(또는 반대) 구멍. 두 세트의 parity 보장 없음.
- **한국어 변형 취약**: 띄어쓰기·조사·동의어("재가동/재기동/다시 돌려", "우회/해제/끄고/무시")에 약함. v6는 위험 차단을 정규식 backstop에 크게 의존하므로 이 커버리지가 안전 하한선.

### G-3. 테스트 커버리지 공백 〔심각도: 중상〕
`intake_gate` 코드에 존재하나 검증 셀이 없는 분기:
| 미검증 분기 | 코드 근거(셀29) |
|---|---|
| features-only 통과 | `not has_text → layer="pass"` |
| empty 차단 | `not has_text and not has_fields → empty` |
| HUMAN_HANDOFF 차단 | `safety_action == "HUMAN_HANDOFF"` |
| hybrid backstop | `_is_forbidden_action`가 ALLOW→BLOCK 승격, `layer="hybrid"` |
| stale-features 방어 | `run_turn`이 매턴 `input_features` 리셋 (내 T6 케이스21) |

- hybrid backstop은 "LLM이 위험을 놓쳐도 정규식이 잡는다"는 v6 안전 설계의 핵심 보루인데 무검증.

### G-4. intake 엣지케이스 〔심각도: 중, 동작변경 수반〕
- **(a)** `has_fields = bool(features)` — 비-dict(Pydantic 등) 입력 시 모든 필드 None이어도 truthy → 빈 입력이 features-only로 오통과(내 T6 known limitation 잔존).
- **(b)** `_is_forbidden_action` deterministic override가 **LLM이 `ALLOW`일 때만** 작동. 위험명령이 `ANSWER_SAFELY`(통과)로 오분류되면 intake backstop이 개입하지 않고 통과(출력 게이트에만 의존) → intake backstop 커버 범위가 의도보다 좁음.

---

## 3. 후속 작업 계획 (우선순위 1·2·3 + 4 옵션, OpenAI 키 사용 가능)

설계 판단: **Phase 1(오프라인 회귀망)을 먼저 깔아야** Phase 2·3·4의 변경을 즉시 회귀 감지할 수 있다. 1~3은 오프라인 가능분이 많아 키 부담 없이 진행 가능.

### Phase 1 — 오프라인 결정론 회귀망 〔G-1, 키 불필요〕
노트북 끝에 신규 셀 `G01 — 가드레일 오프라인 결정론 회귀(no-LLM)`. 순수함수/노드 직접 호출로 검증:
1. `detect_injection` pos/neg
2. `_is_forbidden_action` pos/neg
3. `_contains_unsafe_execution_instruction` pos/neg
4. `_normalize_intake_payload` (invalid→out_of_scope/HUMAN_HANDOFF, bool/str 강제)
5. `_decision_from_intake` (block reason 매핑, dangerous_request/human_handoff/pass)
6. `_normalize_output_safety_payload` (invalid→policy_violation)
7. `sanitize_retrieved_doc` (치환 + `security_flags`)
8. `intake_gate(state)` 직접 호출 — empty/features-only/injection 결정론 분기

### Phase 2 — 정규식 강건성+동기화 〔G-2, 오프라인 부분 키 불필요 / live 회귀는 키 필요〕
> ⚠️ **정규식 수정 = 동작 변경**(차단 범위 확대, 안전 강화 방향). 오프라인 G02로 parity/FP를 검증하되, live T01~T22 회귀는 팀장 키 실행으로 확인.

코드 보강(마커 `# [GUARDRAIL-260622-P2-START/END]`):
- **셀29 `FORBIDDEN_PATTERNS`**: 재기동, 중간 부사/조사(다시·그냥·도), 동의어(돌려·인터록), 경고무시+강행 — **위험 맥락 앵커 유지**.
- **셀30 `OUTPUT_FORBIDDEN_PATTERNS`**: intake와 대칭이 되도록 끄고/무시하고…가동 계열 포착, 단 **어미 affirmation 요구 유지**(진단 답변 False 보존).
- **셀21 `RETRIEVED_DOC_INJECTION_RE`**: 한글 injection(시스템 프롬프트 출력/공개, 역할 변경, you are now, 개발자 지시)을 셀15 canonical과 동기화.
- **셀15 `INJECTION_PATTERNS`**: 셀21과의 공유 canonical 기준 누락분만 소폭 보완.

신규 셀 `G02 — 정규식 커버리지·parity`(노트북 끝, G01 뒤):
1. 한국어 변형 적대적 코퍼스 구성
2. 4세트 커버리지 측정 출력(놓치는 변형 가시화)
3. **danger parity assert** — answer-style canonical 명령문이 intake(`_is_forbidden_action`) ∧ output(`_contains_unsafe_execution_instruction`) **양쪽 True**
4. **injection parity assert** — 공유 canonical injection 문구가 입력(`detect_injection`, 셀15) ∧ 검색문서(`RETRIEVED_DOC_INJECTION_RE`, 셀21) **양쪽 True**
5. **FP 가드(T03 보호)** — 자문/진단 답변은 output **False**, 일반 질문은 injection **False** (보강 후에도 유지)

> 설계 고정: parity는 "세트 동일화"가 아니라 "공유 canonical 코퍼스 양쪽 True"(intake=요청문/output=지시문 답변으로 표면이 다름; 세트 단일화는 G-4(c)대로 후속 보류).

### Phase 3 — 커버리지 공백 보강 〔G-3, 일부 live LLM〕
- 오프라인분(features-only/empty/injection)은 Phase 1에 포함.
- live(`run_turn`, 키 사용): `T23` HUMAN_HANDOFF, `T24` features-only 통과, `T25` stale-features 방어(T6 케이스21 재현), `T26` hybrid backstop(차단 보장 수준).

### Phase 4 — intake 하드닝 〔G-4, 옵션·동작변경〕
Phase 1 회귀망 위에서:
- (a) non-dict `input_features` 가드(`isinstance(features, dict) and bool(features)`)
- (b) ANSWER_SAFELY도 `_is_forbidden_action`이면 BLOCK 승격(T03 FP 가드로 회귀 확인)
- (c) 4정규식 단일 모듈 통합은 범위 커서 보류(후속 제안)

---

## 4. 검증 방법

1. **오프라인(키 불필요)**: 정의 셀 실행 후 G01/G02만 실행 → `✅ ... 통과`. 빠른 반복.
2. **전체 실행(키 사용)**:
   ```bash
   # 레포 루트에서 실행
   jupyter nbconvert --to notebook --execute --ExecutePreprocessor.timeout=600 \
     --output _exec_v6.ipynb manufacturing_agent_v6_noh000.ipynb
   ```
   기존 T01~T22 무변(T03 자문 통과, T01/T02/T07/T12 차단) + 신규 G01/G02 + T23~T26 통과.
3. **회귀 불변**: Phase 2 정규식 보강 후 자문·일반 진단이 오차단(FP)되지 않는지 재확인.

---

## 5. 산출물

- 노트북 신규 셀: `G01`(오프라인 결정론, **완료**), `G02`(정규식 parity), `T23~T26`(live 보강).
- 정규식 보강 diff(셀15/21/29/30), 마커 `# [GUARDRAIL-260622-P2-START/END]`.
- (옵션) Phase 4 코드 하드닝 + 변경 요약.
- 본 문서 갱신(작업 후 테스트 결과·잔여 후속 추가).
- 페이즈별 변경 기록 누적: `docs/2026-06-22-v6-guardrail-phase-changelog.md`.

---

## 6. 잔여 후속 / 메모

- 4정규식 단일 모듈 통합(유지보수성) — 1일 범위 밖, 별도 작업 제안.
- live-LLM 의존 안전 테스트의 결정론화 — Phase 1 순수함수 테스트가 부분 대체하나, LLM 판정 자체(gibberish/out_of_scope/safety_action 분류)의 회귀는 여전히 키 필요. 골든셋 캐싱/녹화 응답 방식은 후속 검토.
- `output_safety_gate`의 LLM judge 프롬프트(`OUTPUT_SAFETY_SYS`) 튜닝 여지 — over/under-block 경계는 live 데이터로 별도 측정.
- **(Phase 2 잔여, 저위험) intake backstop 부정문 FP**: 보강된 셀29 pattern1의 가변 갭(`[\s가-힣]{0,8}`)이 "점검 없이는 …가동이 불가합니다" 같은 부정/진단 진술을 `_is_forbidden_action`=True로 잡음(같은 맥락에서 "…강행하면 사고가 납니다"도 intake=True). 단 output=False이고 intake backstop은 **LLM=ALLOW일 때만** 승격되는 게이트 경로라 실사용 영향 낮음. 부정문 가드(`안 됩니다/불가/위험합니다` 등 종결 시 제외)는 후속 검토.
- **(기존, 신규 아님) 검색문서 RE over-redaction**: 셀21 `RETRIEVED_DOC_INJECTION_RE`의 `규칙\s*무시` 등은 정상 안전문서("안전 규칙 무시 시 사고…")도 flag/redaction 할 수 있음(Phase 2 이전부터 존재). 영향은 매칭 span 치환에 국한.
