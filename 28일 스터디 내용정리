# 로컬 AI 스터디 2주차 정리 (CH4~7)

10/28 2주차 스터디. 교재 Part 2 중 CH4~7까지 진행. Ollama로 모델 받아서 대화 → VS Code/파이썬 환경 → 파이썬 코드로 Ollama 호출까지. 오늘 핵심은 CH7.

## CH4. LLM 다운로드와 첫 대화

기본 모델은 Gemma 3 4B (40억 파라미터, Apache 2.0). 터미널 하나에 `ollama serve` 띄워 놓고, 다른 터미널에서 `ollama pull gemma3:4b` → `ollama run gemma3:4b`. 나갈 때는 `/bye`.

- 같은 질문을 두 번 해도 답이 조금씩 다름 → 비결정적(non-deterministic). 확률로 답을 만들기 때문.
- 파라미터 = 학습하면서 조정된 가중치. 뇌세포 개수라고 생각하면 됨. 많을수록 정교하지만 모델 전체가 메모리에 올라가야 돌아감.
- 맥은 통합 메모리라 메모리의 절반 이하 크기 모델이 안전. 내 맥 16GB → 8B 이하로. 활성 상태 보기에서 메모리 확인.
- 양자화 = 16비트 가중치를 8비트 등으로 압축해서 메모리 줄이기. 너무 낮추면 품질 떨어짐.
- 같은 4B라도 Gemma 3와 Qwen 3.5 답이 다름. 학습이 다르니 가중치가 다름.
- 토큰 = LLM이 다루는 최소 단위. 비용도 컨텍스트 길이도 토큰 기준. 단어 하나 ≠ 토큰 하나.

| 명령 | 용도 |
| --- | --- |
| `ollama pull <모델>` | 모델 다운로드 |
| `ollama list` | 받아 둔 모델 |
| `ollama ps` | 메모리에 올라온 모델 |
| `ollama show <모델>` | 파라미터 수, 컨텍스트 길이, 양자화 |
| `ollama rm <모델>` | 모델 삭제 |

## CH5. VS Code와 파이썬 개발환경

앱을 만들려면 결국 파이썬 코드를 짜야 하니 편집기와 파이썬부터.

- VS Code: `brew install --cask visual-studio-code`. 터미널에서 `code .` 치면 현재 폴더가 열림.
- Python: `brew install python@3.13`. 다른 버전이 이미 있으면 uv로 3.13 환경 따로 만들면 됨.
- Microsoft Python 확장 설치 → 하이라이트, 자동완성, 오류 밑줄, 함수 정보.
- 통합 터미널(보기 → 터미널)로 편집·실행·확인을 VS Code 안에서 끝냄.
- 작업 폴더 만들고 `hello.py`에 print 한 줄 → `python hello.py`.

## VS Code 원격 연결 - 대부분 원격 연결 사용할 것이라 예상

모델은 맥에서 돌리고 코드 작업은 다른 PC에서 하는 방법. Remote - SSH 확장 쓰면 원격 폴더가 로컬처럼 열림.

1. Mac: 시스템 설정 → 일반 → 공유 → 원격 로그인 켜기. IP는 `ipconfig getifaddr en0`.
2. 접속하는 PC의 VS Code에 Microsoft **Remote - SSH** 확장 설치.
3. 왼쪽 하단 `><` → Connect to Host → `사용자명@맥IP` → 맥 비밀번호. 같은 Wi-Fi여야 함.
4. File → Open Folder로 맥의 프로젝트 폴더 열기. 터미널도 맥에서 실행되니 `source .venv/bin/activate`, `python chat_basic.py` 그대로.
5. Streamlit(CH8~)은 Ports 탭에서 8501 포워딩하거나 `http://맥IP:8501`.

비밀번호 매번 치기 싫으면 `ssh-keygen` → `ssh-copy-id 사용자명@맥IP`. Ollama 앱 첫 실행처럼 클릭이 필요한 건 맥 앞에서.

## CH6. 파이썬 가상환경

프로젝트마다 독립된 파이썬을 두는 것. 패키지 버전 충돌 방지 + 환경 공유가 쉬움. A와 B가 같은 패키지의 다른 버전을 쓰면 공용 설치에서는 깨지지만 가상환경에서는 각자 격리.

1. 만들기: `python3.13 -m venv .venv` → `.venv` 안에 bin, include, lib 생김.
2. 활성화: `source .venv/bin/activate` → 프롬프트 앞에 `(.venv)`, `which python`이 `.venv/bin/python`.
3. 해제: `deactivate`.
4. 설치: `pip install ollama streamlit`. pip = PyPI(파이썬 앱스토어)에서 받아서 `.venv/lib`에 넣음. `pip list`로 확인.
5. 고정: `pip freeze > requirements.txt`. 다른 PC에서 `pip install -r requirements.txt`로 같은 버전 복원. 패키지 추가하면 다시 freeze.

## CH7. Ollama 파이썬 라이브러리 (핵심)

터미널 채팅 대신 파이썬 코드로 질문하고 답 받기. LLM 서빙 동작을 가장 짧은 코드로 볼 수 있는 장.

**기본 호출** (`chat_basic.py`)

```python
import ollama

response = ollama.chat(
    model="gemma3:4b",
    messages=[{"role": "user", "content": "What is the capital of France?"}],
)
print(response["message"]["content"])
```

`import`로 라이브러리 불러오고 `ollama.chat()`에 모델 + 메시지 리스트. 로컬 Ollama 서버가 모델에 넘겨서 답을 딕셔너리로 돌려줌. `message` → `content`만 꺼내서 출력.

**메시지 역할 3가지**

| 역할 | 의미 | 예 |
| --- | --- | --- |
| system | AI 성격·규칙(페르소나) | "친절한 과학 선생님, 100단어 이하" |
| user | 내 질문 | "중력은 어떻게 작동해?" |
| assistant | AI의 이전 답 | 대화 기록 유지에 사용 |

system을 "과학 선생님"에서 "셰익스피어풍 배우"로 바꾸니 같은 질문인데 말투가 완전히 달라짐. 한글로 물으면 한글로 답함.

**스트리밍** (`chat_stream.py`)

```python
stream = ollama.chat(model="gemma3:4b", messages=[...], stream=True)
for chunk in stream:
    print(chunk["message"]["content"], end="", flush=True)
print()
```

`stream=True`면 딕셔너리 대신 조각(chunk)이 순서대로 오는 이터레이터. for로 토큰 나올 때마다 바로 출력. 전체 시간은 같은데 기다리는 체감이 줄어서 이탈이 적음.

**대화 기록** (`chat_history.py`)

LLM은 기억이 없음. history 리스트에 user/assistant 메시지를 쌓아서 매번 전체를 보내야 함. "내 이름은 가시다" → "내 이름이 뭐지?" 하면 기억해서 답함. `quit`로 종료.

- 대화가 길어질수록 보내는 컨텍스트도 커짐 → 모델 부담 증가. Claude Code, Codex가 세션 안에서 맥락 유지하는 원리도 같음.
- 컨텍스트 윈도우 넘으면 오래된 것부터 잊음. `ollama show gemma3:4b` 보면 131,072 토큰.

## 해 볼 것

- [ ] CH4: 안 써 본 모델 받아서 같은 질문 → 답과 메모리 변화 비교
- [ ] CH5: 명령 팔레트, 파이썬 파일 여러 개 만들어 실행
- [ ] CH6: 가상환경 만들고 requirements.txt로 복원
- [ ] CH7: system 페르소나 바꿔서 같은 질문 답 비교
- [ ] CH7: 번역기
- [ ] CH7: 스트리밍 on/off 응답 시간 측정
- [ ] CH7: 대화 기록 텍스트 파일로 저장
- [ ] CH7: 두 모델끼리 한 주제로 N회 토론시키기 (샘플 코드 받음)

다음 주 11/4: CH8~11 — Streamlit 웹 UI, MLX Whisper 음성 인식, 음성 AI 채팅 앱, 대화 기록 관리.
