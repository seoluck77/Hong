# watermarks-remover 로컬 서비스 상시 실행 가이드

`/remove-ai-marks` 스킬은 처리 코드가 없는 HTTP 클라이언트입니다. 실제 처리는
[guillaumemeyer/watermarks-remover](https://github.com/guillaumemeyer/watermarks-remover) 의
`service/scripts/server.py` 가 담당합니다. 이 서비스를 PC에 항상 띄워 두면 로컬 Claude Code에서
스킬을 바로 쓸 수 있습니다.

## 1. 서비스 설치

Python 3.10 이상이 필요하고, 다른 의존성은 없습니다.

```bash
git clone https://github.com/guillaumemeyer/watermarks-remover.git ~/watermarks-remover
cd ~/watermarks-remover
python3 service/scripts/server.py --host 127.0.0.1 --port 8765
```

확인:

```bash
curl http://127.0.0.1:8765/health          # {"ok": true, "version": "..."}
curl http://127.0.0.1:8765/capabilities    # 설치된 보조 도구 목록
```

PDF와 DOCX의 메타데이터까지 제대로 지우려면 `exiftool` 과 `qpdf` 를 PATH에 두어야 합니다.
Docker를 쓰면 두 도구가 이미지에 포함되어 있습니다.

```bash
cd ~/watermarks-remover
docker compose up -d        # 127.0.0.1:8765 에 바인딩, restart: unless-stopped
```

## 2. 로그인 시 자동 시작

### Windows
저장소의 `docs/windows-autostart.md` 그대로 따르면 됩니다. 요약하면 `start-service.vbs` 로
창 없이 서버를 띄우고, 작업 스케줄러에 로그온 트리거로 등록합니다. 관리자 권한은 필요 없습니다.

### macOS (launchd)
`~/Library/LaunchAgents/com.watermarks-remover.plist` 를 만들고 로드합니다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.watermarks-remover</string>
  <key>ProgramArguments</key><array>
    <string>/usr/bin/python3</string>
    <string>/Users/USERNAME/watermarks-remover/service/scripts/server.py</string>
    <string>--host</string><string>127.0.0.1</string>
    <string>--port</string><string>8765</string>
  </array>
  <key>WorkingDirectory</key><string>/Users/USERNAME/watermarks-remover</string>
  <key>EnvironmentVariables</key><dict>
    <!-- Layer B 백엔드를 쓰려면 아래 주석을 풀고 값을 채웁니다 -->
    <!-- <key>WATERMARKS_REWRITE_BACKEND</key><string>ollama</string> -->
    <!-- <key>WATERMARKS_REWRITE_MODEL</key><string>llama3.2</string> -->
    <!-- <key>WATERMARKS_REWRITE_BASE_URL</key><string>http://127.0.0.1:11434</string> -->
  </dict>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/tmp/watermarks-remover.log</string>
  <key>StandardErrorPath</key><string>/tmp/watermarks-remover.log</string>
</dict></plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.watermarks-remover.plist
```

### Linux (systemd user service)
`~/.config/systemd/user/watermarks-remover.service`:

```ini
[Unit]
Description=watermarks-remover HTTP service

[Service]
WorkingDirectory=%h/watermarks-remover
ExecStart=/usr/bin/python3 %h/watermarks-remover/service/scripts/server.py --host 127.0.0.1 --port 8765
Restart=always
# Layer B 백엔드를 쓰려면 아래 주석을 풀고 값을 채웁니다
#Environment=WATERMARKS_REWRITE_BACKEND=ollama
#Environment=WATERMARKS_REWRITE_MODEL=llama3.2
#Environment=WATERMARKS_REWRITE_BASE_URL=http://127.0.0.1:11434

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now watermarks-remover
```

### Docker (어느 OS든 동일)
`docker compose up -d` 는 `restart: unless-stopped` 로 올라가므로 Docker Desktop이 로그인 시
시작되도록 해 두면 서비스도 같이 올라옵니다.

## 3. Layer B(문체 재작성) 백엔드 설정

텍스트에 `/clean` 을 호출하면 서비스는 Layer A 뒤에 Layer B 재작성을 반드시 수행하며,
백엔드가 없으면 400으로 거부합니다. 서버를 띄우는 셸(또는 위의 서비스 정의)에 환경 변수를
넣어야 합니다. `.env` 파일은 `docker compose` 만 자동으로 읽습니다.

가장 간단한 구성은 로컬 Ollama 입니다.

```bash
ollama pull llama3.2          # 또는 원하는 모델
export WATERMARKS_REWRITE_BACKEND=ollama
export WATERMARKS_REWRITE_MODEL=llama3.2
export WATERMARKS_REWRITE_BASE_URL=http://127.0.0.1:11434
python3 service/scripts/server.py --host 127.0.0.1 --port 8765
```

외부 API를 쓸 때는 `openai-compatible` 백엔드와 `WATERMARKS_REWRITE_API_KEY`,
`WATERMARKS_REWRITE_ALLOW_REMOTE=1` 을 추가합니다. 키는 환경 변수로만 넘깁니다.

기본 전략은 `paraphrase@0.8,mlm@0.2` 인데 `mlm` 단계는 `transformers` 와 `roberta-large` 가
필요합니다. 이를 피하려면 전략 파일을 만들어 서버에 넘깁니다.

```bash
echo '{"default_strategy": "paraphrase@0.5"}' > ~/wm-strategy.json
python3 service/scripts/server.py --host 127.0.0.1 --port 8765 --strategy-config ~/wm-strategy.json
```

재작성 모델은 원문을 작성한 모델과 다른 것을 고르는 편이 좋습니다.

## 4. 클라우드 세션에서 이 서비스를 쓰려면

클라우드 세션은 PC의 `127.0.0.1` 에 닿지 못합니다. 외부에서 접근하려면:

1. 서버에 `--api-key <값>` (또는 `WATERMARKS_SERVER_API_KEY`) 을 반드시 설정합니다.
2. Cloudflare Tunnel, Tailscale Funnel, ngrok 같은 터널로 `https://` 주소를 만듭니다.
3. 클라우드 환경 설정의 환경 변수에 `WATERMARKS_SERVICE_URL` 과 `WATERMARKS_SERVER_API_KEY`
   를 넣고, 네트워크 정책에서 그 호스트를 허용합니다.

이 과정이 부담스러우면 플러그인이 이미 설치된 로컬 Claude Code에서 작업하는 것이
가장 단순합니다.
