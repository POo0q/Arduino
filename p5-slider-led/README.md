# p5.js 슬라이더로 LED 밝기 조절하기

p5.js 화면에 슬라이더를 추가하고, 슬라이더 값을 시리얼 통신으로 아두이노에 보내 LED 밝기를 조절하도록 기존 코드를 수정했습니다.


---

## 회로

아두이노는 아래 사진처럼 연결했습니다.

<img width="1982" height="1198" alt="1793605593787_image" src="https://github.com/user-attachments/assets/4aa10894-9741-4f9d-a1eb-4a51c4028d71" />

I2C LCD의 SDA, SCL은 A4, A5에, 가변저항 가운데 다리는 A0에, 버튼은 2번 핀에 연결했습니다. ON/OFF용 LED는 13번 핀에, 이번에 추가한 밝기 조절용 LED는 10번 핀에 저항을 거쳐 연결했습니다.

---

## 코드 설명 – p5.js

### 1. 슬라이더 변수 추가

```javascript
let slider;
```

슬라이더를 `setup()`, `draw()`와 다른 함수에서도 함께 쓰기 위해 전역 변수로 선언했습니다.

### 2. 슬라이더 만들기

```javascript
slider = createSlider(0, 255, 0);
slider.input(callbackFunction);
```

`setup()`에 추가한 코드입니다. `createSlider(0, 255, 0)`은 최솟값 0, 최댓값 255, 처음 값 0인 슬라이더를 만듭니다. 아두이노의 `analogWrite()`가 0~255 값을 받기 때문에 범위를 똑같이 맞췄습니다. `slider.input(callbackFunction)`으로 슬라이더를 움직일 때마다 `callbackFunction`이 실행되도록 연결했습니다.

### 3. 슬라이더 값 보내기

```javascript
async function callbackFunction() {
  await serial.send(slider.value());
}
```

슬라이더가 움직이면 현재 값(`slider.value()`)을 시리얼 통신으로 아두이노에 보냅니다.

### 4. 화면에 슬라이더 표시

```javascript
fill(25, 40, 60); textSize(15); textAlign(CENTER);
text('슬라이더: ' + slider.value() + '/255', 420, 265);
slider.position(310, 295);
slider.size(220);
```

`draw()`에 추가한 코드입니다. 현재 슬라이더 값을 `슬라이더: 값/255` 형태로 화면에 보여줍니다. 슬라이더는 가변저항 막대 아래에 배치하고, 막대와 같은 길이(220)로 맞췄습니다.

---

## 코드 설명 – Arduino

### 1. 숫자를 받으면 LED 밝기 조절

```cpp
else if (command[0] >= '0' && command[0] <= '9') {
  analogWrite(10, atoi(command));
}
```

`taskSerialCallback()`에 추가한 코드입니다. 기존에는 `ON`, `OFF`만 처리하고 나머지는 `Invalid command`로 처리했는데, 받은 명령의 첫 글자가 숫자이면 슬라이더 값으로 판단하도록 조건을 추가했습니다. `atoi()`로 `"128"` 같은 문자열을 숫자로 바꾸고, `analogWrite()`로 10번 핀에 0~255 값을 출력해 LED 밝기를 조절합니다. 0이면 꺼지고 255이면 가장 밝습니다.

### 2. 10번 핀 출력 설정

```cpp
pinMode(10, OUTPUT);
```

`setup()`에 추가한 코드입니다. 밝기 조절용 LED를 연결한 10번 핀을 출력으로 설정했습니다. 10번 핀은 PWM(`~`)을 지원하는 핀이라 `analogWrite()`로 밝기 조절이 가능합니다.

---

## 전체 코드
<details>
<summary>p5.js 수정 코드 보기</summary>

```javascript
// 복잡한 통신은 serial-helper.js가 처리합니다.
let serial, connectButton;
let ledOn = false;
let potValue = 0;
let statusText = 'Arduino를 연결해 주세요.';
let slider;

function setup() {
  createCanvas(600, 400);
  textFont('sans-serif');
  serial = new SerialHelper(9600, receiveLine, message => statusText = message);
  connectButton = createButton('Arduino 연결 / 해제');
  connectButton.mousePressed(() => serial.toggle());
  slider = createSlider(0, 255, 0);
  slider.input(callbackFunction);
}

function draw() {
  background(242, 246, 250);
  fill(25, 40, 60); noStroke(); textAlign(CENTER);
  textSize(24); text('Arduino Uno R3 · Web Serial', width / 2, 36);
  textSize(14); text(statusText, width / 2, 66);
  fill(ledOn ? '#22c55e' : '#64748b');
  circle(160, 175, 110);
  fill(255); textSize(22); text(ledOn ? 'ON' : 'OFF', 160, 183);
  fill(25, 40, 60); textSize(15); text('원을 클릭하면 LED 토글', 160, 260);
  text('가변저항 A0', 420, 120);
  fill('#dbe3ed'); rect(310, 155, 220, 40, 10);
  fill('#3b82f6'); rect(310, 155, map(potValue, 0, 1023, 0, 220), 40, 10);
  fill(25, 40, 60); textSize(24); text(potValue + ' / 1023', 420, 235);
  textSize(14); text('9600 baud · ON / OFF + 줄바꿈', 300, 325);
  text('수신: 0~1023 또는 POT:값 · LED:ON / LED:OFF', 300, 352);
  fill(25, 40, 60); textSize(15); textAlign(CENTER);
  text('슬라이더: ' + slider.value() + '/255', 420, 265);
  slider.position(310, 295);  
  slider.size(220);
}

async function mousePressed() {
  if (dist(mouseX, mouseY, 160, 175) > 55) return;
  const next = !ledOn;
  if (await serial.send(next ? 'ON' : 'OFF')) ledOn = next;
}

async function callbackFunction() {
  await serial.send(slider.value());
}

function receiveLine(line) {
  // Arduino가 보내는 한 줄을 화면에 반영합니다.
  const message = line.trim().toUpperCase();
  if (message === 'LED: ON' || message === 'ON') ledOn = true;
  else if (message === 'LED: OFF' || message === 'OFF') ledOn = false;
  else {
    const match = message.match(/^(?:POT\s*[:=]\s*)?(\d+)$/);
    if (match) potValue = constrain(Number(match[1]), 0, 1023);
  }
}
```

</details>

p5.js 전체 코드: [p5.js 에디터에서 보기](https://editor.p5js.org/POo0q/sketches/E-TcmSe1d)

<details>
<summary>Arduino 전체 코드 보기</summary>

```cpp
#include <TaskScheduler.h>
#include <LiquidCrystal_I2C.h>

// =====================================
// 1. Pin Settings
// =====================================
const uint8_t BUTTON_PIN = 2;
const uint8_t LED_PIN    = 13;
const uint8_t POT_PIN    = A0;

// LCD: I2C Address 0x27, 16 columns, 2 rows
LiquidCrystal_I2C lcd(0x27, 16, 2);

// =====================================
// 2. Global Variables
// =====================================
bool ledState = false;

// Last reported potentiometer value
int potValue = -1;

// Minimum potentiometer change
const int POT_THRESHOLD = 5;

// Interrupt variables
volatile bool buttonEvent = false;
volatile unsigned long lastInterruptTime = 0;

// Button debounce interval
const unsigned long DEBOUNCE_MS = 50;

// =====================================
// 3. Function Declarations
// =====================================
void buttonISR();
void taskButtonCallback();
void taskPotCallback();
void taskSerialCallback();
void taskLCDCallback();
void setLED(bool state);
// =====================================
// 4. TaskScheduler
// =====================================
Scheduler runner;

// Button event processing: 10 ms
Task tButton(
  10, TASK_FOREVER,
  &taskButtonCallback, &runner, true
);


// Potentiometer measurement: 100 ms
Task tPot(
  100, TASK_FOREVER,
  &taskPotCallback, &runner, true
);

// Serial command processing: 500 ms
Task tSerial(
  500, TASK_FOREVER,
  &taskSerialCallback, &runner, true
);

// LCD updating: 100 ms
Task tLCD(
  100, TASK_FOREVER,
  &taskLCDCallback, &runner, true
);

// =====================================
// 5. External Interrupt (D2)
// =====================================
void buttonISR() {
  unsigned long now = millis();

  if (now - lastInterruptTime >= DEBOUNCE_MS) {
    buttonEvent = true;
    lastInterruptTime = now;
  }
}

// =====================================
// 6. LED Control
// =====================================
void setLED(bool state) {

  // Do nothing if LED state is unchanged
  if (ledState == state) {
    return;
  }

  ledState = state;

  digitalWrite(
    LED_PIN,
    ledState ? HIGH : LOW
  );

  // Report LED status
  Serial.print("LED: ");
  Serial.println(ledState ? "ON" : "OFF");
}

// =====================================
// 7. Task 1: Button Event
// =====================================
void taskButtonCallback() {
  bool event = false;

  // Safely read and clear interrupt flag
  noInterrupts();

  if (buttonEvent) {
    event = true;
    buttonEvent = false;
  }

  interrupts();

  if (event) {
    setLED(!ledState);
  }
}

// =====================================
// 8. Task 2: Potentiometer
//    Measurement interval: 100 ms
//    Report only if change >= 3
// =====================================
void taskPotCallback() {

  int value = analogRead(POT_PIN);

  // First measurement OR change >= 3
  if (potValue == -1 ||
      abs(value - potValue) >= POT_THRESHOLD) {

    // Update last reported value
    potValue = value;

    // Report potentiometer value
    Serial.print("POT: ");
    Serial.println(potValue);
  }
}

// =====================================
// 9. Task 3: Serial Communication
//    Check interval: 500 ms
//    Commands: ON / OFF
// =====================================
void taskSerialCallback() {

  static char command[8];
  static uint8_t index = 0;
  static bool overflow = false;

  while (Serial.available() > 0) {

    char c = Serial.read();

    // End of command
    if (c == '\n' || c == '\r') {

      if (overflow) {
        Serial.println("Command too long");
      }
      else if (index > 0) {

        command[index] = '\0';

        if (strcmp(command, "ON") == 0) {
          setLED(true);
        }
        else if (strcmp(command, "OFF") == 0) {
          setLED(false);
        }
        else if(command[0]>='0'&&command[0]<='9'){
          analogWrite(10,atoi(command));
        }
        else {
          Serial.println("Invalid command");
        }
      }

      index = 0;
      overflow = false;
    }
    else if (!overflow) {

      // Convert lowercase to uppercase
      if (c >= 'a' && c <= 'z') {
        c -= ('a' - 'A');
      }

      if (index < sizeof(command) - 1) {
        command[index++] = c;
      }
      else {
        index = 0;
        overflow = true;
      }
    }
  }
}

// =====================================
// 10. Task 4: LCD Display
//     Update interval: 100 ms
// =====================================
void taskLCDCallback() {

  static bool previousLED = false;
  static int previousPOT = -1;
  static bool firstUpdate = true;

  // LCD Line 1: LED Status
  if (firstUpdate || ledState != previousLED) {

    lcd.setCursor(0, 0);

    if (ledState) {
      lcd.print("LED: ON         ");
    }
    else {
      lcd.print("LED: OFF        ");
    }

    previousLED = ledState;
  }

  // LCD Line 2: Potentiometer Value
  if (firstUpdate || potValue != previousPOT) {

    lcd.setCursor(0, 1);

    lcd.print("POT: ");
    if (potValue < 0) {
      lcd.print("----");
    }
    else {
      lcd.print(potValue);
    }
    lcd.print("          ");

    previousPOT = potValue;
  }

  firstUpdate = false;
}

// =====================================
// 11. Setup
// =====================================
void setup() {

  // Serial communication
  Serial.begin(9600);

  // Pin configuration
  pinMode(BUTTON_PIN, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
  pinMode(POT_PIN, INPUT);
  pinMode(10, OUTPUT);

  digitalWrite(LED_PIN, LOW);

  // LCD initialization
  lcd.init();
  lcd.backlight();
  lcd.clear();

  lcd.setCursor(0, 0);
  lcd.print("LED: OFF");

  lcd.setCursor(0, 1);
  lcd.print("POT: ----");

  // Initialize debounce timer
  lastInterruptTime = millis() - DEBOUNCE_MS;

  // External interrupt
  attachInterrupt(
    digitalPinToInterrupt(BUTTON_PIN),
    buttonISR,
    FALLING
  );

  // Startup messages
  Serial.println("System Ready");
  Serial.println("Commands: ON / OFF");
  Serial.println("LED: OFF");
}

// =====================================
// 12. Main Loop
// =====================================
void loop() {

  runner.execute();

}
```

</details>

---

## 동작 영상
https://github.com/user-attachments/assets/d495406a-6e24-4238-8a60-8243dfed2b6a
