# tailscale-macos-vm 프로젝트 분석 & 활용 정리

> **저장소 주소:** https://github.com/bmshin94/tailscale-macos-vm
> **작성일:** 2026-09-22
> **문서 성격:** 저장소 전수조사 결과 + 설치/사용법 + 로컬 AI 에이전트 활용 방안 + 수익화 아이디어 정리

---

## 목차

1. [프로젝트 개요](#1-프로젝트-개요)
2. [파일별 전수조사](#2-파일별-전수조사)
3. [동작 흐름](#3-동작-흐름)
4. [쉽게 이해하기 (비유)](#4-쉽게-이해하기-비유)
5. [설치 및 사용법](#5-설치-및-사용법)
6. [자주 묻는 질문](#6-자주-묻는-질문)
7. [로컬 AI 에이전트 구축 활용](#7-로컬-ai-에이전트-구축-활용)
8. [React / PHP 로 GUI 만들기](#8-react--php-로-gui-만들기)
9. [수익화 아이디어](#9-수익화-아이디어)
10. [추천 로드맵](#10-추천-로드맵)

---

## 1. 프로젝트 개요

### 한 줄 요약

> macOS에서 **OrbStack**으로 Ubuntu VM을 만들고, 그 VM을 **Tailscale** 개인 VPN(테일넷)에 등록해서,
> 포트 개방 없이 전 세계 어디서든 `ssh dev-server` 한 줄로 접속하게 만드는 **인프라 자동화 레시피**.

### 정체

- 애플리케이션 / 라이브러리 / 프레임워크가 **아님**
- **zsh 셸 스크립트 4개 + cloud-init YAML 1개 + README** 로 구성된 IaC(Infrastructure as Code) 예제 저장소
- 하위 폴더 없이 루트에 파일 6개 (+ Claude Code용 `CLAUDE.md`)

### 핵심 설계 사상

| 결정 | 이유 |
|---|---|
| 컨테이너 대신 **VM** | Tailscale이 `/dev/net/tun` 커널 모듈을 써야 제 성능. 컨테이너는 userspace-networking 우회가 필요해 느리고 불안정 |
| **Tailscale SSH** 사용 | VM에 비밀번호도 SSH 공개키도 안 넣음. 인증을 전부 테일넷 신원(ACL)에 위임 |
| **Apple Keychain** 에 인증키 저장 | 코드/`.env`에 시크릿 하드코딩 방지 |
| **호스트가 키를 주입** | macOS 샌드박스 때문에 게스트 VM이 `security` 명령으로 호스트 키체인을 직접 못 읽음 |

---

## 2. 파일별 전수조사

```
tailscale-macos-vm/
├── README.md                    설명서 (사실상 본체)
├── dev-server.yml               VM 설계도 (cloud-init)
├── build.sh                     ① VM 생성 및 프로비저닝
├── store-ts-key-keychain.sh     ② 인증키를 키체인에 저장
├── run.sh                       ③ 테일넷 합류 + Tailscale SSH 활성화
├── cleanup.sh                   ④ 전체 정리
├── CLAUDE.md                    Claude Code 퍼소나 지침 (프로젝트 기능과 무관)
└── docs/project-analysis-ko.md  본 문서
```

### `dev-server.yml` — VM 설계도

| 블록 | 내용 |
|---|---|
| `environment` | `DEBIAN_FRONTEND=noninteractive` (설치 중 대화형 프롬프트 차단) |
| `apt` | `preserve_sources_list: true` |
| `packages` | `ca-certificates`, `curl`, `unzip`, `gnupg`, `sudo`, `passwd`, `zsh`, `lsb-release`, `locales`, `git` |
| `write_files` | `/etc/skel/.zshenv` 에 `TERM=wezterm`, `LANG/LC_ALL=en_US.UTF-8` |
| `users` | `player1`(uid 1001), `player2`(uid 1002) — 둘 다 `zsh` + `sudo` 그룹 + `NOPASSWD:ALL` |
| `runcmd` | 로케일 컴파일 → WezTerm terminfo 설치 → Tailscale apt 저장소 등록 → `tailscale` 설치 → `tailscaled` 활성화 |

> 참고: 사용자 계정에 비밀번호/SSH 키가 없음. 접속 인증은 전부 Tailscale SSH가 담당.

### `build.sh`

```zsh
orb create ubuntu:25.10 dev-server -c dev-server.yml
orb -m dev-server cloud-init status --wait
```

OrbStack이 임시 VM을 띄워 cloud-init으로 설정을 적용한 뒤 정지. `--wait` 덕분에 프로비저닝 완료 전 접속하는 사고를 방지.

### `store-ts-key-keychain.sh`

```zsh
read -rs TEMP_KEY   # 화면에 표시하지 않고 입력받기
security add-generic-password -a "$USER" -s "tailscale-auth-key-dev-server" -w "$TEMP_KEY"
```

- 키체인 항목 이름: `tailscale-auth-key-dev-server`
- 주의: 이 항목은 "암호" 앱이 아니라 **Keychain Access** 앱에서 확인 가능
- 주의: SSH 원격 세션에서는 실행 불가 (맥 앞에서 직접 실행)

### `run.sh`

```zsh
TS_KEY_VAL="$(security find-generic-password -s "tailscale-auth-key-dev-server" -w)"

ORBENV=TS_KEY_VAL orb -m dev-server sudo tailscale up \
      --ssh --advertise-tags=tag:myservers --authkey="$TS_KEY_VAL"

unset TS_KEY_VAL
```

| 옵션 | 역할 |
|---|---|
| `--ssh` | Tailscale SSH 활성화 (키 없이 신원 기반 접속) |
| `--advertise-tags=tag:myservers` | 서버 태그 부여 → ACL 정책 적용 대상이 됨 |
| `--authkey` | 브라우저 로그인 없이 자동 등록 |

### `cleanup.sh`

```zsh
orb -m dev-server sudo tailscale logout || true   # 테일넷 탈퇴
orb delete dev-server                             # VM 삭제
security delete-generic-password -s "tailscale-auth-key-dev-server"
```

### `README.md` 가 알려주는 macOS 특유의 함정

1. **키체인 제약** — Apple Security Framework는 대화형 GUI 로그인 세션 기준. SSH 원격 세션에서는 잠긴 로그인 키체인에 접근 불가 → `run.sh`는 원격 실행 불가.
2. **DNS 혼란** — 맥미니 + VM 양쪽에 오픈소스 Tailscale 패키지를 쓰는 구성에서는 이름 해석이 헷갈릴 수 있음. `~/.ssh/config`에 직접 등록하는 게 가장 간단.

```
Host dev-server
    HostName w.x.y.z
    User player1
```

---

## 3. 동작 흐름

```
[① store-ts-key-keychain.sh]  인증키 → Apple Keychain
            ↓
[② build.sh]  OrbStack 임시 VM 기동 → cloud-init 적용 → 정지
            ↓
[③ run.sh]    키체인에서 키 추출 → VM에 주입 → tailscale up --ssh → 테일넷 합류
            ↓
[④ 접속]      ssh player1@dev-server  (어디서든)
            ↓
[⑤ cleanup.sh] VM + 테일넷 등록 + 키체인 항목 제거
```

### 접속 방법 3가지

| 방법 | 명령 | 조건 |
|---|---|---|
| MagicDNS | `ssh player1@dev-server` | 어디서든 (테일넷 연결 시) |
| OrbStack 로컬 프록시 | `ssh player1@dev-server@orb` | 같은 맥에서 |
| OrbStack CLI | `orb -m dev-server` | 같은 맥에서 |

---

## 4. 쉽게 이해하기 (비유)

맥북을 **건물주**라고 생각하면:

| 요소 | 비유 |
|---|---|
| OrbStack | 방을 뚝딱 지어주는 시공업체 |
| `dev-server.yml` | 인테리어 주문서 (자재, 입주자, 설비 목록) |
| `build.sh` | "시공 시작!" |
| Tailscale auth key | 우리 동네 출입 열쇠 |
| Apple Keychain | 맥 안의 금고 (본인이 앞에 앉아 있을 때만 열림) |
| `run.sh` | 열쇠를 새 방에 넣어줘서 동네 주민 등록 |
| `tag:myservers` | 방 이마에 붙인 "나는 서버다" 이름표 |
| ACL | 동네 출입 규칙 |

### 기존 포트포워딩 방식과의 차이

| 항목 | 포트포워딩 | Tailscale |
|---|---|---|
| 공유기 설정 | 필수 | 불필요 |
| 인터넷 노출 | 포트가 공개적으로 열림 | 외부에서 보이지 않음 |
| 봇 스캔/공격 | 상시 노출 | 사실상 0 |
| 비밀번호 관리 | 필요 | 불필요 (신원 기반) |
| IP 변동 | DDNS 필요 | 자동 처리 |

### 헷갈리기 쉬운 포인트

1. `player1` / `player2`는 예시 이름. 바꾸면 **Tailscale ACL의 `users` 배열도 함께** 수정해야 함.
2. `CLAUDE.md`는 프로젝트 기능과 무관한 Claude Code 퍼소나 설정 파일.
3. `run.sh`의 `ORBENV=TS_KEY_VAL`은 이미 `--authkey`로 값이 전달되므로 사실상 이중 안전장치.

---

## 5. 설치 및 사용법

### 사전 준비물

| 준비물 | 방법 | 비용 |
|---|---|---|
| macOS (Apple Silicon 권장) | - | - |
| OrbStack | `brew install orbstack` | 개인 무료 |
| Tailscale 계정 | https://tailscale.com | 개인 무료 (기기 100대, 유저 3명) |
| 맥에도 Tailscale | 오픈소스 패키지(brew) 권장 — App Store 버전은 샌드박스 제약 | 무료 |

### 단계별

**0) 저장소 클론**

```bash
git clone https://github.com/bmshin94/tailscale-macos-vm
cd tailscale-macos-vm
chmod +x *.sh
```

**1) 태그 생성** — [Access controls > Tags](https://login.tailscale.com/admin/acls/visual/tags)
- Tag name: `myservers`
- Tag owners: 본인 이메일

**2) SSH ACL 정책 수정** — [Access controls > Tailscale SSH](https://login.tailscale.com/admin/acls/visual/tailscale-ssh/)

```json
"ssh": [
  {
    "src":    ["autogroup:member"],
    "dst":    ["tag:myservers"],
    "users":  ["autogroup:nonroot", "player1", "player2"],
    "action": "accept"
  }
]
```

> `"action": "check"` 는 접속할 때마다 브라우저 승인이 필요. 무마찰 자동화를 원하면 `"accept"`.

**3) 인증키 발급** — [Keys 패널](https://login.tailscale.com/admin/settings/keys)
- Reusable 활성화 / Pre-authorized 활성화 / Tags: `tag:myservers`
- 발급 직후 한 번만 노출되므로 즉시 복사

**4) 키 저장** — `./store-ts-key-keychain.sh` (입력이 화면에 보이지 않는 것이 정상)

**5) VM 빌드** — `./build.sh` (2~5분)

**6) 테일넷 연결** — `./run.sh` (맥 앞에서 직접 실행)

**7) 접속** — `ssh player1@dev-server`

**8) 정리** — `./cleanup.sh`

### 트러블슈팅

| 증상 | 원인 | 해결 |
|---|---|---|
| `security: ... not found` | 키체인에 항목 없음 / SSH 원격 실행 | 맥 앞에서 직접 실행, 4단계 재수행 |
| `Tailscale SSH: permission denied` | ACL `users`에 계정 누락 | ACL의 `users` 배열 확인 |
| `requested tags are invalid` | 태그 소유자 미지정 | Tags 화면에서 tag owner 지정 |
| `dev-server` 이름 해석 실패 | MagicDNS 미설정 | `~/.ssh/config`에 IP 직접 등록 |

---

## 6. 자주 묻는 질문

### Q. 플러그인인가요? 스킬인가요? MCP인가요?

**셋 다 아님.** 순수 인프라 자동화 스크립트(IaC)이며, 사용자가 터미널에서 직접 실행합니다.

| 구분 | 정체 | 실행 주체 | 해당 여부 |
|---|---|---|---|
| 플러그인 | Claude Code 기능 확장 묶음 | Claude Code | X |
| 스킬 | `SKILL.md` 기반 작업 매뉴얼 | Claude | X |
| MCP | AI가 외부 도구를 쓰게 하는 표준 프로토콜 서버 | MCP 서버 프로세스 | X |
| 본 저장소 | zsh 스크립트 + cloud-init YAML | 사용자 | O |

다만 **스킬이나 MCP로 감쌀 수는 있음**:
- 스킬화: `.claude/skills/vm-manager/SKILL.md` → "VM 만들어줘"로 `build.sh` 실행
- MCP화: `vm_create` / `vm_list` / `vm_destroy` / `vm_exec` 툴 제공 → AI가 직접 VM 생성·폐기

### Q. API 토큰이 필요한가요?

Anthropic/OpenAI 같은 **LLM API 키는 불필요**. 대신 **Tailscale auth key**가 필요합니다.

| | auth key (사용함) | API token (사용 안 함) |
|---|---|---|
| 용도 | 기기를 테일넷에 등록 | 관리 자동화(ACL 수정 등) |
| 형태 | `tskey-auth-...` | `tskey-api-...` |
| 필요 여부 | 필수 | 불필요 |

**비용:** Tailscale 개인 플랜 무료 (기기 100대 / 유저 3명).

**보안 체크리스트**
- 키를 코드나 git에 넣지 않기 (본 저장소는 키체인 사용 — 양호)
- Reusable 키는 만료일을 짧게
- Pre-authorized는 편리하지만 유출 시 누구나 테일넷 합류 가능
- 태그 덕분에 유출되어도 권한이 서버 범위로 제한됨
- 개선 여지: `--authkey=<값>` 은 VM 내 `ps` 에 순간 노출될 수 있음 → `--authkey=file:<경로>` 방식이 더 안전

### Q. 왜 GitHub에서 유명할까요?

**정확히 말하면 이 저장소 자체가 유명한 프로젝트는 아닙니다.** 커밋 9개 규모의 개인 저장소이고, 검색 인덱스에서도 조회되지 않았습니다. 다만 이 **조합**이 최근 개발자 커뮤니티에서 인기 있는 이유는 분명합니다.

1. **포트포워딩 불필요** — Tailscale(WireGuard 기반 메시 VPN)은 공유기 설정·공인 IP·DDNS·인증서를 전부 제거
2. **Docker Desktop 유료화 이후 OrbStack 인기** — 맥 개발자들의 대체제로 자리잡음
3. **커널 모듈 문제를 정공법으로 해결** — 대부분의 글이 `--tun=userspace-networking` 우회로 끝나는데, 여기서는 "진짜 VM을 써라"라는 근본 해법을 제시
4. **macOS 키체인 함정을 명시적으로 문서화** — 실전에서 수 시간을 날릴 수 있는 지식
5. **AI 에이전트 샌드박스 수요 폭발** — "자율 에이전트를 어디서 안전하게 돌릴까"라는 최근 최대 화두에 정확히 부합

---

## 7. 로컬 AI 에이전트 구축 활용

### 이 구성이 해결해주는 것

| 난제 | 해답 |
|---|---|
| 격리 — 에이전트가 파일을 삭제하면? | VM 내부에서만 동작. 망가지면 `cleanup.sh` → `build.sh` 로 수 분 내 초기화 |
| 접근 — 외부에서 상태 확인 | Tailscale SSH로 어디서든. 포트 노출 0 |
| 시크릿 — API 키 보관 | 키체인 패턴을 그대로 확장 |
| 재현성 — 환경 편차 | `dev-server.yml` 하나로 동일 환경 N개 복제 |

### 아키텍처 예시

```
        사용자 (카페 / 아이패드 / 폰)
                 |  Tailscale 암호화 터널
                 v
        맥미니 (24시간 상시 가동)
                 |  OrbStack
    +------------+------------+
    v            v            v
 agent-web   agent-data   agent-test
 (프론트)     (수집/분석)   (파괴 실험용)
  각각 독립 VM + 독립 테일넷 노드 + 독립 MagicDNS 이름
```

각 VM 내부에서 에이전트를 공격적인 권한으로 실행해도 호스트 맥은 안전합니다.

### 확장 4단계

**1) VM 이름 파라미터화**

```zsh
# build.sh
VM_NAME="${1:-dev-server}"
orb create ubuntu:25.10 "$VM_NAME" -c dev-server.yml
orb -m "$VM_NAME" cloud-init status --wait
```

**2) 에이전트 런타임 추가 (`dev-server.yml`)**

```yaml
packages:
  - ca-certificates
  - curl
  - git
  - python3-pip
  - tmux           # 장시간 세션 유지
runcmd:
  - curl -fsSL https://deb.nodesource.com/setup_22.x | bash -
  - apt-get install -y nodejs
  - npm install -g @anthropic-ai/claude-code
```

**3) API 키도 키체인 패턴으로**

```zsh
ANTHROPIC_KEY="$(security find-generic-password -s "anthropic-api-key" -w)"
orb -m dev-server env ANTHROPIC_API_KEY="$ANTHROPIC_KEY" claude ...
```

**4) Tailscale Serve / Funnel**

```bash
tailscale serve 3000    # 테일넷 내부에만 HTTPS 공개 (인증서 자동)
tailscale funnel 3000   # 인터넷 전체 공개 (신중히)
```

> 한계: `run.sh`가 키체인 때문에 로컬 실행만 가능하므로, 완전 무인 자동화를 원하면
> 시크릿 보관을 1Password CLI / 권한 600 파일 / Tailscale OAuth client 등으로 대체해야 합니다.

---

## 8. React / PHP 로 GUI 만들기

### 가능 여부

가능합니다. 단, React/PHP는 브라우저·웹서버 언어라 `orb` 명령이나 키체인을 **직접** 다룰 수 없습니다.
따라서 **"웹 UI → 맥에서 상주하는 로컬 데몬 → orb/tailscale/security 명령"** 구조로 만듭니다.

```
[React 대시보드]   브라우저
      |  HTTPS + WebSocket
      v
[로컬 데몬]        맥미니 상주 (Node / Bun / Go / PHP CLI)
      |  child_process
      v
 orb create / orb delete / tailscale up / security find-generic-password
      v
[OrbStack VM 들]  <- Tailscale로 어디서든 접속
```

브라우저 접근을 `tailscale serve` 로 테일넷 내부에만 노출하면 로그인 기능을 따로 만들지 않아도 됩니다.

### React 버전 (권장)

스택: React + Vite + TanStack Query + shadcn/ui + xterm.js / 백엔드 Node(Hono·Express) 또는 Bun

```js
// server/vm.js
import { execFile } from 'node:child_process'
import { promisify } from 'node:util'
const run = promisify(execFile)

export async function listVMs() {
  const { stdout } = await run('orb', ['list', '--format', 'json'])
  return JSON.parse(stdout)
}

export async function createVM(name) {
  // 문자열 결합(exec) 금지. 배열 인자(execFile)로 커맨드 인젝션 차단
  await run('orb', ['create', 'ubuntu:25.10', name, '-c', 'dev-server.yml'])
}
```

장점: WebSocket 로그 스트리밍이 자연스럽고, xterm.js로 브라우저 내 터미널 구현 가능. Tauri/Electron으로 데스크탑 앱 포장도 용이.

### PHP 버전

```php
<?php
use Symfony\Component\Process\Process;

function runCmd(array $args): string {
    $proc = new Process($args);   // 배열 인자 사용
    $proc->mustRun();
    return $proc->getOutput();
}

$vms = json_decode(runCmd(['orb', 'list', '--format', 'json']), true);
echo json_encode($vms);
```

장점: PHP/Laravel에 익숙하면 가장 빠름. Laravel Queue로 장시간 빌드 처리에 적합.
단점: 실시간 스트리밍은 SSE나 Reverb 등 추가 구성이 필요.

### 보안 3원칙

1. **커맨드 인젝션 방어** — `execFile` / `Process` 배열 인자만 사용. VM 이름은 `^[a-z0-9-]{1,32}$` 로 검증
2. **키체인 접근은 GUI 세션에서만** — 데몬을 launchd의 **LaunchAgent**(GUI 도메인)로 등록. LaunchDaemon(시스템 도메인)은 키체인 접근 불가
3. **바인딩 주소** — `0.0.0.0` 금지, `127.0.0.1` 바인딩 후 `tailscale serve` 로 노출

### MVP 일정 (React 기준)

| 주차 | 작업 |
|---|---|
| 1주 | Node 백엔드로 `orb list/create/delete` REST API |
| 2주 | React 대시보드 (목록 / 생성 / 삭제) |
| 3주 | WebSocket 로그 스트리밍 + xterm.js 웹 터미널 |
| 4주 | 템플릿 프리셋 시스템 + Tailscale Serve 연동 |

---

## 9. 수익화 아이디어

### 비교표

| # | 아이디어 | 난이도 | 초기비용 | 수익 잠재력 | 첫 수익까지 | 추천도 |
|---|---|---|---|---|---|---|
| 1 | 유료 템플릿/강의 패키지 | 낮음 | 0 | 중 | 2주 | 높음 |
| 2 | OrbBoard (VM 관리 GUI 앱) | 중 | 0 | 중상 | 2개월 | 높음 |
| 3 | 오픈코어 CLI (`vmctl`) | 중상 | 소액 | 상 | 6개월 | 중 |
| 4 | **AI 에이전트 샌드박스 SaaS** | 높음 | 중간 | **최상** | 6~12개월 | **최고** |
| 5 | Dev Box 호스팅 구독 | 높음 | 하드웨어 | 상 | 3개월 | 중 |
| 6 | 구축 대행 / 컨설팅 | 낮음 | 0 | 중 | **1주** | 높음 |

### 1) 유료 템플릿 & 강의 패키지

구성: 템플릿 8종(Node/Python/Go/Rust/PHP/DB/AI-agent/풀스택) + 개선 스크립트(멀티 VM·스냅샷·백업) + 한국어 가이드 PDF + 영상 강의 + 디스코드 채널 + 평생 업데이트

| 티어 | 가격 | 구성 |
|---|---|---|
| Basic | 29,000원 | 템플릿 + PDF |
| Pro | 79,000원 | + 영상강의 + 디스코드 |
| Team | 290,000원 | + 5인 라이선스 + 1:1 셋업 지원 |

채널: Gumroad / 크몽 / 인프런. 유입은 유튜브·블로그·Reddit(r/macapps, r/tailscale).
목표 시나리오: 월 30건 x 평균 5만원 = **월 150만원**. 무료 오픈소스 버전은 반드시 유지(신뢰 = 유입).

### 2) OrbBoard — VM 관리 GUI 앱

컨셉: "OrbStack용 Docker Desktop". 터미널이 부담스러운 개발자를 위한 클릭 기반 VM 관리.

| 기능 | Free | Pro |
|---|---|---|
| VM 생성/삭제/목록 | O | O |
| Tailscale 원클릭 연결 | O | O |
| 브라우저 내장 터미널 | O | O |
| 템플릿 마켓플레이스 | 3개 | 무제한 |
| 스냅샷 / 롤백 | X | O |
| 팀 공유 템플릿 | X | O |
| AI 에이전트 런처 | X | O |
| 모바일 웹 대시보드 | X | O |

가격: Pro $8/월 또는 평생 $79, Team $15/유저/월. (맥 개발자 툴은 평생 라이선스 선호. Setapp 입점 검토)
시나리오: 유료 500명 x $8 = **월 $4,000**. 런칭 채널은 Product Hunt / Hacker News.
스택: React + Tauri + Node 데몬 + SQLite.

### 3) 오픈코어 CLI (`vmctl`)

```bash
vmctl create agent-01 --template python-ai --tailnet
vmctl snapshot agent-01 --name before-experiment
vmctl restore agent-01 before-experiment
vmctl exec agent-01 -- claude "이 버그 고쳐줘"
vmctl destroy agent-01
```

- 무료(OSS): 기본 CRUD + 템플릿 3종
- 유료(Team): 팀 템플릿 레지스트리, 감사 로그, SSO, 정책 강제, 중앙 대시보드 — $20/유저/월
- 전략: 깃허브 스타 확보 자체가 마케팅 깔때기

### 4) AI 에이전트 샌드박스 SaaS (최우선 유망)

해결하는 문제: **"자율 AI 에이전트를 어디서 안전하게 실행할 것인가"**
(내 노트북 = 사고 위험 / 클라우드 VM = 느리고 세팅 부담 / 컨테이너 = 커널·네트워크 제약)

```
        웹 대시보드
             |
   +---------+---------+
   v                   v
[오케스트레이터]    [샌드박스 풀]
   |                   |
   v                   v
"레포 리팩토링"      VM #1 (격리)
"테스트 통과시켜"    VM #2 (격리)
"크롤러 작성"        VM #3 (격리)
             |
             v
       결과를 PR로 제출
```

차별화:
1. 진짜 VM 격리 (커널 단위)
2. **Tailscale 내장** — 에이전트가 사내망 DB/API에 안전 접근 (경쟁사 대비 핵심 무기)
3. 스냅샷 타임머신 — 망가진 시점으로 롤백
4. BYO-Mac 하이브리드 — 사내 맥미니를 워커로 등록 → 비용 절감 + 데이터 외부 반출 없음 (금융·의료권 소구)

| 플랜 | 가격 | 내용 |
|---|---|---|
| Hobby | 무료 | 샌드박스 1개, 시간 제한 |
| Pro | $29/월 | 5개 동시, 스냅샷 |
| Team | $99/월~ | 무제한, 사내망 연결, 감사 로그 |
| Enterprise | 협의 | 온프레미스, SSO, SLA |

시나리오: 유료 200팀 x $99 = **월 $19,800**.
리스크: E2B·Daytona·Modal 등 경쟁자 존재 / 맥 기반은 확장성 한계(리눅스 확장 필요) / 보안 검증 부담.

### 5) Personal Cloud Dev Box 호스팅

맥미니를 구매·코로케이션하여 전용 개발 서버를 구독제로 임대. 월 39,000~99,000원.

셀링 포인트: Apple Silicon 네이티브 성능 / **iOS 빌드 가능 (AWS·GCP 불가)** / Tailscale로 5분 연결 / 아이패드에서도 원격 빌드.
리스크: 하드웨어 선투자, 정전·회선 관리, MacStadium 등 경쟁.
전략: **"iOS 개발자 전용 원격 빌드머신"** 으로 니치를 좁히면 승산.

### 6) 구축 대행 & 컨설팅 (가장 빠른 첫 수익)

| 상품 | 가격 | 소요 |
|---|---|---|
| 개인 셋업 대행 | 150,000원 | 2시간 |
| 소규모 팀(5인) 구축 | 800,000원 | 1~2일 |
| 기업 인프라 설계 | 3,000,000원~ | 1~2주 |
| 월 유지보수 | 200,000원/월 | - |

채널: 크몽, 숨고, 위시켓, 링크드인, 개발자 커뮤니티. **1주 내 첫 수익 가능**하며, 여기서 얻은 고객 피드백이 1~4번 제품의 재료가 됨.

---

## 10. 추천 로드맵

```
지금 ~ 1개월   저장소 다듬어 무료 공개 + 블로그/영상 콘텐츠
               구축 대행(6번)으로 첫 수익 + 고객 피드백 수집
      |
1 ~ 3개월      유료 템플릿 패키지(1번) 출시
               OrbBoard MVP 개발(2번, React + Tauri)
      |
3 ~ 6개월      OrbBoard 정식 출시 + Product Hunt 런칭
               vmctl 오픈소스 공개(3번)로 스타 확보
      |
6 ~ 12개월     AI 에이전트 샌드박스(4번)로 확장
```

시장 규모만 보면 **4번(AI 에이전트 샌드박스)** 이 압도적이지만, 1인으로 시작한다면
**6 → 1 → 2** 순서로 현금 흐름을 만들며 가는 편이 안전합니다.

### 성공의 열쇠

1. **무료 버전 유지** — 신뢰가 곧 유입이고 유입이 곧 매출
2. **문서화가 곧 제품** — 이 저장소의 가치도 결국 README에서 나옴
3. **니치를 좁힐 것** — "모든 개발자"가 아니라 "AI 에이전트를 돌리는 맥 개발자"

---

## 참고 링크

- 저장소: https://github.com/bmshin94/tailscale-macos-vm
- OrbStack: https://orbstack.dev
- Tailscale: https://tailscale.com
- Tailscale 관리자 — 태그: https://login.tailscale.com/admin/acls/visual/tags
- Tailscale 관리자 — SSH ACL: https://login.tailscale.com/admin/acls/visual/tailscale-ssh/
- Tailscale 관리자 — 인증키: https://login.tailscale.com/admin/settings/keys
- Tailscale 관리자 — 기기 목록: https://login.tailscale.com/admin/machines
- cloud-init 문서: https://cloudinit.readthedocs.io
