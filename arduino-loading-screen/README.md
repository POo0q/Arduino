# 아두이노 LCD로 로딩 애니메이션 만들기

아두이노와 I2C LCD(16x2)를 이용해 동작 중임을 보여주는 로딩 애니메이션을 만들어 봤습니다.
LCD의 한 칸에 공 → 하트 → 별 → 불꽃놀이 순서로 모양이 바뀌며 계속 반복됩니다.

![로딩 애니메이션](images/loading.gif)

---

## 동작 순서

LCD 동작 순서는 아래와 같습니다.

| 순서 | 동작 | 시간 |
|---|---|---|
| 1 | 공이 점점 낮게 튀어 오름 | 약 1.9초 |
| 2 | 공이 하트로 변신 | 약 0.3초 |
| 3 | 하트가 두 번 튕기다가 별로 변신 | 약 2.1초 |
| 4 | 별이 위아래로 흔들리며 반짝임 | 약 2.8초 |
| 5 | 떨리다가 폭발하고 사라짐 (마지막 1.2초는 빈 화면) | 약 2.8초 |

한 바퀴는 약 10초이고, 끝나면 처음부터 다시 반복합니다.

---

## 회로

아두이노는 아래 사진처럼 연결했습니다.

<img width="1148" height="690" alt="회로 사진" src="https://github.com/user-attachments/assets/83517240-ed3e-44d8-a1b1-24b6f42107e7" />

I2C LCD의 GND, VCC, SDA, SCL을 각각 아두이노의 GND, 5V, A4, A5에 연결했습니다.

---

## 코드 설명

### 1. 라이브러리와 LCD 설정

```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);
```

를 이용해 I2C LCD를 사용합니다.
0x27은 LCD의 I2C 주소이고, 모듈에 따라 0x3F인 경우도 있습니다.

### 2. 표시할 위치

```cpp
const int COL = 7;
const int ROW = 0;
```

를 이용해 모양을 표시할 칸을 정합니다.
LCD는 16칸 2줄이라서 COL은 0~15, ROW는 0~1 중에서 고르면 됩니다.

### 3. 모양 만들기

```cpp
const byte B_TOP[8] = {0x0E, 0x1F, 0x1F, 0x0E, 0x00, 0x00, 0x00, 0x00};
```

LCD 한 칸은 가로 5픽셀, 세로 8픽셀입니다.
배열의 숫자 하나가 한 줄이고, 이진수로 바꿨을 때 1인 픽셀이 켜집니다.

| 줄 | 값 | 이진수 | 모양 |
|---|---|---|---|
| 1 | 0x0E | 01110 | ・■■■・ |
| 2 | 0x1F | 11111 | ■■■■■ |
| 3 | 0x1F | 11111 | ■■■■■ |
| 4 | 0x0E | 01110 | ・■■■・ |
| 5~8 | 0x00 | 00000 | ・・・・・ |

이렇게 위쪽에 둥근 공이 있는 모양이 됩니다.
같은 방법으로 공, 하트, 별, 불꽃놀이 모양을 총 30여 개 만들었습니다.

### 4. 모양 바꿔 보여주기

```cpp
void show(const byte *f, int ms) {
  lcd.createChar(0, (byte *)f);
  lcd.setCursor(COL, ROW);
  lcd.write((byte)0);
  delay(ms);
}
```

LCD에는 직접 만든 글자를 8개까지만 저장할 수 있습니다.
그래서 0번 슬롯 하나에 모양을 계속 덮어쓰고, 같은 칸에 다시 표시하는 방식으로 모양을 바꿉니다.
`createChar()`를 하면 커서 위치가 바뀌기 때문에 `setCursor()`로 다시 칸을 지정해야 합니다.
`show(모양, 시간)`을 순서대로 부르면 그 시간(ms)만큼 보여주고 다음 모양으로 넘어갑니다.

### 5. 공 튀기기

```cpp
show(B_TOP, 250);
show(B_F1, 120);
show(B_F2, 70);
show(B_LAND, 50);
show(B_SQUASH, 90);
```

를 이용해 공이 위에서 떨어져 바닥에 닿고 찌그러지는 모습을 만듭니다.
떨어질수록 시간을 짧게, 정점에서는 길게 해서 중력처럼 보이게 했습니다.
이 동작을 높이만 낮춰가며 세 번 반복합니다.

### 6. 모양 바꾸기 (공 → 하트 → 별)

```cpp
show(M_HEART, 130);
show(H2, 150);
...
show(DIAMOND, 140);
show(STAR1, 220);
```

를 이용해 중간 모양을 한 번 거쳐서 자연스럽게 변신하게 했습니다.
공에서 하트로 갈 때는 `M_HEART`, 하트에서 별로 갈 때는 `DIAMOND`(마름모)를 사용합니다.

### 7. 별 반짝이기

```cpp
for (int n = 0; n < 2; n++) {
  show(STAR1, 160);
  show(TW0, 160);
  show(STAR2, 160);
  show(TW1, 160);
  ...
}
```

를 이용해 별이 위아래로 흔들리면서 반짝이게 합니다.
반복문으로 같은 동작을 두 번 반복합니다.

### 8. 폭발

```cpp
for (int i = 0; i < 6; i++) {
  show(STAR1, 55);
  show(STAR2, 55);
}
show(STAR_SM, 170);
show(S1, 90);
...
show(BLANK, 1200);
```

를 이용해 별이 빠르게 떨리다가 작게 줄어든 뒤 터지는 모습을 만듭니다.
`S1`~`S4`로 퍼지고, `F1`~`F3`으로 불씨가 떨어지며 사라지고, `BLANK`로 빈 화면을 잠깐 보여줍니다.

---

## 전체 코드

```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// PCF8574(NXP)는 0x27, PCF8574A(TI)는 0x3F
LiquidCrystal_I2C lcd(0x27, 16, 2);

const int COL = 7;   // 표시할 칸 (0~15)
const int ROW = 0;   // 표시할 줄 (0~1)

// ---------- 공 ----------
const byte B_TOP[8]    = {0x0E, 0x1F, 0x1F, 0x0E, 0x00, 0x00, 0x00, 0x00};
const byte B_F1[8]     = {0x00, 0x0E, 0x0E, 0x0E, 0x0E, 0x00, 0x00, 0x00};
const byte B_F2[8]     = {0x00, 0x00, 0x00, 0x0E, 0x0E, 0x0E, 0x0E, 0x00};
const byte B_LAND[8]   = {0x00, 0x00, 0x00, 0x00, 0x0E, 0x1F, 0x1F, 0x0E};
const byte B_SQUASH[8] = {0x00, 0x00, 0x00, 0x00, 0x00, 0x0E, 0x1F, 0x1F};
const byte B_MID[8]    = {0x00, 0x00, 0x0E, 0x1F, 0x1F, 0x0E, 0x00, 0x00};
const byte B_LOW[8]    = {0x00, 0x00, 0x00, 0x0E, 0x1F, 0x1F, 0x0E, 0x00};

// ---------- 하트 (H0이 가장 높고 H3이 가장 낮음) ----------
const byte M_HEART[8]  = {0x00, 0x00, 0x1B, 0x1F, 0x1F, 0x0E, 0x04, 0x00};  // 공 -> 하트 변신
const byte H0[8]       = {0x0A, 0x1F, 0x1F, 0x0E, 0x04, 0x00, 0x00, 0x00};
const byte H1[8]       = {0x00, 0x0A, 0x1F, 0x1F, 0x0E, 0x04, 0x00, 0x00};
const byte H2[8]       = {0x00, 0x00, 0x0A, 0x1F, 0x1F, 0x0E, 0x04, 0x00};
const byte H3[8]       = {0x00, 0x00, 0x00, 0x0A, 0x1F, 0x1F, 0x0E, 0x04};
const byte HEART_SQ[8] = {0x00, 0x00, 0x00, 0x00, 0x00, 0x1B, 0x1F, 0x0E};  // 찌그러진 하트

// ---------- 별 (STAR0이 가장 높고 STAR2가 가장 낮음) ----------
const byte DIAMOND[8]  = {0x00, 0x04, 0x0E, 0x1F, 0x0E, 0x04, 0x00, 0x00};  // 하트 -> 별 중간
const byte STAR0[8]    = {0x04, 0x1F, 0x0E, 0x0A, 0x11, 0x00, 0x00, 0x00};
const byte STAR1[8]    = {0x00, 0x04, 0x1F, 0x0E, 0x0A, 0x11, 0x00, 0x00};
const byte STAR2[8]    = {0x00, 0x00, 0x04, 0x1F, 0x0E, 0x0A, 0x11, 0x00};
const byte TW0[8]      = {0x04, 0x15, 0x0E, 0x1F, 0x0E, 0x15, 0x04, 0x00};  // 반짝이는 별 (높음)
const byte TW1[8]      = {0x00, 0x04, 0x15, 0x0E, 0x1F, 0x0E, 0x15, 0x04};  // 반짝이는 별 (낮음)
const byte STAR_SM[8]  = {0x00, 0x00, 0x04, 0x0E, 0x04, 0x00, 0x00, 0x00};  // 터지기 전 응축

// ---------- 불꽃놀이 ----------
const byte S1[8] = {0x00, 0x00, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00};
const byte S2[8] = {0x00, 0x00, 0x04, 0x0A, 0x04, 0x00, 0x00, 0x00};
const byte S3[8] = {0x00, 0x04, 0x00, 0x15, 0x00, 0x04, 0x00, 0x00};
const byte S4[8] = {0x04, 0x0A, 0x00, 0x11, 0x11, 0x00, 0x0A, 0x04};
const byte F1[8] = {0x00, 0x00, 0x0A, 0x00, 0x00, 0x11, 0x00, 0x04};
const byte F2[8] = {0x00, 0x00, 0x00, 0x0A, 0x00, 0x00, 0x04, 0x11};
const byte F3[8] = {0x00, 0x00, 0x00, 0x00, 0x00, 0x0A, 0x00, 0x04};
const byte BLANK[8] = {0, 0, 0, 0, 0, 0, 0, 0};

// 같은 슬롯(0번)에 모양을 덮어쓰고 같은 칸에 다시 표시
void show(const byte *f, int ms) {
  lcd.createChar(0, (byte *)f);
  lcd.setCursor(COL, ROW);
  lcd.write((byte)0);
  delay(ms);
}

void setup() {
  lcd.init();
  lcd.backlight();
  lcd.clear();
}

void loop() {
  // ===== 1) 공 튀기기 (점점 낮게) =====
  show(B_TOP, 250);
  show(B_F1, 120);
  show(B_F2, 70);
  show(B_LAND, 50);
  show(B_SQUASH, 90);
  show(B_LAND, 50);
  show(B_F2, 60);
  show(B_F1, 90);

  show(B_MID, 220);
  show(B_F2, 80);
  show(B_LAND, 50);
  show(B_SQUASH, 80);
  show(B_LAND, 50);
  show(B_F2, 70);

  show(B_LOW, 170);
  show(B_LAND, 50);
  show(B_SQUASH, 70);
  show(B_LAND, 50);
  show(B_LOW, 110);
  show(B_MID, 150);

  // ===== 2) 공 -> 하트 변신 =====
  show(M_HEART, 130);
  show(H2, 150);

  // ===== 3) 하트 튕기기 =====
  // 낙하 -> 착지
  show(H3, 70);
  show(HEART_SQ, 90);       // 찌그러짐
  show(H3, 50);
  // 높이 튀어오름
  show(H2, 70);
  show(H1, 110);
  show(H0, 260);            // 정점
  show(H1, 110);
  show(H2, 70);
  show(H3, 50);
  show(HEART_SQ, 80);
  show(H3, 50);
  // 낮게 한 번 더
  show(H2, 80);
  show(H1, 200);            // 정점
  show(H2, 90);
  show(H3, 50);
  show(HEART_SQ, 70);
  // 마지막에 떠오르며 별로 변신
  show(H3, 60);
  show(H2, 80);
  show(H1, 130);
  show(DIAMOND, 140);       // 하트 -> 마름모
  show(STAR1, 220);         // 별 완성

  // ===== 4) 별이 둥실 떠 있기 (위아래로 흔들 + 반짝) =====
  for (int n = 0; n < 2; n++) {
    show(STAR1, 160);
    show(TW0, 160);
    show(STAR2, 160);
    show(TW1, 160);
    show(STAR2, 160);
    show(TW1, 160);
    show(STAR1, 160);
    show(TW0, 160);
  }
  show(STAR1, 200);

  // ===== 5) 터지기 직전 떨림 -> 폭발 =====
  for (int i = 0; i < 6; i++) {          // 부르르
    show(STAR1, 55);
    show(STAR2, 55);
  }
  show(STAR_SM, 170);                    // 응축
  show(S1, 90);                          // 번쩍
  show(S2, 70);
  show(S3, 70);
  show(S4, 90);                          // 최대로 퍼짐
  show(F1, 110);                         // 불씨가 떨어지며 사라짐
  show(F2, 130);
  show(F3, 160);
  show(BLANK, 1200);

  delay(500);
}
```

---

## 동작 영상

[![동작 영상](images/video_thumbnail.png)](동작_영상_링크)
