# 로컬 AI 스터디 3주차 정리 (CH8\~11)

Nov 8, 2026 · @Kiyong Kang

11/4 3주차 스터디. 교재 Part 2 마지막 CH8\~11. 지난주까지 만든 파이썬 + Ollama 호출에 웹 UI(Streamlit)와 음성 인식(MLX Whisper)을 붙여서 드디어 "말로 질문하는 로컬 AI 챗봇"이 됨. 순서는 CH8 웹 UI → CH9 오디오 → CH10 둘을 합친 음성 챗봇 → CH11 Streamlit의 단점(매번 처음부터 재실행)을 세션 상태로 보완. 오늘부터는 좀 써먹을 만한 앱 느낌.

## CH8. Streamlit 웹 UI

지난주엔 파이썬 스크립트로 LLM에 질문하고 답을 받았는데, 일반 사용자는 웹 화면이 익숙하니 Streamlit으로 웹 인터페이스를 만든다. HTML/CSS/JS 없이 파이썬 함수만 호출하면 Streamlit이 웹 페이지를 자동 생성. 데이터 앱·대시보드·챗봇 UI용이지 쇼핑몰 같은 건 아님.

**첫 앱** — 지난주 가상환경 들어가서 `pip install streamlit` → `pip freeze > requirements.txt` 갱신 (의존성이 꽤 많이 깔림). `app.py` 만들고:

```python
import streamlit as st

st.title("Hello, Streamlit!")
st.write("This is my first web application.")
```

실행은 `python app.py`가 아니라 `streamlit run app.py`. 브라우저가 자동으로 뜨고 포트는 8501(변경 가능). 두 번째 앱을 또 띄우면 8502, 8503… 하나씩 올라감. 멈출 땐 Ctrl+C.

- 입력 받기: `name = st.text_input("What is your name?")` → `if name:` 로 비었는지 검사. 빈 문자열은 False라 입력 전엔 인사말이 안 나오고, 입력하면 True가 되어 나옴.
- 파일 저장하면 브라우저 오른쪽 위에 "코드 바뀜" 알림 → Rerun 또는 Always rerun. 클릭하기 싫으면 `runOnSave = true`.
- **핵심 동작**: 입력이 생길 때마다 스크립트를 **맨 위부터 다시 실행**함. 입력 하나 바꿨는데 페이지 전체를 다시 렌더링하니 비효율적이지만, 콜백·이벤트 핸들러 없이 위에서 아래로 쭉 쓰면 되니까 코드가 단순. 이 단점은 CH11 세션 상태로 보완.
- 통계 전송: Streamlit은 기본으로 사용 통계를 본사 서버로 보냄. 찜찜하면 `~/.streamlit/config.toml`에 `[browser] gatherUsageStats = false` 넣어두면 안 나감.

**컴포넌트** — 하나씩 코드에 넣고 새로고침하면서 눈으로 익히는 게 빠름.

| 함수 | 역할 |
| --- | --- |
| `st.title / header / subheader` | 제목 크기 3단계 |
| `st.write` | 만능. 문자·숫자·데이터프레임·차트 다 됨. 모르겠으면 이걸로 |
| `st.markdown / st.code` | 볼드·기울임, 코드 블록 |
| `st.text_input / text_area` | 한 줄 / 여러 줄 입력 |
| `st.number_input / slider` | 숫자 (+/-), 슬라이더 |
| `st.selectbox / radio / checkbox` | 드롭다운, 라디오, 체크 |
| `st.button` | 클릭하면 True, 아니면 False. 다음 rerun에 다시 False라 출력이 사라짐 |
| `with st.sidebar:` | 왼쪽 접히는 패널. 설정은 사이드바, 대화는 메인 — 챗봇도 이 구조 |
| `st.columns(3)` | 가로 3칸 레이아웃, `st.metric`과 같이 씀 |

교재의 `components_demo.py`를 같은 폴더에 만들어 돌리면 전부 한 화면에서 볼 수 있음. 종합 실습 `greeting_app.py`는 사이드바에서 언어(English/Italian/French)와 격식(Casual/Formal) 고르고 이름 넣으면 맞춤 인사. 이게 그대로 챗봇 레이아웃.

유용한 옵션: `streamlit run app.py --server.port 8502`, 브라우저 안 열려면 `--server.headless true`.

## CH9. MLX Whisper 음성 인식

오디오 부분을 추가하는 장. 이걸 하고 나면 CH10에서 음성 챗봇이 됨.

- 음성 인식 = ASR(Automatic Speech Recognition). 사람 말을 문자로. 예전엔 음소 규칙을 손으로 만들었는데 지금은 신경망.
- **Whisper** = OpenAI가 2022년 공개. 인터넷의 다국어 오디오 68만 시간(팟캐스트, 영상, 강의)으로 학습. 언어의 통계적 패턴까지 배워서 영어는 사람 수준, 다른 언어도 근접. 완전 오프라인으로 돌아감.
- 구조는 인코더-디코더 트랜스포머. GPT가 텍스트→텍스트라면 Whisper는 오디오용으로 변형한 것. 음성 → 스펙트럼(주파수 그림) → 인코더가 오디오 패턴 인식 → 디코더가 텍스트를 한 토큰씩 생성.
- **왜 MLX Whisper?** 원본은 PyTorch라 NVIDIA GPU 기준. 맥에서 그냥 쓰면 CPU만 씀. MLX는 애플이 만든 Apple silicon용 프레임워크라 통합 메모리 + GPU + 뉴럴 엔진을 다 활용. 맥미니 M2 기준 표준 대비 4\~6배 빠름(tiny 8초→2초, small 28초→6초). 결론 한 줄: 맥이면 무조건 MLX Whisper.

**설치**: `pip install mlx-whisper` + `pip install sounddevice numpy scipy`. sounddevice = 맥 마이크 접근, numpy = 오디오를 숫자 배열로, scipy = WAV 파일 쓰기. 깔고 나서 `pip freeze | grep -E "mlx-whisper|sounddevice|numpy|scipy" >> requirements.txt`.

주의: 지금 설치하면 mlx-whisper 1.4.3인데 교재의 `check_versions.py`(`mlx_whisper.__version__`)는 실패함. `from importlib.metadata import version` 으로 바꿔 쓰면 됨.

**디지털 오디오 숫자 3개** (코드마다 나옴)

| 항목 | 값 | 메모 |
| --- | --- | --- |
| 샘플레이트 | 16 kHz | 초당 측정 횟수. 음악은 44.1 kHz, Whisper는 16 kHz로 학습됐으니 처음부터 16으로 녹음 |
| 채널 | mono (1) | 스테레오는 좌우 분리. Whisper는 모노 |
| 비트 깊이 | float32 → int16 | 파이썬에서는 -1.0\~1.0 float32로 다루고, WAV 표준은 int16이라 저장할 때 변환 |

**record.py** — 5초 녹음해서 `recording.wav` 저장. 핵심 세 줄:

```python
audio = sd.rec(int(duration * sample_rate), samplerate=16000, channels=1, dtype="float32")
sd.wait()                                   # 녹음 끝날 때까지 대기
return audio.squeeze()                      # (80000, 1) → (80000,)
```

- 5초 × 16,000 = 80,000개 샘플. `sd.rec`가 돌려주는 배열은 열이 하나인 표 모양 `(80000, 1)`인데 MLX Whisper는 평평한 리스트 `(80000,)`를 원해서 `.squeeze()`로 채널 차원 제거. 내용은 같고 모양만 바뀜.
- WAV 저장: `(audio * 32767).astype(np.int16)`. float32에 32767 곱하고 소수점 버려서 정수로.
- 처음 실행하면 맥에서 마이크 권한 창이 뜸. VS Code 터미널이면 VS Code에, 터미널이면 터미널에 허용.

**transcribe.py** — `mlx_whisper.transcribe(str(audio_path), path_or_hf_repo=model)` 한 줄. 결과는 딕셔너리, `result["text"]`만 쓰면 됨 (`segments`, `language`도 있음). 처음엔 ffmpeg 없다고 에러 → `brew install ffmpeg`. 모델은 Hugging Face에서 첫 실행 때 받고 `~/.cache/huggingface/hub/`에 캐시.

교재엔 없지만 `LANGUAGE = "ko"` 상수 추가해서 `language=language`로 넘김. None이면 자동 감지인데 어차피 한국말 할 거니까 지정하는 게 빠르고 정확.

**모델 고르기** — 메모리가 제약. Ollama gemma3:4b가 2\~3GB 먹으니 그 위에 Whisper 메모리가 얹힘.

| 모델 | 파라미터 | 메모리 | 30초 처리 | 정확도 |
| --- | --- | --- | --- | --- |
| whisper-tiny-mlx | 39M | \~200MB | \~1초 | 기본 |
| whisper-small-mlx | 244M | \~800MB | \~4초 | 꽤 좋음 |
| whisper-medium-mlx | 769M | \~2GB | \~8초 | 우수 |
| whisper-large-v3-mlx | 1550M | \~4GB | \~18초 | 사람 근접 |

통합 메모리 8GB면 small, 16GB면 medium부터, 그 이상이면 large-v3. 모델명은 `mlx-community/whisper-{size}-mlx`. 실제로 medium은 "안녕하세요 가시다입니다"를 정확히 받아썼는데 tiny로 바꾸니 이름이 엉뚱하게 나옴. 모델 바꿔가며 직접 비교해 볼 것.

**voice\_input.py** — 녹음 + 받아쓰기를 한 파일로 합친 재사용 모듈. `python voice_input.py --duration 5 --model mlx-community/whisper-small-mlx`로 바로 쓰거나, CH10에서 `from voice_input import record_and_transcribe`로 import. 임시 WAV는 자동 삭제. my-ai-chatbot 폴더 바로 아래에 둘 것.

트러블슈팅: 마이크 권한(시스템 설정 → 개인정보 보호 → 마이크), 빈 텍스트면 입력 볼륨·소음·녹음 시작 전에 말했는지 확인, import 안 되면 가상환경 확인 후 `pip install --upgrade mlx-whisper`.

## CH10. 음성 AI 채팅 앱 (핵심)

지금까지 배운 걸 전부 엮은 핵심 앱. 마이크에 말하면 텍스트가 되고, LLM 답이 스트리밍으로 나옴. 전부 내 맥에서, 인터넷 없이.

**파이프라인** (동작에 순서가 있음)

1. 오디오 캡처 — 브라우저. Streamlit의 `st.audio_input()`이 Web Audio API로 마이크 녹음 → WAV 바이트를 파이썬으로 넘김
2. 음성 인식 — MLX Whisper가 WAV → 텍스트 (맥 GPU)
3. LLM 추론 — 텍스트 + 대화 기록을 Ollama에 넣고 토큰 단위 스트리밍 → Streamlit 채팅 UI에 출력

준비: `my-ai-chatbot/ch10/` 폴더 만들고 `voice_chat.py`, `model_config.json`(사이드바 모델 목록), `setup_ollama.sh`. `voice_input.py`는 상위 폴더에. 가상환경에 ollama 없으면 `pip install ollama`.

**1단계: Streamlit에서 받아쓰기만**

```python
audio_value = st.audio_input("Click the microphone to record your message")
if audio_value is not None:
    with tempfile.NamedTemporaryFile(suffix=".wav", delete=False) as f:
        f.write(audio_value.read())
        tmp_path = f.name
    with st.spinner("Transcribing..."):
        result = mlx_whisper.transcribe(tmp_path, path_or_hf_repo=WHISPER_MODEL)
    os.unlink(tmp_path)
    st.success(f"**You said:** {result['text'].strip()}")
```

임시 WAV 파일을 일부러 떨구는 이유: MLX Whisper가 파일 경로로 받기 때문. `delete=False` 안 주면 with 블록 끝나자마자 지워져서 Whisper가 못 읽음. 그래서 받아쓰기 끝나고 `os.unlink`로 직접 삭제. 브라우저에서 마이크 권한 허용하고 "안녕하세요 가시다입니다" → 바로 텍스트로 나옴.

**2단계: Ollama 연결** — `st.session_state.messages` 리스트에 user/assistant를 쌓고, 받아쓴 텍스트를 `ollama.chat(..., messages=st.session_state.messages, stream=True)`로. `placeholder = st.empty()` 만들어 두고 chunk마다 `placeholder.markdown(response_text + "▌")`로 커서 효과.

실행 전에 **`ollama serve` 켜져 있어야 함**. 안 켜고 녹음하니 연결 에러 났음. 켜고 다시 "안녕하세요 저는 가시다입니다" → Whisper 텍스트 → Ollama 답변. "내 이름이 뭐라고?" 하니 기억함. `ollama serve` 터미널에 요청 로그도 찍힘.

**3단계: 완성본** — 사이드바에 Ollama 모델 선택, Whisper 모델 선택, temperature 슬라이더, Clear Conversation 버튼. 메인에 마이크 + 텍스트 입력. 코드 구조:

| 함수 | 역할 |
| --- | --- |
| `st.set_page_config(layout="wide")` | 첫 번째 Streamlit 호출이어야 함. 넓은 화면 |
| 사이드바 selectbox ×2 | `format_func`로 사용자에겐 친절한 라벨, 코드엔 `gemma3:4b` 같은 실제 태그 |
| `transcribe_audio()` | Streamlit ↔ Whisper 다리. 임시 파일 → 받아쓰기 → `try/finally`로 반드시 삭제 → `.strip()`으로 앞뒤 공백 제거 |
| `stream_response()` | Ollama 스트리밍을 함수로 뺌(리팩토링). 자주 쓰는 로직은 함수 하나로 |
| `handle_user_message()` | 통합 진입점. 음성이든 텍스트든 여기로. LLM 입장에선 어디서 온 텍스트인지 상관없음 |
| `st.chat_input()` | 텍스트 폴백. 조용한 곳이거나 받아쓰기 틀렸을 때 타이핑 |

temperature 낮으면 보수적, 높으면 무작위. 0.7이 무난. "안녕 나 가시다야, 서울 여행 추천해줘" 음성으로 넣고, 그 다음 텍스트로 이어서 물어도 맥락이 이어짐.

**테스트 체크리스트**: 기본 음성 → 기술 용어(MLX Whisper, Ollama) 받아쓰기 정확도 → 텍스트 폴백 → "내 이름은 Sam" 세 번 뒤 "내 이름이 뭐야?" → 모델 바꿔서 재시도 → Clear.

**레이턴시 버짓** — IT 쪽에서 지연을 "예산"으로 표현. 말 끝나고 답이 시작될 때까지 사용자는 참을성이 없음. 5초 음성, M2 8GB, whisper-small + gemma3:4b 기준: 업로드 0.1초 + Whisper 1.5초 + 첫 토큰 0.5초 = 약 2.1초. large-v3로 바꾸면 4\~5초. 단계가 길어서(업로드 → STT → LLM) 텍스트 채팅보다 느릴 수밖에 없음. `st.spinner("Transcribing...")`가 "죽은 거 아님"을 알려줘서 중요.

문제 생기면: 녹음 2초 이상·마이크 볼륨, 받아쓰기 부정확하면 medium/large-v3, 연결 거부면 `ollama serve`, 첫 실행 멈춤은 모델 다운로드 중, 브라우저 마이크 권한.

## CH11. 세션 상태와 대화 기록

CH10 앱을 보완하는 기능들. 시간이 없어서 빠르게 넘어갔지만 핵심은 두 가지 문제를 푸는 것: (1) 웹 앱에 보이는 대화를 어떻게 유지하나 (2) 그중 얼마만큼을 LLM에 보내나. 세션 상태는 **앱의 기억**이지 모델의 기억이 아님.

**1. Streamlit 세션 상태** — Streamlit은 상호작용마다 스크립트를 처음부터 다시 돌리니 일반 변수는 매번 초기화됨. 음성 챗봇에서 두 번째 질문을 넣으면 첫 질문부터 다시 실행되는 비효율도 이것 때문. `st.session_state`는 rerun을 넘어 살아남는 특별한 딕셔너리.

`counter_demo.py`로 확인: Increment/Decrement/Reset 버튼. 원래는 누를 때마다 새로 시작해서 기록이 없어야 하는데 세션 상태에 넣으니 값이 쌓임.

```python
if "counter" not in st.session_state:   # 처음 한 번만 초기화
    st.session_state.counter = 0
if st.button("Increment"):
    st.session_state.counter += 1
    st.rerun()
```

챗봇의 `if "messages" not in st.session_state: st.session_state.messages = []`도 똑같은 패턴. 문자열·숫자·리스트·딕셔너리·불린 다 저장 가능. 탭 닫기, 새로고침, 서버 재시작 때 리셋됨. 사용자마다 세션이 따로라 여러 명이 써도 대화가 안 섞임.

**2. LLM은 상태가 없다** — 2주차에 봤던 것. `messages=st.session_state.messages`로 매번 대화 전체를 보내니 기억하는 것처럼 보일 뿐. 10번째 질문엔 1\~9번 질문과 답이 전부 같이 감. 파이썬 리스트는 mutable이라 append만 하면 다음 호출에 자동 포함. 에이전틱 AI(Claude Code, Codex)가 맥락을 이해하는 원리도 같음.

**3. 컨텍스트 윈도우** — 모델이 한 번에 처리할 수 있는 최대 토큰. gemma3:4b는 기본 131,072 (`ollama ps`로 실제 할당값 확인, 변경 가능). 토큰 ≈ 영어 3\~4글자, 한국어는 영어보다 토큰 효율이 나쁨. 대충 글자 수 ÷ 4로 추정. 넘어가면 오래된 메시지를 조용히 버리거나 모순된 답을 냄 → 챗봇이 정신 놓은 것처럼 보임. 넘지 않아도 컨텍스트가 길면 매 답변이 느려짐. 유료 SaaS(Claude Code 등)는 알아서 해주지만 로컬은 내가 관리해야 함.

**4. 트리밍** — 사용자는 브라우저에서 전체 대화를 보지만, 모델에는 최근 N쌍만 보냄.

```python
max_history = st.slider("Max History (message pairs)", 1, 50, 10)
max_messages = max_history * 2 + 1          # user+assistant = 1쌍, +1은 지금 질문
messages_for_model = st.session_state.messages[-max_messages:]
```

100개 주고받았는데 50으로 잡으면 앞 50개는 안 보냄. 단순한 휴리스틱이라 유난히 긴 메시지는 여전히 넘칠 수 있음. 더 나은 방법은 오래된 대화를 요약해서 넣기, 또는 RAG로 저장해 두고 필요한 부분만 꺼내기. 대부분은 10\~20쌍 트리밍이면 충분.

**5. 편의 기능** — `chat_context.py` 사이드바에 넣어본 것들:

- 대화 통계: `st.metric("Messages", 총 개수)`, `st.metric("Approx. Tokens", 총 글자수 // 4)`. 메시지 2개 보내니 토큰 21개, 계속 올라가는 게 보임. 넘어가기 전에 초기화할지 판단 가능.
- Clear Chat 2단계 확인: `confirm_clear` 플래그를 세션 상태에 두고, 처음 누르면 True + rerun → "정말 지울까?" 경고와 Yes/Cancel 버튼. 원래는 바로 지워졌는데 실수 방지. 어떤 파괴적 동작에도 쓸 수 있는 패턴.
- 대화 시작 시각 저장해서 "지금 몇 분째 대화 중" 표시, 모델 바뀌면 감지해서 안내.

**전체 데이터 흐름** (타이핑 → 답변): 입력 → 스크립트 전체 rerun → 사이드바 렌더 → 세션 상태 확인 → 저장된 메시지 전부 표시 → 새 메시지 감지 → 세션에 append → 말풍선 → max\_history로 트리밍 → `ollama.chat()` → LLM이 토큰 생성 → placeholder에 스트리밍 → 완료 후 세션에 저장. 느린 건 LLM 생성 한 단계뿐인데 스트리밍 덕에 빠르게 느껴짐.

## 해 볼 것

연습문제가 꽤 괜찮으니 직접 해 보기. 앱 하나 만드는 느낌이 나는 주.

- [ ] CH8: Hello 앱에 이름·서브헤더·markdown 볼드 넣기, 숫자 두 개 + 연산 selectbox로 계산기
- [ ] CH9: `voice_input.py --duration 10`으로 5\~6문장 받아쓰기 정확도 확인, tiny/small/medium 속도·정확도 비교(`compare_models.py`), 배경 음악 켜고 테스트, 내 음성 파일 받아쓰기(`ffmpeg -i in.mp3 -ar 16000 -ac 1 out.wav`), 언어 자동 감지
- [ ] CH10: 받아쓴 텍스트를 채팅 기록에 남기기, 사이드바에 감지 언어 표시(`result.get("language")`), 시스템 프롬프트 "음성 비서니까 2\~3문장으로" 넣고 비교, 받아쓰기 신뢰도 경고, LLM 없이 받아쓰기만 하는 `voice_memo.py` + `st.download_button`
- [ ] CH11: 세션 상태로 to-do 리스트, 모델에 실제 보내는 토큰 수 표시 + 컨텍스트 75% 넘으면 경고, 대화 다운로드 버튼, max\_history=3으로 "내 이름은 Alex" 실험, 오래된 대화 요약해서 system에 넣기
- [ ] 컨텍스트 넘길 때까지 대화해서 얼마나 멍청해지는지, 트리밍이 잘 되는지 직접 확인

다음 주: 여기서 더 발전한 내용(Part 3).
