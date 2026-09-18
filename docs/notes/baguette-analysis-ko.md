# 🥖 baguette 프로젝트 분석 정리 (한국어)

> 작성일: 2026-09-18
> 작성: Claude Code 세션에서의 코드베이스 분석 대화 정리

## 📍 GitHub 주소

| 구분 | 주소 |
|---|---|
| **원본 (upstream)** | https://github.com/tddworks/baguette |
| **포크 (이 저장소)** | https://github.com/bmshin94/baguette |
| 프로젝트 홈페이지 | https://tddworks.github.io/baguette/ |
| 라이선스 | Apache License 2.0 |
| 버전 (플러그인 매니페스트 기준) | 0.1.75 |

---

## 1. 이게 뭐하는 프로젝트인가

**`baguette`는 Xcode나 Simulator.app을 열지 않고 iOS 시뮬레이터를 완전히 원격
조종하는 macOS 전용 Swift CLI + 웹 서버.**

핵심은 "시뮬레이터 조작"을 **텍스트 명령 = 스크립트 = AI가 호출 가능한 것**으로
바꿔준다는 점이다. 사람의 마우스가 필요했던 일이 프로그래밍 가능해진다.

### 폴더 구조가 말해주는 정체

```
Sources/Baguette/
├── App/            CLI 명령 30개 (Boot/Tap/Serve/Screenshot/DescribeUI…)
├── Domain/         순수 Swift 값타입 — 26개 바운디드 컨텍스트
│                   (Input/Screen/Camera/Motion/Network/Location/Chrome…)
├── Infrastructure/ 애플 비공개 프레임워크(SimulatorKit/CoreSimulator) 호출부
└── Resources/Web/  브라우저 UI — 번들러 없는 바닐라 JS IIFE

Injected/            앱에 주입하는 ObjC dylib 3개
                     (VirtualCamera / VirtualMotion / VirtualNetwork)
Companion/DeviceTwin/ 실기기 화면 미러링용 동반 앱
skills/baguette/     Claude Code용 Agent Skill (+ evals)
.claude-plugin/      Claude Code 플러그인 매니페스트 + 마켓플레이스
plugins/deeplink/    자체 플러그인 시스템 예제 (Python 스크립트)
docs/features/       기능별 리버스 엔지니어링 기록 30개
```

규모: Swift 소스 **296개 파일**, 테스트 **225개 파일 / 460+ 케이스**,
CHANGELOG **114KB**.

### 주요 기능 (CLI 기준)

| 분류 | 명령 | 설명 |
|---|---|---|
| 라이프사이클 | `list` `boot` `shutdown` `orientation` `heal` | 헤드리스 부팅/종료/회전/복구 |
| 입력 | `tap` `double-tap` `swipe` `pinch` `pan` `key` `type` `press` | 실제 HID 이벤트 |
| 스트리밍 | `stream` `serve` | 60fps MJPEG / H.264 (stdout 또는 WebSocket) |
| 캡처 | `screenshot` `record` `render-3d` | App Store 규격, MP4, 3D 목업 |
| 검사 | `describe-ui` `logs` | 접근성 트리 JSON, os_log 실시간 |
| 환경 시뮬 | `motion` `network` `location` `shake` `status-bar` `interface` | 센서·네트워크·외관 |
| 기타 | `paste` `clipboard` `openurl` `schemes` | 클립보드, 딥링크 |
| 확장 | `plugin` `bakery` | 자체 플러그인 생태계 (git repo = bakery) |

### 왜 특별한가 — iOS 26 호출 규약 변경

iOS 26 / Xcode 26에서 애플이 내부 호출 규약을 바꿔 기존 도구들
(`idb`, `AXe`, `simctl io`)이 전부 깨졌다. baguette이 뚫어낸 3가지:

1. **`IndigoHIDMessageForMouseNSEvent`가 5인자 → 9인자로 변경.**
   옛 시그니처는 메시지가 조용히 버려지거나 `backboardd`를 크래시시킨다
   (시뮬레이터가 갑자기 재부팅되는 증상). baguette은 Xcode 26 preview-kit의
   9인자 버전을 써서 digitizer target `0x32`로 라우팅한다.
   또한 이 함수는 NSEvent 스레드로컬 상태를 읽으므로 **반드시 MainActor에서
   실행**해야 한다.
2. **스트리밍 터치·엣지 제스처는 실제 `IOHIDEvent`가 필요.**
   `IOHIDEventCreateDigitizerEvent` 부모 + finger 자식 이벤트를 만들고,
   래퍼가 비워둔 바이트 슬롯(`IndigoHIDTouchTarget`, `IndigoHIDEdge`)을 직접
   패치. 덕분에 홈 인디케이터 스와이프, 앱 스위처 드래그, 알림센터
   끌어내리기가 실제 iOS 제스처 인식기로 동작한다.
3. **카메라는 dylib 주입.** SimulatorKit에 카메라를 속이는 심볼이 없어
   `VirtualCamera.dylib`을 `DYLD_INSERT_LIBRARIES`로 주입하고
   `AVCaptureVideoPreviewLayer` 등을 훅. 맥 웹캠 영상이 iOS 앱 카메라로 들어간다.

기록된 함정들: `IndigoHIDTargetForScreen`은 "정답처럼 보이지만 부르면
시뮬이 죽는 덫", CarPlay 코드네임이 "Stark", `CMGyroData`가 도(degree)를
받는데 프로퍼티는 라디안 반환, 쿼터니언이 내부 `w,x,y,z` vs 공개 `x,y,z,w`,
`CLLocation.course`가 `cos(latitude)` 보정 누락으로 위도 37도에서 45° 대신
51.52° 보고 — 전부 측정해서 문서화되어 있다.

### 어떨 때 쓰는가 / 어떤 도움이 되는가

- **AI 에이전트에게 "손과 눈"을 달아줄 때** — 코드 수정 → 시뮬에서 직접 탭 →
  스크린샷 / AX 트리로 검증하는 루프
- **CI에서 UI 스모크 테스트** — Xcode UI Test보다 가볍고 빠름
- **App Store 스크린샷 / 데모 영상 자동 생성** — `--size appstore-6.9` 내장
- **원격 / 무헤드 개발** — Mac mini에 띄우고 브라우저로 조작, `/farm`에서 다중 기기
- **RN / Expo / Flutter 개발 루프** — `examples/expo-bakery`에 Expo 플러그인 예제

핵심 이득: `describe-ui`가 화면을 **JSON 텍스트**로 주므로 LLM 토큰이
스크린샷 대비 압도적으로 적게 든다. 에이전트 구축 시 결정적 요소.

> ⚠️ 전제조건: **Apple Silicon Mac + Xcode 26 필수.** 비공개 프레임워크에
> 링크하므로 리눅스 / 윈도우 / 인텔맥에서는 동작하지 않는다.

---

## 2. 쉬운 비유로 다시

> "아이폰 시뮬레이터에 **로봇 손가락 + CCTV**를 달아주는 도구"

평소: `Xcode 열기 → 시뮬 창 → 마우스 클릭 → 눈으로 확인` (사람만 가능)

baguette: `baguette tap --udid ABC --x 219 --y 478 --width 438 --height 954`
= "화면 가운데를 톡 눌러". 사람 손이 불필요 → 쉘/파이썬/**AI**가 누를 수 있다.

### 4개의 로봇 부품

| 부품 | 명령 | 사람으로 치면 |
|---|---|---|
| 🖐️ 손가락 | `tap` `swipe` `pinch` `type` `press` | 터치, 타이핑, 버튼 |
| 👁️ 눈(그림) | `screenshot` `stream` `record` | 화면 보기, 녹화 |
| 🧠 눈(글자) | `describe-ui` | "로그인 버튼이 x=100,y=200에 있다"를 JSON으로 읽기 |
| 👂 귀 | `logs` | 앱 로그 실시간 청취 |

👁️와 🧠의 차이가 중요하다. 스크린샷은 AI에게 비싸고 좌표가 부정확하지만,
`describe-ui`는 `{"role":"Button","label":"로그인","frame":{...}}`를 주므로
AI가 **읽고 정확히 누른다.**

### 자물쇠 비유

애플은 공식 문(public API)을 안 열어줬고, Xcode가 쓰는 뒷문의 **자물쇠를
iOS 26에서 바꿨다.** 옛 도구는 옛 열쇠를 꽂아 실패하거나 집(backboardd)을
무너뜨린다. baguette은 Xcode 26 내부를 뜯어 **새 열쇠 모양(9인자)** 을 찾아냈다.

### 데이터 흐름

```
[ 사람 / 스크립트 / AI ]
        ↓ CLI 한 줄 또는 NDJSON 한 줄
    baguette
        ↓ 애플 비공개 SimulatorKit (9-arg)
 iOS 시뮬레이터의 backboardd
        ↓ 진짜 UITouch 이벤트로 변환
   앱은 "사람이 눌렀다"고 믿음
        ↓ 60fps 프레임
    baguette → WebSocket → 브라우저 (localhost:8421)
```

앱은 사람인지 로봇인지 구분할 수 없다. 진짜 HID 이벤트이기 때문.
(Appium 등은 앱 안에 에이전트를 심어야 하는 것과 대비된다.)

---

## 3. 질문별 정리

### ① 설치 및 사용법

**전제조건**

| 항목 | 요구사항 |
|---|---|
| CPU | Apple Silicon (M1~) — 인텔맥 불가 |
| OS | macOS 15+ |
| Xcode | **26 필수** (비공개 프레임워크 링크) |
| 빌드 타깃 | `arm64e-apple-macos26.0` |
| Homebrew | 네이티브 arm64 brew (`/opt/homebrew/bin/brew`) |

**설치**

```bash
# A. Homebrew
brew install baguette
/opt/homebrew/bin/brew install baguette   # Rosetta 문제 시

# B. 소스 빌드
git clone https://github.com/tddworks/baguette && cd baguette
make          # ./build.sh → ./Baguette
swift test    # 460+ 케이스, 시뮬레이터 불필요
```

**세 가지 사용 모드**

(a) 웹 UI — 사람용
```bash
baguette serve                          # 기본 8421
open http://localhost:8421/simulators   # 목록 + Boot/Shutdown
open http://localhost:8421/farm         # 다중 기기 벽면 뷰
# 개발 중 재빌드 없이 JS 수정:
BAGUETTE_WEB_DIR=./Sources/Baguette/Resources/Web baguette serve
```

(b) 원샷 CLI — 스크립트 / CI용
```bash
UDID=$(baguette list --json | jq -r '.running[0].udid')
read W H < <(baguette chrome layout --udid $UDID | jq -r '.screen | "\(.width) \(.height)"')
baguette tap --udid $UDID --x $((W/2)) --y $((H/2)) --width $W --height $H
baguette screenshot --udid $UDID -o /tmp/after.png
baguette describe-ui --udid $UDID | jq '.[] | select(.role=="Button")'
```

(c) 롱리브 파이프 — 에이전트 / 플러그인용 (가장 빠름)
```bash
baguette input --udid $UDID    # stdin NDJSON
# {"type":"tap","x":219,"y":478,"width":438,"height":954}
```
`baguette serve`의 WebSocket도 내부적으로 같은 경로
(`GestureDispatcher → Input → IndigoHIDInput`)를 탄다.

**알려진 함정**

1. 좌표는 **device points** — 정규화(0~1)도 raw 픽셀도 아니다.
   반드시 `chrome layout`의 width/height를 함께 넘길 것
2. Xcode 27의 Device Hub가 입력을 죽인다 — 탭이 ack되지만 반응이 없으면
   `baguette heal --udid <UDID>` (SpringBoard 재시작). `boot`는 자동 수행
3. `siri` 버튼은 모든 경로로 `backboardd`를 크래시시켜 아예 거부됨

### ② 플러그인 / 스킬 / MCP 중 무엇인가

**본체는 네이티브 CLI. 그 위에 플러그인 껍데기와 스킬 껍데기가 둘 다 있고,
MCP는 없다.** (코드 전체 grep 결과 MCP 언급 0건)

| 형태 | 존재 | 근거 | 정체 |
|---|---|---|---|
| ① 네이티브 CLI | ✅ **본체** | `Sources/Baguette/` 296파일 | 모든 형태가 결국 이걸 호출 |
| ② Claude Code 플러그인 | ✅ | `.claude-plugin/plugin.json`, `marketplace.json` | v0.1.75, Apache-2.0 |
| ③ Agent Skill | ✅ | `skills/baguette/SKILL.md` + `references/` + `evals/` | AI용 사용 설명서 |
| ④ 자체 플러그인 시스템 | ✅ | `plugins/deeplink/`, `baguette.json` | baguette이 플러그인 **호스트** |
| ⑤ MCP 서버 | ❌ **없음** | — | 빈 자리 (직접 만들 기회) |

③과 ④의 구분:
- ③ Agent Skill = "이런 상황에 baguette을 이렇게 써라"는 **AI용 프롬프트 문서**
- ④ baguette 플러그인 = 웹 UI에 **새 패널 / 버튼을 추가하는 서브프로세스**.
  `deeplink`는 Python 스크립트이고 매니페스트에 `capabilities: ["open-url"]`로
  권한을 선언한다 (언어 무관, 보안 모델 존재)

MCP로 감싸면 Claude Desktop / Cursor / 다른 에이전트 프레임워크에서도 쓸 수
있다. `baguette input`의 NDJSON 파이프가 이미 있으니 얇은 Node/Python 래퍼로 가능.

### ③ API 토큰이 필요한가

**baguette 자체는 토큰이 전혀 필요 없다.** 완전 로컬, 오프라인 동작.

| 상황 | 필요 | 설명 |
|---|---|---|
| `tap` / `serve` / `screenshot` 등 전부 | ❌ | 로컬 IPC + 비공개 프레임워크 호출 |
| `brew install baguette` | ❌ | 공개 탭 |
| `bakery add owner/repo` | ⚠️ | 공개 repo는 불필요, 비공개면 git 자격증명 |
| Claude Code / Desktop에서 사용 | ✅ | **Claude 쪽** 구독·API 키 (baguette 것이 아님) |
| Device Twin (실기기 미러링) | ❌ | 단 실기기 앱 설치에 Apple Developer 계정 |

**보안**: `baguette serve`는 기본 `--host 127.0.0.1`(루프백)이고 **인증이
없다.** 공개 인터넷에 노출하면 시뮬레이터 완전 제어권이 넘어간다. 외부
사용 시 반드시 리버스 프록시 + 인증을 앞에 둘 것. (이 레이어가 곧 수익화 지점)

### ④ 왜 GitHub에서 유명한가

> 이 세션은 포크 저장소에만 접근 권한이 있어 원본의 실제 스타 수는 확인하지
> 못했다. 아래는 코드·문서가 가진 객관적 강점 기반 분석이다.

1. **타이밍 — 경쟁자가 전멸한 순간에 등장.** iOS 26으로 `idb`(사실상 방치),
   `AXe`, `simctl io`가 깨진 상황에서 동작하는 사실상 유일한 해답
2. **리버스 엔지니어링 서사.** "target 1073741826 not in known targets" 크래시
   메시지 분석, `IndigoHIDTargetForScreen` is a trap, Stark = CarPlay 코드네임,
   자이로 단위 불일치, course 계산의 `cos(latitude)` 누락까지 측정·기록.
   "만들었다"가 아니라 "해부했다" 수준의 콘텐츠
3. **AI 에이전트 시대와 정확히 맞물림.** 플러그인 키워드가
   `["ios","simulator","automation","testing","agent"]`
4. **엔지니어링 품질.** 460+ 테스트(시뮬 없이 통과), 엄격한 DDD 3계층,
   Codecov/CI 배지, `docs/features/` 30개 문서, "TDD is non-negotiable" 문화
5. **비주얼.** README 데모 영상 3개, `/farm`의 다중 시뮬 60fps 벽면 뷰
6. **폭발적 확장.** 입력 → 스트리밍 → 카메라 → 모션 → 네트워크 → 위치 →
   3D 렌더 → 실기기 미러링 → 플러그인 생태계. CHANGELOG 114KB
7. **브랜딩.** 이름이 기억에 남고 "Bon appétit." 태그라인

### ⑤ 로컬 에이전트 구축에 도움이 되는가

**iOS를 다루는 로컬 에이전트라면 사실상 필수. 대체재가 없다.**

에이전트 루프와의 정합성:
```
1. 관찰: baguette describe-ui       → JSON 텍스트 (저렴)
2. 사고: 로컬 LLM 또는 Claude       → "로그인 버튼을 눌러야 함"
3. 행동: baguette tap --x .. --y .. → 실제 HID 이벤트
4. 검증: describe-ui + baguette logs → 화면 변화 / 에러 확인
```

강점 5개
1. **토큰 효율** — 비전 호출(이미지당 수천 토큰) 대비 AX JSON은 수백 토큰.
   비전이 약한 로컬 소형 모델로도 동작한다는 점이 핵심
2. **롱리브 프로세스** — `baguette input`으로 서브프로세스 재생성 오버헤드 0
3. **전부 JSON 출력** — `list --json`, `describe-ui`, `chrome layout`,
   `logs --style ndjson`
4. **에이전트용 문서가 이미 존재** — `skills/baguette/SKILL.md`에 실패 케이스
   (좌표 함정)까지 정리, `evals/evals.json`으로 평가셋도 제공
5. **완전 로컬·오프라인** — 로컬 LLM과 조합 시 전체 파이프라인 오프라인

권장 아키텍처
```
[ Local Agent (Python/Node) ]
  · 로컬 LLM(Ollama) 또는 Claude API
  · 툴: observe / tap / swipe / type
  · 메모리: 스텝별 스크린샷 + AX 스냅샷
        ↓ subprocess stdin NDJSON (핫 파이프)
  [ baguette ] → SimulatorKit 9-arg → [ iOS Simulator ]
```
MCP 서버로 감싸면 툴 정의를 재사용하고 Claude Desktop / Cursor에서도 쓸 수 있다.

제약
- macOS + Apple Silicon + Xcode 26 필수 → 실행 환경이 맥에 묶임
  (원격 Mac mini + 브라우저 프론트로 우회 가능)
- 비공개 API 의존 → iOS/Xcode 업데이트로 깨질 수 있음 (Xcode 27 Device Hub
  이슈 #77가 실제 사례). 버전 핀 고정 전략 필수
- 인증 / 멀티테넌시 없음 → 상용화 시 직접 구현해야 함 (= 사업 기회)

### ⑥ 수익화 아이디어 → 4장 참조

### ⑦ React나 PHP로 만들 수 있는가

**"감싸는 것은 완벽히 가능, 핵심 대체는 불가능."**

❌ 불가능 — HID 코어
```
Infrastructure/Input/IndigoHIDInput.swift
├─ IndigoHIDMessageForMouseNSEvent (9-arg C 심볼)
├─ IOHIDEventCreateDigitizerEvent
├─ dlopen + class_getMethodImplementation
└─ MainActor 필수 (AppKit/NSEvent 스레드로컬 상태)
```
이유: Xcode 비공개 `.framework`에 직접 링크해야 하고, MainActor 실행이
강제되며, 주입 dylib은 ObjC 메서드 스위즐링이 필요하다. PHP/Node FFI로는
arm64e 포인터 인증 + MainActor 요건을 맞출 수 없다.

✅ 가능 — 그 위의 모든 레이어

1. **React로 웹 UI 전면 교체 (난이도 ★★)**
   현재 `Resources/Web/`은 번들러 없는 바닐라 JS IIFE이고, README에
   "`Server` is intentionally dumb — UI lives in `Resources/Web/`"라고
   명시되어 있다. 즉 **서버 프로토콜이 곧 공개 API**.
   ```js
   const ws = new WebSocket(`ws://localhost:8421/simulators/${udid}/stream.avcc`);
   ws.onmessage = e => decoder.decode(e.data);          // 서버→브라우저: 바이너리
   const tap = (x,y) => ws.send(JSON.stringify({        // 브라우저→서버: JSON
     type:'tap', x, y, width, height }));
   ```
   `BAGUETTE_WEB_DIR`로 서빙 루트를 바꿀 수 있어 baguette 재빌드 없이 React
   빌드 산출물을 꽂을 수 있다.

2. **PHP 백엔드 오케스트레이션 (난이도 ★★, 맥에서 실행 전제)**
   ```php
   $udid = escapeshellarg($udid);
   $tree = json_decode(shell_exec("baguette describe-ui --udid $udid"), true);

   // 롱리브 파이프 (빠름)
   $p = proc_open("baguette input --udid $udid",
       [0=>['pipe','r'], 1=>['pipe','w']], $pipes);
   fwrite($pipes[0], json_encode([
       'type'=>'tap','x'=>219,'y'=>478,'width'=>438,'height'=>954])."\n");
   ```
   Laravel로 큐 + 인증 + 멀티테넌시 + 과금을 얹으면 SaaS 백엔드가 된다.
   단 PHP도 macOS에서 실행되어야 하므로, 리눅스 API 서버 → 맥 워커로
   job queue(Redis/SQS)를 넘기는 하이브리드가 정석.

3. **baguette 플러그인 (난이도 ★, 가장 쉬움)**
   `plugins/deeplink/`가 Python이므로 언어 무관 서브프로세스.
   ```json
   { "name":"my-plugin", "apiVersion":1,
     "contributes": {
       "commands":[{"id":"run","run":["node","bin/run.js"]}],
       "panels":[{"id":"run","body":{"kind":"list","source":"run"}}] } }
   ```
   JSON 매니페스트로 UI 패널까지 선언되고 `bakery`(git repo)로 배포.

4. **MCP 서버 (난이도 ★★, Node/TS)**
   ```ts
   server.tool("ios_tap", { x: z.number(), y: z.number() }, async ({x,y}) => {
     pipe.stdin.write(JSON.stringify({type:'tap',x,y,width:W,height:H})+"\n");
     return { content: [{ type:"text", text:"tapped" }] };
   });
   ```

| 레이어 | React | PHP | 난이도 |
|---|---|---|---|
| HID 코어 (SimulatorKit) | ❌ | ❌ | 불가 — Swift/ObjC만 |
| 주입 dylib | ❌ | ❌ | 불가 — ObjC만 |
| 웹 UI 전면 교체 | ✅✅ | — | ★★ |
| REST/WS 클라이언트 | ✅✅ | ✅✅ | ★★ |
| SaaS 백엔드 (인증/과금/큐) | — | ✅✅ | ★★★ |
| baguette 플러그인 | ✅ | ✅ | ★ |
| MCP 서버 | ✅(TS) | ✅ | ★★ |
| 에이전트 오케스트레이터 | ✅ | ✅ | ★★★ |

요약: **baguette은 엔진, 우리가 만드는 것은 차체와 계기판.**

---

## 4. 수익화 아이디어 상세

전제: Apache-2.0이라 상용 이용·수정·재배포가 합법(원저작권 표시 유지).
가장 어려운 엔진은 이미 완성돼 있으므로, 팔아야 할 것은 **엔진에 없는 것** —
인증, 멀티테넌시, 큐, 리포트, 통합, UX. 이것이 정확히 React/PHP 영역이다.

### 🥇 1. App Store 에셋 자동 생성 SaaS — "스크린샷 공장" (최우선 추천)

**문제**: 심사 제출마다 기기 사이즈(6.9"/6.5"/iPad) × 언어(10개국) × 화면(5장)
= 약 150장을 수동 촬영·가공해야 한다. 업데이트마다 반복.
Fastlane snapshot이 있지만 UI 테스트 코드 작성과 설정이 부담.

**baguette 적합성 (기능이 이미 다 있음)**
- `screenshot --size appstore-6.9` — App Store 규격 프리셋 내장
- `render-3d --variant --rotation` — 3D 기기 목업 렌더 (마케팅 이미지)
- `interface appearance light|dark` — 라이트/다크 두 벌
- `interface text-size` — 12개 Dynamic Type (접근성 스크린샷)
- `describe-ui` — 목표 화면 도달 여부 자동 검증
- `status-bar` — 시간/배터리/신호를 애플 권장값으로 고정

**제품 형태**
```
[React 대시보드]  시나리오 노코드 편집 (탭→입력→스크롤→촬영)
      ↓
[Laravel API]     인증 + 프로젝트 + 과금 + job queue
      ↓ Redis
[Mac mini 워커]   baguette 실행 → 150장 생성 → 프레임/텍스트 합성
      ↓
[결과]            ZIP 다운로드 또는 App Store Connect API 직접 업로드
```

**가격**: 무료(1앱/월 10장) → Pro $29/월 → Team $99/월(CI+API) → Enterprise

**장점**: 가치가 즉각적·측정 가능("3시간 → 3분"), 배치 작업이라 Mac mini
1대로 시작 가능, 제품의 90%가 React+PHP, 이미 디자이너에게 비용을 쓰는 일,
결과물이 예뻐 SNS 바이럴.
**차별점**: 경쟁자(Previewed, AppLaunchpad)는 사용자가 스크린샷을 업로드해야
한다. 여기는 **실제 앱을 실행해 자동 촬영**한다 — 이것이 진짜 moat.

**MVP 4주**: 1주 CLI 래퍼 + PHP job runner → 2주 React 시나리오 에디터 →
3주 프레임 합성(Canvas/Sharp) + ZIP → 4주 Stripe + 랜딩.

### 🥈 2. AI QA 에이전트 SaaS — "자연어로 쓰는 iOS 테스트"

**문제**: UI 테스트는 작성 비용이 크고 flaky하며 QA 인력은 비싸다.

**해결**: 테스트를 문장으로 작성.
```
"로그인 화면에서 test@a.com / pw1234 로 로그인 → 홈 탭 확인 →
 프로필에서 닉네임 변경 후 저장 확인"
```
에이전트가 `describe-ui`로 화면을 읽고 LLM이 다음 행동을 결정,
`tap`/`type`으로 실행 후 검증. **좌표 하드코딩이 없어 UI 변경에 강하다.**

**결정적 기여**: `describe-ui`의 토큰 절감이 단가를 사업 가능하게 만든다.
비전 모델로 수백 스텝을 돌리면 원가가 폭발하지만 AX JSON은 수백 토큰.

**증거물**: 실패 시 `record` MP4 + `logs` os_log + 스텝별 스크린샷을
Slack/Jira로 자동 리포트.

**가격**: 실행 횟수 기반 $0.5/run 또는 $99/월 200run, Team $299/월(CI).
**주의**: LLM 원가 관리 필수(AX 캐싱 + 로컬 모델 하이브리드).
경쟁(Mobot, QA Wolf) 대비 차별점은 iOS 26 지원 + 저가.

### 🥉 3. 시뮬레이터 클라우드 (Simulator-as-a-Service)

**문제**: 윈도우/리눅스 개발자, 디자이너, PM, 해외 QA는 맥이 없어 iOS 앱을
볼 수 없다. BrowserStack은 실기기라 비싸다($199/월+).

**해결**: `baguette serve`의 `/farm`이 이미 제품의 90%. 추가할 것은
인증 + 세션 격리, 세션 타이머 + 큐잉, 과금, React 제품급 UI + 공유 링크.

**가격**: 시간당 $1~3 또는 팀 $149/월(동시 3세션).
부수 상품: "고객에게 앱 데모 링크 전송" (영업팀 수요).

**⚠️ 리스크**: 자본 집약적(Mac mini + 전기 + 대역폭)이며, **Xcode/iOS SDK
EULA는 애플 하드웨어에서만 실행을 허용한다.** 하드웨어를 직접 소유/렌탈
(MacStadium, EC2 Mac)하면 하드웨어 요건은 충족하지만, **"시뮬레이터를
제3자에게 서비스로 제공"하는 것이 EULA 허용 범위인지는 변호사 검토가
필수다.** 1·2번은 결과물만 팔고 세션을 팔지 않으므로 훨씬 안전하다.

### 4. RN/Expo/Flutter 개발 루프 도구

`examples/expo-bakery/`에 이미 Expo 플러그인 예제(expo-devmenu, expo-reload)가
들어있다.

**제품**: VSCode 확장 + 로컬 대시보드
- 저장 시 여러 기기 동시 리로드 + **제스처 시나리오 리플레이로 같은 화면 복귀**
- 같은 조작을 iPhone SE / 17 Pro Max / iPad에 동시 전송 (`/farm`)
- `network`로 3G/오프라인 즉시 토글, `motion`/`location`으로 센서 시뮬

**가격**: 개인 $9/월, 팀 $19/seat — 저가 대량 모델.
**장점**: 로컬 실행이라 서버 원가 거의 0(라이선스 검증만), RN 개발자 풀이
거대, VSCode 마켓플레이스가 무료 배포 채널.
**주의**: 개인 지불 의지가 낮으므로 팀 플랜 중심 + 넓은 무료 티어.

### 5. MCP 서버 + 프리미엄 스킬 팩

오픈소스 MCP 서버를 무료 배포해 **유입 채널**을 만들고, "iOS 에이전트 프로 팩"
(고급 시나리오, 회귀 감지, 팀 공유)을 유료화. 직접 수익보다 신뢰·유입 자산으로
가치가 크다 — 1·2번의 미끼 상품.

### 6. 마케팅 영상·GIF 생성 서비스

`record` + `render-3d` + 브라우저 합성 녹화(베젤 + 제스처 오버레이 포함)로
Product Hunt 런치 영상, 문서용 GIF, 릴스를 자동 생성. **1번의 애드온**으로
붙여 ARPU를 올리는 것이 단독 창업보다 유리.

### 7. CI/CD 러너 통합

`baguette-action` + 셀프호스트 macOS 러너 관리. "PR마다 스크린샷 diff 자동
코멘트"는 팀이 실제로 결제하는 기능 — 1번의 Team 플랜 기능으로 편입.

### 8. 콘텐츠·교육 (가장 빠른 현금화)

- "iOS 비공개 API 리버스 엔지니어링" 강의 (인프런/유데미)
- "TDD·DDD 실전" — 460개 테스트 + CLAUDE.md의 엄격한 게이트가 살아있는 교재
- "AI 에이전트에게 손 달아주기" — 현재 최고 관심 주제
- 한국어 콘텐츠 경쟁이 거의 없음
- 자본 0, 법적 리스크 0, 2주면 첫 매출 → 다른 제품 개발 중 현금흐름 확보

### 🎯 권장 실행 순서

```
[0~1개월]  ⑧ 콘텐츠로 즉시 현금 + 인지도
             ⑤ MCP 서버 오픈소스로 신뢰 자산      (자본 0, 리스크 0)
[1~3개월]  ① App Store 스크린샷 공장 MVP
             (Mac mini 1대 + React + Laravel)     (첫 유료 고객)
[3~6개월]  ⑥ 영상 애드온 + ⑦ CI 통합을 ①에 번들   (ARPU 상승, 팀 플랜)
[6~12개월] ② AI QA 에이전트 (①의 고객에게 업셀)
[선택]     ③ 시뮬레이터 클라우드 — 변호사 검토 후에만
```

### ⚠️ 반드시 짚어야 할 리스크 3개

1. **기술 리스크** — 비공개 API 의존으로 iOS/Xcode 업데이트에 깨질 수 있다.
   Xcode 27 Device Hub 이슈(#77)가 실제 사례이며 대응 코드가 들어가 있다.
   버전 핀 고정 + 업스트림 추적 필수. 제품 SLA에 지원 iOS 버전을 명시할 것
2. **법적 리스크** — Apple EULA는 애플 하드웨어에서만 실행을 허용한다.
   1·2·4·8번은 결과물/도구를 판매하므로 상대적으로 안전하나,
   **3번(세션 대여)은 변호사 검토 후에만.** 순서를 바꾸지 말 것
3. **업스트림 의존** — 핵심 엔진은 tddworks 소유. Apache-2.0이라 포크는
   자유롭지만, **우리 가치는 반드시 래퍼 계층(인증·큐·UI·리포트)에 있어야
   한다.** 엔진에만 의존하면 언제든 대체될 수 있다

### 우리에게 유리한 이유

- baguette이 가장 어려운 부분(HID 리버스 엔지니어링)을 이미 완성해뒀다
- 남은 90%(인증, 과금, 큐, 대시보드, 리포트)가 정확히 React/PHP 영역이다
- 서버가 "의도적으로 멍청하게" 설계되어 프로토콜이 곧 공개 API다
- AI 에이전트 타이밍이 좋고, 한국어 콘텐츠 시장은 거의 비어 있다

---

## 참고 문서

| 문서 | 내용 |
|---|---|
| `README.md` | 퀵스타트, 전체 CLI 레퍼런스, 와이어 프로토콜 JSON 예시 |
| `CLAUDE.md` | TDD 게이트, 아키텍처 규칙, iOS 26 제약 목록 |
| `AGENTS.md` | 에이전트용 가이드 |
| `docs/ARCHITECTURE.md` | tap → `UITouch` 전체 흐름, 계층 다이어그램, 라우트 표 |
| `docs/features/*.md` | 기능별 리버스 엔지니어링 기록 30개 |
| `Sources/Baguette/Infrastructure/Input/IndigoHIDInput.swift` | 9-arg 레시피 (상세 주석) |
| `skills/baguette/SKILL.md` | 에이전트용 사용 지침 + 좌표 함정 |
