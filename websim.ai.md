# Claude Sonet 4 [putem websim.com](https://websim.com/p/9x6_6208fqie04p1xvqk)

# 🚢 Submarine Wars - VIDI X ESP32

## Opis igre

Submarine Wars je uzbudljiva podmorska igra za VIDI X mikroračunalo na ESP32 platformi. Upravljajte svojom podmornicom kroz opasne vode, izbjegavajte neprijateljske torpeda i uništavajte neprijateljske podmornice!

### Kontrole

- **GPIO34:** Lijevo/Desno (analogni)
- **GPIO35:** Zaroni/Izroni (analogni)
- **GPIO0 &amp; GPIO13:** Brzina (5 stupnjeva: -1, 0, 1, 2, 3)
- **GPIO32:** Ispaljivanje torpeda
- **GPIO27 &amp; GPIO39:** Upravljanje torpedom (lijevo/desno)

### 🎯 Game Features

- 10 neprijateljskih podmornica
- Upravljivi torpedi
- Realistična fizika podmornice
- Pametna AI za neprijatelje
- Ograničen broj torpeda (10 po podmornicu)

### 🎮 Controls System

- Analogne kontrole za precizno upravljanje
- 5-stupanjski sustav brzine
- Rotacija za 5° po kliku
- Sporo zaranjanje/izranjanje
- Konstanta brzina torpeda (200)

### ⚠️ Važne napomene

- Neprijateljske podmornice neće sve pucati istovremeno
- Neprijateljski torpedi mogu pogoditi vlastite podmornice
- Kolizija između neprijateljskih podmornica ih uništava
- Torpedi se mogu međusobno uništiti

📋 Copy Code

```
#define LGFX_USE_V1
#include <LovyanGFX.hpp>

class LGFX : public lgfx::LGFX_Device {
  lgfx::Panel_ILI9341 _panel_instance;
  lgfx::Bus_SPI _bus_instance;
public:
  LGFX(void) {
    { // Bus config
      auto cfg = _bus_instance.config();
      cfg.spi_host = VSPI_HOST;
      cfg.spi_mode = 0;
      cfg.freq_write = 40000000;
      cfg.freq_read = 16000000;
      cfg.spi_3wire = true;
      cfg.use_lock = true;
      cfg.dma_channel = 1;
      cfg.pin_sclk = 18;
      cfg.pin_mosi = 23;
      cfg.pin_miso = 19;
      cfg.pin_dc = 21;
      _bus_instance.config(cfg);
      _panel_instance.setBus(&_bus_instance);
    }
    { // Panel config
      auto cfg = _panel_instance.config();
      cfg.pin_cs = 5;
      cfg.pin_rst = -1;
      cfg.pin_busy = -1;
      cfg.memory_width = 240;
      cfg.memory_height = 320;
      cfg.panel_width = 240;
      cfg.panel_height = 320;
      cfg.offset_x = 0;
      cfg.offset_y = 0;
      cfg.offset_rotation = 1;
      cfg.dummy_read_pixel = 8;
      cfg.dummy_read_bits = 1;
      cfg.readable = true;
      cfg.invert = false;
      cfg.rgb_order = false;
      cfg.dlen_16bit = false;
      cfg.bus_shared = true;
      _panel_instance.config(cfg);
    }
    setPanel(&_panel_instance);
  }
};

static LGFX lcd;
static LGFX_Sprite canvas(&lcd);

#define SPEAKER_PIN 25
#define LEDC_CHANNEL 0
#define LEDC_RESOLUTION 8
#define LEDC_BASE_FREQ 2000

#define SCREEN_WIDTH 320
#define SCREEN_HEIGHT 240

// GPIO pins for controls
#define BTN_LR         34  // Left/Right analog
#define BTN_UD         35  // Dive/Surface analog
#define BTN_SPEED_UP   0   // Speed increase
#define BTN_SPEED_DOWN 13  // Speed decrease
#define BTN_FIRE       32  // Fire torpedo
#define BTN_TORPEDO_L  27  // Torpedo left
#define BTN_TORPEDO_R  39  // Torpedo right

#define MAX_SUBMARINES 10
#define MAX_TORPEDOS 50
#define TORPEDO_SPEED 200
#define MAX_SUB_SPEED 100

/* @tweakable number of enemy submarines */
const int enemySubmarineCount = 10;

/* @tweakable player submarine max speed */
const int playerMaxSpeed = 100;

/* @tweakable torpedo speed */
const int torpedoSpeed = 200;

/* @tweakable rotation step in degrees */
const float rotationStep = 5.0;

/* @tweakable dive/surface speed */
const float diveSpeed = 1.0;

struct Submarine {
  float x, y;
  float angle;        // Direction in degrees
  int speed;          // -1 to 3 (with 0 = stop)
  int torpedos;       // Remaining torpedos
  bool alive;
  bool isPlayer;
  float depth;        // For dive/surface animation
  unsigned long lastFire;
  float targetX, targetY; // AI target for enemies
};

struct Torpedo {
  float x, y;
  float dx, dy;
  bool active;
  bool isPlayer;
  bool controllable; // Only player torpedos are controllable
  unsigned long spawnTime;
};

Submarine submarines[MAX_SUBMARINES + 1]; // +1 for player
Torpedo torpedos[MAX_TORPEDOS];
int playerIndex = 0;

bool gameOver = false;
unsigned long lastEnemyFire = 0;
int currentEnemyFirer = 0;

// Speed control
int currentSpeed = 0; // -1, 0, 1, 2, 3
bool lastSpeedUp = HIGH;
bool lastSpeedDown = HIGH;

// Torpedo control
bool lastTorpedoLeft = HIGH;
bool lastTorpedoRight = HIGH;

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.setRotation(0);
  lcd.fillScreen(0x0000);
  canvas.setPsram(true);
  bool ok = canvas.createSprite(SCREEN_WIDTH, SCREEN_HEIGHT);
  if (!ok) {
    Serial.println("Canvas creation failed!");
    while(1) delay(1000);
  }
  
  ledcSetup(LEDC_CHANNEL, LEDC_BASE_FREQ, LEDC_RESOLUTION);
  ledcAttachPin(SPEAKER_PIN, LEDC_CHANNEL);
  
  // Setup GPIO pins
  pinMode(BTN_LR, INPUT);
  pinMode(BTN_UD, INPUT);
  pinMode(BTN_SPEED_UP, INPUT_PULLUP);
  pinMode(BTN_SPEED_DOWN, INPUT_PULLUP);
  pinMode(BTN_FIRE, INPUT_PULLUP);
  pinMode(BTN_TORPEDO_L, INPUT_PULLUP);
  pinMode(BTN_TORPEDO_R, INPUT_PULLUP);
  
  initGame();
}

void initGame() {
  // Initialize player submarine
  submarines[playerIndex].x = SCREEN_WIDTH / 2;
  submarines[playerIndex].y = SCREEN_HEIGHT / 2;
  submarines[playerIndex].angle = 0;
  submarines[playerIndex].speed = 0;
  submarines[playerIndex].torpedos = 10;
  submarines[playerIndex].alive = true;
  submarines[playerIndex].isPlayer = true;
  submarines[playerIndex].depth = 0.5; // Mid depth
  submarines[playerIndex].lastFire = 0;
  
  // Initialize enemy submarines
  for(int i = 1; i <= MAX_SUBMARINES; i++) {
    submarines[i].x = random(50, SCREEN_WIDTH - 50);
    submarines[i].y = random(50, SCREEN_HEIGHT - 50);
    submarines[i].angle = random(0, 360);
    submarines[i].speed = random(1, 4); // 1-3 speed
    submarines[i].torpedos = 10;
    submarines[i].alive = true;
    submarines[i].isPlayer = false;
    submarines[i].depth = random(0, 100) / 100.0;
    submarines[i].lastFire = millis() + random(2000, 5000);
    submarines[i].targetX = random(0, SCREEN_WIDTH);
    submarines[i].targetY = random(0, SCREEN_HEIGHT);
  }
  
  // Initialize torpedos
  for(int i = 0; i < MAX_TORPEDOS; i++) {
    torpedos[i].active = false;
  }
  
  gameOver = false;
  currentSpeed = 0;
  lastEnemyFire = millis();
  currentEnemyFirer = 1;
}

void updatePlayerMovement() {
  Submarine &player = submarines[playerIndex];
  
  // Read analog controls for rotation and dive/surface
  int analogLR = analogRead(BTN_LR);
  int analogUD = analogRead(BTN_UD);
  
  // Left rotation
  if (analogLR > 4000) {
    player.angle -= rotationStep;
    if (player.angle < 0) player.angle += 360;
  }
  // Right rotation
  if (analogLR > 1800 && analogLR < 2200) {
    player.angle += rotationStep;
    if (player.angle >= 360) player.angle -= 360;
  }
  
  // Dive (increase depth)
  if (analogUD > 4000) {
    player.depth += diveSpeed * 0.02;
    if (player.depth > 1.0) player.depth = 1.0;
  }
  // Surface (decrease depth)
  if (analogUD > 1800 && analogUD < 2200) {
    player.depth -= diveSpeed * 0.02;
    if (player.depth < 0.0) player.depth = 0.0;
  }
  
  // Speed control
  bool nowSpeedUp = digitalRead(BTN_SPEED_UP);
  bool nowSpeedDown = digitalRead(BTN_SPEED_DOWN);
  
  if (lastSpeedUp == HIGH && nowSpeedUp == LOW) {
    currentSpeed++;
    if (currentSpeed > 3) currentSpeed = 3;
  }
  if (lastSpeedDown == HIGH && nowSpeedDown == LOW) {
    currentSpeed--;
    if (currentSpeed < -1) currentSpeed = -1;
  }
  
  lastSpeedUp = nowSpeedUp;
  lastSpeedDown = nowSpeedDown;
  
  player.speed = currentSpeed;
  
  // Move submarine based on speed and angle
  if (player.speed != 0) {
    float radians = player.angle * PI / 180.0;
    float actualSpeed = (player.speed > 0) ? player.speed * 25 : player.speed * 15; // Reverse slower
    player.x += cos(radians) * actualSpeed * 0.1;
    player.y += sin(radians) * actualSpeed * 0.1;
    
    // Wrap around screen
    if (player.x < 0) player.x = SCREEN_WIDTH;
    if (player.x > SCREEN_WIDTH) player.x = 0;
    if (player.y < 0) player.y = SCREEN_HEIGHT;
    if (player.y > SCREEN_HEIGHT) player.y = 0;
  }
}

void updateEnemyAI() {
  for(int i = 1; i <= MAX_SUBMARINES; i++) {
    if (!submarines[i].alive) continue;
    
    Submarine &enemy = submarines[i];
    
    // Simple AI: move towards target, occasionally change target
    if (random(100) < 2) { // 2% chance to change target
      enemy.targetX = random(0, SCREEN_WIDTH);
      enemy.targetY = random(0, SCREEN_HEIGHT);
    }
    
    // Calculate angle to target
    float dx = enemy.targetX - enemy.x;
    float dy = enemy.targetY - enemy.y;
    float targetAngle = atan2(dy, dx) * 180.0 / PI;
    if (targetAngle < 0) targetAngle += 360;
    
    // Gradually turn towards target
    float angleDiff = targetAngle - enemy.angle;
    if (angleDiff > 180) angleDiff -= 360;
    if (angleDiff < -180) angleDiff += 360;
    
    if (abs(angleDiff) > 5) {
      if (angleDiff > 0) {
        enemy.angle += 3;
      } else {
        enemy.angle -= 3;
      }
      if (enemy.angle < 0) enemy.angle += 360;
      if (enemy.angle >= 360) enemy.angle -= 360;
    }
    
    // Move enemy
    float radians = enemy.angle * PI / 180.0;
    float speed = enemy.speed * 20 * 0.1;
    enemy.x += cos(radians) * speed;
    enemy.y += sin(radians) * speed;
    
    // Wrap around screen
    if (enemy.x < 0) enemy.x = SCREEN_WIDTH;
    if (enemy.x > SCREEN_WIDTH) enemy.x = 0;
    if (enemy.y < 0) enemy.y = SCREEN_HEIGHT;
    if (enemy.y > SCREEN_HEIGHT) enemy.y = 0;
    
    // Random depth changes
    if (random(1000) < 5) {
      enemy.depth += (random(-20, 21) / 100.0);
      if (enemy.depth < 0) enemy.depth = 0;
      if (enemy.depth > 1) enemy.depth = 1;
    }
  }
}

void fireTorpedo(int subIndex) {
  if (submarines[subIndex].torpedos <= 0) return;
  
  for(int i = 0; i < MAX_TORPEDOS; i++) {
    if (!torpedos[i].active) {
      Submarine &sub = submarines[subIndex];
      float radians = sub.angle * PI / 180.0;
      
      torpedos[i].x = sub.x + cos(radians) * 15; // Spawn ahead of submarine
      torpedos[i].y = sub.y + sin(radians) * 15;
      torpedos[i].dx = cos(radians) * torpedoSpeed * 0.1;
      torpedos[i].dy = sin(radians) * torpedoSpeed * 0.1;
      torpedos[i].active = true;
      torpedos[i].isPlayer = sub.isPlayer;
      torpedos[i].controllable = sub.isPlayer;
      torpedos[i].spawnTime = millis();
      
      sub.torpedos--;
      playTorpedoSound();
      break;
    }
  }
}

void updateTorpedos() {
  bool nowTorpedoLeft = digitalRead(BTN_TORPEDO_L);
  bool nowTorpedoRight = digitalRead(BTN_TORPEDO_R);
  
  for(int i = 0; i < MAX_TORPEDOS; i++) {
    if (!torpedos[i].active) continue;
    
    // Control player torpedos
    if (torpedos[i].controllable) {
      float currentAngle = atan2(torpedos[i].dy, torpedos[i].dx);
      bool angleChanged = false;
      
      if (lastTorpedoLeft == HIGH && nowTorpedoLeft == LOW) {
        currentAngle -= rotationStep * PI / 180.0;
        angleChanged = true;
      }
      if (lastTorpedoRight == HIGH && nowTorpedoRight == LOW) {
        currentAngle += rotationStep * PI / 180.0;
        angleChanged = true;
      }
      
      if (angleChanged) {
        float speed = sqrt(torpedos[i].dx * torpedos[i].dx + torpedos[i].dy * torpedos[i].dy);
        torpedos[i].dx = cos(currentAngle) * speed;
        torpedos[i].dy = sin(currentAngle) * speed;
      }
    }
    
    // Move torpedo
    torpedos[i].x += torpedos[i].dx;
    torpedos[i].y += torpedos[i].dy;
    
    // Remove torpedo if off screen or too old
    if (torpedos[i].x < -50 || torpedos[i].x > SCREEN_WIDTH + 50 ||
        torpedos[i].y < -50 || torpedos[i].y > SCREEN_HEIGHT + 50 ||
        millis() - torpedos[i].spawnTime > 15000) {
      torpedos[i].active = false;
    }
  }
  
  lastTorpedoLeft = nowTorpedoLeft;
  lastTorpedoRight = nowTorpedoRight;
}

void checkCollisions() {
  // Torpedo vs Submarine collisions
  for(int t = 0; t < MAX_TORPEDOS; t++) {
    if (!torpedos[t].active) continue;
    
    for(int s = 0; s <= MAX_SUBMARINES; s++) {
      if (!submarines[s].alive) continue;
      
      float dx = torpedos[t].x - submarines[s].x;
      float dy = torpedos[t].y - submarines[s].y;
      float distance = sqrt(dx * dx + dy * dy);
      
      if (distance < 15) { // Hit
        torpedos[t].active = false;
        submarines[s].alive = false;
        playExplosionSound();
        
        if (s == playerIndex) {
          gameOver = true;
        }
        break;
      }
    }
  }
  
  // Torpedo vs Torpedo collisions
  for(int t1 = 0; t1 < MAX_TORPEDOS; t1++) {
    if (!torpedos[t1].active) continue;
    
    for(int t2 = t1 + 1; t2 < MAX_TORPEDOS; t2++) {
      if (!torpedos[t2].active) continue;
      
      float dx = torpedos[t1].x - torpedos[t2].x;
      float dy = torpedos[t1].y - torpedos[t2].y;
      float distance = sqrt(dx * dx + dy * dy);
      
      if (distance < 8) {
        torpedos[t1].active = false;
        torpedos[t2].active = false;
        playCollisionSound();
        break;
      }
    }
  }
  
  // Submarine vs Submarine collisions
  for(int s1 = 1; s1 <= MAX_SUBMARINES; s1++) {
    if (!submarines[s1].alive) continue;
    
    for(int s2 = s1 + 1; s2 <= MAX_SUBMARINES; s2++) {
      if (!submarines[s2].alive) continue;
      
      float dx = submarines[s1].x - submarines[s2].x;
      float dy = submarines[s1].y - submarines[s2].y;
      float distance = sqrt(dx * dx + dy * dy);
      
      if (distance < 20) {
        submarines[s1].alive = false;
        submarines[s2].alive = false;
        playExplosionSound();
      }
    }
  }
}

void updateEnemyFiring() {
  // Only one enemy fires at a time, with delays
  if (millis() - lastEnemyFire > 3000) { // 3 second minimum between enemy fires
    
    // Find next living enemy to fire
    for(int attempts = 0; attempts < MAX_SUBMARINES; attempts++) {
      currentEnemyFirer++;
      if (currentEnemyFirer > MAX_SUBMARINES) currentEnemyFirer = 1;
      
      if (submarines[currentEnemyFirer].alive && submarines[currentEnemyFirer].torpedos > 0) {
        // Calculate predicted player position
        Submarine &enemy = submarines[currentEnemyFirer];
        Submarine &player = submarines[playerIndex];
        
        float dx = player.x - enemy.x;
        float dy = player.y - enemy.y;
        float distance = sqrt(dx * dx + dy * dy);
        float timeToReach = distance / (torpedoSpeed * 0.1);
        
        // Predict where player will be
        float playerRadians = player.angle * PI / 180.0;
        float playerSpeed = (player.speed > 0) ? player.speed * 25 * 0.1 : player.speed * 15 * 0.1;
        float predictedX = player.x + cos(playerRadians) * playerSpeed * timeToReach;
        float predictedY = player.y + sin(playerRadians) * playerSpeed * timeToReach;
        
        // Aim at predicted position
        float aimDx = predictedX - enemy.x;
        float aimDy = predictedY - enemy.y;
        float aimAngle = atan2(aimDy, aimDx) * 180.0 / PI;
        if (aimAngle < 0) aimAngle += 360;
        
        enemy.angle = aimAngle;
        fireTorpedo(currentEnemyFirer);
        lastEnemyFire = millis();
        break;
      }
    }
  }
}

void updateGame() {
  if (gameOver) return;
  
  updatePlayerMovement();
  updateEnemyAI();
  updateTorpedos();
  updateEnemyFiring();
  checkCollisions();
  
  // Check for fire button
  if (digitalRead(BTN_FIRE) == LOW && submarines[playerIndex].alive) {
    if (millis() - submarines[playerIndex].lastFire > 500) { // 500ms cooldown
      fireTorpedo(playerIndex);
      submarines[playerIndex].lastFire = millis();
    }
  }
  
  // Check win condition
  bool enemiesAlive = false;
  for(int i = 1; i <= MAX_SUBMARINES; i++) {
    if (submarines[i].alive) {
      enemiesAlive = true;
      break;
    }
  }
  
  if (!enemiesAlive) {
    gameOver = true; // Player wins
  }
}

void drawSubmarine(Submarine &sub) {
  // Color based on depth and type
  uint16_t subColor;
  if (sub.isPlayer) {
    subColor = 0x07E0; // Green for player
  } else {
    subColor = 0xF800; // Red for enemies
  }
  
  // Adjust brightness based on depth
  float brightness = 1.0 - sub.depth * 0.5;
  
  // Draw submarine body
  float radians = sub.angle * PI / 180.0;
  float cos_a = cos(radians);
  float sin_a = sin(radians);
  
  // Main body
  int x1 = sub.x - cos_a * 8;
  int y1 = sub.y - sin_a * 8;
  int x2 = sub.x + cos_a * 8;
  int y2 = sub.y + sin_a * 8;
  
  canvas.drawLine(x1, y1, x2, y2, subColor);
  canvas.drawCircle(sub.x, sub.y, 3, subColor);
  
  // Direction indicator
  int front_x = sub.x + cos_a * 12;
  int front_y = sub.y + sin_a * 12;
  canvas.drawLine(sub.x, sub.y, front_x, front_y, 0xFFFF);
}

void drawTorpedo(Torpedo &torpedo) {
  uint16_t color = torpedo.isPlayer ? 0xFFE0 : 0xF81F; // Yellow for player, Magenta for enemy
  
  // Draw torpedo as a small line
  float angle = atan2(torpedo.dy, torpedo.dx);
  int x1 = torpedo.x - cos(angle) * 2;
  int y1 = torpedo.y - sin(angle) * 2;
  int x2 = torpedo.x + cos(angle) * 2;
  int y2 = torpedo.y + sin(angle) * 2;
  
  canvas.drawLine(x1, y1, x2, y2, color);
  canvas.drawPixel(torpedo.x, torpedo.y, 0xFFFF);
}

void drawGame() {
  canvas.fillScreen(0x001F); // Dark blue sea
  
  // Draw grid lines for reference
  for(int x = 0; x < SCREEN_WIDTH; x += 40) {
    canvas.drawLine(x, 0, x, SCREEN_HEIGHT, 0x0010);
  }
  for(int y = 0; y < SCREEN_HEIGHT; y += 40) {
    canvas.drawLine(0, y, SCREEN_WIDTH, y, 0x0010);
  }
  
  // Draw submarines
  for(int i = 0; i <= MAX_SUBMARINES; i++) {
    if (submarines[i].alive) {
      drawSubmarine(submarines[i]);
    }
  }
  
  // Draw torpedos
  for(int i = 0; i < MAX_TORPEDOS; i++) {
    if (torpedos[i].active) {
      drawTorpedo(torpedos[i]);
    }
  }
  
  // Draw UI
  canvas.setTextColor(0xFFFF);
  canvas.setTextSize(1);
  
  // Speed and direction display
  canvas.setCursor(5, 5);
  canvas.printf("Speed: %d", currentSpeed);
  canvas.setCursor(5, 15);
  canvas.printf("Dir: %.0f°", submarines[playerIndex].angle);
  canvas.setCursor(5, 25);
  canvas.printf("Torpedos: %d", submarines[playerIndex].torpedos);
  
  // Enemy count
  int enemyCount = 0;
  for(int i = 1; i <= MAX_SUBMARINES; i++) {
    if (submarines[i].alive) enemyCount++;
  }
  canvas.setCursor(SCREEN_WIDTH - 80, 5);
  canvas.printf("Enemies: %d", enemyCount);
  
  if (gameOver) {
    canvas.setTextSize(2);
    canvas.setCursor(SCREEN_WIDTH/2 - 50, SCREEN_HEIGHT/2);
    if (submarines[playerIndex].alive) {
      canvas.print("YOU WIN!");
    } else {
      canvas.print("GAME OVER");
    }
    canvas.setTextSize(1);
    canvas.setCursor(SCREEN_WIDTH/2 - 60, SCREEN_HEIGHT/2 + 30);
    canvas.print("Press SPEED UP to restart");
  }
  
  canvas.pushSprite(0, 0);
}

void playTorpedoSound() {
  ledcWriteTone(LEDC_CHANNEL, 800);
  delay(50);
  ledcWriteTone(LEDC_CHANNEL, 0);
}

void playExplosionSound() {
  for(int i = 0; i < 10; i++) {
    ledcWriteTone(LEDC_CHANNEL, random(100, 1000));
    delay(20);
  }
  ledcWriteTone(LEDC_CHANNEL, 0);
}

void playCollisionSound() {
  ledcWriteTone(LEDC_CHANNEL, 400);
  delay(100);
  ledcWriteTone(LEDC_CHANNEL, 0);
}

void loop() {
  if (gameOver && digitalRead(BTN_SPEED_UP) == LOW) {
    initGame();
    delay(500);
  }
  
  updateGame();
  drawGame();
  delay(50); // ~20 FPS
}
```

### 📋 Instalacija

1. Otvorite Arduino IDE
2. Instalirajte LovyanGFX biblioteku (Tools → Manage Libraries)
3. Kopirajte kod u novi sketch
4. Odaberite ESP32 board
5. Uploadajte na VIDI X mikroračunalo
