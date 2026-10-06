# Hong

논문 작성 중 AI 생성 문장의 흔적(워터마크)을 제거하기 위한 작업 공간입니다.

## 용도

- 초안 텍스트를 이 저장소에 두고 `/watermarks-remover:remove-ai-marks` 스킬로 AI 흔적을 제거합니다.
- 원문과 수정본을 함께 보관해 수정 이력을 추적합니다.

## 디렉터리 구성 (제안)

- `drafts/` : 수정 전 원문
- `cleaned/` : 스킬 적용 후 수정본

## 사용 방법

1. 수정할 텍스트를 `drafts/` 아래에 `.md` 또는 `.txt` 파일로 저장합니다.
2. Claude Code에서 `/watermarks-remover:remove-ai-marks <파일 경로>` 를 실행합니다.
3. 결과를 `cleaned/` 아래에 같은 파일명으로 저장하고 커밋합니다.

## 스킬 설치 상태

`.claude/skills/` 아래에 [guillaumemeyer/watermarks-remover](https://github.com/guillaumemeyer/watermarks-remover) (MIT) 의
두 스킬을 복사해 두었습니다. 클라우드 세션은 저장소 안의 스킬을 자동으로 읽습니다.

- `remove-ai-marks` : 얇은 HTTP 클라이언트. 별도의 서비스가 떠 있어야 동작합니다.
- `clean-user-facing-text` : 서비스 없이 동작하는 텍스트 전용 스킬 (Layer A + 재작성 가이드).

## 서비스 실행 (remove-ai-marks 용)

서비스는 Python 3.10+ 표준 라이브러리만으로 동작합니다.

```bash
git clone --depth 1 https://github.com/guillaumemeyer/watermarks-remover
cd watermarks-remover
python3 service/scripts/server.py --host 127.0.0.1 --port 8765
```

- Layer A(보이지 않는 유니코드·메타데이터 제거)는 추가 설정 없이 동작합니다.
- Layer B(문체 재작성, `paraphrase@…`)는 LLM 백엔드가 필요합니다. 서비스 실행 전에
  `WATERMARKS_REWRITE_BACKEND`(`ollama` 또는 `openai-compatible`), `WATERMARKS_REWRITE_MODEL`,
  `WATERMARKS_REWRITE_BASE_URL`, 필요 시 `WATERMARKS_REWRITE_API_KEY` 를 환경 변수로 설정합니다.
  원격 엔드포인트를 쓰려면 `WATERMARKS_REWRITE_ALLOW_REMOTE=1` 도 필요합니다.
- 서비스 주소가 기본값(`http://127.0.0.1:8765`)이 아니면 `WATERMARKS_SERVICE_URL` 을 설정합니다.
