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
