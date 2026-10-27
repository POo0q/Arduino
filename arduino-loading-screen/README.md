# 아두이노 LCD로 로딩 애니메이션 만들기

아두이노와 I2C LCD(16x2)를 이용해 동작 중임을 보여주는 로딩 애니메이션을 만들어 봤습니다.
LCD의 한 칸에 공이 튀어오릅니다.

---

## 회로

아두이노는 아래 사진처럼 연결했습니다.

<img width="1148" height="690" alt="회로 사진" src="https://github.com/user-attachments/assets/83517240-ed3e-44d8-a1b1-24b6f42107e7" />

I2C LCD의 GND, VCC, SDA, SCL을 각각 아두이노의 GND, 5V, A4, A5에 연결했습니다.

---

## 코드 설명

### 1. 라이브러리와 LCD 설정

```c
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);
```

를 이용해 I2C LCD를 사용합니다.
0x27은 LCD의 I2C 주소이입니다

### 2.모양 만들기

```c
byte c1[8] = { 0x0E, 0x1F, 0x1F, 0x0E, 0x00, 0x00, 0x00, 0x00 };
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
같은 방법으로 공 모양을 총 5 개 만들었습니다.

### 3. 모양 바꿔 보여주기

```c
void show(byte *f, int ms) {
  lcd.createChar(0, f);
  lcd.setCursor(0, 0);
  lcd.write(0);
  delay(ms);
}
```

0번 슬롯 하나에 모양을 계속 덮어쓰고, 같은 칸에 다시 표시하는 방식으로 모양을 바꿉니다.
`createChar()`를 하면 커서 위치가 바뀌기 때문에 `setCursor()`로 다시 칸을 지정해야 합니다.
`show(모양, 시간)`을 순서대로 부르면 그 시간(ms)만큼 보여주고 다음 모양으로 넘어갑니다.

### 4. 공 튀기기

```c
void loop() {
  show(c1, 250);
  show(c2, 120);
  show(c3, 70);
  show(c4, 50);
  show(c5, 90);
  show(c4, 50);
  show(c3, 60);
  show(c2, 90);
}
```

를 이용해 공이 위에서 떨어져 바닥에 닿고 찌그러지는 모습을 만듭니다.
떨어질수록 시간을 짧게, 정점에서는 길게 해서 중력처럼 보이게 했습니다.
이 동작을 계속 반복합니다.


---

## 전체 코드

```c
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

byte c1[8] = { 0x0E, 0x1F, 0x1F, 0x0E, 0x00, 0x00, 0x00, 0x00 };
byte c2[8] = { 0x00, 0x0E, 0x0E, 0x0E, 0x0E, 0x00, 0x00, 0x00 };
byte c3[8] = { 0x00, 0x00, 0x00, 0x0E, 0x0E, 0x0E, 0x0E, 0x00 };
byte c4[8] = { 0x00, 0x00, 0x00, 0x00, 0x0E, 0x1F, 0x1F, 0x0E };
byte c5[8] = { 0x00, 0x00, 0x00, 0x00, 0x00, 0x0E, 0x1F, 0x1F };

void show(byte *f, int ms) {
  lcd.createChar(0, f);
  lcd.setCursor(0, 0);
  lcd.write(0);
  delay(ms);
}

void setup() {
  lcd.init();
  lcd.backlight();
  lcd.clear();
}

void loop() {
  show(c1, 250);
  show(c2, 120);
  show(c3, 70);
  show(c4, 50);
  show(c5, 90);
  show(c4, 50);
  show(c3, 60);
  show(c2, 90);
}
```

---

## 동작 영상

[![동작 영상]https://github.com/user-attachments/assets/88892b00-ca61-425a-9c7a-acafdb1256f3


