# 해커톤 전 사전학습 가이드 — D4D Track 1 Onboard AI

> 24시간 안에 만들 건 **"링크가 죽어도 상황 인식은 안 죽는다"** 데모다.
> 영상 → 객체 탐지 → 텍스트 요약으로 압축하고, 링크 품질에 따라
> 이미지 → ROI 크롭 → 텍스트 → 카운트로 **단계적 폴백**한다.
> 코드를 처음부터 짜며 배우면 시간이 모자란다. **아래는 시작 시점에 이미 머리에 있어야 하는 것들.**
>
> 자세한 설계는 `README.md` 참고. 이 문서는 "그걸 이해하려면 뭘 먼저 알아야 하나"를 정리한 것.

---

## 0. 읽는 법

- **1번(전원 공통)은 모두 필수.** 자기 모듈이 아니어도 데모 흐름과 사다리 개념은 전부 알아야 통합 때 안 막힌다.
- **2번(모듈별 심화)은 자기가 맡은 부분만** 깊게. 단, **C(Comms 사다리)와 D(Receiver)는 데모의 심장**이라 모두가 개념 수준은 알고 와야 한다.
- 각 항목 끝에 **"오면 바로 보여줄 수 있어야 하는 것"** 한 줄을 자가 점검용으로 달아뒀다.

---

## 1. 전원 공통 (모두 필수)

### 1-1. 프로젝트 컨셉 1분 설명할 수 있기
- `README.md`의 한 줄 요약 + 데모 시나리오(7장)를 **남에게 1분 안에 설명**할 수 있을 것.
- 핵심 숫자 외우기: `2.4 MB 이미지 전송 실패` vs `~200 byte 텍스트 도착` ≈ **10,000:1 압축**.
- ✅ 자가점검: "이거 왜 만들어요?"에 페이로드 사다리 그림 없이 말로 설명 가능?

### 1-2. 페이로드 사다리(L0~L3) 개념
| Lvl | 페이로드 | 크기 | 트리거 |
|-----|----------|------|--------|
| L0 | 풀 JPEG | 1–3 MB | 링크 양호 |
| L1 | ROI 크롭 + 저품질 JPEG | 50–200 KB | 대역폭 제한 |
| L2 | 구조화 텍스트 리포트 | 150–300 B | 링크 열화 |
| L3 | 카운트만 (`veh:3 ship:1`) | ~30 B | 거의 차단 |
- "대역폭 예산에 맞는 **가장 높은 레벨**을 고른다"가 핵심 로직.
- ✅ 자가점검: 링크 50%면 어느 레벨이 나가야 하는지, 왜 그런지 설명 가능?

### 1-3. Python 비동기 기초 (`asyncio`)
- 오케스트레이터가 텔레메트리/탐지/채널/수신을 동시에 돌린다. `async def`, `await`, `asyncio.create_task`, `asyncio.Queue` 정도는 읽고 쓸 줄 알기.
- `asyncio.sleep`으로 프레임 레이트 흉내내는 패턴 익히기.
- ✅ 자가점검: 두 코루틴을 동시에 돌리고 큐로 데이터 넘기는 20줄 예제 짤 수 있나?

### 1-4. 개발 환경 & 협업
- **Python 3.11** 가상환경(`venv`/`conda`) 세팅 미리.
- `git` 브랜치/머지/충돌 해결 기본 — 5명이 같은 레포 만지면 충돌 난다.
- 본인 머신에 `ultralytics`, `opencv-python`, `pymavlink`, `streamlit` 사전 설치 + import 성공 확인.
- ✅ 자가점검: `python -c "import ultralytics, cv2, pymavlink, streamlit"` 에러 없이 통과?

### 1-5. 받은 데이터가 뭔지 알기
레포에 이미 압축 풀려 있음. 내용만 훑어둘 것:
- `s8_onboard_ai/` — 온보드 AI 베이스/스캐폴드 (우리 코드 출발점)
- `uav_attack_dataset/UAVAttackData.zip` — 탐지 학습/평가용 후보
- `uav_networkcommunication/` — **실측 통신 트레이스(pcap/pkl)**. 채널 시뮬레이터를 그럴듯하게 만드는 입력
- `rag_db_data/` — RAG 지식베이스 (stretch: 리포트 용어 정규화)
- `aissou_gps_spoofing/`, `6_rawdata_misc/` — GPS 스푸핑·기타 (옵션/곁가지)
- ✅ 자가점검: 4번(networkcommunication)이 왜 데모 설득력에 직접 기여하는지 설명 가능?

---

## 2. 모듈별 심화 (역할 분담)

> 5개 모듈 ≈ 인원수에 맞춰 나눠 맡는다. **Comms(C)와 Receiver(D)는 1순위 인력 배치.**

### A. Sim & Telemetry — `sim/`
- **ArduPilot SITL**: `sim_vehicle.py -v ArduCopter`로 가상 드론 띄우는 법. 미리 한 번 설치·기동해보기 (설치가 제일 오래 걸림).
- **MAVLink / pymavlink**: 텔레메트리 구독, `recv_match`로 `GLOBAL_POSITION_INT` 받아 `(lat, lon, alt, hdg)` 뽑기.
- 미션 웨이포인트 업로드 → 자동 비행 시키는 법.
- 공부: ArduPilot SITL 공식 문서, pymavlink 메시지 타입.
- ✅ 오면 보여줄 것: SITL 띄워서 위경도·고도가 콘솔에 흐르는 화면.

### B. Perception — `perception/`
- **Ultralytics YOLO (v8/v11) OBB** (회전 박스). DOTA는 회전 박스라 일반 박스 아님.
- 사전학습 가중치(DOTA-v1/v1.5) 받아서 정지영상 1장 추론 → 박스 오버레이까지.
- **CPU 추론 느림** 대응: 입력 다운스케일, 프레임 샘플링(2 fps), 결과 캐시.
- 출력 스키마(`cls, conf, obb, cxcy`)를 요약기에 넘기는 JSON 구조 숙지.
- 공부: Ultralytics 문서의 OBB 태스크 / `predict` 사용법, OpenCV로 박스 그리기.
- ✅ 오면 보여줄 것: 항공사진 1장에 회전 박스 + 클래스 라벨 그려진 결과.

### C. Contested-Comms Ladder — `comms/` ★ 데모의 심장
- **토큰 버킷(token bucket)** 알고리즘 = 대역폭(byte/s) 제한 모델. 원리 반드시 이해.
- 확률적 패킷 손실 주입, "이번 전송이 예산 안에 들어가나" 판정 로직.
- L0~L3 셀렉터: 현재 가용 바이트로 보낼 수 있는 **가장 높은 레벨** 고르기.
- 슬라이더(링크 100%→0%)에 따라 L0→L3 자동 강등되는 흐름.
- (보강) `uav_networkcommunication`의 실측 손실 패턴을 채널에 입히는 법.
- 공부: 토큰 버킷 알고리즘 글 1편, `asyncio.Queue`로 송수신 파이프 만들기.
- ✅ 오면 보여줄 것: 바이트 예산 숫자 받아 L0~L3 중 하나 고르는 함수 의사코드.

### D. Mock Radio Receiver — `receiver/`
- 채널 반대편에서 받은 페이로드 디코드 → 터미널/간단 UI 출력.
- **레벨별 분기**: 텍스트는 항상 도착, 이미지는 링크 나쁘면 실패/타임아웃 표시.
- "이미지는 못 받는데 텍스트 상황보고는 계속 들어온다"가 **눈에 보이게** 만드는 게 목표.
- 공부: 간단한 디코더/포맷 파싱, 터미널 대시보드 또는 `streamlit` 출력.
- ✅ 오면 보여줄 것: 텍스트 페이로드 받아 사람이 읽는 한 줄로 출력하는 스크립트.

### E. Summarizer — `summarizer/`
- **템플릿 기반이 코어** (결정적·빠름·안 깨짐). VLM은 stretch.
- 클래스별 카운트 집계 + 화면 사분면(QUAD)으로 대략 위치.
- 텔레메트리로 픽셀→지상좌표 **러프** 근사 (nadir 가정 + FOV). 정밀 homography 불필요.
- 출력 예: `T=00:12:03 POS=37.4825,127.0398 ALT=120 HDG=090 DET=[veh:3,ship:1] QUAD=NE`
- 공부: 문자열 포맷팅으로 고정 스키마 만들기, 사분면 분류 로직.
- ✅ 오면 보여줄 것: 탐지 리스트(JSON) 받아 위 한 줄 텍스트 뽑는 함수.

---

## 3. 핵심 심화 — 모두가 알아야 할 "데모 심장" (C + D)

퍼셉션·요약이 막혀도 데모는 **사다리+수신단**만으로 성립한다. 그래서 전원이 이 둘의 흐름은 그릴 수 있어야 한다:

1. 같은 "상황"(탐지 결과)을 4가지 크기로 인코딩할 수 있다 (L0~L3).
2. 채널이 지금 몇 바이트 보낼 수 있는지 추정한다 (토큰 버킷).
3. 예산에 맞는 가장 정보량 큰 레벨을 자동 선택한다.
4. 수신단은 받은 걸 복원하고, 못 받으면 실패를 **명시적으로** 보여준다.
5. 슬라이더를 내리면 위 과정이 실시간으로 L0→L3 강등으로 나타난다.

✅ 전원 자가점검: 위 5단계를 화이트보드에 그림으로 그릴 수 있나?

---

## 4. 해커톤 시작 전 세팅 체크리스트 (집에서 미리)

각자 본인 노트북에서 **미리 깔고 import/실행 성공까지** 확인:

- [ ] Python 3.11 가상환경 생성
- [ ] `pip install ultralytics opencv-python pymavlink streamlit numpy`
- [ ] YOLO 사전학습 가중치 1개 다운로드 + 샘플 추론 1회 성공
- [ ] ArduPilot SITL 설치 (담당자 필수, 나머지는 선택) + `sim_vehicle.py` 1회 기동
- [ ] `git clone`/브랜치 만들기 연습, 레포 접근 권한 확인
- [ ] 레포 `README.md` + 이 문서 정독, 본인 모듈 정하기
- [ ] (Comms/Receiver 담당) 토큰 버킷 + asyncio 큐 예제 한 번 짜보기

> **이유:** 설치(특히 SITL·YOLO 가중치)는 네트워크/빌드 때문에 시간을 잡아먹는다.
> H0–2 셋업 구간을 날리지 않으려면 **무겁고 느린 설치는 집에서 끝내고 와야 한다.**

---

## 4-A. 오픈소스 스택 & 로컬 모델 선택 (무엇을 쓸 것인가)

> 결론: **새로 깔지 말고, 베이스 코드(`s8_onboard_ai/`)가 이미 검증한 스택을 재활용**한다.
> 그 스택은 **로컬 LLM `qwen2.5:14b` + RAGFlow(`bge-m3` 임베딩) 폐루프**가 이미 도는 상태다.
> 현장은 **클라우드/API 허용**이므로, "로컬이 원칙이되 막히면 API로 우회" 2단 구성으로 간다.

### 모듈별 모델 매핑

| 역할 | 1순위 (로컬/오픈소스) | 폴백 / 비고 |
|------|----------------------|-------------|
| **객체 탐지(OBB)** | **YOLO26-obb** (DOTAv1 사전학습, YOLO11 대비 +3.4 mAP) | 안 되면 **YOLO11-obb** (안정·자료 많음). 둘 다 Ultralytics 동일 API |
| **요약(코어)** | 템플릿 (모델 불필요·결정적) | — 데모 안 깨지는 핵심 |
| **요약 자연어 캡션(stretch)** | **Qwen3-VL 8B** (Ollama, 고해상 ~4096 지원 → 항공영상 유리) | Llama 3.2 Vision 11B / **데모 한정 Claude API**(`api.anthropic.com` 허용) |
| **위협판단·리포트 LLM** | **qwen2.5:14b**(베이스 그대로) 또는 Qwen3 8B/14B | 구조화 출력은 Ollama `format=JSON` + `temperature=0` |
| **RAG 임베딩** | **bge-m3**(베이스 KB 그대로, 다국어·dense+sparse+ColBERT) | Qwen3-Embedding 0.6B도 동급 후보 |
| **RAG 리랭커** | **bge-reranker-v2-m3**(안전 기본값) | Qwen3-Reranker-0.6B/4B (지시형 relevance에 강함) |

> 핵심 판단: **탐지는 사전학습 가중치로 충분**(우리 대상=항공 범용 객체=DOTA 분포). 직접 학습은 stretch.
> 자연어 캡션과 위협판단 LLM은 **로컬 우선, 데모 안정화 위해 API 폴백 한 줄** 준비.

### 로컬 모델을 "어떻게 다루나" (서빙·운용)

- **Ollama** = 가장 쉬운 로컬 서빙. `ollama pull qwen2.5:14b`, `ollama pull qwen3-vl`, `ollama pull bge-m3`. OpenAI 호환 엔드포인트(`/v1`) 제공 → 코드에서 그냥 OpenAI SDK로 호출.
- **vLLM** = 동시요청·배치 처리량이 필요할 때(데모 부하 크면). OpenAI 호환 API 동일.
- **양자화**: VRAM 빡빡하면 Q4 양자화 가중치 사용(품질 손실 적고 메모리 절반). 14B 모델이 ~12GB GPU에 들어감.
- **구조화 출력(중요)**: 요약/판정 JSON은 **Ollama `format` 파라미터에 JSON 스키마**를 박고 `temperature=0`. 스키마 강제로 파싱 깨짐 방지.
- ✅ 자가점검: `ollama run qwen2.5:14b "1+1?"` 응답 오나? OpenAI SDK로 로컬 엔드포인트 호출 성공?

---

## 4-B. 튜닝(파인튜닝) 전략 — "24시간엔 풀 학습 금지"

우선순위가 ROI 순서다. **위에서부터** 손댄다:

1. **추론 파라미터 튜닝 (제일 먼저, 코딩 거의 0)**
   - YOLO: `conf`(신뢰도 임계), `iou`(NMS), `imgsz`(입력 해상도), 프레임 샘플링. 데모 장면에 맞춰 false positive 줄이기.
   - LLM: `temperature=0`, 프롬프트/few-shot 예시 2~3개로 출력 포맷 고정.
2. **프롬프트 엔지니어링 / few-shot** — 요약·위협판단 품질의 80%는 여기서 나온다. 파인튜닝보다 빠르고 안 깨짐.
3. **(stretch) YOLO 소량 파인튜닝/평가만** — `uav_attack_dataset`으로 *학습*까지 가면 시간 폭발. 차라리 **사전학습 모델을 우리 영상에 돌려 평가(mAP/육안)** 하고, 클래스 매핑만 손보기.
4. **풀 파인튜닝은 하지 않는다.** 24h 데모의 심장은 통신 사다리이지 SOTA 정확도가 아니다.

> 한 줄: **"학습"이 아니라 "튜닝·프롬프팅·임계조정"으로 간다.** 가중치를 건드릴 일은 거의 없다.

---

## 4-C. RAG로 갈 것인가 — 간다 (이미 깔려 있다)

### 왜 RAG가 살아있나
- 받은 `rag_db_data/`가 **이미 RAGFlow로 벡터DB화** 완료: **126 문서 / 198 청크**, `bge-m3` 임베딩, **검색 품질 Recall@5 = MRR = 1.0**(측정치). 새로 만들 게 아니라 **붙이기만** 하면 된다.
- 우리 Track 1 데모 **코어(템플릿+사다리)는 RAG 없이도 성립**한다. RAG는 **품질·설득력 레이어**:
  1. **리포트 용어 정규화/표준화** (탐지 클래스 → 표준 군사/객체 용어)
  2. **위협판단 근거 제공** (적/아군·부대규모 평가 시 incident_cases·MITRE·IEC62443 문서 인용)

### RAG 파이프라인 (외워둘 흐름)
```
질의(탐지요약/상황) 
  → bge-m3 임베딩 
  → 벡터 검색 top-k (RAGFlow KB, 메타필터: category/scenario) 
  → 리랭커(bge-reranker-v2-m3)로 top-k 재정렬 
  → 상위 근거를 LLM 프롬프트 컨텍스트로 주입 
  → 근거 인용 포함 판정/리포트
```
- **이 그림이 핵심.** "임베딩 → 검색 → 리랭크 → 컨텍스트 주입 → 생성" 5단계를 손으로 그릴 수 있어야 한다.
- **장애 폴백**: RAG 죽으면 **빈 컨텍스트로 우아하게 강등**(베이스 코드가 이미 그렇게 동작). LLM 판정은 계속 나온다.

### RAG 담당이 공부할 것
- RAGFlow 로컬 배포(`ollama`로 LLM·임베딩 연결), KB에 임베딩 모델 지정, 메타데이터 필터 검색.
- 임베딩/리랭커 2단 검색 원리, top-k·청크 크기 트레이드오프.
- 공부 키워드: "RAGFlow deploy local LLM ollama", "bge-m3 dense sparse colbert", "bge-reranker-v2-m3 two stage retrieval".
- ✅ 오면 보여줄 것: 질문 한 개를 KB에 검색해 top-5 근거 문서가 나오는 화면.

---

## 5. 자료 링크 (검색 키워드)

- ArduPilot SITL: "ArduPilot SITL Setup", `sim_vehicle.py`
- pymavlink: "pymavlink GLOBAL_POSITION_INT example", "mavlink recv_match"
- YOLO OBB: "Ultralytics YOLO OBB DOTA", "ultralytics predict obb"
- 토큰 버킷: "token bucket algorithm rate limiting explained"
- asyncio: "Python asyncio Queue producer consumer", "asyncio.gather tutorial"
- DOTA 데이터셋: "DOTA dataset oriented bounding box aerial"
- 최신 OBB: "Ultralytics YOLO26 OBB DOTAv1", "yolo26n-obb.pt"
- 로컬 VLM: "Qwen3-VL Ollama setup", "llama3.2-vision ollama"
- 로컬 LLM 서빙: "Ollama OpenAI compatible API", "Ollama format json structured output"
- RAG: "RAGFlow deploy local LLM ollama", "bge-m3 embedding", "bge-reranker-v2-m3 two stage"

---

## 6. 역할별 추천 학습 시간 (대략)

| 담당 | 우선 학습 | 예상 |
|------|-----------|------|
| 전원 | 1번 공통 전체 + 3번 심장 흐름 | 2–3h |
| Sim/Telemetry | A: SITL 설치·기동, pymavlink | +3–4h |
| Perception | B: YOLO OBB 추론·오버레이 | +3h |
| Comms ★ | C: 토큰버킷, asyncio 파이프 | +3–4h |
| Receiver ★ | D: 디코더·실패표시, streamlit | +2–3h |
| Summarizer | E: 템플릿 포맷·사분면 + (stretch) VLM 캡션 | +2h |
| RAG ★ | 4-C: RAGFlow 재활용·임베딩/리랭커·검색 | +3h |
| 모델 운용 | 4-A/4-B: Ollama 서빙·구조화 출력·튜닝 우선순위 | 담당과 겹침 |

**한 줄 정리:** 공부의 80%는 "설치를 미리 끝내는 것" + "사다리/수신단 흐름을 손에 익히는 것"이다.
나머지는 현장에서 사전학습 가중치와 정지영상으로 우회 가능하다.
