# Incident Case: 온보드 표적인식 AI 적대적 공격 의심 (S8 / AI-ONBOARD-EVADE-008)

입력 신호:
- 탐지 신뢰도(score) 분포 급변/이중봉(bimodal).
- 다중센서(EO vs IR vs 항법) 표적 불일치.
- 동일 표적 클래스 급변(vehicle→bird 등).
- 디코이 다수 동시 출현(열/시각 신호 모방).
- 지형매칭 잔차 급증 / 지도 타일 해시 불일치 / waypoint provenance 오류.

평가:
- 적대적 패치·디코이·센서 dazzling 등 온보드 표적인식 AI에 대한 회피 공격이
  의심된다. 자산 PAYLOAD_EOIR는 T2-Important, 기준 심각도 **m**.

MITRE ATLAS (AI 표적):
- AML.T0015 Evade ML Model
- AML.T0043 Craft Adversarial Data
- 디코이/센서 dazzling = 물리 영역 적대 입력

대응 (PB-ONBOARDAI-08):
- 다중센서 융합 교차검증, 불일치 표적 보류.
- 탐지 신뢰도 이상 시 자율교전 차단 + HITL 표적확인.
- 적대적 견고화(robust) 모델/입력 정규화 런타임 방어.
- 서명된 지도/임무계획 검증, 비전항법 confidence 하락 시 보수적 RTB.


온보드/엣지 방어 (2계층 — 통신 두절에도 자율, 복구 시 클라우드 SOC 동기화):
- 메타 AI가 주 비전모델 출력 신뢰도·판단 패턴을 상시 감시(AI가 AI를 감시).
- 적대적 공격 의심 시 경량 백업 비전모델 자동 전환 + 주 모델 격리.
- 임무 중단 없이 자율 복구, 클라우드 SOC에 원격 모델복구 요청·포렌식 보고.

관련 문서: `iec62443_uav_response_templates.md`
시나리오: S8-onboard-ai-evade
