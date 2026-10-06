# 로컬 Claude Code용 설정 프롬프트

아래 내용을 통째로 복사해, 확보해 둔 폴더에서 연 로컬 Claude Code 세션에 붙여 넣으면 됩니다.

---

# 목표
이 폴더 아래에 watermarks-remover HTTP 서비스를 설치하고, `/watermarks-remover:remove-ai-marks`
스킬이 Layer A(보이지 않는 유니코드·메타데이터 제거)와 Layer B(문체 재작성, paraphrase@0.5)를
모두 수행할 수 있는 상태로 만들어라. 끝나면 실제 호출로 두 레이어가 동작함을 증명하라.

# 제약
- 모든 파일은 현재 폴더 아래에만 만들어라. 홈 디렉터리나 시스템 경로는 건드리지 마라.
- 시스템 전역 설치(Ollama, exiftool, qpdf 등)가 필요하면 설치 명령을 보여 주고 내 확인을 받은
  뒤에 실행하라. 확인 없이 설치하지 마라.
- API 키가 필요하면 내가 직접 `.env.local` 파일에 넣겠다. 채팅으로 키를 요구하지 마라.
- 재작성(Layer B) 모델은 Claude가 아닌 모델을 써야 한다. 원고 초안을 Claude로 썼기 때문이다.

# 1단계: 환경 점검
다음을 확인하고 표로 보고하라: OS, `python3 --version`(3.10 이상 필요), `git`, `docker`,
`ollama`, `exiftool`, `qpdf`, 가용 RAM, GPU 유무. 없는 것은 "없음"으로 표시하라.

# 2단계: 서비스 설치
```
git clone --depth 1 https://github.com/guillaumemeyer/watermarks-remover.git ./watermarks-remover
```
서비스는 `./watermarks-remover/service/scripts/server.py` 이며 Python 표준 라이브러리만 쓴다.
`exiftool`과 `qpdf`가 없으면 PDF/DOCX 메타데이터 제거가 best-effort로 떨어진다는 점을 보고하고,
설치할지 내게 물어라. 텍스트와 마크다운 작업에는 둘 다 필요 없다.

# 3단계: Layer B 백엔드 결정
우선순위대로 시도하라.

(a) Ollama가 이미 있으면: `ollama list`로 모델을 확인하고, 없으면 RAM에 맞춰 하나를 추천하라
    (16GB 이상: `qwen2.5:7b` 또는 `llama3.1:8b`, 8GB 수준: `qwen2.5:3b` 또는 `llama3.2:3b`).
    내 확인 후 `ollama pull`. 백엔드 설정:
    WATERMARKS_REWRITE_BACKEND=ollama
    WATERMARKS_REWRITE_MODEL=<선택한 모델>
    WATERMARKS_REWRITE_BASE_URL=http://127.0.0.1:11434

(b) Ollama가 없고 설치를 원치 않으면: OpenAI 호환 API를 쓴다. `.env.local`에 다음 키를 비워 둔
    템플릿으로 만들어 주고, 내가 채운 뒤 알려 주겠다고 안내하라.
    WATERMARKS_REWRITE_BACKEND=openai-compatible
    WATERMARKS_REWRITE_MODEL=
    WATERMARKS_REWRITE_BASE_URL=
    WATERMARKS_REWRITE_API_KEY=
    WATERMARKS_REWRITE_ALLOW_REMOTE=1
    `.env.local`은 `.gitignore`에 넣어라.

# 4단계: 전략 파일
기본 전략 `paraphrase@0.8,mlm@0.2`의 `mlm` 단계는 transformers와 roberta-large를 요구하므로
쓰지 않는다. 다음 파일을 만들어라.
```
./config/wm-strategy.json  →  {"default_strategy": "paraphrase@0.5"}
```

# 5단계: 실행 스크립트
OS에 맞는 시작 스크립트를 만들어라.
- macOS/Linux: `./start-wm.sh`
- Windows: `./start-wm.ps1`
스크립트는 (1) `.env.local`이 있으면 읽어 환경 변수로 내보내고, (2) 다음 명령으로 서버를 띄운다.
```
python3 ./watermarks-remover/service/scripts/server.py --host 127.0.0.1 --port 8765 \
  --strategy-config ./config/wm-strategy.json
```
`./stop-wm.sh`(또는 `.ps1`)와 로그 파일 `./logs/wm-server.log`도 만들어라.
로그인 시 자동 시작은 별도 선택지로 안내만 하고, 내가 원할 때 설정하겠다고 하라.

# 6단계: 기동과 검증
서버를 백그라운드로 띄우고 다음을 순서대로 실행해 결과를 그대로 보여 줘라.
```
curl -s http://127.0.0.1:8765/health
curl -s http://127.0.0.1:8765/capabilities
```
그다음 `./samples/test.txt`에 영어 학술 문장 3개짜리 샘플을 만들고(한 문장에 U+200B 제로폭
공백을 일부러 넣어라), base64로 인코딩해 `/inspect`와 `/clean`을 호출하라.
- `/inspect` 결과에서 U+200B가 탐지되어야 한다.
- `/clean` 응답이 200이고 `report.layer_b`가 채워져 있어야 Layer B가 작동한 것이다.
  400이 나오면 백엔드 설정 문제이므로 원인을 로그에서 찾아 고치고 다시 시도하라.
- 디코딩한 결과를 `./samples/test.cleaned.txt`에 저장하고 원문과 나란히 보여 줘라.

# 7단계: 작업 폴더와 메모
`./revision_texts/`와 `./revision_texts_cleaned/`를 만들고, `./WORKFLOW.md`에 다음을 적어라.
- 서버 시작/중지 방법, 로그 위치
- `/watermarks-remover:remove-ai-marks` 호출 예시
- Layer B가 "탐지 불가"를 보장하지 않는다는 주의 문구
- Elsevier의 생성형 AI 사용 선언이 필요하다는 메모

# 보고
끝나면 한 화면 안에 요약하라: 설치 위치, 선택한 백엔드와 모델, health/capabilities 결과,
샘플 테스트에서 Layer A가 제거한 것과 Layer B가 바꾼 것, 남은 수동 작업.
