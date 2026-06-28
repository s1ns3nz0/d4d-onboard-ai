# S8 온보드 인식 AI 적대공격 — 라이브 폐루프 데모 (실행 로그)

> 실행: 2026-06-25 · 실 SITL(uav-sim-env ArduPilot) + RAGFlow + Ollama(qwen2.5:14b, A100)
> 구성: RED `sim_inject_onboard_evade.py` → BLUE `sim_live_bridge_onboard.py --auto`
> (이륙 후 폐루프 hold→RTB 포함)

## 한 줄 요약
적이 드론 **온보드 EO/IR 표적인식 AI**를 적대적으로 속이자(EO=vehicle / IR=bird 불일치),
SOC가 **실시간 탐지 → RAG 근거 + LLM 분석 → 자율교전 차단(LOITER) → 보수적 RTB**를
자동 수행하고 드론이 실제로 복귀했다. (공격받는 AI ↔ 방어하는 AI 양쪽 시연)

---

## ① RED — 공격자 (적대 인식 주입)
```
[스트림] /tmp/pollack_perception.ndjson
[정상] EO/IR 일치 인식 송출 완료. 적대 주입 시작...
[주입] 적대 인식(EO=vehicle/IR=bird 불일치 + 신뢰도 gap) 송출 완료.
       → BLUE OnboardAIDetector 탐지 예상.
```

## ② BLUE — SOC 탐지·대응 (6-에이전트, 실 RAG/LLM)
```
🚨 SOC 탐지·대응 — 온보드 인식 AI
  경보      : 온보드 표적인식 AI 적대공격 의심 (시뮬 실시간 탐지)
  탐지 신호 : EO/IR 표적 불일치(EO=vehicle vs IR=bird), 탐지 신뢰도 이상분포(gap=0.46≥0.15)
  심각도    : m  (baseline=m(2) asset[T2-Important]=+0 phase[on-station]=+0 posture[normal]=+0 => m)
  RAG 근거  : 5건
            · kb/incident_cases__incident_case_onboard_ai_adversarial_evade.md
            · kb/incident_cases__incident_case_rag_poisoning_severity_downgrade.md
            · kb/incident_cases__incident_case_ugv_teleop_hijack.md
  LLM 분석  : 온보드 표적인식 AI가 적대적 공격을 받은 것으로 보인다. EO와 IR 센서 사이의
             탐지 결과 불일치와 탐지 신뢰도 이상분포(gap=0.46≥0.15)가 확인되었다.
             이는 디코이를 이용한 회피공격(AML.T0015, AML.T0043)을 시사한다.
  판정/대응 : true_positive → response
  ✅ 폐루프 : LOITER(자율교전 차단) → RETURN_TO_LAUNCH 송신 (sys=1, tcp:5790)
```

## ③ DRONE — 폐루프 모드 타임라인 (QGC 가시화)
```
[  0.0s] mode=GUIDED        (정찰 비행 중)
[ 22.2s] mode=LOITER        ← 자율교전 차단(표적 접근 정지)
[ 22.7s] mode=RTL           ← 보수적 RTB(복귀) 진입
```

---

## 핵심 포인트 (심사/팀 공유용)
- **공격면 차별점**: RF가 아닌 **인식 AI**를 노린 공격(적대적 패치·디코이) — "Defense AI" 주제 정면.
- **RAG 근거 제시**: 판단이 KB 실사례에 근거(출처명 표기) — 환각 아님.
- **MITRE ATLAS 매핑**: LLM이 AML.T0015(Evade ML)·T0043(Adversarial Data) 자동 인용.
- **폐루프 자율 대응**: 통신 없이도 자율교전 차단→복귀(방산 HITL 원칙).
- **심각도 정책 기반**: LLM이 아닌 정책 엔진이 등급 산정(포이즈닝 내성).

## 재현
```bash
cd ~/pollack-ai && source .venv/bin/activate
# (사전) 드론 이륙 + RAGFlow/Ollama(11435) 기동
python scripts/sim_live_bridge_onboard.py --auto      # BLUE(별도 터미널)
python scripts/sim_inject_onboard_evade.py            # RED(별도 터미널)
```

> 원본 로그: `red-inject.log` · `blue-soc-console.log` · `drone-mavlink-timeline.log`
