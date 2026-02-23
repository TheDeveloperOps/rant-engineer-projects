# 🎥 SSD1306 OLED Dialogue Animation (Arduino)

This project demonstrates how to interface a 0.96" SSD1306 OLED Display with Arduino using I2C communication and create a cinematic word-by-word dialogue animation inspired by Master Oogway’s iconic quote.

Instead of printing a full sentence at once, each word appears with natural timing — simulating movie-style dialogue delivery on a 128x32 OLED screen.

---

## 📦 Components Required

- Arduino UNO / Nano
- 0.96" SSD1306 OLED (I2C – 4 pin)
- Jumper wires
- Breadboard
- USB cable
- Arduino IDE
- Adafruit GFX Library
- Adafruit SSD1306 Library

---

## 🔌 Pin Diagram (Arduino UNO)

| OLED Pin | Arduino UNO |
|----------|-------------|
| VCC      | 5V          |
| GND      | GND         |
| SCL      | A5          |
| SDA      | A4          |

---

## ⚙ If Using ESP32

| OLED Pin | ESP32 |
|----------|--------|
| VCC      | 3.3V   |
| GND      | GND    |
| SCL      | GPIO 22 |
| SDA      | GPIO 21 |

Add this in setup():

```cpp
Wire.begin(21, 22);

📚 Install Required Libraries

In Arduino IDE:

Sketch → Include Library → Manage Libraries

Install:

Adafruit GFX

Adafruit SSD1306

🧠 How It Works

Words are stored in an array

Each word is displayed individually

Delays are added based on punctuation

The screen is cleared before showing the next word

Creates a cinematic dialogue animation effect



````markdown
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 32
#define SCREEN_ADDRESS 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

String dialogue[] = {
  "Yesterday", "is", "history,",
  "tomorrow", "is", "a", "mystery,",
  "but", "today", "is", "a", "gift.",
  "That", "is", "why", "it", "is",
  "called", "the", "present."
};

int totalWords = sizeof(dialogue) / sizeof(dialogue[0]);

void setup() {
  Wire.begin();
  display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS);
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
}

void loop() {
  for (int i = 0; i < totalWords; i++) {
    display.clearDisplay();
    display.setCursor(0, 10);
    display.println(dialogue[i]);
    display.display();

    if (dialogue[i].endsWith(",")) {
      delay(600);
    } 
    else if (dialogue[i].endsWith(".")) {
      delay(1000);
    } 
    else {
      delay(350);
    }
  }
  delay(2000);
}
