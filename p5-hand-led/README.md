# 손가락으로 LED 밝기 제어하기

ml5.js의 handPose로 웹캠 속 손을 인식하고, 엄지와 검지 사이 거리로 슬라이더 값을 바꾼 뒤 그 값을 시리얼 통신으로 아두이노에 보내 LED 밝기를 조절하도록 기존 코드를 수정했습니다. 가운데 손가락을 구부린 상태에서만 밝기가 조절됩니다.

---

## 회로

아두이노는 아래 사진처럼 연결했습니다.

<img width="1266" height="806" alt="s" src="https://github.com/user-attachments/assets/47105ae5-9112-4ad0-b29a-afc9ac65bf5f" />

밝기 조절용 LED를 PWM을 지원하는 10번 핀에 저항을 거쳐 연결했습니다.

---

## 코드 설명 – p5.js

### 1. 변수 추가

```javascript
let handPose, video, slider, serial, connectButton;
let lastSent = -1;
let statusText = "연결 안 됨";
```

시리얼 통신을 위한 `serial`, 연결 버튼 `connectButton`, 마지막으로 보낸 값을 저장하는 `lastSent`, 연결 상태를 표시할 `statusText`를 추가했습니다.

### 2. 슬라이더 범위 변경

```javascript
slider = createSlider(0, 255, 50, 1);   // 기존: createSlider(0, 100, 50, 1)
```

아두이노의 `analogWrite()`가 0\~255 값을 받기 때문에 슬라이더 범위를 0\~100에서 0\~255로 바꿨습니다.

### 3. 아두이노 연결

```javascript
serial = new SerialHelper(
  9600,
  () => {},
  (message) => (statusText = message)
);
connectButton = createButton("Arduino 연결 / 해제");
connectButton.mousePressed(() => serial.toggle());
```

`setup()`에 추가한 코드입니다. 9600 속도로 시리얼 통신을 준비하고, 연결 상태 메시지가 오면 `statusText`에 저장합니다. 버튼을 누르면 아두이노와 연결하거나 연결을 해제합니다.

### 4. 손가락 거리 계산 범위 변경

```javascript
let s = round(200*(v/p)-30);   // 기존: round(100*(v/p))-15
return constrain(s,0,255);     // 기존: constrain(s,0,100)
```

`getNormalizedDistance()`에서 엄지-검지 거리(`v`)를 손바닥 길이(`p`)로 나눠 손과 카메라 사이 거리와 상관없이 비슷한 값이 나오게 합니다. 슬라이더 범위가 0\~255로 바뀌었기 때문에 계산 결과도 0\~255가 되도록 바꿨습니다.

### 5. 슬라이더 값 부드럽게 바꾸기

```javascript
let v = getNormalizedDistance(hands);
sliderValue += 0.05 * (v - sliderValue);
slider.value(sliderValue);
send();
```

`draw()`의 `if (adjustMode)` 안을 수정한 코드입니다. 기존에는 `slider.value()`를 읽어서 계산했는데, 슬라이더는 1 단위로 반올림되기 때문에 작은 변화가 사라졌습니다. 그래서 소수점까지 저장되는 `sliderValue` 변수로 계산한 뒤 슬라이더에 넣도록 바꿨습니다. 목표값과의 차이의 5%씩만 다가가서 손이 떨려도 밝기가 부드럽게 바뀝니다. 값을 바꾼 뒤 `send()`로 아두이노에 보냅니다.

### 6. 값이 바뀔 때만 보내기

```javascript
async function send() {
  if (!serial) return;
  let value = slider.value();
  if (value === lastSent) return;
  lastSent = value;
  await serial.send(value);
}
```

새로 추가한 함수입니다. `draw()`는 1초에 약 60번 실행되기 때문에 매번 보내면 아두이노가 처리하지 못하고 밀릴 수 있습니다. 그래서 이전에 보낸 값(`lastSent`)과 다를 때만 보내도록 했습니다.

### 7. 슬라이더를 직접 움직여도 보내기

```javascript
function updateSliderValue() {
  sliderValue = Number(slider.value());
  send();
}
```

마우스로 슬라이더를 직접 움직였을 때도 값이 아두이노로 전송되도록 `send()`를 추가했습니다.

### 8. 화면에 상태와 값 표시

```javascript
textSize(14);
text(statusText, width / 2, 66);
textSize(18);
text(slider.value(), 38, height / 2);
```

`draw()` 마지막에 추가한 코드입니다. 화면 위쪽에 아두이노 연결 상태를, 슬라이더 옆에 현재 밝기 값을 표시합니다.

---

## 코드 설명 – Arduino

```cpp
void loop() {
  if (Serial.available() > 0) {
    int s = Serial.parseInt();
    if (Serial.read() == '\n') {
      analogWrite(10, constrain(s, 0, 255));
    }
  }
}
```

시리얼로 데이터가 들어오면 `Serial.parseInt()`로 `"123"` 같은 문자열을 숫자로 읽습니다. 그 다음 글자가 줄바꿈(`\n`)이면 한 줄을 끝까지 받은 것이므로, `constrain()`으로 값을 0~255 사이로 제한한 뒤 `analogWrite()`로 10번 핀에 출력해 LED 밝기를 조절합니다.

---

## 전체 코드

<details>
<summary>p5.js 코드 보기 (sketch.js)</summary>

```javascript
let handPose, video, slider,serial, connectButton;;
let hands = [];
let sliderValue = 50;
let connections;

async function setup() {
  createCanvas(640, 480);
  
  slider = createSlider(0, 100, 50, 1);
  slider.style('writing-mode', 'vertical-rl');
  slider.style('height', '440px');
  slider.style('width', '20px');    
  slider.style('transform: rotate(180deg)'); 
  slider.position(10, 20);
  slider.input(updateSliderValue);
  
  video = createCapture(VIDEO, {flipped:true});
  video.size(640, 480);
  video.hide();
  
  handPose = await ml5.handPose({flipped:true});
  handPose.detectStart(video, gotHands);
  connections = handPose.getConnections();
  serial = new SerialHelper(
    9600,
    {},
    (message) => (statusText = message)
  );
  connectButton = createButton("Arduino 연결 / 해제");
  connectButton.mousePressed(() => serial.toggle());
}

function draw() {
  image(video, 0, 0, width, height);

  drawSkeletons(hands);
  drawKeyPoints(hands);

  let adjustMode = isGrabMode(hands);
  if (adjustMode) {
    drawConnectedLine(hands);
    let c = slider.value();
    let v = getNormalizedDistance(hands);
    slider.value(c + 0.05*(v-c)); // 부드럽게 조정
  }
}

function gotHands(results) {
  hands = results;
}

function drawKeyPoints( hs) {
  for (let i = 0; i < hs.length; i++) {
    let hand = hs[i];
    for (let j = 0; j < hand.keypoints.length; j++) {
      let keypoint = hand.keypoints[j];
      fill(255, 0, 0);
      noStroke();
      circle(keypoint.x, keypoint.y, 10);
    }
  }
}

function drawSkeletons(hs) {
  for (let i = 0; i < hs.length; i++) {
    let hand = hs[i];
    stroke(0, 255, 0);
    strokeWeight(2);
    for (const [a, b] of connections) {
      const start = hand.keypoints[a];
      const end = hand.keypoints[b];
      if (start && end) {
        line(start.x, start.y, end.x, end.y);
      }
    }
  }  
}

// 가운데 손가락을 구부렸는지 확인
function isGrabMode(hs) {
  if( hs == null ) return false;
  if( hs.length == 0) return false;
  let ks = hs[0].keypoints;
  if( ks.length != 21 ) return false;

  let palmLength   = dist(ks[0].x,ks[0].y,ks[9].x,ks[9].y);
  let grabDistance = dist(ks[0].x,ks[0].y,ks[12].x,ks[12].y);
  if( grabDistance > palmLength ) return false;
  return true;
}

function drawConnectedLine(hs) {
  if(hs.length > 0) {
    let hand = hs[0];
    const a = hand.keypoints[4]; // 엄지 끝
    const b = hand.keypoints[8]; // 검지 끝
    stroke(255, 140, 0);
    strokeWeight(4);
    line(a.x,a.y,b.x,b.y);
    noFill();
    circle(a.x,a.y,22);
    circle(b.x,b.y,22);
  }
}

function getNormalizedDistance(hs) {
  if(hs.length > 0) {
    let ks = hs[0].keypoints;
    const a = ks[4]; // 엄지 끝
    const b = ks[8]; // 검지 끝
    const c = ks[0]; // 손목
    const d = ks[9]; // 중지 시작점
    let v = dist(a.x,a.y,b.x,b.y); // 엄지-검지 거리
    let p = dist(c.x,c.y,d.x,d.y); // 손바닥 길이
    let s = round(100*(v/p))-15;
    return constrain(s,0,100);
  }
  return 0;
}
  
function updateSliderValue() {
  sliderValue = Number(slider.value());
}

```

</details>

p5.js 전체 파일(serial-helper.js 포함): [p5.js 에디터에서 보기]https://editor.p5js.org/POo0q/sketches/mIFePq2HV



<details>
<summary>Arduino 전체 코드 보기</summary>

```cpp
void setup() {
  Serial.begin(9600);
  pinMode(10, OUTPUT);
}

void loop() {
  if (Serial.available() > 0) {
    int s = Serial.parseInt();   // "123" → 123
    if (Serial.read() == '\n') { // 줄 끝까지 읽었으면
      analogWrite(10, constrain(s, 0, 255));
    }
  }
}
```

</details>

---

## 동작 영상

https://github.com/user-attachments/assets/6a457116-aba3-4104-b2f6-1c3df84c2f48
