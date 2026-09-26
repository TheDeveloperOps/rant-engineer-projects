# WiFi Robot Eyes 🤖👀

Cosmo-style animated robot eyes on a tiny OLED, controlled from your phone over WiFi — no app needed, just a browser. Built for the **ESP32-C3 Super Mini** and a **0.96" I2C OLED (SSD1306, 128x64)**.

Tap a button on your phone to switch expressions: normal/idle, happy, surprised, angry, sleepy, wink, love, and look-left/right. Hold the board's built-in BOOT button to put it to sleep, tap it again to wake it back up — no extra hardware button needed.

---

## Table of contents

- [What you need](#what-you-need)
- [Wiring](#wiring)
- [Arduino IDE setup](#arduino-ide-setup)
- [Uploading the code](#uploading-the-code)
- [First run](#first-run)
- [Using it](#using-it)
- [Sleep / wake (power button)](#sleep--wake-power-button)
- [The full code](#the-full-code)
- [Beginner mistakes & troubleshooting](#beginner-mistakes--troubleshooting)
- [How it works (short version)](#how-it-works-short-version)
- [License](#license)

---

## What you need

| Part | Notes |
|---|---|
| ESP32-C3 Super Mini | Any ESP32-C3 dev board works, pins may differ slightly |
| 0.96" I2C OLED display | SSD1306 driver, 128x64, 4-pin (GND/VCC/SCL/SDA) |
| USB-C cable | Data-capable, not charge-only |
| A phone or laptop | To connect to the board's WiFi and control it |

No extra buttons, resistors, or breadboard strictly required — the OLED wires directly to the board, and power on/off uses the button that's already built into the board.

---

## Wiring

| OLED pin | Connects to (ESP32-C3) |
|---|---|
| GND | GND |
| VCC | 3V3 *(not 5V — most of these OLEDs are 3.3V logic only)* |
| SCL | GPIO5 |
| SDA | GPIO4 |

```
OLED            ESP32-C3 Super Mini
----            -------------------
GND     ------->  GND
VCC     ------->  3V3
SCL     ------->  GPIO5
SDA     ------->  GPIO4
```

**Board-specific notes:**
- GPIO4 and GPIO5 are safe general-purpose pins on the C3 — no boot conflicts.
- Avoid using GPIO2, GPIO8, or GPIO9 for anything else you add later — these are strapping pins tied to the board's boot mode selection, and misusing them can cause boot loops.
- The onboard I2C address used in the code is `0x3C` (the standard for most SSD1306 modules). If the screen stays blank, some clone boards use `0x3D` instead — see the troubleshooting section.

---

## Arduino IDE setup

1. Install the **ESP32 board package** if you haven't already: `File > Preferences > Additional Board Manager URLs`, add:
   `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`
   Then `Tools > Board > Boards Manager`, search "esp32", install the Espressif package.
2. Install libraries via `Sketch > Include Library > Manage Libraries`:
   - **Adafruit GFX Library**
   - **Adafruit SSD1306**
   - (`WiFi.h` and `WebServer.h` come bundled with the ESP32 board package — nothing to install for those.)
3. Under `Tools > Board`, select your ESP32-C3 board (e.g. "ESP32C3 Dev Module" or the closest Super Mini match).
4. Under `Tools > Port`, select the port your board shows up as after plugging in via USB.

---

## Uploading the code

1. Copy the full sketch below into a new file named `robot_eyes_wifi.ino` (or download it from this repo).
2. Near the top, change the WiFi name/password if you like:
   ```cpp
   const char* ssid     = "RobotEyes";
   const char* password = "eyes1234";   // must be 8+ characters, or "" for an open network
   ```
3. Click **Upload**.
4. Open `Tools > Serial Monitor`, set the baud rate to **115200**. You should see:
   ```
   softAP() returned: true
   AP started. Connect phone to WiFi: RobotEyes
   Then open: http://192.168.4.1
   Hold BOOT button ~1s to sleep. Press BOOT again to wake.
   ```

If you see `softAP() returned: false`, or nothing shows up in your phone's WiFi list at all, jump to [Beginner mistakes & troubleshooting](#beginner-mistakes--troubleshooting) — this was the very first bug this project ran into.

---

## First run

1. On your phone, go to WiFi settings and connect to the **RobotEyes** network using the password you set.
2. Open a browser and go to: **http://192.168.4.1**
3. You should see a **"Pick a face"** page with buttons.

---

## Using it

Tap any button on the web page to change the expression:

- Normal / Idle — default animated blinking + occasional glance
- Happy
- Surprised
- Angry
- Sleepy
- Wink
- Love (hearts)
- Look Left / Look Right

Each tap sends a request to `/set?mode=...` on the board, which updates the animation shown on the OLED.

---

## Sleep / wake (power button)

There's no separate physical power switch — the board's **built-in BOOT button** doubles as one:

1. **Hold BOOT for about 1 second, then let go.** The OLED blanks and the WiFi network disappears from your phone's list — the board is now in light sleep, drawing much less power.
2. **Press BOOT again (a quick tap)** to wake it back up. The OLED lights back up on the *same face it was showing before*, and the WiFi network reappears within a few seconds.

Important detail if you're modifying this: the wake condition is level-triggered (checks "is the pin LOW right now"), so the code deliberately waits for you to **release** the long-press before it actually sleeps — otherwise it would see your finger still on the button and wake up instantly. This was a real bug during development; see the troubleshooting section.

---

## The full code

```cpp
/*
  Cosmo-style animated robot eyes — WiFi controlled from your phone
  ESP32-C3 Super Mini + 0.96" I2C OLED (SSD1306, 128x64)

  HOW IT WORKS:
  - ESP32 creates its own WiFi network (Access Point mode)
  - Connect your phone to that WiFi network
  - Open a browser to 192.168.4.1
  - Tap a button to switch the face/animation
  - Hold the built-in BOOT button (GPIO9) for ~1 second to power down
    (light sleep). Press BOOT again to wake it back up.

  WIRING (unchanged from before):
    OLED GND -> GND
    OLED VCC -> 3V3
    OLED SCL -> GPIO5
    OLED SDA -> GPIO4

  LIBRARIES NEEDED (Arduino IDE -> Library Manager):
    - Adafruit GFX Library
    - Adafruit SSD1306
    (WiFi.h and WebServer.h are built into the ESP32 board package already)

  SETUP:
    1. Change ssid/password below to whatever you want.
    2. Upload.
    3. On your phone, go to WiFi settings, connect to that network.
    4. Open a browser, go to: http://192.168.4.1
    5. Tap buttons to change expressions.
    6. Hold BOOT (GPIO9) ~1s to sleep. Press it again to wake up.

  FIX NOTES (AP not showing up in WiFi list):
  Root cause was the radio browning out during AP beaconing at full TX power
  on ESP32-C3 Super Mini boards. The fix is to explicitly drop TX power
  right after WiFi.mode(WIFI_AP) and before WiFi.softAP(). This makes the
  brownout-detector-disable hack unnecessary, so it's been removed.

  POWER-OFF NOTES:
  The ESP32 can't cut its own power from software, so "off" here means
  light sleep: OLED is blanked and WiFi is shut down while sleeping.
  GPIO9 (BOOT) isn't RTC-capable on the C3, so deep-sleep ext0/ext1 wakeup
  can't use it — light sleep with a plain GPIO wakeup works on any pin
  instead, and keeps RAM intact, so it resumes on the same face it was
  showing rather than rebooting back to "normal".
*/

#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <WiFi.h>
#include <WebServer.h>
#include "esp_wifi.h"   // usually pulled in by WiFi.h already, but explicit here
#include "esp_sleep.h"
#include "driver/gpio.h"

// ---------- WiFi settings (edit these) ----------
const char* ssid     = "RobotEyes";
const char* password = "eyes1234";   // must be 8+ characters, or set to "" for open network

// ---------- OLED / I2C ----------
#define SDA_PIN 4
#define SCL_PIN 5
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define SCREEN_ADDRESS 0x3C

// ---------- Built-in BOOT button (power off) ----------
#define BOOT_BUTTON 9          // active LOW
#define HOLD_MS_TO_SLEEP 1000  // how long to hold BOOT before sleeping

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
WebServer server(80);

// ---------- Eye geometry ----------
int eyeWidth = 34;
int eyeHeight = 40;
int eyeRadius = 10;
int eyeGap = 12;
int leftEyeX, rightEyeX, eyeY;
int centerY = SCREEN_HEIGHT / 2;

// current face mode, changed by web requests
String currentMode = "normal";
String lastDrawnMode = "";

// ---------- Web page ----------
const char PAGE_HTML[] PROGMEM = R"rawliteral(
<!DOCTYPE html>
<html>
<head>
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Robot Face Control</title>
<style>
  body { font-family: sans-serif; background:#111; color:#eee; text-align:center; padding:20px; }
  h2 { margin-bottom: 20px; }
  button {
    display:block; width:90%; max-width:320px; margin:10px auto;
    padding:16px; font-size:18px; border:none; border-radius:10px;
    background:#2b8aef; color:white; }
  button:active { background:#1c5fa8; }
</style>
</head>
<body>
  <h2>Pick a face</h2>
  <button onclick="setMode('normal')">Normal / Idle</button>
  <button onclick="setMode('happy')">Happy</button>
  <button onclick="setMode('surprised')">Surprised</button>
  <button onclick="setMode('angry')">Angry</button>
  <button onclick="setMode('sleepy')">Sleepy</button>
  <button onclick="setMode('wink')">Wink</button>
  <button onclick="setMode('love')">Love (hearts)</button>
  <button onclick="setMode('lookleft')">Look Left</button>
  <button onclick="setMode('lookright')">Look Right</button>
  <script>
    function setMode(m) {
      fetch('/set?mode=' + m);
    }
  </script>
</body>
</html>
)rawliteral";

void handleRoot() {
  server.send(200, "text/html", PAGE_HTML);
}

void handleSet() {
  if (server.hasArg("mode")) {
    currentMode = server.arg("mode");
  }
  server.send(200, "text/plain", "ok");
}

// ---------- Drawing helpers ----------
void drawEyes(int offsetX, int offsetY, int leftH, int rightH) {
  display.clearDisplay();
  int lY = eyeY + offsetY + (eyeHeight - leftH) / 2;
  int rY = eyeY + offsetY + (eyeHeight - rightH) / 2;
  display.fillRoundRect(leftEyeX + offsetX, lY, eyeWidth, leftH, eyeRadius, SSD1306_WHITE);
  display.fillRoundRect(rightEyeX + offsetX, rY, eyeWidth, rightH, eyeRadius, SSD1306_WHITE);
  display.display();
}

// checks web requests + power button frequently during an animation
// so buttons stay responsive and a long-press can still be caught
bool stayInMode(unsigned long ms) {
  unsigned long start = millis();
  while (millis() - start < ms) {
    server.handleClient();
    checkPowerButton();
    if (currentMode != lastDrawnMode) return false; // mode changed, break out
    delay(4);
  }
  return true;
}

void blink() {
  for (int h = eyeHeight; h >= 4; h -= 10) { drawEyes(0, 0, h, h); if (!stayInMode(4)) return; }
  for (int h = 4; h <= eyeHeight; h += 10) { drawEyes(0, 0, h, h); if (!stayInMode(4)) return; }
}

void lookAt(int dx, int dy, int holdMs) {
  for (int i = 0; i <= 10; i += 2) { drawEyes((dx*i)/10, (dy*i)/10, eyeHeight, eyeHeight); if (!stayInMode(4)) return; }
  if (!stayInMode(holdMs)) return;
  for (int i = 10; i >= 0; i -= 2) { drawEyes((dx*i)/10, (dy*i)/10, eyeHeight, eyeHeight); if (!stayInMode(4)) return; }
}

void angryBrows() {
  // draw normal eyes then overlay two black diagonal wedges to fake angled eyebrows
  drawEyes(0, 0, eyeHeight, eyeHeight);
  display.fillTriangle(leftEyeX - 4, eyeY - 2, leftEyeX + eyeWidth, eyeY - 2, leftEyeX - 4, eyeY + 14, SSD1306_BLACK);
  display.fillTriangle(rightEyeX + eyeWidth + 4, eyeY - 2, rightEyeX, eyeY - 2, rightEyeX + eyeWidth + 4, eyeY + 14, SSD1306_BLACK);
  display.display();
}

void sleepyEyes() {
  drawEyes(0, 6, eyeHeight / 3, eyeHeight / 3);
}

void winkEyes() {
  drawEyes(0, 0, 6, eyeHeight);
}

void heartEyes() {
  display.clearDisplay();
  drawHeart(leftEyeX + eyeWidth / 2, eyeY + eyeHeight / 2, 16);
  drawHeart(rightEyeX + eyeWidth / 2, eyeY + eyeHeight / 2, 16);
  display.display();
}

void drawHeart(int cx, int cy, int size) {
  int r = size / 3;
  display.fillCircle(cx - r, cy - r / 2, r, SSD1306_WHITE);
  display.fillCircle(cx + r, cy - r / 2, r, SSD1306_WHITE);
  display.fillTriangle(cx - size, cy - r / 3, cx + size, cy - r / 3, cx, cy + size, SSD1306_WHITE);
}

// ---------- Mode runners (each runs until currentMode changes) ----------
void runNormal() {
  drawEyes(0, 0, eyeHeight, eyeHeight);
  if (!stayInMode(700)) return;
  blink();
  if (currentMode != "normal") return;
  int r = random(0, 4);
  if (r == 0) lookAt(-15, 0, 200);
  else if (r == 1) lookAt(15, 0, 200);
  else if (r == 2) lookAt(0, -8, 150);
  // r==3: just blink again, no look
}

void runHappy() {
  for (int h = eyeHeight; h >= eyeHeight/2; h -= 8) { drawEyes(0, eyeHeight/4, h, h); if (!stayInMode(6)) return; }
  if (!stayInMode(400)) return;
  for (int h = eyeHeight/2; h <= eyeHeight; h += 8) { drawEyes(0, eyeHeight/4, h, h); if (!stayInMode(6)) return; }
  stayInMode(300);
}

void runSurprised() {
  int bigH = eyeHeight + 10, bigW = eyeWidth + 6;
  display.clearDisplay();
  display.fillRoundRect(leftEyeX - 3, eyeY - 5, bigW, bigH, eyeRadius, SSD1306_WHITE);
  display.fillRoundRect(rightEyeX - 3, eyeY - 5, bigW, bigH, eyeRadius, SSD1306_WHITE);
  display.display();
  stayInMode(400);
}

void runAngry() {
  angryBrows();
  stayInMode(400);
}

void runSleepy() {
  sleepyEyes();
  stayInMode(400);
}

void runWink() {
  winkEyes();
  stayInMode(400);
}

void runLove() {
  heartEyes();
  stayInMode(400);
}

void runLookLeft() {
  lookAt(-18, 0, 500);
}

void runLookRight() {
  lookAt(18, 0, 500);
}

// ---------- Power button (BOOT, GPIO9) ----------
// Non-blocking check: called from loop() and from inside stayInMode()
// so a long-press is caught even mid-animation.
unsigned long buttonPressStart = 0;
bool buttonHeld = false;

void checkPowerButton() {
  bool pressed = (digitalRead(BOOT_BUTTON) == LOW);

  if (pressed && !buttonHeld) {
    buttonHeld = true;
    buttonPressStart = millis();
  } else if (pressed && buttonHeld) {
    if (millis() - buttonPressStart >= HOLD_MS_TO_SLEEP) {
      goToSleep();  // does not return — chip goes to deep sleep here
    }
  } else if (!pressed) {
    buttonHeld = false;
  }
}

void goToSleep() {
  // Wait for the current long-press to be released FIRST. The wakeup below
  // is level-triggered (LOW), so if we entered sleep while still held down,
  // the wake condition is already true and it wakes instantly — which is
  // the "flickers off then immediately back on" bug.
  while (digitalRead(BOOT_BUTTON) == LOW) delay(10);
  delay(50); // simple debounce after release

  display.clearDisplay();
  display.display();                          // blank the OLED before sleeping
  display.ssd1306_command(SSD1306_DISPLAYOFF); // panel off, saves a little more power

  WiFi.softAPdisconnect(true);
  WiFi.mode(WIFI_OFF);

  gpio_wakeup_enable(GPIO_NUM_9, GPIO_INTR_LOW_LEVEL);
  esp_sleep_enable_gpio_wakeup();

  esp_light_sleep_start();   // now genuinely blocks until BOOT is pressed again

  // ---------- execution resumes here after waking ----------
  gpio_wakeup_disable(GPIO_NUM_9);

  display.ssd1306_command(SSD1306_DISPLAYON);

  WiFi.mode(WIFI_AP);
  delay(100);
  WiFi.setTxPower(WIFI_POWER_8_5dBm);
  WiFi.softAP(ssid, password);

  lastDrawnMode = "";   // force a redraw of the current face on wake

  // debounce: wait for the wake-press to be released too, so it doesn't
  // immediately re-trigger sleep on the next loop() pass
  while (digitalRead(BOOT_BUTTON) == LOW) delay(10);
  buttonHeld = false;
}

void setup() {
  Serial.begin(115200);
  Wire.begin(SDA_PIN, SCL_PIN);
  Wire.setClock(400000);

  pinMode(BOOT_BUTTON, INPUT_PULLUP);

  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println("OLED not found!");
    while (true) { delay(1000); }
  }

  int totalW = eyeWidth * 2 + eyeGap;
  leftEyeX = (SCREEN_WIDTH - totalW) / 2;
  rightEyeX = leftEyeX + eyeWidth + eyeGap;
  eyeY = centerY - eyeHeight / 2;

  display.clearDisplay();
  display.display();
  randomSeed(analogRead(0));

  WiFi.mode(WIFI_AP);
  delay(100);
  WiFi.setTxPower(WIFI_POWER_8_5dBm);   // the fix: prevents radio brownout during AP beaconing
  bool apOk = WiFi.softAP(ssid, password);
  Serial.print("softAP() returned: ");
  Serial.println(apOk ? "true" : "false");
  Serial.print("AP started. Connect phone to WiFi: ");
  Serial.println(ssid);
  Serial.print("Then open: http://");
  Serial.println(WiFi.softAPIP());
  Serial.println("Hold BOOT button ~1s to sleep. Press BOOT again to wake.");

  server.on("/", handleRoot);
  server.on("/set", handleSet);
  server.begin();
}

void loop() {
  server.handleClient();
  checkPowerButton();
  lastDrawnMode = currentMode;

  if (currentMode == "normal") runNormal();
  else if (currentMode == "happy") runHappy();
  else if (currentMode == "surprised") runSurprised();
  else if (currentMode == "angry") runAngry();
  else if (currentMode == "sleepy") runSleepy();
  else if (currentMode == "wink") runWink();
  else if (currentMode == "love") runLove();
  else if (currentMode == "lookleft") runLookLeft();
  else if (currentMode == "lookright") runLookRight();
  else runNormal();
}
```

---

## Beginner mistakes & troubleshooting

These are the actual issues hit while building this project, in the order they came up — if you're following along and something's not working, check here first.

### 1. WiFi network never shows up in the phone's WiFi list
**Symptom:** Code uploads fine, Serial Monitor might even look normal, but "RobotEyes" never appears when you scan for WiFi networks on your phone.
**Cause:** On ESP32-C3 Super Mini boards, running the radio at full TX power during AP beaconing can brown out the chip's power rail. It's a known issue on these specific boards, not a code bug in the usual sense.
**Fix:** Explicitly lower the TX power right after switching to AP mode and before starting the access point:
```cpp
WiFi.mode(WIFI_AP);
delay(100);
WiFi.setTxPower(WIFI_POWER_8_5dBm);   // the fix
bool apOk = WiFi.softAP(ssid, password);
```
Don't reach for disabling the brownout detector (`WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0)`) as a "fix" — that just hides the symptom instead of solving the actual power issue, and can mask other real problems.

### 2. Compile error: `'esp_sleep_enable_ext0_wakeup' was not declared in this scope`
**Cause:** `ext0`/`ext1` deep-sleep wakeup functions only work on **RTC-capable GPIOs**. On the ESP32-C3, only **GPIO0–GPIO5** are RTC-capable — GPIO9 (the BOOT button) is not one of them, so this function can't be used with it at all, on any core version.
**Fix:** Use **light sleep** instead of deep sleep, with a plain GPIO wakeup (`gpio_wakeup_enable` + `esp_sleep_enable_gpio_wakeup`), which works on any digital pin — not just RTC-capable ones. As a bonus, light sleep keeps RAM intact, so the board resumes on the exact face it was showing instead of rebooting back to "normal".

### 3. Board turns off while holding the button, then immediately turns back on while still holding it
**Cause:** The GPIO wakeup used here is level-triggered (`GPIO_INTR_LOW_LEVEL`) — it wakes as soon as the pin *reads* LOW. If you enter sleep while your finger is still holding the button down, the pin is already LOW, so it wakes up instantly, before you've even let go.
**Fix:** Wait for the button to be **released** before actually calling `esp_light_sleep_start()`:
```cpp
while (digitalRead(BOOT_BUTTON) == LOW) delay(10);  // wait for release first
delay(50);                                          // debounce
// ...then actually go to sleep
```
With this in place, the real sequence is: hold ~1s → **release** → it sleeps → tap again → it wakes.

### 4. OLED shows nothing at all (blank screen, nothing crashes)
**Likely causes, in order of likelihood:**
- Wrong I2C address. This code assumes `0x3C`; some clone boards use `0x3D`. Run a basic I2C scanner sketch to confirm.
- SDA/SCL swapped, or on the wrong pins. Double check against the [wiring](#wiring) table above.
- OLED powered from 5V instead of 3V3 — some modules survive this, some don't display correctly, some are damaged over time. Always use 3V3 unless the module explicitly says 5V-tolerant.

### 5. `softAP() returned: false` in Serial Monitor
**Cause:** Usually the same TX-power brownout issue as #1, just caught before the AP even reports success. Occasionally caused by a `password` shorter than 8 characters — WiFi requires 8+ characters for WPA2, or an empty string `""` for a fully open network.

### 6. Buttons on the web page don't seem to do anything
**Cause:** Usually the phone is still connected to the board's AP but the animation is mid-frame, or the phone silently reconnected to your home WiFi instead of staying on "RobotEyes" (some phones do this automatically since the AP has no internet access).
**Fix:** Check your phone is still showing "RobotEyes" as the active WiFi network, not just in range. Some phones have a setting like "stay connected to network with no internet" that needs to be toggled on for the AP to be picked and held.

---

## How it works (short version)

- On boot, the ESP32 creates its own WiFi access point (no router needed) and starts a tiny web server on port 80.
- Visiting `http://192.168.4.1` in a phone browser loads a page (`handleRoot()`) with buttons.
- Each button tap calls `/set?mode=...` (`handleSet()`), which updates a `currentMode` variable.
- The main `loop()` continuously redraws the OLED based on whatever `currentMode` currently is, and checks the BOOT button on every pass so it stays responsive even mid-animation.
- Holding BOOT long enough puts the chip into light sleep (screen + WiFi off); a later tap wakes it back up and restarts the access point.

---

## License

Free to use, modify, and share for personal or educational projects.
