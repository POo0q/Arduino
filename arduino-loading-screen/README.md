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


https://github.com/user-attachments/assets/413f9699-aed0-4588-98d9-25e0556aaff1


---

## 추가

ai를 이용해 추가 버전도 만들어 보았습니다.

<details>
<summary>전체 코드 보기</summary>

```c
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <math.h>

// PCF8574(NXP)는 0x27, PCF8574A(TI)는 0x3F
LiquidCrystal_I2C lcd(0x27, 16, 2);

// 화면 전체를 80x16 픽셀 캔버스로 쓰는 방식
//  - 매 프레임 캔버스에 도형을 그리고
//  - 칸(5x8)별 모양을 모아서 같은 모양은 합치고
//  - CGRAM 8슬롯에 배정해서 LCD에 표시한다.

const int N = 40;                       // 도형 꼭짓점 수
const unsigned long FRAME_MS = 66;      // 프레임 간격 (약 15fps)

byte tile[2][16][8];                    // 캔버스 (칸 단위)
byte shown[2][16];                      // 지금 LCD에 표시 중인 문자 코드
byte slotPat[8][8];                     // 지금 CGRAM에 들어 있는 모양
int8_t shp[3][N][2];                    // 0=원 1=하트 2=별 (좌표 x100)
unsigned long lastT = 0;

// ===================== 캔버스 =====================
void px(int x, int y) {
  if (x < 0 || x >= 80 || y < 0 || y >= 16) return;
  tile[y >> 3][x / 5][y & 7] |= 1 << (4 - (x % 5));
}

void pxf(float x, float y) {
  px((int)floor(x + 0.5), (int)floor(y + 0.5));
}

void spark(int cx, int cy) {            // + 모양 반짝임
  px(cx, cy); px(cx - 1, cy); px(cx + 1, cy); px(cx, cy - 1); px(cx, cy + 1);
}

void fillPoly(float *xs, float *ys, int n) {
  float ymin = 1e9, ymax = -1e9;
  for (int i = 0; i < n; i++) {
    if (ys[i] < ymin) ymin = ys[i];
    if (ys[i] > ymax) ymax = ys[i];
  }
  int y0 = (int)ceil(ymin); if (y0 < 0) y0 = 0;
  int y1 = (int)floor(ymax); if (y1 > 15) y1 = 15;
  for (int y = y0; y <= y1; y++) {
    float xi[16];
    int k = 0;
    for (int i = 0; i < n; i++) {
      int j = (i + 1) % n;
      float ya = ys[i], yb = ys[j];
      if ((ya <= y && yb > y) || (yb <= y && ya > y)) {
        if (k < 16) xi[k++] = xs[i] + (y - ya) * (xs[j] - xs[i]) / (yb - ya);
      }
    }
    for (int a = 1; a < k; a++) {       // 작은 정렬
      float v = xi[a];
      int b = a - 1;
      while (b >= 0 && xi[b] > v) { xi[b + 1] = xi[b]; b--; }
      xi[b + 1] = v;
    }
    for (int p = 0; p + 1 < k; p += 2) {
      int xa = (int)ceil(xi[p] - 0.3);
      int xb = (int)floor(xi[p + 1] + 0.3);
      for (int x = xa; x <= xb; x++) px(x, y);
    }
  }
}

// 도형 a -> b 로 t만큼 변형. s = 반지름(px), sx/sy = 가로/세로 늘이기
void drawShape(int a, int b, float t, float cx, float cy, float s, float sx, float sy) {
  if (s * (sx > sy ? sx : sy) < 1.6) { pxf(cx, cy); return; }
  float xs[N], ys[N];
  for (int i = 0; i < N; i++) {
    float ux = shp[a][i][0] + (shp[b][i][0] - shp[a][i][0]) * t;
    float uy = shp[a][i][1] + (shp[b][i][1] - shp[a][i][1]) * t;
    xs[i] = cx + s * sx * ux / 100.0;
    ys[i] = cy + s * sy * uy / 100.0;
  }
  fillPoly(xs, ys, N);
}

void buildShapes() {
  const float DEG = 57.29578;
  for (int k = 0; k < 3; k++) {
    float gx[N], gy[N];
    float minx = 9, maxx = -9, miny = 9, maxy = -9;
    for (int i = 0; i < N; i++) {
      float th = i * 6.2831853 / N;     // 9도 간격
      float r = 1;
      if (k == 1) {                     // 하트 (끝이 아래쪽)
        float sn = sin(th), cs = cos(th);
        r = 2 - 2 * sn + sn * sqrt(fabs(cs)) / (sn + 1.4);
      } else if (k == 2) {              // 별 (뾰족한 끝이 위쪽)
        float t = fmod(th * DEG - 90 + 360, 72);
        if (t > 36) t = 72 - t;
        float ph = t / DEG, dl = 36 / DEG;
        r = 0.42 * sin(dl) / (sin(ph) + 0.42 * sin(dl - ph));
      }
      gx[i] = r * cos(th);
      gy[i] = r * sin(th);
      if (gx[i] < minx) minx = gx[i];
      if (gx[i] > maxx) maxx = gx[i];
      if (gy[i] < miny) miny = gy[i];
      if (gy[i] > maxy) maxy = gy[i];
    }
    float cx = (minx + maxx) / 2, cy = (miny + maxy) / 2;
    float w = maxx - minx, h = maxy - miny;
    float s = (w > h ? w : h) / 2;
    for (int i = 0; i < N; i++) {
      shp[k][i][0] = (int8_t)floor((gx[i] - cx) / s * 100 + 0.5);
      shp[k][i][1] = (int8_t)floor(-(gy[i] - cy) / s * 100 + 0.5);
    }
  }
}

// ===================== 화면 출력 =====================
int ham(const byte *a, const byte *b) {
  int d = 0;
  for (int i = 0; i < 8; i++) {
    byte x = a[i] ^ b[i];
    while (x) { d += x & 1; x >>= 1; }
  }
  return d;
}

void present() {
  byte pats[32][8];
  byte cnt[32];
  bool alive[32];
  int8_t idx[32];
  int np = 0;

  // 1) 칸마다 모양을 모으고 같은 모양은 하나로
  for (int r = 0; r < 2; r++) {
    for (int c = 0; c < 16; c++) {
      int ci = r * 16 + c;
      byte any = 0;
      for (int k = 0; k < 8; k++) any |= tile[r][c][k];
      if (!any) { idx[ci] = -1; continue; }
      int f = -1;
      for (int p = 0; p < np; p++) {
        if (memcmp(pats[p], tile[r][c], 8) == 0) { f = p; break; }
      }
      if (f < 0) {
        memcpy(pats[np], tile[r][c], 8);
        cnt[np] = 0;
        alive[np] = true;
        f = np++;
      }
      cnt[f]++;
      idx[ci] = f;
    }
  }

  // 2) 8개를 넘으면 가장 적게 쓰인 모양을 가장 비슷한 모양에 합침
  int na = np;
  while (na > 8) {
    int a = -1;
    for (int p = 0; p < np; p++) {
      if (alive[p] && (a < 0 || cnt[p] < cnt[a])) a = p;
    }
    int b = -1, bd = 99;
    for (int p = 0; p < np; p++) {
      if (!alive[p] || p == a) continue;
      int d = ham(pats[a], pats[p]);
      if (b < 0 || d < bd || (d == bd && cnt[p] > cnt[b])) { bd = d; b = p; }
    }
    alive[a] = false;
    na--;
    int to = (bd <= 6) ? b : -1;        // 너무 다르면 그냥 지움
    for (int ci = 0; ci < 32; ci++) if (idx[ci] == a) idx[ci] = to;
    if (to >= 0) cnt[to] += cnt[a];
  }

  // 3) CGRAM 슬롯 배정 (같은 모양이 이미 있으면 재사용)
  int8_t slotOf[32];
  bool taken[8];
  for (int s = 0; s < 8; s++) taken[s] = false;
  for (int p = 0; p < np; p++) slotOf[p] = -1;
  for (int p = 0; p < np; p++) {
    if (!alive[p]) continue;
    for (int s = 0; s < 8; s++) {
      if (!taken[s] && memcmp(slotPat[s], pats[p], 8) == 0) {
        slotOf[p] = s; taken[s] = true; break;
      }
    }
  }
  for (int p = 0; p < np; p++) {
    if (!alive[p] || slotOf[p] >= 0) continue;
    for (int s = 0; s < 8; s++) {
      if (!taken[s]) {
        memcpy(slotPat[s], pats[p], 8);
        lcd.createChar(s, slotPat[s]);
        slotOf[p] = s; taken[s] = true; break;
      }
    }
  }

  // 4) 바뀐 칸만 LCD에 쓰기
  for (int r = 0; r < 2; r++) {
    for (int c = 0; c < 16; c++) {
      int ci = r * 16 + c;
      byte code = (idx[ci] < 0) ? ' ' : (byte)slotOf[idx[ci]];
      if (code != shown[r][c]) {
        lcd.setCursor(c, r);
        lcd.write(code);
        shown[r][c] = code;
      }
    }
  }
}

// 지금 캔버스를 화면에 내보내고 다음 프레임 시간까지 기다림
void flip() {
  present();
  while (millis() - lastT < FRAME_MS) { }
  lastT = millis();
  memset(tile, 0, sizeof(tile));
}

float easeOut(float t) { return 1 - (1 - t) * (1 - t); }
float easeIn(float t) { return t * t; }
float smooth(float t) { return t * t * (3 - 2 * t); }
float lerpf(float a, float b, float t) { return a + (b - a) * t; }

// ===================== 장면 =====================
// 1) 반짝임 -> 공이 떨어져 튕기며 굴러옴
void sceneBall() {
  for (int i = 0; i < 6; i++) flip();
  px(5, 3); flip();
  spark(7, 3); flip();
  px(7, 3); flip();

  const float R = 3.3;
  float bx = 5, by = 3, vy = 0;
  bool done = false;
  while (!done) {
    vy += 0.5; by += vy; bx += 1.2;
    float sx = 1, sy = 1, cy = by;
    if (by >= 12) {                     // 바닥에 닿음: 납작하게
      by = 12;
      float sq = vy * 0.09;
      if (sq > 0.4) sq = 0.4;
      sy = 1 - sq; sx = 1 + sq * 0.9;
      cy = 12 + R * (1 - sy);
      if (vy < 1.6) done = true; else vy = -vy * 0.72;
    } else {                            // 공중: 속도에 따라 길쭉하게
      sy = 1 + fabs(vy) * 0.05;
      if (sy > 1.25) sy = 1.25;
      sx = 1 / sy;
    }
    drawShape(0, 0, 0, bx, cy, R, sx, sy);
    flip();
  }
  float x0 = bx;
  for (int i = 1; i <= 10; i++) {       // 가운데로 굴러감
    drawShape(0, 0, 0, lerpf(x0, 40, easeOut(i / 10.0)), 12, R, 1, 1);
    flip();
  }
}

// 2) 웅크렸다 뛰어오르며 하트로 변신
void sceneMorph() {
  const float R = 3.3;
  const float SQ[3] = {0.15, 0.25, 0.2};
  for (int i = 0; i < 3; i++) {
    float sy = 1 - SQ[i], sx = 1 + SQ[i] * 0.9;
    drawShape(0, 0, 0, 40, 12 + R * (1 - sy), R, sx, sy);
    flip();
  }
  for (int i = 0; i <= 12; i++) {
    float t = i / 12.0, e = easeOut(t);
    float sy = 1 + 0.2 * (1 - t), sx = 1 / sy;
    drawShape(0, 1, smooth(t), 40, lerpf(12, 8, e), lerpf(R, 5.4, e), sx, sy);
    flip();
  }
  const float POP[4] = {6.2, 5.8, 5.55, 5.4};
  for (int i = 0; i < 4; i++) {
    drawShape(1, 1, 0, 40, 8, POP[i], 1, 1);
    flip();
  }
}

// 3) 두근두근
void sceneBeat() {
  const byte BEAT[14] = {0, 40, 100, 60, 20, 50, 80, 40, 10, 0, 0, 0, 0, 0};
  for (int b = 0; b < 3; b++) {
    for (int i = 0; i < 14; i++) {
      float s = 5.4 * (1 + 0.16 * BEAT[i] / 100.0);
      drawShape(1, 1, 0, 40, 8, s, 1, 1);
      flip();
    }
  }
}

// 4) 하트가 내려앉아 튕기다가 별로 변신하며 떠오름
void sceneHops() {
  const float S = 4.0, FLOOR = 15.3;
  for (int i = 0; i <= 6; i++) {        // 작아지며 바닥으로
    float t = i / 6.0;
    float s = lerpf(5.4, S, t);
    float sy = 1 + 0.15 * t, sx = 1 / sy;
    drawShape(1, 1, 0, 40, lerpf(8, FLOOR - S * sy, easeIn(t)), s, sx, sy);
    flip();
  }
  for (int h = 0; h < 2; h++) {
    const float SQ[2] = {0.3, 0.15};
    for (int i = 0; i < 2; i++) {       // 착지 찌그러짐
      float sy = 1 - SQ[i], sx = 1 + SQ[i] * 0.9;
      drawShape(1, 1, 0, 40, FLOOR - S * sy, S, sx, sy);
      flip();
    }
    if (h == 0) {                       // 첫 번째 튕김
      float cy = FLOOR - S, vy = -2.6;
      for (;;) {
        cy += vy; vy += 0.45;
        if (cy >= FLOOR - S) break;
        float sy = 1 + fabs(vy) * 0.06;
        if (sy > 1.2) sy = 1.2;
        drawShape(1, 1, 0, 40, cy, S, 1 / sy, sy);
        flip();
      }
    }
  }
  for (int i = 0; i <= 9; i++) {        // 두 번째 튕김: 별로 변신하며 위로
    float t = i / 9.0, e = easeOut(t);
    float sy = 1 + 0.2 * (1 - t), sx = 1 / sy;
    drawShape(1, 2, smooth(t), 40, lerpf(FLOOR - S, 6, e), lerpf(S, 5.2, e), sx, sy);
    flip();
  }
  for (int i = 0; i <= 5; i++) {        // 살짝 내려와 자리잡음
    float t = smooth(i / 5.0);
    drawShape(2, 2, 0, 40, lerpf(6, 7.5, t), 5.2, 1, 1);
    flip();
  }
}

// 5) 별이 둥실 떠서 반짝
void sceneFloat() {
  for (int i = 0; i < 40; i++) {
    float cy = 7.5 + sin(i * 0.25) * 1.0;
    float s = 5.2 + 0.4 * sin(i * 0.6);
    drawShape(2, 2, 0, 40, cy, s, 1, 1);
    if (i % 12 < 6) spark(27, 3); else spark(52, 11);
    flip();
  }
}

// 6) 부르르 떨다가 응축 -> 폭발 -> 불꽃놀이
void sceneBurst() {
  for (int i = 0; i < 10; i++) {
    float dx = (i % 2 ? 1 : -1) * (0.6 + i * 0.12);
    drawShape(2, 2, 0, 40 + dx, 7.5, 5.2 - i * 0.1, 1, 1);
    flip();
  }
  const float SH[4] = {4.0, 2.8, 1.8, 1.0};
  for (int i = 0; i < 4; i++) {
    drawShape(2, 2, 0, 40, 7.5, SH[i], 1, 1);
    flip();
  }
  drawShape(0, 0, 0, 40, 7.5, 6.5, 1, 1);   // 번쩍
  flip();

  float x[20], y[20], vx[20], vy[20];
  byte life[20];
  for (int i = 0; i < 20; i++) life[i] = 0;
  for (int i = 0; i < 10; i++) {
    float a = i * 0.6283 + random(-12, 12) / 100.0;
    float sp = random(18, 36) / 10.0;
    x[i] = 40; y[i] = 7.5;
    vx[i] = cos(a) * sp * 1.6;
    vy[i] = sin(a) * sp * 0.45;
    life[i] = random(22, 32);
  }
  for (int f = 0; f < 46; f++) {
    if (f == 10) {                      // 2차 폭발(자글자글)
      for (int i = 0; i < 10; i++) {
        int c = 10 + i;
        float a = random(0, 628) / 100.0;
        float sp = random(8, 16) / 10.0;
        x[c] = x[i]; y[c] = y[i];
        vx[c] = cos(a) * sp * 1.4;
        vy[c] = sin(a) * sp * 0.5;
        life[c] = 14;
      }
    }
    for (int i = 0; i < 20; i++) {
      if (life[i] == 0) continue;
      if (life[i] > 7 || life[i] % 2 == 0) {     // 사라지기 전 깜빡임
        pxf(x[i], y[i]);
        if (f < 14 && i < 10) pxf(x[i] - vx[i] * 0.7, y[i] - vy[i] * 0.7);
      }
      x[i] += vx[i];
      y[i] += vy[i];
      vx[i] *= 0.93;
      vy[i] = vy[i] * 0.93 + 0.06;
      life[i]--;
    }
    flip();
  }
}

void setup() {
  lcd.init();
  lcd.backlight();
  Wire.setClock(400000);               // 통신을 빠르게 (프레임 속도 향상)
  buildShapes();
  randomSeed(analogRead(A0));
  lastT = millis();
}

void loop() {
  lcd.clear();
  memset(shown, ' ', sizeof(shown));
  memset(tile, 0, sizeof(tile));

  sceneBall();
  sceneMorph();
  sceneBeat();
  sceneHops();
  sceneFloat();
  sceneBurst();
  for (int i = 0; i < 14; i++) flip();  // 잠깐 정적 후 처음부터 반복
}
```

</details>

##  AI버전 동작 영상
https://github.com/user-attachments/assets/318cc78b-3084-4f10-8150-735537419909









