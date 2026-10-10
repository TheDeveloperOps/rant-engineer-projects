# Tilt Racer

A simple car-dodging game for a 128x64 OLED, steered by tilting an MPU6050 accelerometer. Tilt left and right to steer, avoid the oncoming cars, and survive as the road speeds up.

It grew out of an earlier experiment, **Tilt Balls**, where dots roll around the OLED like marbles on a tray. Tilt Racer reuses the same sensor setup, axis calibration and fast display code.

## Features

- Tilt steering with a low-pass filter and dead zone, so the car goes straight when the board is flat
- Scrolling road with lane dividers and rumble strips
- Oncoming traffic that spawns at random positions
- Score counter and best score (kept in RAM until the board restarts)
- Difficulty ramps up as your score increases
- Game over screen; tilt hard to either side to restart
- Time-based physics, so motion stays consistent regardless of frame rate
- Fast I2C display updates (800 kHz) for smoother animation

## Hardware

| Part | Notes |
|------|-------|
| Microcontroller | ESP32-style board with I2C on GPIO4 / GPIO5 |
| OLED display | SSD1306, 128x64, I2C address `0x3C` |
| Accelerometer | MPU6050, I2C address `0x68` |

### Wiring

Both devices share one I2C bus.

| OLED | MPU6050 | Board |
|------|---------|-------|
| GND | GND | GND |
| VCC | VCC | 3V3 |
| SCL | SCL | GPIO5 |
| SDA | SDA | GPIO4 |

## Libraries

Install from the Arduino Library Manager:

- Adafruit GFX Library
- Adafruit SSD1306

The MPU6050 is read directly over I2C registers, so no MPU6050 library is needed.

## Full code (`tilt_racer.ino`)

Create a new sketch in the Arduino IDE, delete the default code, paste everything below, and save it as `tilt_racer`.

```cpp
/*
  Tilt Racer — steer a car by tilting the board.

  WIRING (same as before):
    OLED  GND->GND, VCC->3V3, SCL->GPIO5, SDA->GPIO4
    MPU6050 GND->GND, VCC->3V3, SCL->GPIO5, SDA->GPIO4 (shared bus)

  LIBRARIES:
    - Adafruit GFX Library
    - Adafruit SSD1306

  HOW TO PLAY:
    Tilt left / right to steer. Avoid the oncoming cars.
    Score = cars passed. After a crash, tilt the board hard to either side to retry.
*/

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SDA_PIN 4
#define SCL_PIN 5
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C
#define MPU_ADDR 0x68

// ---- Direction calibration (same values that worked for you) ----
#define SWAP_XY   true
#define INVERT_X  true

// ---- I2C speed (lower 800000 to 400000 if the screen shows garbage) ----
#define I2C_DISPLAY_CLOCK 800000UL
#define I2C_BUS_CLOCK     400000UL

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1,
                         I2C_DISPLAY_CLOCK, I2C_BUS_CLOCK);

// ---------- MPU6050 ----------
void mpuWriteReg(uint8_t reg, uint8_t value) {
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(reg);
  Wire.write(value);
  Wire.endTransmission();
}

bool mpuInit() {
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x75); // WHO_AM_I
  if (Wire.endTransmission(false) != 0) return false;
  Wire.requestFrom((int)MPU_ADDR, 1);
  if (Wire.available() < 1) return false;
  uint8_t whoami = Wire.read();
  if (whoami != 0x68 && whoami != 0x70 && whoami != 0x98 && whoami != 0x72) return false;

  mpuWriteReg(0x6B, 0x00); // wake up
  delay(100);
  mpuWriteReg(0x1A, 0x03); // built-in low-pass filter
  mpuWriteReg(0x1C, 0x00); // +/- 2g
  return true;
}

bool mpuReadAccel(float &ax, float &ay) {
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x3B);
  if (Wire.endTransmission(false) != 0) return false;
  if (Wire.requestFrom((int)MPU_ADDR, 4) != 4) return false;

  int16_t rawX = (Wire.read() << 8) | Wire.read();
  int16_t rawY = (Wire.read() << 8) | Wire.read();
  ax = rawX / 16384.0f;
  ay = rawY / 16384.0f;
  return true;
}

// ---------- Game layout ----------
const int ROAD_L = 24;            // road left edge (px)
const int ROAD_R = 104;           // road right edge (px)
const int CAR_W  = 10;
const int CAR_H  = 14;
const int PLAYER_Y = 47;          // player's fixed row

// ---------- Tuning ----------
const float STEER_GAIN  = 400.0f; // px/s of sideways speed per 1 g of tilt
const float MAX_STEER   = 170.0f; // px/s cap on sideways speed
const float TILT_FILTER = 0.35f;  // lower = smoother but laggier steering
const float DEAD_ZONE   = 0.04f;  // g, ignore tiny tilt so the car goes straight when flat
const float BASE_SPEED  = 50.0f;  // starting road speed (px/s)
const float MAX_SPEED   = 135.0f; // top road speed (px/s)
const float SPEED_PER_POINT = 2.5f;
const float RESTART_TILT = 0.5f;  // g, tilt this hard (after a crash) to retry
const uint32_t RESTART_LOCKOUT_MS = 1500;

const int MAX_OBS = 4;

// ---------- Game state ----------
float playerX;
float obsX[MAX_OBS];
float obsY[MAX_OBS];
bool  obsActive[MAX_OBS];
bool  obsPassed[MAX_OBS];

int   score = 0;
int   bestScore = 0;
bool  gameOver = false;
float roadSpeed = BASE_SPEED;
float scrollPos = 0;
float distSinceSpawn = 0;
float nextGap = 40;
float filtGx = 0;
uint32_t lastMicros = 0;
uint32_t gameOverAt = 0;

void resetGame() {
  playerX = (ROAD_L + ROAD_R) / 2.0f - CAR_W / 2.0f;
  for (int i = 0; i < MAX_OBS; i++) {
    obsActive[i] = false;
    obsPassed[i] = false;
    obsY[i] = -CAR_H;
  }
  score = 0;
  roadSpeed = BASE_SPEED;
  scrollPos = 0;
  distSinceSpawn = 0;
  nextGap = 40;
  gameOver = false;
}

void spawnObstacle() {
  for (int i = 0; i < MAX_OBS; i++) {
    if (!obsActive[i]) {
      obsActive[i] = true;
      obsPassed[i] = false;
      obsY[i] = -CAR_H;
      obsX[i] = random(ROAD_L + 2, ROAD_R - CAR_W - 1);
      return;
    }
  }
}

// ---------- Drawing ----------
// player = true: filled car with windshield at the top.
// player = false: outlined oncoming car, windshield at the bottom.
void drawCar(int x, int y, bool player) {
  display.fillRect(x - 1, y - 1, CAR_W + 2, CAR_H + 2, SSD1306_BLACK); // clear border

  // wheels
  display.fillRect(x,     y + 2, 2, 3, SSD1306_WHITE);
  display.fillRect(x + 8, y + 2, 2, 3, SSD1306_WHITE);
  display.fillRect(x,     y + 9, 2, 3, SSD1306_WHITE);
  display.fillRect(x + 8, y + 9, 2, 3, SSD1306_WHITE);

  if (player) {
    display.fillRoundRect(x + 1, y, 8, CAR_H, 2, SSD1306_WHITE);
    display.fillRect(x + 2, y + 3, 6, 3, SSD1306_BLACK);   // windshield
    display.fillRect(x + 3, y + 10, 4, 2, SSD1306_BLACK);  // rear window
  } else {
    display.drawRoundRect(x + 1, y, 8, CAR_H, 2, SSD1306_WHITE);
    display.fillRect(x + 3, y + 8, 4, 3, SSD1306_WHITE);   // windshield (faces down)
  }
}

void drawRoad() {
  // Curbs (solid edge lines)
  display.drawFastVLine(ROAD_L - 1, 0, SCREEN_HEIGHT, SSD1306_WHITE);
  display.drawFastVLine(ROAD_R,     0, SCREEN_HEIGHT, SSD1306_WHITE);

  // Scrolling rumble strips outside the curbs
  int off8 = (int)scrollPos % 8;
  for (int y = off8 - 8; y < SCREEN_HEIGHT; y += 8) {
    display.fillRect(ROAD_L - 4, y, 2, 4, SSD1306_WHITE);
    display.fillRect(ROAD_R + 2, y, 2, 4, SSD1306_WHITE);
  }

  // Scrolling lane dividers
  int off16 = (int)scrollPos % 16;
  for (int y = off16 - 16; y < SCREEN_HEIGHT; y += 16) {
    display.drawFastVLine(44, y, 8, SSD1306_WHITE);
    display.drawFastVLine(64, y, 8, SSD1306_WHITE);
    display.drawFastVLine(84, y, 8, SSD1306_WHITE);
  }
}

void drawHud() {
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("SC");
  display.setCursor(0, 10);
  display.print(score);

  display.setCursor(110, 0);
  display.print("HI");
  display.setCursor(110, 10);
  display.print(bestScore);
}

void drawGameOver() {
  display.fillRect(14, 6, 100, 52, SSD1306_BLACK);
  display.drawRect(14, 6, 100, 52, SSD1306_WHITE);
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(37, 11);
  display.print("GAME OVER");
  display.setCursor(25, 24);
  display.print("Score: ");
  display.print(score);
  display.setCursor(25, 34);
  display.print("Best:  ");
  display.print(bestScore);
  display.setCursor(25, 46);
  display.print("Tilt to retry");
}

void setup() {
  Serial.begin(115200);
  delay(500);

  Wire.begin(SDA_PIN, SCL_PIN);
  Wire.setClock(I2C_BUS_CLOCK);

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED not found!");
    while (true) delay(1000);
  }
  if (!mpuInit()) {
    Serial.println("MPU6050 not found!");
    while (true) delay(1000);
  }

  randomSeed(analogRead(0) ^ micros());

  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(2);
  display.setCursor(4, 16);
  display.println("TILT RACER");
  display.setTextSize(1);
  display.setCursor(22, 44);
  display.println("Tilt to steer!");
  display.display();
  delay(1800);

  resetGame();
  lastMicros = micros();
}

void loop() {
  // ---- Time step ----
  uint32_t now = micros();
  float dt = (now - lastMicros) / 1000000.0f;
  lastMicros = now;
  if (dt > 0.05f) dt = 0.05f;
  if (dt <= 0.0f) dt = 0.001f;

  // ---- Read + filter tilt (sideways only) ----
  float rawAx, rawAy;
  if (mpuReadAccel(rawAx, rawAy)) {
    float gx = SWAP_XY ? rawAy : rawAx;
    if (INVERT_X) gx = -gx;
    filtGx += (gx - filtGx) * TILT_FILTER;
  }
  float tilt = (fabsf(filtGx) < DEAD_ZONE) ? 0 : filtGx;

  if (!gameOver) {
    // ---- Steering ----
    float steer = constrain(tilt * STEER_GAIN, -MAX_STEER, MAX_STEER);
    playerX += steer * dt;
    playerX = constrain(playerX, ROAD_L + 1, ROAD_R - CAR_W - 1);

    // ---- Road + obstacles ----
    scrollPos += roadSpeed * dt;
    if (scrollPos > 1000) scrollPos -= 1000;

    distSinceSpawn += roadSpeed * dt;
    if (distSinceSpawn >= nextGap) {
      spawnObstacle();
      distSinceSpawn = 0;
      nextGap = 30.0f + roadSpeed * 0.25f + random(0, 15);
    }

    for (int i = 0; i < MAX_OBS; i++) {
      if (!obsActive[i]) continue;
      obsY[i] += roadSpeed * dt;

      // Passed the player -> score
      if (!obsPassed[i] && obsY[i] > PLAYER_Y + CAR_H) {
        obsPassed[i] = true;
        score++;
        roadSpeed = min(MAX_SPEED, BASE_SPEED + score * SPEED_PER_POINT);
      }
      if (obsY[i] > SCREEN_HEIGHT) obsActive[i] = false;

      // Collision (1 px forgiveness on each side)
      if (obsX[i] + 1 < playerX + CAR_W - 1 &&
          obsX[i] + CAR_W - 1 > playerX + 1 &&
          obsY[i] + 1 < PLAYER_Y + CAR_H - 1 &&
          obsY[i] + CAR_H - 1 > PLAYER_Y + 1) {
        gameOver = true;
        gameOverAt = millis();
        if (score > bestScore) bestScore = score;
      }
    }
  } else {
    // Restart by tilting hard to either side after a short lockout
    if (millis() - gameOverAt > RESTART_LOCKOUT_MS && fabsf(filtGx) > RESTART_TILT) {
      resetGame();
    }
  }

  // ---- Draw ----
  display.clearDisplay();
  drawRoad();
  for (int i = 0; i < MAX_OBS; i++) {
    if (obsActive[i]) drawCar((int)lroundf(obsX[i]), (int)lroundf(obsY[i]), false);
  }
  drawCar((int)lroundf(playerX), PLAYER_Y, true);
  drawHud();
  if (gameOver) drawGameOver();
  display.display();
}
```

## How to play

1. Upload the sketch from the [Full code](#full-code-tilt_racerino) section below and hold the board flat in front of you.
2. Tilt left or right to steer.
3. Avoid the oncoming cars. Each car you pass adds one point.
4. After a crash, tilt the board hard to either side to retry.

## Calibration

The tilt direction depends on how the MPU6050 is mounted relative to the screen. These flags at the top of the sketch control it:

```cpp
#define SWAP_XY   true   // sensor X/Y swapped relative to the screen
#define INVERT_X  true   // flip if steering goes the wrong way
```

**Test:** tilt the board right and the car should move right. If it moves left, flip `INVERT_X`. If left/right tilt does nothing but forward/back tilt steers, flip `SWAP_XY`.

## Tuning

| Setting | Effect |
|---------|--------|
| `STEER_GAIN` | Steering sensitivity (lower = calmer, higher = sharper) |
| `TILT_FILTER` | Smoothing of the tilt signal (lower = smoother but laggier) |
| `DEAD_ZONE` | Tilt ignored near level; raise it if the car drifts when flat |
| `BASE_SPEED` / `MAX_SPEED` | Starting and top road speed |
| `SPEED_PER_POINT` | How quickly the game speeds up |
| `RESTART_TILT` | How hard you must tilt to restart after a crash |
| `I2C_DISPLAY_CLOCK` | Lower from `800000UL` to `400000UL` if the screen shows garbage |

## Project history

1. **Tilt Balls:** 10 balls rolling on the OLED, direction-calibrated for the sensor mounting.
2. **Smoothness pass:** raised I2C speed, removed the fixed delay, added time-based physics, tilt filtering and a dead zone, and improved ball collisions. Balls increased to 30.
3. **Tilt Racer:** a simple tilt-controlled car game built on the same foundation.

## Known notes

- Arduino auto-generates function prototypes, so helper functions that take a custom `struct` as a parameter can fail to compile. The sketches avoid this by passing plain `float` values instead.
- The best score is not saved to flash.

## Possible improvements

- Save the best score using `Preferences` or EEPROM
- Add a start screen and a pause option
- Add a buzzer for crash and score sounds
- Add different obstacle types or a fuel/bonus pickup
