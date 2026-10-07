# 티처블머신으로 조종하는 웹캠 뱀게임

> **강의노트** · 2026-10-07
> 웹캠에 비친 손짓·몸짓을 **Teachable Machine 이미지 모델**로 분류하고, 그 결과(left / right / up / down / neutral)로 브라우저 뱀게임을 조종합니다. 모델 교체부터 GitHub Pages 배포까지 하루 동안 진행한 과정을 정리했습니다.

**플레이: https://leesm3528-netizen.github.io/snake/**

---

## 📋 목차

1. [학습 목표](#1-학습-목표)
2. [준비물: Teachable Machine 모델](#2-준비물-teachable-machine-모델)
3. [실습 1: 포즈 모델을 이미지 모델로 바꾸기](#3-실습-1-포즈-모델을-이미지-모델로-바꾸기)
4. [실습 2: 인식 결과를 게임 조작으로 바꾸기](#4-실습-2-인식-결과를-게임-조작으로-바꾸기)
5. [실습 3: 뱀게임 로직](#5-실습-3-뱀게임-로직)
6. [GitHub 업로드와 GitHub Pages 배포](#6-github-업로드와-github-pages-배포)
7. [트러블슈팅](#7-트러블슈팅)
8. [정리 및 복습 퀴즈](#8-정리-및-복습-퀴즈)
9. [빠른 실행 가이드](#9-빠른-실행-가이드)

---

## 1. 학습 목표

이 강의를 마치면 다음을 할 수 있습니다.

- [ ] Teachable Machine 모델의 `metadata.json`을 보고 **모델 종류와 클래스**를 확인할 수 있다.
- [ ] `@teachablemachine/pose` 코드를 `@teachablemachine/image` 코드로 바꿀 수 있다.
- [ ] 확률 기준값과 연속 프레임 조건으로 **흔들리는 인식 결과를 안정화**할 수 있다.
- [ ] 웹캠 페이지가 `file://`에서는 안 되고 **https 또는 localhost**에서만 되는 이유를 설명할 수 있다.
- [ ] `gh` CLI로 저장소를 만들고 **GitHub Pages**로 배포할 수 있다.

---

## 2. 준비물: Teachable Machine 모델

사용한 모델: `https://teachablemachine.withgoogle.com/models/U93WCxIU8/`

모델 주소 뒤에 `metadata.json`을 붙이면 모델 정보를 볼 수 있습니다.

```bash
curl -sL https://teachablemachine.withgoogle.com/models/U93WCxIU8/metadata.json
```

```json
{
  "packageName": "@teachablemachine/image",
  "labels": ["left", "right", "up", "down", "neutral"],
  "imageSize": 224
}
```

| 항목 | 값 | 의미 |
|---|---|---|
| `packageName` | `@teachablemachine/image` | **이미지 프로젝트**로 학습한 모델 |
| `labels` | left, right, up, down, neutral | 게임 방향 4개 + 가만히 있기 |
| `imageSize` | 224 | 모델 입력 이미지 크기 (224×224) |

> 📌 **핵심 포인트**: `-L` 옵션이 없으면 `Found. Redirecting to ...`만 나옵니다. 모델 파일은 실제로 `storage.googleapis.com`에 있어서 리다이렉트를 따라가야 합니다.

---

## 3. 실습 1: 포즈 모델을 이미지 모델로 바꾸기

📄 파일: [`index.html`](index.html)

처음 코드는 **포즈 모델**(`@teachablemachine/pose`)용이었습니다. 새 모델은 **이미지 모델**이라 라이브러리와 추론 방식을 바꿔야 합니다.

### 개념: 포즈 모델 vs 이미지 모델

| 구분 | 포즈 모델 | 이미지 모델 |
|---|---|---|
| 라이브러리 | `@teachablemachine/pose` | `@teachablemachine/image` |
| 전역 객체 | `tmPose` | `tmImage` |
| 입력 | PoseNet이 뽑은 **관절 좌표** | 웹캠 **이미지 전체** |
| 추론 단계 | `estimatePose()` → `predict()` (2단계) | `predict()` (1단계) |
| 화면 표시 | 캔버스에 관절·뼈대를 직접 그림 | `webcam.canvas`를 그대로 붙임 |

### 바뀐 코드

**① 라이브러리와 모델 주소**

```html
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@1.3.1/dist/tf.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@teachablemachine/image@0.8/dist/teachablemachine-image.min.js"></script>
```

```js
const MODEL_URL = "https://teachablemachine.withgoogle.com/models/U93WCxIU8/";
model = await tmImage.load(MODEL_URL + "model.json", MODEL_URL + "metadata.json");
```

**② 웹캠 화면**: 별도 캔버스를 만들 필요 없이 웹캠 캔버스를 바로 붙입니다.

```js
webcam = new tmImage.Webcam(300, 300, true);   // 너비, 높이, 좌우반전
await webcam.setup();                          // 카메라 권한 요청
await webcam.play();
container.appendChild(webcam.canvas);
```

**③ 추론 루프**

```js
// 이전 (포즈)
const { pose, posenetOutput } = await model.estimatePose(webcam.canvas);
const prediction = await model.predict(posenetOutput);

// 이후 (이미지)
const prediction = await model.predict(webcam.canvas);
```

`prediction`은 클래스마다 `{ className, probability }`가 들어 있는 배열입니다.

> 📌 **핵심 포인트**: 모델이 바뀌어도 `predict()`가 돌려주는 결과 형식은 같습니다. 그래서 **게임 쪽 코드는 하나도 바꾸지 않았습니다.** 인식 부분과 게임 부분을 나눠 두면 이렇게 교체가 쉽습니다.

---

## 4. 실습 2: 인식 결과를 게임 조작으로 바꾸기

### 개념: 왜 그냥 1등 클래스를 쓰면 안 될까?

웹캠 인식은 프레임마다 조금씩 흔들립니다. 손을 움직이는 중간에 엉뚱한 클래스가 잠깐 1등이 되면 뱀이 갑자기 꺾여 벽에 부딪힙니다. 그래서 **두 가지 조건**을 겁니다.

```js
const THRESHOLD = 0.8;   // 이 확률 이상일 때만 방향 전환
const STABLE_FRAMES = 2; // 같은 동작이 연속으로 이 프레임 수만큼 나와야 인정

if (top.probability >= THRESHOLD) {
  const label = top.className.toLowerCase();
  stableCount = label === lastLabel ? stableCount + 1 : 1;
  lastLabel = label;
  if (stableCount >= STABLE_FRAMES) handleCommand(label);
}
```

| 조건 | 거르는 것 |
|---|---|
| 확률 ≥ 0.8 | 애매한 프레임 (여러 클래스 확률이 비슷할 때) |
| 같은 결과 2프레임 연속 | 한 프레임만 튀는 잘못된 인식 |

### neutral은 어떻게 처리할까?

```js
const DIRS = { up: [0, -1], down: [0, 1], left: [-1, 0], right: [1, 0] };

function handleCommand(label) {
  if (!DIRS[label]) return;             // neutral 등은 무시 → 현재 방향 유지
  const [dx, dy] = DIRS[label];
  const [cx, cy] = DIRS[dir];
  if (dx === -cx && dy === -cy) return; // 정반대 방향 금지
  nextDir = label;
}
```

- `neutral`은 `DIRS`에 없으므로 무시됩니다. 뱀은 하던 방향으로 계속 갑니다.
- 오른쪽으로 가는 중에 `left`가 나오면 자기 몸에 바로 부딪히므로 무시합니다.

> 📌 **핵심 포인트**: `neutral` 클래스가 있어서 "아무 동작도 안 함"을 따로 인식할 수 있습니다. 이 클래스가 없으면 가만히 있어도 네 방향 중 하나로 분류됩니다.

---

## 5. 실습 3: 뱀게임 로직

| 요소 | 구현 |
|---|---|
| 보드 | 480×480 캔버스, 20×20 칸 |
| 뱀 | 칸 좌표 배열 `[{x, y}, ...]`, 0번이 머리 |
| 이동 | `setInterval(tick, 속도)` (느림 260ms · 보통 180ms · 빠름 120ms) |
| 먹이 | 뱀과 겹치지 않는 칸에 무작위 배치 |
| 최고 점수 | `localStorage`에 저장 |

### 한 칸 이동 (`tick`)

```
방향 확정 ─▶ 새 머리 좌표 계산 ─▶ 벽/몸 충돌? ──예──▶ GAME OVER
                                        │
                                       아니오
                                        ▼
                       머리를 앞에 추가 (unshift)
                                        │
                     먹이를 먹었나? ──예──▶ 점수 +1, 새 먹이 (꼬리 유지 = 길어짐)
                                        │
                                       아니오 ──▶ 꼬리 제거 (pop)
```

> 📌 **핵심 포인트**: 뱀이 "움직이는" 것은 앞에 머리를 하나 붙이고 뒤에서 꼬리를 하나 떼는 것입니다. 먹이를 먹었을 때 꼬리를 안 떼면 길어집니다.

---

## 6. GitHub 업로드와 GitHub Pages 배포

### 업로드

```powershell
git branch -M main                       # 브랜치 이름을 main으로 변경
git add index.html
git commit -m "Add webcam-controlled snake game"
gh repo create snake --public --source . --remote origin --push
```

### GitHub Pages 켜기

```powershell
gh api -X POST repos/leesm3528-netizen/snake/pages -f "source[branch]=main" -f "source[path]=/"
gh api repos/leesm3528-netizen/snake/pages/builds/latest --jq ".status"   # building → built
```

- 저장소 최상위의 `index.html`이 그대로 사이트 첫 화면이 됩니다.
- 이후 `main`에 푸시하면 자동으로 다시 배포됩니다.
- 처음 켠 직후에는 1분 정도 404가 나올 수 있습니다. 빌드 상태가 `built`가 되면 접속됩니다.

> 📌 **핵심 포인트**: 브라우저는 **https 또는 localhost**에서만 웹캠을 허용합니다. GitHub Pages는 https로 서비스되므로 따로 서버를 켜지 않아도 웹캠이 동작합니다.

---

## 7. 트러블슈팅

### 🐛 `gh: command not found`

| 항목 | 내용 |
|---|---|
| **증상** | `gh`로 로그인까지 해 두었는데 터미널에서 `gh`를 찾지 못함 |
| **원인** | `gh`를 설치하기 전에 열린 터미널이라 **PATH가 갱신되지 않음** |
| **확인** | `Test-Path "C:\Program Files\GitHub CLI\gh.exe"` → `True` |
| **해결** | 터미널을 새로 열거나, 전체 경로로 실행: `& "C:\Program Files\GitHub CLI\gh.exe" auth status` |

### 🐛 코드 자동 치환이 실패함

| 항목 | 내용 |
|---|---|
| **증상** | 파이썬 스크립트로 여러 줄 코드를 찾아 바꾸려 했는데 "찾을 수 없음" |
| **원인 1** | Windows 파일의 줄바꿈이 `\r\n`(CRLF)이라 `\n`으로 쓴 검색어와 다름 |
| **원인 2** | `str.replace()`는 **일치하는 곳을 모두** 바꿈. `requestAnimationFrame(poseLoop)`가 두 군데 있어서 앞의 치환이 뒤의 검색 대상까지 바꿔 버림 |
| **해결** | 한 곳만 정확히 지정해서 고치기 (편집기에서 직접 수정, 또는 `replace(a, b, 1)`) |

### 🐛 웹캠이 안 켜짐

| 원인 | 해결 |
|---|---|
| `index.html`을 더블클릭해서 `file://`로 열었음 | GitHub Pages 주소로 열거나 로컬 서버 사용 (아래 실행 가이드) |
| 카메라 권한을 거부했음 | 주소창 왼쪽 자물쇠 아이콘 → 카메라 허용 → 새로고침 |
| 다른 앱이 카메라 사용 중 | Zoom, 카메라 앱 등을 닫기 |

### 참고: Claude 아티팩트 버전

같은 게임을 Claude 아티팩트로도 만들었습니다. 아티팩트에서는 **웹캠을 쓸 수 없고 외부 모델도 불러올 수 없어서**, 방향키·스와이프·화면 버튼으로 조종하는 버전입니다. (비공개 링크라 공유 설정을 해야 다른 사람이 볼 수 있습니다.)

---

## 8. 정리 및 복습 퀴즈

### 오늘 배운 내용

| 주제 | 핵심 |
|---|---|
| 모델 확인 | `metadata.json`의 `packageName`, `labels` |
| 모델 교체 | `tmPose` → `tmImage`, `predict(webcam.canvas)` 한 번으로 추론 |
| 인식 안정화 | 확률 기준값 0.8 + 2프레임 연속 |
| neutral | 방향 목록에 없으면 무시 → 현재 방향 유지 |
| 배포 | `gh repo create --push` + Pages API, https라서 웹캠 가능 |

### 🧠 복습 퀴즈

<details>
<summary><b>Q1.</b> 받은 Teachable Machine 모델이 이미지 모델인지 포즈 모델인지 어떻게 알 수 있나요?</summary>

모델 주소 뒤에 `metadata.json`을 붙여 열고 `packageName`을 봅니다. `@teachablemachine/image`면 이미지 모델, `@teachablemachine/pose`면 포즈 모델입니다.
</details>

<details>
<summary><b>Q2.</b> 이미지 모델에서는 왜 <code>estimatePose()</code>가 필요 없나요?</summary>

이미지 모델은 관절 좌표가 아니라 웹캠 이미지 자체를 입력으로 받기 때문입니다. `model.predict(webcam.canvas)` 한 번이면 됩니다.
</details>

<details>
<summary><b>Q3.</b> <code>STABLE_FRAMES</code>를 1로 줄이면 어떤 일이 생기나요? 5로 늘리면요?</summary>

1이면 반응은 빨라지지만 한 프레임만 잘못 인식돼도 방향이 바뀝니다. 5면 오인식은 줄지만 동작을 바꾼 뒤 방향이 바뀌기까지 시간이 걸려서 반응이 굼뜹니다.
</details>

<details>
<summary><b>Q4.</b> <code>index.html</code>을 더블클릭하면 웹캠이 안 켜지는데 GitHub Pages에서는 켜지는 이유는?</summary>

브라우저는 보안 때문에 https 또는 localhost 페이지에서만 카메라를 허용합니다. 더블클릭하면 `file://`로 열리고, GitHub Pages는 https입니다.
</details>

<details>
<summary><b>Q5.</b> 오른쪽으로 가는 뱀에게 <code>left</code>가 인식되면 어떻게 되나요?</summary>

무시됩니다. 정반대 방향으로 꺾으면 바로 자기 몸에 부딪히므로 `handleCommand()`에서 막습니다.
</details>

### 🚀 도전 과제

- [ ] 화면에 슬라이더를 달아 `THRESHOLD`를 실시간으로 바꿔 보세요.
- [ ] 먹이를 먹을 때마다 조금씩 빨라지게 만들어 보세요.
- [ ] 벽에 부딪히면 반대편으로 나오는 "벽 통과 모드"를 추가해 보세요.
- [ ] Teachable Machine에서 `pause` 클래스를 추가로 학습해서 일시정지 동작을 만들어 보세요.

---

## 9. 빠른 실행 가이드

### 바로 플레이

**https://leesm3528-netizen.github.io/snake/** 접속 → **웹캠 켜고 시작** → 카메라 허용

### 내 PC에서 실행

```bash
python -m http.server 8000
```

브라우저에서 `http://localhost:8000` 을 엽니다.

### 조작

| 입력 | 동작 |
|---|---|
| `left` / `right` / `up` / `down` | 그 방향으로 전환 |
| `neutral` | 현재 방향 유지 |
| 키보드 방향키 | 같은 방향 전환 (웹캠 없이 테스트할 때) |
| **다시 시작** 버튼 | 새 게임 |
| 속도 선택 | 느림 / 보통 / 빠름 (바꾸면 게임이 다시 시작됨) |

### 설정값 (`index.html` 위쪽)

| 상수 | 설명 | 기본값 |
|---|---|---|
| `MODEL_URL` | Teachable Machine 모델 주소 | `.../models/U93WCxIU8/` |
| `THRESHOLD` | 이 확률 이상일 때만 방향 전환 | `0.8` |
| `STABLE_FRAMES` | 같은 결과가 연속으로 나와야 하는 프레임 수 | `2` |
