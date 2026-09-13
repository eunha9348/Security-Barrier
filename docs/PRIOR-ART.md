# Prior Art & Research Survey
## Security Barrier — 사내 보안문서 AI 유입 차단 / 흔적 추적 및 폐기 스킬

조사일: 2026-09-13
조사 범위: Claude Skill 마켓플레이스(4곳), GitHub 큐레이션 목록, 상용 AI-DLP 제품, arXiv 논문

---

## 0. 한 줄 결론

**동일한 스킬은 존재하지 않는다.** 다만 기능을 4개 층위로 쪼개면 3개 층위는 이미 포화 상태이고,
**진짜 빈 공간은 "유입 이후의 양방향 추적 + 파생물 전파 삭제 + 검증 가능한 폐기 증적"** 한 층위뿐이다.
그리고 요구사항 중 **"LLM이 그 문서를 잊도록 한다"는 스킬 레이어에서 구현 불가능**하다 (§2 참조).
이 점을 설계에 반영하지 않으면 스킬 자체가 Hallucination 생성기가 된다.

---

## 1. 중복 판정 (층위별)

### 층위 A — 사전 차단 (입력 DLP): 🔴 심각한 중복, 진입 비권장

| 이름 | 형태 | 겹치는 부분 |
|---|---|---|
| [Nightfall AI](https://www.nightfall.ai/) | 상용 | 프롬프트/파일 업로드를 모델 도달 전 실시간 검사·차단. Claude 포함 |
| [Prompt Security](https://www.cbinsights.com/compare/lasso-security-vs-prompt-security) / [Lasso Security](https://www.lasso.security/) | 상용 | GenAI 인터랙션 모니터링, 데이터 유출 차단 |
| [Microsoft Purview DLP for Copilot](https://learn.microsoft.com/en-us/purview/dlp-microsoft365-copilot-location-learn-about) | 상용 | 민감도 레이블 기반으로 프롬프트를 **모델 도달 전 차단**. 2026-07 라벨 적용 확대 |
| [Strac](https://www.strac.io/blog/claude-dlp) / [dope.security](https://dope.security/post/claude-dlp-stop-data-leaks-2026) / [Aona](https://aona.ai/solutions/dlp-for-claude/) | 상용 | Claude 전용 DLP를 이미 제품명으로 판매 중 |
| [Cloud DLP skill](https://mcpmarket.com/tools/skills/cloud-data-loss-prevention-dlp) | Claude Skill | AWS/Azure/GCP DLP 자동화, 분류 라벨링, 마스킹/토큰화 |
| [sanitize](https://github.com/BehiSecc/awesome-claude-skills) (agentward-ai) | Claude Skill | 15개 카테고리 PII/API키 로컬 탐지·마스킹 |
| [ironclaw-agent-guard](https://github.com/wd041216-bit/ironclaw-agent-guard) | Claude Skill | 위험 툴 호출 리뷰, 프롬프트 인젝션, 시크릿 레닥션, 감사 로그 |
| [varlock-claude-skill](https://github.com/wrsmith108/varlock-claude-skill) | Claude Skill | 시크릿이 세션·터미널·로그·커밋에 안 나타나게 |
| [shellward-security-guide](https://github.com/jnMetaCode/shellward) | Claude Skill | 프롬프트 인젝션 방지 + DLP + PII 스캔 가이드 |

> **판정**: "탐지해서 막는다"만 만들면 100% 중복이다. 차별화 없음.

### 층위 B — 정책 기반 1차 판단 (= "보안 Axis"): 🟡 개념 중복, 구현 형태는 차별화 여지 있음

- [policy-opa skill](https://www.skillsdirectory.com/skills/agentsecops-policy-opa) — OPA/Rego 정책 집행을 스킬화. SOC2/PCI-DSS/GDPR/HIPAA 대응
- rego-skill — RBAC/ABAC 패턴 114개 테스트 포함 Rego 개발 스킬
- Purview 민감도 레이블 — "이 라벨이 붙은 문서는 AI 처리 제외" 라는 판단축이 이미 상용화됨
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks) — `UserPromptSubmit`(프롬프트 거부), `PreToolUse`(툴 호출 차단, exit code 2). **권한 모드보다 먼저 실행되며 `bypassPermissions`로도 못 넘김**

> **판정**: "정책 파일 기반 판단"은 새롭지 않다. 단, **"회사 인가자만 수정 가능한 서명된 정책축"을
> 에이전트가 읽되 절대 못 고치게 강제하는 구조**는 스킬로 패키징된 사례를 찾지 못했다.
> "Security Axis" / "Policy Axis"는 기존 표준 용어가 아니다 → 용어 선점 가능.

### 층위 C — 유입 이후 양방향 추적 & 파생물 전파 삭제: 🟢 **빈 공간. 여기가 핵심**

연구는 활발하지만 **에이전트 스킬로 구현된 사례 없음**:

- **파생물 문제**: 원본 문서를 지워도 그것이 낳은 청크·임베딩·요약·캐시된 답변·리랭커 입력은 안 지워진다.
  삭제는 모든 파생 아티팩트로 **전파**되어야 하며, 각 파생물은 원본으로 되돌아가는 안정적 참조를 들고 있어야 한다.
  ([Oracle](https://blogs.oracle.com/developers/how-to-detect-rag-index-drift-deleted-docs-stale-chunks-and-duplicate-embeddings), [vdf.ai](https://vdf.ai/blog/data-deletion-private-rag-retention-erasure/))
- **Forward/Backward Lineage**는 데이터 거버넌스에서 이미 확립된 개념이다
  (backward = 근본원인 분석/사고 대응, forward = 영향 분석). 사용자가 말한 "Reverse/Forward Tracing 병렬 수행"은
  **데이터 리니지의 양방향 분석을 LLM 컨텍스트에 적용한 것**으로 보면 된다. → 개념은 기존, 적용 대상이 새로움.
- 가장 가까운 선행: **MemLineage** (arXiv:2605.14421) — LLM 에이전트 메모리의 리니지 기반 집행.
  삭제를 파괴적으로 하지 않고 **tombstone 마커**(원본 엔트리 id + 삭제 사유)를 남겨 포함 증명 가능하게 유지.
- 실무에서 가장 가까운 기존 도구 조합: `gitleaks`/`trufflehog`(탐지) + `git-filter-repo`/BFG(히스토리 삭제) + 키 로테이션.
  Claude Code로 10년치 히스토리에서 121개 시크릿을 제거한 [사례](https://dev.to/yureki_lab/how-i-purged-121-leaked-secrets-from-10-years-of-git-history-with-claude-code-37dk)는 있으나 애드혹이고 스킬화되지 않음.

> **판정**: 중복 없음. 다만 "새로 발명"이 아니라 **"리니지 + tombstone + 감사"라는 기존 연구 패턴을
> 에이전트 워크스페이스(전사/메모리파일/툴로그/git)로 옮기는 것**으로 포지셔닝해야 정직하다.

### 층위 D — "LLM이 스스로 잊게 만들기": ⛔ 스킬로 구현 불가

§2 참조. 연구 영역(machine unlearning)이며 가중치 접근이 필요하다.
스킬(=프롬프트+스크립트)은 모델 파라미터도, 제공자 서버측 로그도 건드릴 수 없다.

---

## 2. 기술적 현실 체크 — "Hallucination 절대 금지" 요구사항과 직결

사용자 요구: *"LLM이 자체 폐기하고 (…) LLM이 그 문서를 잊도록"*
→ **이 문장을 그대로 구현하면 스킬이 거짓말을 하게 된다.** 이유:

1. **컨텍스트는 append-only다.** 이미 모델에 들어간 토큰을 소급해서 "안 본 것"으로 만들 수 없다.
   할 수 있는 건 (a) 세션 폐기, (b) 서버측 컨텍스트 편집(`clear_tool_uses_20250919`)으로 **이후 턴에서 제외**,
   (c) 전사/메모리 파일 삭제뿐이다. ([Context editing 문서](https://platform.claude.com/docs/en/build-with-claude/context-editing))
2. **제공자 측 보존은 스킬 밖이다.** 프롬프트가 트레이싱 백엔드에 들어갔다면 그 시스템의 보존정책·접근제어·익스포트 연동을 따른다.
   계약(ZDR) 영역이지 스킬 영역이 아니다.
3. **크리덴셜은 되돌릴 수 없다.** "모델을 통과한 크리덴셜은 un-leak 할 수 없다. 회전은 가능하지만 사본은 회전 못 한다."
   → 보안문서/시크릿 유입 시 **정답은 삭제가 아니라 "노출 가정 + 회전/권한회수"**다.
4. **unlearning은 검증 자체가 미해결이다.** (ε,δ)-인증은 "잊었다"를 직접 검증하지 않고 참조 모델과의 구분불가능성만 정량화한다.
   기존 검증 스킴은 보조 계측에 의존하거나 파이프라인 변경에 취약해 스푸핑이 쉽다.

### ⇒ 설계에 반영해야 할 규칙 (Hallucination 0 을 위한 강제 조항)

| 규칙 | 내용 |
|---|---|
| R1. 1차 판단은 결정론적으로 | Security Axis 매칭은 **정규식/해시/라벨/경로 규칙**으로 수행. LLM은 1차 판단에 관여 금지. LLM은 "결정론적으로 걸린 후보"에 대한 2차 문맥 판단만 |
| R2. 증거 없으면 미판정(abstain) | 파일 경로·라인번호·해시 같은 **인용 가능한 증거 없이는 "유출됨/삭제됨" 어느 쪽도 단정 금지**. 출력 스키마에 `UNVERIFIED` 상태 필수 |
| R3. 삭제는 검증으로 닫는다 | 삭제 후 동일 탐지기를 재실행해 0 hit 확인 + 삭제 전후 해시 기록. 검증 실패 시 "삭제됨" 보고 금지 |
| R4. "잊었다"고 말하지 않는다 | 보고서는 `모델 망각` 대신 **`컨텍스트 격리 / 아티팩트 폐기 / 자격증명 회전 필요`** 로만 표현. 모델 가중치·제공자 로그는 **out-of-scope로 명시** |
| R5. Axis는 불변 입력 | 정책 파일은 읽기 전용 + 서명(GPG/sigstore) + CODEOWNERS. 에이전트가 Axis를 수정하는 경로 자체를 차단 |
| R6. 미도달 범위 명시 | 리포트 말미에 **"추적 불가 영역"**(제공자 서버 로그, 타인 세션, 이미 전송된 외부 SaaS)을 항상 열거. 조용히 생략하면 그것이 곧 hallucination |

---

## 3. 관련 논문 리스트

> ⚠️ 각 항목은 검색 메타데이터로 존재를 확인했다. 전문을 읽고 검증한 것은 아니므로,
> 인용 전 원문 확인 필요. arXiv ID가 26xx인 것은 2026년 발표분이다.

### 3-1. LLM Unlearning — 기본기 / 벤치마크
| ID | 제목 | 왜 필요한가 |
|---|---|---|
| [2401.06121](https://arxiv.org/abs/2401.06121) | TOFU: A Task of Fictitious Unlearning for LLMs | 망각 평가의 사실상 표준 벤치마크 |
| [2407.06460](https://arxiv.org/abs/2407.06460) | MUSE: Machine Unlearning Six-Way Evaluation for Language Models | 망각을 6개 속성(축자암기·지식암기·프라이버시 누출·효용보존·확장성·지속성)으로 분해. **우리 리포트의 평가 축 설계에 직접 차용 가능** |
| [2310.07579](https://arxiv.org/abs/2310.07579) | In-Context Unlearning: Language Models as Few-Shot Unlearners (Pawelczyk, Neel, Lakkaraju, ICML 2024) | **파라미터 수정 없이 컨텍스트만으로 망각** — 스킬 레이어에서 가능한 유일한 방향 |
| [2402.00751](https://arxiv.org/abs/2402.00751) | Fast Exact Unlearning for In-Context Learning Data for LLMs | ICL 데이터에 대한 **정확(exact) 망각**. 근사가 아닌 보장이 필요할 때 |
| [2510.17620](https://arxiv.org/pdf/2510.17620) | Forget to Know, Remember to Use: Context-Aware Unlearning for LLMs | "지식은 잊되 주어지면 쓸 수 있게" — 업무 효용 손실 최소화 |
| [2605.27138](https://arxiv.org/pdf/2605.27138) | ICCU: In-Context Continual Unlearning via Pattern-Induced Refusal Rules | 반복적·연속적 망각 요청 처리 |
| [2510.25117](https://arxiv.org/pdf/2510.25117) | A Survey on Unlearning in Large Language Models | 최신 서베이 |
| [2511.09855](https://arxiv.org/abs/2511.09855) | Unlearning Imperative: Securing Trustworthy and Responsible LLMs through Engineered Forgetting | 민감정보 영구삭제 보장의 부재를 정면으로 다룸 |
| [2209.02299](https://arxiv.org/pdf/2209.02299) | A Survey of Machine Unlearning (ACM TIST 2025) | 전통 ML 포함 총괄 |

### 3-2. 망각 **검증/감사** — "Hallucination 금지" 요구의 근거 문헌
| ID | 제목 | 왜 필요한가 |
|---|---|---|
| [2506.15115](https://arxiv.org/html/2506.15115v3) | Towards Reliable Forgetting: A Survey on Machine Unlearning Verification | **검증 방법론 총정리. 필독** |
| [2210.11334](https://arxiv.org/pdf/2210.11334) | Proof of Unlearning: Definitions and Instantiation | "삭제 증명"의 형식적 정의 |
| [2003.04247](https://arxiv.org/pdf/2003.04247) | Towards Probabilistic Verification of Machine Unlearning | 확률적 검증의 한계 |
| [2602.14553](https://arxiv.org/html/2602.14553v1) | Governing AI Forgetting: Auditing for Machine Unlearning Compliance | 인증 이론과 규제 집행 사이의 간극 |
| [2606.16110](https://arxiv.org/html/2606.16110v1) | Auditing Machine Unlearning: Whether Models Truly Forget | 모델이 정말 잊는지 감사 |
| [2506.06112](https://arxiv.org/pdf/2506.06112) | Lifecycle Unlearning Commitment Management: Sample-level Unlearning Completeness | 샘플 단위 완전성 측정 |

### 3-3. RAG / 메모리 삭제 **전파** — 층위 C의 핵심 근거
| ID | 제목 | 왜 필요한가 |
|---|---|---|
| [2410.15267](https://arxiv.org/pdf/2410.15267) | When Machine Unlearning Meets RAG: Keep Secret or Forget Knowledge? | RAG 환경에서의 망각 |
| [SIGIR 2026](https://marksanderson.org/files/papers/SIGIR2026_Leila_Main__Copy_.pdf) | Deletion Isn't Enough: Auditing RAG for Selective Forgetting | **"삭제만으로 부족"** — 검색/인용 아티팩트에 잔존 여부를 감사 문제로 정식화. 우리 검증 단계 설계의 직접 근거 |
| [2605.14421](https://arxiv.org/pdf/2605.14421) | **MemLineage: Lineage-Guided Enforcement for LLM Agent Memory** | **가장 가까운 선행 연구.** tombstone 기반 비파괴 삭제 + 포함 증명 |
| [2605.28732](https://arxiv.org/html/2605.28732) | Tracing and Attributing Errors in LLM Memory Systems | 메모리 생성/갱신/삭제 연산의 사후 추적 |

### 3-4. 정보흐름 추적(IFC) / Taint — "Forward Tracing"의 이론적 기반
| ID | 제목 | 왜 필요한가 |
|---|---|---|
| [2503.18813](https://arxiv.org/pdf/2503.18813) | **CaMeL: Defeating Prompt Injections by Design** (Google DeepMind/ETH) | 제어흐름과 데이터흐름 분리. 신뢰 경계 설계의 레퍼런스 |
| [2505.23643](https://arxiv.org/pdf/2505.23643) | **Securing AI Agents with Information-Flow Control (FIDES)** (Microsoft, [코드](https://github.com/microsoft/fides)) | **기밀성·무결성 라벨을 메시지/액션/툴호출/결과에 전파하고 정책 만족 시에만 실행.** Security Axis 설계의 가장 강력한 참조 구현 |
| [2604.23374](https://arxiv.org/abs/2604.23374) | Ghost in the Agent: Redefining Information Flow Tracking for LLM Agents (NeuroTaint) | 명시적 전달뿐 아니라 **의미 변형·결정에 대한 인과 영향·세션 간 메모리 지속**까지 taint 전파로 봄. 사용자의 "Forward Tracing" 개념과 거의 일치 |
| [2603.22868](https://arxiv.org/html/2603.22868v2) | Agent-Sentry: Bounding LLM Agents via Execution Provenance | 구조적 분류기 + 결정론적 allowlist + LLM 판사 3층. **결정론 우선 + LLM 보조 구조의 실제 사례** |
| [2606.04990](https://arxiv.org/pdf/2606.04990) | From Agent Traces to Trust: A Survey of Evidence Tracing and Execution Provenance in LLM Agents | 이 분야 서베이. 출발점으로 적합 |
| [2606.05679](https://arxiv.org/pdf/2606.05679) | Data Flow Control: Data Safety Policies for AI Agents | 에이전트용 데이터 안전 정책 |
| [2607.24625](https://arxiv.org/html/2607.24625) | APPA: Recoverable Information-Flow Control for Real-World LLM Agents | **복구 가능한** IFC — 차단 후 되돌리기 |
| [2607.00440](https://arxiv.org/pdf/2607.00440) | Minos: Multi-Agent Collaborative Framework for Provenance-Based Backward Tracking | **"Reverse Tracing"의 직접 선행.** 멀티에이전트 역추적 |

### 3-5. Contextual Integrity — "보안 Axis 기반 1차 판단"의 이론적 정당화
| ID | 제목 | 왜 필요한가 |
|---|---|---|
| Nissenbaum (2004) | Privacy as Contextual Integrity | 이론적 원점. "정보 흐름이 그 맥락의 규범에 부합하는가" |
| [2502.17041](https://arxiv.org/html/2502.17041v1) | PrivaCI-Bench: Evaluating Privacy with Contextual Integrity and Legal Compliance | 법규 준수 결합 벤치마크 |
| [2604.21308](https://arxiv.org/abs/2604.21308) | **CI-Work: Benchmarking Contextual Integrity in Enterprise LLM Agents** | **기업 워크플로 전용.** 위반율 15.8~50.9%, 유출 최대 26.7%. **"효용이 높을수록 프라이버시 위반이 증가"라는 트레이드오프** — 우리 스킬의 존재 이유를 뒷받침하는 수치 |
| [2506.04245](https://arxiv.org/pdf/2506.04245) | Contextual Integrity in LLMs via Reasoning and Reinforcement Learning | 추론·RL로 CI 강화 |
| [2505.14585](https://arxiv.org/pdf/2505.14585) | Context Reasoner: Contextualized Privacy and Safety Compliance via RL | 맥락화된 컴플라이언스 |
| [2606.04067](https://arxiv.org/pdf/2606.04067) | Need to Know: CI-Grounded Query Rewriting for Privacy-Conscious LLM Delegation | **차단 대신 재작성** — 업무 중단 없이 보호. 층위 A 대안으로 검토 가치 |
| [2606.21710](https://arxiv.org/pdf/2606.21710) | PrivacyAlign: Contextual Privacy Alignment for LLM Agents | |

### 3-6. 새니타이즈의 한계 — 반드시 알아야 할 반례
| ID | 제목 | 왜 필요한가 |
|---|---|---|
| [2508.18976](https://arxiv.org/pdf/2508.18976) | The Double-edged Sword of LLM-based Data Reconstruction: Contextual Vulnerability in Word-level DP Text Sanitization | **마스킹해도 주변 문맥으로 원문 복원 가능.** "레닥션했으니 안전"이라는 주장이 왜 hallucination인지의 근거 |
| [2506.15076](https://arxiv.org/pdf/2506.15076) | Learning-Time Encoding Shapes Unlearning in LLMs | 학습 시 인코딩 방식이 망각 가능성을 좌우 |

---

## 4. 새로움 판정 — 무엇이 진짜 새로운가

### ❌ 새롭지 않은 것
- 민감정보 탐지·차단 (상용 제품 다수 + 스킬 다수)
- 정책 파일 기반 판단 (OPA/Rego 스킬, Purview 라벨)
- Forward/Backward 리니지 개념 (데이터 거버넌스 표준 개념)
- 정보흐름 라벨 전파 (FIDES, CaMeL, NeuroTaint가 이미 함)
- LLM 망각 (연구 분야이며, 스킬로는 불가)

### ✅ 새로운 것 (= 이 스킬의 존재 이유)
1. **에이전트 워크스페이스를 대상으로 한 사후 purge 파이프라인.**
   기존 연구(MemLineage, Minos)는 프레임워크·논문 수준이고, 기존 스킬은 전부 "사전 차단"에서 멈춘다.
   **전사(transcript)·CLAUDE.md·메모리 파일·툴 로그·캐시·git 히스토리·생성된 산출물**을 한 단위로 묶어
   양방향 추적 후 폐기하는 스킬은 조사 범위 내에 없었다.
2. **"Security Axis" — 에이전트가 읽되 수정 불가능한 서명된 정책축.**
   OPA 스킬은 정책을 *작성*하게 돕지만, 정책을 *불변 입력*으로 강제하지 않는다.
   `PreToolUse` 훅이 권한 모드보다 먼저 실행되어 `bypassPermissions`로도 우회 불가하다는 점이 이를 기술적으로 가능하게 한다.
3. **결정론 1차 / LLM 2차 라는 2단 판정 + abstain 강제.**
   Agent-Sentry가 유사 구조를 쓰지만 스킬화된 사례는 없다.
4. **폐기 증적(tombstone + 해시 + 재검증)을 산출물로 내는 것.**
   "지웠다"가 아니라 "지웠음을 증명한다"로 출력을 정의.

### ⚠️ 반드시 바꿔야 할 것
- 스킬 이름/설명에서 **"unlearning", "망각", "LLM이 잊는다"** 표현 제거
  → **"Containment & Verified Purge"** 로 재정의.
  그렇게 하지 않으면 §2의 기술적 한계 때문에 스킬이 구조적으로 거짓 보고를 하게 된다.

---

## 5. 권장 스코프

```
[Security Axis]  ← 서명·읽기전용·CODEOWNERS 보호. 인가자만 수정
      │
      ├─ (1) GATE     사전 차단   : 결정론적 매칭 → PreToolUse/UserPromptSubmit 훅으로 차단
      │                             ※ 이 부분은 기존 스킬과 중복. 최소 구현만
      │
      └─ (2) BREACH   유입 감지 시
             ├─ REVERSE TRACE : 이 자료가 어디서 들어왔나 (출처·경로·최초 유입 턴)  ┐
             │                                                                    ├ 병렬
             ├─ FORWARD TRACE : 들어온 뒤 무엇을 낳았나 (파생 파일·커밋·요약·캐시) ┘
             │
             ├─ PURGE        : 파생물 우선 → 원본 순. tombstone 기록
             ├─ VERIFY       : 동일 탐지기 재실행 0-hit + 해시 대조. 실패 시 "삭제됨" 금지
             ├─ ROTATE       : 크리덴셜은 삭제 불가 전제 → 회전/권한회수 지시
             └─ REPORT       : 도달 불가 영역 명시 (제공자 로그·외부 SaaS·타 세션)
```

---

## 6. 조사 출처

**스킬/마켓플레이스**
- [BehiSecc/awesome-claude-skills](https://github.com/BehiSecc/awesome-claude-skills)
- [travisvn/awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills)
- [aiskillstore/marketplace](https://github.com/aiskillstore/marketplace) / [skillstore.io](https://skillstore.io/)
- [MCP Market Skills](https://mcpmarket.com/tools/skills)
- [Eyadkelleh/awesome-skills-security](https://github.com/Eyadkelleh/awesome-skills-security)

**플랫폼 문서**
- [Claude Code Hooks](https://code.claude.com/docs/en/hooks)
- [Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing) / [Memory tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool)

**실무 근거**
- [What breaks when secrets are pasted into LLM prompts?](https://nhimg.org/faq/what-breaks-when-secrets-are-pasted-into-llm-prompts/)
- [Your Secrets Manager Ends Where the Context Window Begins](https://tianpan.co/blog/2026/07/02/your-secrets-manager-ends-where-the-context-window-begins)
- [ATR-2026-00021: Credential and Secret Exposure in Agent Output](https://agentthreatrule.org/en/rules/ATR-2026-00021)
