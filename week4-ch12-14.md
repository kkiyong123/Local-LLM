# 로컬 AI 스터디 4주차 정리 (CH12~14)

11/11 4주차 스터디. 교재 Part 3 시작, CH12~14. 지난주까지 음성 챗봇을 완성했고 이번 주는 그 위에 얹는 내용: 어떤 모델을 골라 쓸지(CH12), 같은 모델의 말투와 답변 성향을 어떻게 바꿀지(CH13), 굳이 로컬로 돌리는 이유(CH14). 시간 대부분은 CH12 모델 계열·MoE와 CH13 temperature/top-k/top-p 원리에 썼고 CH14는 5분 정도로 짧게 넘어감. 다음 주가 마지막.

## CH12. LLM 모델 비교 및 선택

오픈 웨이트 = 가중치를 공개한 모델. 누구든 내려받아 로컬이나 폐쇄망에서 돌릴 수 있다는 뜻. 크게 5개 계열이 있고, 이걸 쉽게 서빙해 주는 런타임 중 하나가 Ollama.

**모델 계열 5가지**

| 계열 | 만든 곳 | 특징 | 요즘 소식 |
| --- | --- | --- | --- |
| Llama | Meta | 2023년 2월 공개로 오픈 모델 흐름의 시작. 이걸 베이스로 파인튜닝한 파생 모델이 많이 나옴. 범용·코딩·추론 무난 | Llama 4(2025년 4월)부터 MoE. 최근엔 Muse Spark를 밀고 있어 Llama 쪽은 뜸한 느낌 |
| Gemma | Google | 크기 대비 성능이 좋고 경량 모델이 있어서 Qwen과 함께 로컬에서 자주 씀. 교재 기본 모델 | 꾸준히 새 버전이 나오는 중 |
| Qwen | Alibaba | 처음부터 영어·중국어·일본어·한국어 등 다국어 어휘로 학습. 아시아권 언어에 강해서 한국에서 가장 많이 씀. 코딩·수학도 좋음 | 올해 릴리스가 쏟아짐: 3.5(2월) → 3.6(4월) → 3.7(5월) → 3.8(7월) |
| Mistral | Mistral AI (프랑스) | 프랑스어·독일어·스페인어·이탈리아어 같은 유럽 언어에 강함. Mixtral 8x7B로 MoE를 일찍 씀 | 9월에 삼성전자와 전략적 파트너십. 한국어도 좀 넣어 주면 좋겠음 |
| DeepSeek | DeepSeek AI (중국) | 최신 GPU가 아닌 중저가 GPU로 돌릴 방법을 고민해 나온 아키텍처. V3(671B 중 37B 활성)가 충격이었음. R1은 추론 모델 | 올해 V4는 파라미터가 훨씬 더 커짐 |

**크기와 메모리** — 이름 뒤의 B는 10억 단위 파라미터(2B = 20억 개 가중치). 많을수록 품질은 좋지만 느려지고 메모리를 먹음. **RAM이 절대적인 한계**.

| 모델 | 필요 RAM | 속도(토큰/초) | 용도 |
| --- | --- | --- | --- |
| gemma2:2b, llama3.2:3b, qwen2.5:3b | 약 4GB | 30~60 | 가벼운 작업, 저사양 |
| mistral:7b, llama3.1:8b, gemma2:9b | 약 8GB | 10~25 | 범용, 추론 |
| qwen2.5:14b | 약 16GB | 5~10 | 복잡한 작업 |

- 맥은 통합 메모리라 모델 올리고 나면 KV 캐시도 쓰고 OS도 써야 함. 계산은 **내 RAM ÷ 2**. 16GB면 mistral:7b ~ gemma2:9b 구간이 적당하고 14b는 못 올림.
- 메모리를 넘기면 디스크 스왑으로 돌아가긴 하는데 50~100배 느려짐. 빠른 작은 모델이 느린 큰 모델보다 훨씬 쓸모 있음.
- 체감 속도: 30 토큰/초 이상이면 매끄럽고, 10~15는 참을 만하고, 5 미만은 답답.
- 토큰은 영어 기준 단어의 3/4 정도. 한국어는 토큰을 더 많이 씀. 텍스트를 토큰으로 쪼개는 게 토크나이저(패키지 설치할 때 보이던 tokenizers가 그것).
- 전체/여유 메모리 확인: `sysctl -n hw.memsize | awk '{print $0/1073741824 " GB"}'` 또는 활성 상태 보기.

**벤치마크** — 모델 성능의 표준 시험(지식·코딩·추론·창의력·대화). 2B와 8B 차이는 크고 7B와 9B 차이는 작음. 제조사 블로그의 벤치마크는 걸러서 봐야 함. 점수가 잘 나오게 하는 꼼수가 있음. 최종 답으로 삼지 말고 내 로컬에 받아서 직접 돌려 보는 게 답.

**다국어** — 학습 데이터에 그 언어가 얼마나 들어갔느냐가 결정. 영어는 아무 모델이나, 아시아 언어는 Qwen, 유럽 언어는 Mistral. 같은 질문을 모델별로 던져서 비교:

```bash
ollama run gemma2:2b "Describe the four seasons of Japan briefly."
ollama run qwen2.5:3b "한국의 사계절을 짧게 설명해주세요."
```

qwen2.5:3b는 한국어 질문을 잘 이해하고 자연스럽게 답함. 학습 어휘에 한국어가 들어 있어서.

**챗봇에서 모델 바꾸기** — API가 동일해서 코드의 모델 이름 한 줄만 바꾸면 됨. 더 편하게는 사이드바에 `ollama list` 결과를 그대로 띄우는 선택기:

```python
available_models = [m.model for m in ollama.list().models]
selected_model = st.sidebar.selectbox("Choose a model:", available_models)
```

따로 지정 안 해도 내가 받아 둔 모델 목록이 나오고, 골라서 바로 질문.

- 관리 명령: `ollama list`, `ollama show <모델>`, `ollama rm <모델>`, 용량은 `du -sh ~/.ollama/models`.
- `ollama show qwen2.5:3b` → 파라미터 3.1B, 양자화 방식, 컨텍스트 길이. `qwen3.5:4b`는 4.5B이고 thinking이 기본으로 켜져 있는 모델.
- `test_models.py`: 모델 리스트를 for로 돌면서 같은 질문을 던지고 답과 걸린 시간 출력. 내용은 비슷한데 말투와 구성이 모델마다 다름.

**MoE (Mixture of Experts)** — 요즘 큰 모델이 전부 쓰는 구조.

- 기존 dense 모델: 토큰 하나 만들 때마다 모든 파라미터가 활성화. 700억 파라미터면 700억이 다 계산에 참여. RAM이 금값인데 감당이 안 됨.
- MoE: 전문가(expert) 그룹 여러 개 + 앞단의 라우터. 토큰이 오면 라우터가 잘 답할 전문가 쪽으로만 보냄. 전문가가 7개면 1/7만 활성화.
- 전문가와 라우터도 처음엔 아무것도 모르고, 학습을 반복하면서 각자 전문 분야와 보낼 곳이 정해짐.
- 총 파라미터 = 모델이 가진 지식의 양, 활성 파라미터 = 토큰 하나당 필요한 계산량. MoE 모델은 활성 파라미터를 보고 내 맥에서 돌릴 만한지 판단.
- sparse는 dense의 반대말(희소). 활성 비율이 4% 미만이면 ultra-sparse.
- 주의: 교재는 활성이 3B라고 80B 모델이 16GB나 32GB 맥에 들어간다고 가정하지 말라고 함. Qwen3-Coder-Next는 다운로드만 약 52GB. 받기 전에 파일 크기부터 확인.

**2026년 프론티어 오픈 모델** — 2월에 8일 사이 3개가 나오면서 판이 바뀜. 전부 중국, 전부 MoE.

| 모델 | 총 / 활성 | 특징 |
| --- | --- | --- |
| Zhipu GLM-5 | 744B / 40B | 전문가 256개, 200K 컨텍스트, MIT. SWE-bench 77.8%, Humanity's Last Exam 50.4%. 상반기에 굉장히 핫했고 국내 회사 내부 서빙으로도 많이 돌릴 듯 |
| Qwen3-Coder-Next | 80B / 3B | 전문가 512개, 256K. MoE를 극단까지 밀어붙인 ultra-sparse. 코딩 에이전트용, SWE-bench 70.6% |
| MiniMax M2.5 | 230B / 약 10B | SWE-bench 80.2% (당시 Claude Opus 4.6이 80.8%). 자사 API 가격이 Opus 4.6의 1/20, GPU 서버가 있으면 전기요금만 |

상용 모델이 여전히 앞서긴 하지만 이제 진짜로 경쟁하는 형태가 됨. Qwen도 8월에 Qwen3.8-27B가 나왔고 Qwen 팀 주장으로는 Opus 4.6급을 로컬에서 돌리는 수준.

## CH13. 시스템 메시지와 매개변수 조정

2주차에 써 본 system 역할을 본격적으로 다룸. 자주 쓰는 페르소나를 프리셋으로 만들고, temperature / top-k / top-p가 답변을 어떻게 바꾸는지 원리를 보고, 이 설정들을 Modelfile로 묶어 커스텀 모델로 만들고, 마지막에 Streamlit 챗봇 사이드바에 붙임.

**시스템 프롬프트** — 대화가 시작되기 전에 전달되는 특별한 지시. 모델이 따라야 할 규칙·성격·한계를 정함. 모델은 이걸 가장 먼저 읽고 이후 모든 응답에서 따름. messages의 맨 앞에 `{"role": "system", "content": "..."}`.

핵심: 시스템 프롬프트는 관점·스타일·우선순위를 바꾸지만 **가중치는 안 바뀜**. 해적처럼 답하라고 해도 답의 내용(맥락)은 그대로고 설명하는 방식만 달라짐.

**프리셋 5가지 실습** — 터미널에서 바로 해 보기.

```bash
# 터미널 1: Flash Attention + KV 캐시 8비트 양자화로 메모리 절약
OLLAMA_FLASH_ATTENTION="1" OLLAMA_KV_CACHE_TYPE="q8_0" ollama serve

# 터미널 2
ollama run qwen3.5:4b
/set system "You are a professional business consultant with 20 years of experience. ... 답변은 한글로 해"
대한민국 서울에 대해서 설명해줘
```

- Flash Attention: 어텐션 계산을 메모리 효율적으로 처리해서 긴 컨텍스트에서 메모리를 줄이고 속도를 높임. KV 캐시 `q8_0`: 대화 문맥을 기억하는 메모리를 기본 f16 대신 8비트로 저장해 용량이 절반 정도. 정확도 손실은 작음.
- qwen3.5:4b는 thinking 모델이라 `/set system` 하면 "앞으로 이렇게 답하겠다"는 생각 과정을 먼저 출력함.
- 책 원문 뒤에 "답변은 한글로 해"를 붙여서 실습.

| 프리셋 | 지시 내용 | 서울 설명 결과 |
| --- | --- | --- |
| 전문 컨설턴트 | 20년 경력, 명확하고 실행 가능한 조언, 글머리 기호, 모르면 모른다고 | 보고서처럼 개요·핵심 요소로 구조화 |
| 인내심 있는 스터디 친구 | 쉽게, 일상 비유로, 격려하며 | 이모지 달고 비유로 친근하게 |
| 해적 튜터 | 해적 말투지만 정확하게, 바다에 빗대서 | "야르, 반갑다" 식으로 시작 |
| 엄격한 코딩 멘토 | 정답을 바로 주지 않고 질문으로 유도(소크라테스식) | — |
| 간결한 전문가 | 군더더기 없이 최단 답변 | — |

**한국어 모델 Motif** — 추가로 해 본 것. Motif 2.6B는 학습 언어에 한국어가 들어간 국산 경량 모델. Ollama 라이브러리에는 없음(런타임마다 지원하는 모델 규격이 달라서). 대신 MLX로 돌리는 `mlx-motif`를 사용. 4bit 변환본이 1.5~2GB라 16GB 맥에서도 부담 없음.

```bash
git clone https://github.com/junhoyeo/mlx-motif && cd mlx-motif
uv venv --python 3.13 && source .venv/bin/activate
uv pip install -e .

mlx-motif generate --model junhoyeo/Motif-2.6B-MLX-q4 --prompt "Hello, world."
mlx-motif serve --port 8080 --model junhoyeo/Motif-2.6B-MLX-q4   # OpenAI 호환 서버
```

서버를 띄우면 `curl http://localhost:8080/v1/chat/completions`에 system/user 메시지와 `temperature`, `top_p`, `max_tokens`를 넣어 호출 가능. 한국어를 잘 이해해서 답변이 좋음. 별도 폴더에 설치해서 기존 `.venv`와 섞이지 않게. 더 큰 Motif 3(8월, 총 314B / 활성 13.2B, 256K, MIT)도 있음. Hugging Face에 한국어 모델이 많으니 여러 개 테스트해 볼 것.

**왜 같은 질문에 답이 매번 다른가** — 일부러 그렇게 만듦. 다양성을 위해 **확률 기반 샘플링**으로 다음 단어를 뽑기 때문. 항상 확률 1위만 뽑으면 뻔한 답만 나오는데, 창의적인 답이 필요할 때도 있으니까. 대신 테스트처럼 고정된 답이 필요할 때를 위해 조절 파라미터를 열어 둠.

참고 자료: 유튜브 임커밋 "LLM 답이 매번 다른 이유", [Transformer Explainer](https://poloclub.github.io/transformer-explainer/)(슬라이더를 움직이면 다음 단어 확률이 바뀌는 게 보임).

| 파라미터 | 하는 일 | 낮추면 | 높이면 |
| --- | --- | --- | --- |
| temperature (예: 0.7) | 각 단어의 점수(logit)를 이 값으로 나눔. 확률 분포를 뾰족하게 또는 평평하게 | 당연한 다음 단어만. 냉철하고 일관됨 | 창의적이지만 앞뒤가 안 맞을 수 있음 |
| top_k (예: 40) | 후보 단어를 상위 k개로 고정. 후보에 들어야 뽑힐 기회가 있음 | k=1이면 무조건 1위만 | 더 다양 |
| top_p (예: 0.9) | 확률을 높은 순으로 더해서 합이 p가 될 때까지만 후보. 개수가 상황에 따라 변함 | 후보가 적어 보수적 | 어휘가 다양 |
| repeat_penalty (예: 1.1) | 같은 단어·문구 반복 억제 | 반복 허용 | 반복은 줄지만 부자연스러워짐 |

- temperature 범위: 0.0~0.3은 코드·수학·사실 질문, 0.5~0.8은 일반 대화·글쓰기, 1.0~2.0은 브레인스토밍·시. 기본값 0.7에서 출발(모델과 런타임마다 다름).
- 같은 프롬프트 예: 0.0 "해가 수평선 아래로 내려가며 하늘을 주황과 분홍으로 물들였다" → 1.5 "해가 마멀레이드 빛 구름 사이로 옆으로 무너져 내렸다". 아이디어가 필요하면 올려서 받기.
- top_k는 고정 크기 후보 목록, top_p는 유연한 후보 목록. 대부분은 temperature만 조절해도 충분.
- 넣는 방법 3가지: 파이썬 `options={"temperature": 0.7, "top_p": 0.9, "top_k": 40, "repeat_penalty": 1.1}`, 터미널 대화 중 `/set parameter temperature 0.7`, 또는 API 호출 본문에 직접.

**Modelfile로 커스텀 모델** — 위 설정을 매번 넣지 말고 묶어서 이름 붙이기.

```
FROM gemma3:4b
PARAMETER temperature 0.8
PARAMETER top_p 0.9
SYSTEM """You are a friendly study buddy who explains concepts simply and
encourages the learner. Always use analogies and examples from everyday life."""
```

```bash
ollama create studybuddy -f Modelfile.studybuddy
ollama run studybuddy
ollama show studybuddy     # System, Parameters 확인
ollama rm studybuddy
```

- `FROM`만 필수. `PARAMETER`로 temperature, top_p, top_k, repeat_penalty, `num_ctx`(컨텍스트 크기)를 지정. `SYSTEM`은 큰따옴표 3개로 여러 줄.
- 새 모델이 생기는 게 아님. 베이스는 그대로 gemma3:4b이고, 파라미터와 시스템 프롬프트만 얹은 태그 같은 설정 레이어. 용량도 복제되지 않음.
- `ollama create`도 `ollama serve`가 켜져 있어야 함. 안 켜면 서버에 연결 못 한다는 에러.
- 교재 소스에 `Modelfile.studybuddy`, `Modelfile.pirate`, `Modelfile.codereview`가 이미 들어 있음. 코드 리뷰용은 temperature 0.3으로 낮게.

**Streamlit 챗봇에 붙이기** (`chatbot_custom.py`) — 사이드바에 모델 선택, temperature 슬라이더(0.0~2.0), 페르소나 프리셋 selectbox(Default / Study Buddy / Pirate Tutor / Coding Mentor / Concise Expert / Custom), 시스템 프롬프트 text_area, Clear 버튼.

```python
api_messages = []
if system_prompt:
    api_messages.append({"role": "system", "content": system_prompt})
api_messages.extend(st.session_state.messages)
response = ollama.chat(model=selected_model, messages=api_messages,
                       stream=True, options={"temperature": temperature})
```

시스템 메시지는 처음 한 번이 아니라 **매 요청마다 맨 앞에 새로 붙어서** 나감. 모델이 그 기조 아래에서 사용자 메시지를 처리해야 하니 앞에 와야 함. 프리셋을 해적으로 바꾸면 같은 챗봇이 바로 해적 말투로 답함. 내 프리셋을 더 만들어 넣어 보기.

## CH14. 오프라인 LLM의 장점

5분 정도로 짧게 넘어간 장. 대부분 당연한 여담 같은 내용이고, 제일 중요한 건 프라이버시가 걸린 사용 사례.

**오프라인 동작 확인 (비행기 모드 테스트)** — 챗봇이 되는 것 확인 → Wi-Fi 끄기 → `ping google.com`이 안 되는 것 확인 → 챗봇에 질문. 답이 정상으로 오면 완전히 로컬에서 돌아가는 것. 인터넷이 필요한 건 처음 `ollama pull` 할 때뿐.

**데이터 흐름 차이** — 클라우드(SaaS형) AI는 질문이 외부 서버로 나갔다가 답이 돌아옴. 로컬은 내 기기 안에서 끝.

| 항목 | 클라우드 AI | 로컬 AI (Ollama) |
| --- | --- | --- |
| 데이터 위치 | 회사 서버 (주로 미국) | 내 기기 |
| 전송 | 암호화되지만 전송됨 | 전송 없음 |
| 보관 | 30일 이상 저장될 수 있음 | 내가 정함 |
| 학습 활용 | 프롬프트와 답변이 모델 개선에 쓰일 수 있음 | 없음 |
| 유출 위험 | 서버 침해 영향 | 내 기기가 뚫릴 때만 |

고객 데이터, 개인식별정보, 소스 코드, 계약서, 내부 전략을 모델에 넣어야 할 때 이 차이가 결정적. 클라우드 회사가 나쁘다는 게 아니라, 데이터가 기기를 떠나는 구조 자체의 문제.

**비용** — 로컬을 고르는 현실적인 이유.

- 클라우드: 구독은 월 20달러 안팎, API를 많이 쓰면 월 100~500달러 이상.
- 로컬: Ollama·모델·Python·Streamlit 모두 무료. 하드웨어와 전기요금만. 교재 예시로는 개인 1명이 2.5년, 5명 팀이 6개월이면 맥미니 값을 뽑음.
- 종속성도 비용: 가격 인상, 약관 변경, 모델 단종, 장애. 로컬 모델 파일은 내 디스크에 있으니 내일도 그대로 동작.
- 그래도 최고 성능이 필요한 작업은 클라우드. 결론은 **하이브리드**: 일상·민감한 작업은 로컬, 최고 성능이 필요할 때만 클라우드. 오픈 웨이트 모델이 계속 좋아지고 있어서 로컬도 충분히 쓸 만함.

**로컬이 필수인 사례** (이 장에서 제일 중요)

- 의료: 환자 건강정보는 나라마다 규제가 빡셈(미국 HIPAA 등, 국내도 마찬가지). 병원 데이터로 LLM을 쓰려면 로컬밖에 없음.
- 법률: 변호사와 의뢰인 사이 비밀 유지. SaaS형 도구에 올릴 수 없음.
- 교육(학생 데이터), 창작(내 작품이 학습에 쓰이는 것 방지), 언론·연구(취재원·기밀 문서), 개인 재무·정신 건강, 기업 영업비밀.
- 인터넷이 없는 환경: 배, 오지, 재난 현장.

판단 기준: 데이터에 민감 정보가 있나 → 있으면 로컬. 없으면 최신 성능이 필요한가 → 필요하면 클라우드, 아니면 비용이 부담될 때 로컬.

## 로컬 LLM으로 CML 2.10 다루기 (별도 메모)

이번 주 내용(모델 선택, temperature, 오프라인)을 내 CML 랩에 붙여 보면 어디까지 되나 정리. 결론: 랩 조회·노드 시작/중지·show 명령 실행 후 해석·간단한 설정 투입까지는 로컬 LLM으로 충분히 됨. 여러 단계를 이어서 판단해야 하는 트러블슈팅이나 긴 설정 생성은 작은 모델에선 불안정해서 결과를 꼭 show로 검증해야 함.

**CML 2.10에서 AI 관련으로 바뀐 점**

- **MCP 서버 내장**: AI 도구에 CML MCP 서버를 연결하면 스크립트나 API 호출 없이 토폴로지 생성, 노드 명령 실행, 네트워크 점검이 가능. `GET /ai/mcp/configuration` API가 클라이언트 설정값을 돌려줌. 릴리스 노트에 전송 방식(stdio/HTTP)과 인증 방식은 안 나와 있어서 내 서버에서 직접 확인 필요.
- 패킷 캡처를 Breakout UI로 내 PC의 Wireshark에 실시간 스트리밍.
- 랩 자동 시작(서버 부팅 시 순서대로 기동), 노드 기동 순서(staging), 무선 랩(베타).
- 2.10.1(10/1)은 업그레이드 오류 수정과 보안 패치 위주. 온라인 업그레이드 시 `download.docker.com` 접속이 필요.

**연결 구조** — 모델 → 도구 → 장비 순서로 이어짐.

1. Ollama 로컬 모델 (도구 호출 지원 모델이어야 함)
2. MCP 클라이언트: `ollmcp` (터미널) 또는 Open WebUI (웹, 다음 주 CH16)
3. MCP 서버: CML 2.10 내장 서버 또는 오픈소스 `cml-mcp`
4. CML REST API → 노드 콘솔 (CLI는 pyATS 경유)

**MCP 서버/클라이언트 고르기**

| 구성요소 | 선택지 | 메모 |
| --- | --- | --- |
| MCP 서버 | CML 2.10 내장 | 설치 불필요. 설정값은 `/ai/mcp/configuration`으로 확인 |
| MCP 서버 | `cml-mcp` (오픈소스) | CML 2.9 이상, Python 3.12~3.14. 도구 52개(랩·노드·링크·어노테이션·패킷 캡처·사용자/그룹·시스템 정보). CLI 실행 도구는 `cml-mcp[pyats]`로 설치해야 생김. stdio 기본, HTTP 모드는 YAML ACL로 사용자별 도구 제한 가능 |
| 클라이언트 | `ollmcp` | `uv tool install ollmcp`. stdio·SSE·HTTP 전부 지원. 에이전트 모드로 도구를 연쇄 호출(기본 최대 7회). 추천 모델 qwen3.5, gemma4, llama3.2, mistral |
| 클라이언트 | Open WebUI | v0.6.31부터 MCP 내장. 단 Streamable HTTP만 지원해서 stdio 서버는 `mcpo` 프록시로 감싸야 함 |

```json
"Cisco Modeling Labs CML": {
  "type": "stdio",
  "command": "uvx",
  "args": ["cml-mcp[pyats]"],
  "env": {
    "CML_URL": "https://<CML 주소>",
    "CML_USERNAME": "<계정>",
    "CML_PASSWORD": "<비밀번호>",
    "CML_VERIFY_SSL": "false"
  }
}
```

CML은 자체 서명 인증서라 `CML_VERIFY_SSL`을 false로 두거나 CA 번들을 지정. CLI 명령용 장비 계정은 `PYATS_USERNAME`, `PYATS_PASSWORD`, `PYATS_AUTH_PASS`로 따로 넣음. pyATS는 Windows에선 WSL이나 Docker가 필요하지만 맥은 바로 됨.

**작업별로 어디까지 되나** (16GB 맥, 4B~8B 모델 기준 예상)

| 작업 | 로컬 LLM | 메모 |
| --- | --- | --- |
| 랩 목록·상태 조회, 노드 시작/중지 | 잘 됨 | 도구 1~2번 호출로 끝남 |
| show 명령 실행 후 해석 (`show ip route`, `show ip ospf neighbor`) | 잘 됨 | 출력이 길면 `num_ctx`를 늘려야 잘리지 않음 |
| 랩 토폴로지 생성 (라우터 3대 + 링크) | 보통 | 4B급은 인터페이스 이름이나 연결을 자주 틀림. qwen 8B 이상, 단계를 나눠 요청 |
| OSPF·BGP 같은 설정 투입 | 보통 | 명령어를 지어내는 경우가 있음. 투입 후 show로 반드시 검증 |
| 트러블슈팅 (neighbor가 왜 안 올라오나) | 어려움 | 조회 → 비교 → 수정을 여러 번 연쇄. 작은 모델은 루프 한계에 걸리거나 엉뚱한 결론 |
| 패킷 캡처 분석 | 부분적 | PCAP 다운로드는 도구로 되지만 모델에 넣으려면 tshark로 텍스트 변환 후 요약 |
| 사용자·그룹 관리 (실습 계정 생성) | 가능하지만 주의 | 관리자 권한 필요. 실수 시 영향이 커서 HTTP 모드 ACL로 막아 두는 게 안전 |

**이번 주 내용과 연결**

- CH12 모델 선택: 도구 호출 지원 여부부터 확인. 한국어로 지시하면 Qwen 계열이 유리. 16GB 맥이면 8B까지가 현실적.
- 도구 설명 52개가 전부 프롬프트에 들어가서 컨텍스트를 꽤 차지함. 필요한 도구만 켜고 `num_ctx`를 넉넉히(8192 이상).
- CH13 temperature: 설정 생성·투입은 0.1~0.3으로 낮게. 창의성이 필요 없는 작업.
- CH13 Modelfile: 네트워크 엔지니어 역할 시스템 프롬프트 + 낮은 temperature + 큰 `num_ctx`를 묶어 `cml-assistant` 같은 커스텀 모델로 만들어 두면 편함.
- CH14 오프라인: 랩 토폴로지, 장비 설정, 사내 IP 계획이 외부로 안 나감. 실제 망 구성을 그대로 넣어 연습해도 됨.
- 안전: wipe·delete 같은 파괴적 도구는 끄거나 실행 전에 확인을 받게.

**해 볼 것 (CML)**

- [ ] 내 CML에서 `GET /ai/mcp/configuration` 호출해 내장 MCP 서버 설정 확인
- [ ] `ollmcp` + qwen3.5:4b로 "랩 목록 보여줘", "R1 시작해줘", "R1에서 show ip int brief 실행해줘"
- [ ] 같은 요청을 4B와 8B 모델로 비교 (토폴로지 생성 정확도, 도구 호출 횟수)
- [ ] OSPF 설정을 투입시키고 neighbor 확인까지 한 번에 시켜 보기, 어디서 멈추는지 기록

참고: [CML 2.10 Release Notes](https://developer.cisco.com/docs/modeling-labs/cml-release-notes/), [cml-mcp (PyPI)](https://pypi.org/project/cml-mcp), [mcp-client-for-ollama (ollmcp)](https://pypi.org/project/mcp-client-for-ollama/), [Open WebUI MCP](https://docs.openwebui.com/features/mcp)

## 해 볼 것

- [ ] 4주차 과제: 배운 내용이나 Ollama 모델 실행을 정리해서 공개 글로 올리고 스터디 슬랙 채널에 링크 공유
- [ ] 도전: 내 맥 스펙으로 돌릴 수 있는 최대 크기 모델까지 올려 보기. RAM을 넘기는 모델도 일부러 띄워서 어떤 에러가 나는지 확인
- [ ] CH12: gemma / qwen / llama에 같은 한국어 질문 던져 비교, `test_models.py`로 답과 시간 비교, 챗봇에 모델 선택기 넣기
- [ ] CH13: 페르소나 5개 만들어 "하늘은 왜 파란가" 비교, temperature 0.0 / 0.5 / 1.0 / 1.5 실험(어디서부터 말이 안 되는지), Modelfile 3개(창작용·사실 Q&A용·내 공부용), `chatbot_custom.py`에 내 프리셋 2개 추가, `num_ctx` 512와 4096으로 긴 대화 비교
- [ ] CH13 추가: mlx-motif로 Motif 2.6B 돌려서 한국어 답변 비교, Transformer Explainer에서 temperature·top-k·top-p 움직여 보기, 임커밋 영상 보기
- [ ] CH14: Wi-Fi 끄고 챗봇에 질문 5개 + 모델 전환, 최근 AI에 물은 10가지 중 로컬로 될 것 세어 보기, 내 하이브리드 전략 표 만들기

다음 주 11/18: 마지막 주. CH15~18 — 오류와 문제 해결, RAG, Open WebUI, LangChain, 에이전트, 오픈 모델 현황.
