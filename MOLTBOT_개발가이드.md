# Moltbot 개발 가이드

이 문서는 Moltbot을 처음부터 설치하고 개발 환경을 구축하는 전체 과정을 담고 있습니다.

## 📋 목차

1. [초기 설정](#초기-설정)
2. [AI 모델 인증 설정](#ai-모델-인증-설정)
3. [텔레그램 연동](#텔레그램-연동)
4. [시스템 서비스 설치](#시스템-서비스-설치)
5. [개발 워크플로우](#개발-워크플로우)
6. [유용한 명령어](#유용한-명령어)

---

## 초기 설정

### 필수 요구사항

- Node.js 22+
- pnpm 또는 npm
- macOS (이 가이드 기준)

### 프로젝트 디렉토리

```bash
cd /Users/minsungkim/development/clawdbot
pnpm install
```

---

## AI 모델 인증 설정

### Anthropic Setup-Token 방식 (Claude Pro/Max 구독)

#### 1. Setup-Token 생성

실제 터미널(Terminal.app, iTerm2 등)에서:

```bash
claude setup-token
```

브라우저가 열리고 로그인하면 `sk-ant-oat01-`로 시작하는 긴 토큰을 받게 됩니다.

#### 2. Moltbot에 토큰 설정

```bash
pnpm moltbot onboard --non-interactive --accept-risk \
  --auth-choice token \
  --token-provider anthropic \
  --token "sk-ant-oat01-YOUR_TOKEN_HERE" \
  --skip-channels --skip-skills --skip-daemon
```

#### 3. 설정 확인

```bash
pnpm moltbot models status
```

**출력 예시:**
```
Providers w/ OAuth/tokens (1): anthropic (1)
- anthropic:default = token:sk-ant-o...
```

### Setup-Token vs API Key

| 구분 | Setup-Token (현재 설정) | API Key |
|------|------------------------|---------|
| 발급 방법 | Claude Pro/Max 구독 | Anthropic Console |
| 비용 | 추가 비용 없음 (구독료만) | 사용량 기반 과금 |
| 사용 제한 | 구독 플랜의 rate limit | 신용카드 한도 |
| 월 비용 | 구독료 ($20 Pro / $200 Max) | 변동 |

**결론**: Setup-token 사용 시 추가 API 비용 없이 구독 한도 내에서 사용 가능!

---

## 텔레그램 연동

### 1. Gateway 시작 (임시)

```bash
pnpm moltbot gateway run --bind loopback --port 18789 &
```

### 2. 채널 상태 확인

```bash
pnpm moltbot channels status --probe
```

**출력 예시:**
```
- Telegram default: enabled, configured, running, bot:@jjanga_bot, works
```

### 3. 텔레그램 페어링

#### 3-1. 텔레그램에서 봇 시작

텔레그램 앱에서:
1. `@jjanga_bot` 검색
2. `/start` 전송
3. 페어링 코드 받음 (예: LTYKXN6F)

#### 3-2. 페어링 요청 확인

```bash
pnpm moltbot pairing list --channel telegram
```

**출력 예시:**
```
┌──────────┬────────────────┬──────────┬──────────────────────────┐
│ Code     │ telegramUserId │ Meta     │ Requested                │
├──────────┼────────────────┼──────────┼──────────────────────────┤
│ LTYKXN6F │ 6958754007     │ {...}    │ 2026-01-29T02:32:08.777Z │
└──────────┴────────────────┴──────────┴──────────────────────────┘
```

#### 3-3. 페어링 승인

```bash
pnpm moltbot pairing approve LTYKXN6F --channel telegram
```

**출력:**
```
✅ Approved telegram sender 6958754007.
```

### 4. 사용 시작

이제 텔레그램 `@jjanga_bot`에게 메시지를 보내면 Claude가 응답합니다!

**예시:**
```
안녕! 자기소개 해줘
```

```
Python으로 피보나치 함수 만들어줘
```

---

## 시스템 서비스 설치

### 왜 시스템 서비스로 설치하나?

- ✅ Mac 부팅 시 자동 시작
- ✅ 크래시 시 자동 재시작
- ✅ VS Code 닫아도 계속 실행
- ✅ 백그라운드에서 항상 실행

### 설치 과정

#### 1. 기존 Gateway 중지 (실행 중인 경우)

```bash
# PID 확인
ps aux | grep moltbot-gateway

# 중지
kill [PID]
```

#### 2. 시스템 서비스 설치

```bash
pnpm moltbot gateway install --runtime node
```

**출력:**
```
✅ Installed LaunchAgent: /Users/minsungkim/Library/LaunchAgents/bot.molt.gateway.plist
📝 Logs: /Users/minsungkim/.clawdbot/logs/gateway.log
```

#### 3. 설치 확인

```bash
pnpm moltbot channels status --probe
```

### LaunchAgent 설정 확인

서비스가 **개발 디렉토리**를 바라보고 있습니다:

```xml
<key>ProgramArguments</key>
<array>
  <string>/opt/homebrew/bin/node</string>
  <string>/Users/minsungkim/development/clawdbot/dist/index.js</string>
  <string>gateway</string>
  <string>--port</string>
  <string>18789</string>
</array>
```

**중요:** 소스 코드를 수정하고 빌드하면 서비스가 업데이트된 코드를 사용합니다!

---

## 개발 워크플로우

### 코드 수정 → 빌드 → 재시작

#### 1. 소스 코드 수정

```bash
cd /Users/minsungkim/development/clawdbot

# 에디터로 코드 수정
# 예: src/agents/*.ts, src/telegram/*.ts 등
```

#### 2. TypeScript 빌드

```bash
pnpm build
```

#### 3. 서비스 재시작 (변경사항 적용)

```bash
launchctl kickstart -k gui/$UID/bot.molt.gateway
```

또는

```bash
pnpm moltbot gateway restart
```

#### 4. 로그 확인

```bash
tail -f ~/.clawdbot/logs/gateway.log
```

### 개발 모드 (실시간 디버깅)

서비스를 중지하고 직접 실행하여 로그를 바로 볼 수 있습니다:

```bash
# 1. 서비스 중지
launchctl stop bot.molt.gateway

# 2. 개발 모드로 실행
pnpm moltbot gateway run --bind loopback --port 18789

# 3. 테스트 완료 후 Ctrl+C로 중지

# 4. 서비스 재시작
launchctl start bot.molt.gateway
```

### 주요 커스터마이징 포인트

```
src/
├── agents/          # AI 에이전트 로직, 모델 설정
├── telegram/        # 텔레그램 메시지 핸들러
├── discord/         # 디스코드 통합
├── slack/           # 슬랙 통합
├── commands/        # CLI 명령어 정의
├── auto-reply/      # 자동 응답 규칙
├── config/          # 설정 관리
└── infra/           # 인프라 유틸리티
```

---

## 유용한 명령어

### Gateway 관리

```bash
# Gateway 상태 확인
pnpm moltbot channels status --probe

# Gateway 재시작
launchctl kickstart -k gui/$UID/bot.molt.gateway

# Gateway 중지
launchctl stop bot.molt.gateway

# Gateway 시작
launchctl start bot.molt.gateway

# Gateway 언인스톨
launchctl unload ~/Library/LaunchAgents/bot.molt.gateway.plist
rm ~/Library/LaunchAgents/bot.molt.gateway.plist
```

### 로그 확인

```bash
# Gateway 로그 실시간 보기
tail -f ~/.clawdbot/logs/gateway.log

# 에러 로그
tail -f ~/.clawdbot/logs/gateway.err.log

# macOS 시스템 로그
./scripts/clawlog.sh -f
```

### 모델 관리

```bash
# 사용 가능한 모델 목록
pnpm moltbot models list

# 현재 모델 상태
pnpm moltbot models status

# 기본 모델 변경
pnpm moltbot models set anthropic/claude-opus-4-5
```

### 페어링 관리

```bash
# 페어링 요청 목록
pnpm moltbot pairing list --channel telegram

# 페어링 승인
pnpm moltbot pairing approve [CODE] --channel telegram

# 페어링 거부
pnpm moltbot pairing reject [CODE] --channel telegram
```

### 개발 도구

```bash
# 빌드 (TypeScript → JavaScript)
pnpm build

# 타입 체크
pnpm build

# 린트
pnpm lint

# 포맷
pnpm format

# 테스트
pnpm test

# 테스트 커버리지
pnpm test:coverage
```

---

## 설정 파일 위치

```bash
# 메인 설정
~/.moltbot/moltbot.json
~/.clawdbot/moltbot.json  # 동일 (심볼릭 링크)

# 인증 프로필
~/.clawdbot/agents/main/agent/auth-profiles.json

# 세션 데이터
~/.clawdbot/agents/main/sessions/

# 로그
~/.clawdbot/logs/gateway.log
~/.clawdbot/logs/gateway.err.log

# LaunchAgent (시스템 서비스)
~/Library/LaunchAgents/bot.molt.gateway.plist
```

---

## 트러블슈팅

### Gateway가 시작되지 않는 경우

```bash
# 1. 로그 확인
tail -n 100 ~/.clawdbot/logs/gateway.err.log

# 2. 포트 충돌 확인
lsof -i :18789

# 3. 서비스 재시작
launchctl kickstart -k gui/$UID/bot.molt.gateway
```

### 텔레그램 응답이 없는 경우

```bash
# 1. Gateway 상태 확인
pnpm moltbot channels status --probe

# 2. 페어링 확인
pnpm moltbot pairing list --channel telegram

# 3. 로그 확인
tail -f ~/.clawdbot/logs/gateway.log
```

### 모델 인증 오류

```bash
# 인증 상태 확인
pnpm moltbot models status

# setup-token 재설정
claude setup-token
pnpm moltbot models auth paste-token --provider anthropic
```

---

## 다음 단계

### 추가 채널 연동

- **Discord**: `pnpm moltbot configure discord`
- **Slack**: `pnpm moltbot configure slack`
- **WhatsApp**: Pi 세션 설정 필요

### 고급 설정

- **다중 모델 설정**: Fallback 모델 구성
- **자동 응답 규칙**: 특정 키워드에 자동 반응
- **플러그인 개발**: 커스텀 기능 추가
- **웹훅 설정**: 외부 서비스 통합

### 참고 문서

- 공식 문서: https://docs.molt.bot
- GitHub: https://github.com/moltbot/moltbot
- 설정 예시: `docs/gateway/configuration.md`
- 채널 설정: `docs/channels/`

---

## 요약

✅ **설치 완료 항목**
- Node.js + pnpm 설치
- Claude CLI 설치
- Anthropic setup-token 설정
- Moltbot Gateway 설치 (시스템 서비스)
- 텔레그램 봇 연동 (@jjanga_bot)
- 페어링 완료

✅ **개발 환경**
- 소스 코드: `/Users/minsungkim/development/clawdbot`
- 빌드 출력: `dist/`
- 서비스가 개발 디렉토리 사용
- VS Code 닫아도 계속 실행
- 코드 수정 → 빌드 → 재시작으로 업데이트

🎉 **이제 Moltbot을 자유롭게 커스터마이징하세요!**

---

작성일: 2026-01-29
버전: 2026.1.27-beta.1
작성자: Moltbot Setup Assistant
