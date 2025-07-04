# Perplexity.ai s Claude Sonet 4 Razmišljanje modelom
[https://www.perplexity.ai/search/razmisli-o-detaljnim-uputama-z-jufAjMoSSlGOQTA4kE2heQ](https://www.perplexity.ai/search/razmisli-o-detaljnim-uputama-z-jufAjMoSSlGOQTA4kE2heQ)

<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" class="logo" width="120"/>

Analizirajući vaše zahtjeve za igru podmornica na VIDI X mikroračunalu, evo kompletnog koda koji možete copy-paste u Arduino IDE:

```cpp
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
#define UI_HEIGHT 35

// GPIO definicije
#define BTN_LR       34  // Lijevo/Desno (analogni)
#define BTN_UD       35  // Zaroni/Izroni (analogni)
#define BTN_SPEED_UP 0   // Povećaj brzinu
#define BTN_SPEED_DN 13  // Smanji brzinu
#define BTN_FIRE     32  // Ispali torpedo
#define BTN_TORP_L   27  // Torpedo lijevo
#define BTN_TORP_R   39  // Torpedo desno
#define BTN_RESTART  33  // Restart igre

// Konstante igre
#define MAX_ENEMIES 10
#define MAX_TORPEDOES 20
#define TORPEDO_SPEED 200
#define MAX_PLAYER_SPEED 100
#define TURN_ANGLE 5
#define DIVE_SPEED 2

// Boje
#define COLOR_WATER 0x001F
#define COLOR_SUBMARINE 0xFFE0
#define COLOR_ENEMY 0xF800
#define COLOR_TORPEDO 0xFFFF
#define COLOR_UI_BG 0x18E3
#define COLOR_TEXT 0xFFFF

struct Torpedo {
  float x, y;
  float dx, dy;
  float angle;
  bool active;
  bool isPlayer;
  bool isGuided;
  unsigned long fireTime;
};

struct Submarine {
  float x, y;
  float angle;
  float speed;
  float depth;
  float targetDepth;
  int torpedoCount;
  bool active;
  bool isPlayer;
  unsigned long lastFire;
  unsigned long lastTurn;
  float targetX, targetY;
};

// Globalne varijable
Submarine player;
Submarine enemies[MAX_ENEMIES];
Torpedo torpedoes[MAX_TORPEDOES];
int playerSpeedLevel = 2; // 0=unazad, 1=stop, 2=sporije, 3=brže, 4=najbrže
bool gameOver = false;
int score = 0;
int enemiesAlive = MAX_ENEMIES;
unsigned long lastEnemyFire = 0;
int nextEnemyToFire = 0;

// Funkcije
void initGame();
void updatePlayer();
void updateEnemies();
void updateTorpedoes();
void drawGame();
void drawSubmarine(Submarine &sub);
void drawTorpedo(Torpedo &torp);
void fireTorpedo(Submarine &sub, bool isGuided = false);
void checkCollisions();
void playSound(int freq, int duration);
bool isInWater(float x, float y);
float calculateAngle(float x1, float y1, float x2, float y2);
void predictPlayerPosition(float &predX, float &predY, float enemyX, float enemyY);

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.setRotation(0);
  lcd.fillScreen(COLOR_WATER);
  
  canvas.setPsram(true);
  bool ok = canvas.createSprite(SCREEN_WIDTH, SCREEN_HEIGHT);
  Serial.println(ok ? "Canvas created!" : "Canvas FAIL!");
  if (!ok) while(1) delay(1000);
  
  // Setup speaker
  ledcSetup(LEDC_CHANNEL, LEDC_BASE_FREQ, LEDC_RESOLUTION);
  ledcAttachPin(SPEAKER_PIN, LEDC_CHANNEL);
  
  // Setup pins
  pinMode(BTN_LR, INPUT);
  pinMode(BTN_UD, INPUT);
  pinMode(BTN_SPEED_UP, INPUT_PULLUP);
  pinMode(BTN_SPEED_DN, INPUT_PULLUP);
  pinMode(BTN_FIRE, INPUT_PULLUP);
  pinMode(BTN_TORP_L, INPUT_PULLUP);
  pinMode(BTN_TORP_R, INPUT_PULLUP);
  pinMode(BTN_RESTART, INPUT_PULLUP);
  
  initGame();
}

void initGame() {
  gameOver = false;
  score = 0;
  enemiesAlive = MAX_ENEMIES;
  playerSpeedLevel = 2;
  
  // Postavke igrača
  player.x = SCREEN_WIDTH / 2;
  player.y = SCREEN_HEIGHT / 2;
  player.angle = 0;
  player.speed = 0;
  player.depth = 50;
  player.targetDepth = 50;
  player.torpedoCount = 10;
  player.active = true;
  player.isPlayer = true;
  player.lastFire = 0;
  
  // Postavke neprijatelja
  for (int i = 0; i < MAX_ENEMIES; i++) {
    enemies[i].x = random(50, SCREEN_WIDTH - 50);
    enemies[i].y = random(50, SCREEN_HEIGHT - 50);
    enemies[i].angle = random(0, 360);
    enemies[i].speed = random(20, MAX_PLAYER_SPEED);
    enemies[i].depth = random(20, 100);
    enemies[i].targetDepth = enemies[i].depth;
    enemies[i].torpedoCount = 10;
    enemies[i].active = true;
    enemies[i].isPlayer = false;
    enemies[i].lastFire = 0;
    enemies[i].lastTurn = 0;
    enemies[i].targetX = random(50, SCREEN_WIDTH - 50);
    enemies[i].targetY = random(50, SCREEN_HEIGHT - 50);
  }
  
  // Očisti torpeda
  for (int i = 0; i < MAX_TORPEDOES; i++) {
    torpedoes[i].active = false;
  }
  
  lastEnemyFire = 0;
  nextEnemyToFire = 0;
}

void updatePlayer() {
  // Čitaj analogne gumbove
  int analogLR = analogRead(BTN_LR);
  int analogUD = analogRead(BTN_UD);
  
  // Upravljanje smjerom (lijevo/desno)
  static unsigned long lastTurn = 0;
  if (millis() - lastTurn > 100) {
    if (analogLR > 4000) { // Lijevo
      player.angle -= TURN_ANGLE;
      if (player.angle < 0) player.angle += 360;
      lastTurn = millis();
    }
    if (analogLR > 1800 && analogLR < 2200) { // Desno
      player.angle += TURN_ANGLE;
      if (player.angle >= 360) player.angle -= 360;
      lastTurn = millis();
    }
  }
  
  // Upravljanje dubinom (zaroni/izroni)
  if (analogUD > 4000) { // Zaroni
    player.targetDepth += DIVE_SPEED;
    if (player.targetDepth > 100) player.targetDepth = 100;
  }
  if (analogUD > 1800 && analogUD < 2200) { // Izroni
    player.targetDepth -= DIVE_SPEED;
    if (player.targetDepth < 0) player.targetDepth = 0;
  }
  
  // Postepeno zaranjanje/izranjanje
  if (player.depth < player.targetDepth) {
    player.depth += 0.5;
    if (player.depth > player.targetDepth) player.depth = player.targetDepth;
  } else if (player.depth > player.targetDepth) {
    player.depth -= 0.5;
    if (player.depth < player.targetDepth) player.depth = player.targetDepth;
  }
  
  // Upravljanje brzinom
  static bool lastSpeedUp = HIGH, lastSpeedDn = HIGH;
  bool nowSpeedUp = digitalRead(BTN_SPEED_UP);
  bool nowSpeedDn = digitalRead(BTN_SPEED_DN);
  
  if (lastSpeedUp == HIGH && nowSpeedUp == LOW) {
    playerSpeedLevel++;
    if (playerSpeedLevel > 4) playerSpeedLevel = 4;
    playSound(800, 50);
  }
  if (lastSpeedDn == HIGH && nowSpeedDn == LOW) {
    playerSpeedLevel--;
    if (playerSpeedLevel < 0) playerSpeedLevel = 0;
    playSound(600, 50);
  }
  
  lastSpeedUp = nowSpeedUp;
  lastSpeedDn = nowSpeedDn;
  
  // Postavljanje brzine prema stupnju
  switch (playerSpeedLevel) {
    case 0: player.speed = -25; break;  // Unazad
    case 1: player.speed = 0; break;    // Stop
    case 2: player.speed = 25; break;   // Sporije
    case 3: player.speed = 50; break;   // Brže
    case 4: player.speed = MAX_PLAYER_SPEED; break; // Najbrže
  }
  
  // Pomakni igrača
  float angleRad = player.angle * PI / 180.0;
  player.x += cos(angleRad) * player.speed * 0.02;
  player.y += sin(angleRad) * player.speed * 0.02;
  
  // Provjeri granice
  if (player.x < 10) player.x = 10;
  if (player.x > SCREEN_WIDTH - 10) player.x = SCREEN_WIDTH - 10;
  if (player.y < UI_HEIGHT + 10) player.y = UI_HEIGHT + 10;
  if (player.y > SCREEN_HEIGHT - 10) player.y = SCREEN_HEIGHT - 10;
  
  // Ispaljivanje torpeda
  static bool lastFire = HIGH;
  bool nowFire = digitalRead(BTN_FIRE);
  if (lastFire == HIGH && nowFire == LOW && player.torpedoCount > 0) {
    fireTorpedo(player, true);
    playSound(1000, 100);
  }
  lastFire = nowFire;
}

void updateEnemies() {
  for (int i = 0; i < MAX_ENEMIES; i++) {
    if (!enemies[i].active) continue;
    
    // Jednostavna AI - krećemo se prema cilju
    float dx = enemies[i].targetX - enemies[i].x;
    float dy = enemies[i].targetY - enemies[i].y;
    float dist = sqrt(dx * dx + dy * dy);
    
    if (dist < 20 || millis() - enemies[i].lastTurn > 5000) {
      enemies[i].targetX = random(50, SCREEN_WIDTH - 50);
      enemies[i].targetY = random(50, SCREEN_HEIGHT - 50);
      enemies[i].lastTurn = millis();
    }
    
    // Okreni prema cilju
    float targetAngle = atan2(dy, dx) * 180 / PI;
    if (targetAngle < 0) targetAngle += 360;
    
    float angleDiff = targetAngle - enemies[i].angle;
    if (angleDiff > 180) angleDiff -= 360;
    if (angleDiff < -180) angleDiff += 360;
    
    if (abs(angleDiff) > 5) {
      enemies[i].angle += (angleDiff > 0) ? 2 : -2;
      if (enemies[i].angle < 0) enemies[i].angle += 360;
      if (enemies[i].angle >= 360) enemies[i].angle -= 360;
    }
    
    // Pomakni neprijatelja
    float angleRad = enemies[i].angle * PI / 180.0;
    enemies[i].x += cos(angleRad) * enemies[i].speed * 0.02;
    enemies[i].y += sin(angleRad) * enemies[i].speed * 0.02;
    
    // Provjeri granice
    if (enemies[i].x < 10) enemies[i].x = 10;
    if (enemies[i].x > SCREEN_WIDTH - 10) enemies[i].x = SCREEN_WIDTH - 10;
    if (enemies[i].y < UI_HEIGHT + 10) enemies[i].y = UI_HEIGHT + 10;
    if (enemies[i].y > SCREEN_HEIGHT - 10) enemies[i].y = SCREEN_HEIGHT - 10;
    
    // Provjeri kolizije s drugim neprijateljima
    for (int j = i + 1; j < MAX_ENEMIES; j++) {
      if (!enemies[j].active) continue;
      
      float dx = enemies[i].x - enemies[j].x;
      float dy = enemies[i].y - enemies[j].y;
      float dist = sqrt(dx * dx + dy * dy);
      
      if (dist < 20) {
        enemies[i].active = false;
        enemies[j].active = false;
        enemiesAlive -= 2;
        score += 200;
        playSound(400, 200);
      }
    }
  }
  
  // Neprijateljsko pucanje - jedan po jedan
  if (millis() - lastEnemyFire > 2000) {
    for (int attempts = 0; attempts < MAX_ENEMIES; attempts++) {
      int enemyIndex = (nextEnemyToFire + attempts) % MAX_ENEMIES;
      if (enemies[enemyIndex].active && enemies[enemyIndex].torpedoCount > 0) {
        fireTorpedo(enemies[enemyIndex], false);
        lastEnemyFire = millis();
        nextEnemyToFire = (enemyIndex + 1) % MAX_ENEMIES;
        break;
      }
    }
  }
}

void updateTorpedoes() {
  // Upravljanje igračevim torpedom
  static bool lastTorpL = HIGH, lastTorpR = HIGH;
  bool nowTorpL = digitalRead(BTN_TORP_L);
  bool nowTorpR = digitalRead(BTN_TORP_R);
  
  for (int i = 0; i < MAX_TORPEDOES; i++) {
    if (!torpedoes[i].active) continue;
    
    // Upravljanje igračevim torpedom
    if (torpedoes[i].isPlayer && torpedoes[i].isGuided) {
      static unsigned long lastTorpTurn = 0;
      if (millis() - lastTorpTurn > 50) {
        if (lastTorpL == HIGH && nowTorpL == LOW) {
          torpedoes[i].angle -= 10;
          if (torpedoes[i].angle < 0) torpedoes[i].angle += 360;
          lastTorpTurn = millis();
        }
        if (lastTorpR == HIGH && nowTorpR == LOW) {
          torpedoes[i].angle += 10;
          if (torpedoes[i].angle >= 360) torpedoes[i].angle -= 360;
          lastTorpTurn = millis();
        }
      }
      
      // Ažuriraj smjer
      float angleRad = torpedoes[i].angle * PI / 180.0;
      torpedoes[i].dx = cos(angleRad) * TORPEDO_SPEED;
      torpedoes[i].dy = sin(angleRad) * TORPEDO_SPEED;
    }
    
    // Pomakni torpedo
    torpedoes[i].x += torpedoes[i].dx * 0.02;
    torpedoes[i].y += torpedoes[i].dy * 0.02;
    
    // Provjeri granice
    if (torpedoes[i].x < 0 || torpedoes[i].x > SCREEN_WIDTH || 
        torpedoes[i].y < UI_HEIGHT || torpedoes[i].y > SCREEN_HEIGHT) {
      torpedoes[i].active = false;
    }
    
    // Timeout za torpeda
    if (millis() - torpedoes[i].fireTime > 10000) {
      torpedoes[i].active = false;
    }
  }
  
  lastTorpL = nowTorpL;
  lastTorpR = nowTorpR;
}

void checkCollisions() {
  for (int i = 0; i < MAX_TORPEDOES; i++) {
    if (!torpedoes[i].active) continue;
    
    // Torpedo vs podmornica
    if (torpedoes[i].isPlayer) {
      // Igračev torpedo vs neprijatelj
      for (int j = 0; j < MAX_ENEMIES; j++) {
        if (!enemies[j].active) continue;
        
        float dx = torpedoes[i].x - enemies[j].x;
        float dy = torpedoes[i].y - enemies[j].y;
        float dist = sqrt(dx * dx + dy * dy);
        
        if (dist < 15) {
          torpedoes[i].active = false;
          enemies[j].active = false;
          enemiesAlive--;
          score += 100;
          playSound(1200, 150);
        }
      }
    } else {
      // Neprijateljski torpedo vs igrač
      float dx = torpedoes[i].x - player.x;
      float dy = torpedoes[i].y - player.y;
      float dist = sqrt(dx * dx + dy * dy);
      
      if (dist < 15) {
        torpedoes[i].active = false;
        gameOver = true;
        playSound(200, 500);
      }
      
      // Neprijateljski torpedo vs neprijatelj
      for (int j = 0; j < MAX_ENEMIES; j++) {
        if (!enemies[j].active) continue;
        
        float dx = torpedoes[i].x - enemies[j].x;
        float dy = torpedoes[i].y - enemies[j].y;
        float dist = sqrt(dx * dx + dy * dy);
        
        if (dist < 15) {
          torpedoes[i].active = false;
          enemies[j].active = false;
          enemiesAlive--;
          score += 50;
          playSound(800, 100);
        }
      }
    }
    
    // Torpedo vs torpedo
    for (int j = i + 1; j < MAX_TORPEDOES; j++) {
      if (!torpedoes[j].active) continue;
      
      float dx = torpedoes[i].x - torpedoes[j].x;
      float dy = torpedoes[i].y - torpedoes[j].y;
      float dist = sqrt(dx * dx + dy * dy);
      
      if (dist < 8) {
        torpedoes[i].active = false;
        torpedoes[j].active = false;
        playSound(600, 80);
      }
    }
  }
  
  // Provjeri pobjed
  if (enemiesAlive == 0) {
    gameOver = true;
    score += 1000;
  }
}

void fireTorpedo(Submarine &sub, bool isGuided) {
  if (sub.torpedoCount <= 0) return;
  
  for (int i = 0; i < MAX_TORPEDOES; i++) {
    if (!torpedoes[i].active) {
      torpedoes[i].x = sub.x;
      torpedoes[i].y = sub.y;
      torpedoes[i].angle = sub.angle;
      torpedoes[i].isPlayer = sub.isPlayer;
      torpedoes[i].isGuided = isGuided;
      torpedoes[i].fireTime = millis();
      torpedoes[i].active = true;
      
      if (!isGuided && !sub.isPlayer) {
        // Predvidi poziciju igrača
        float predX, predY;
        predictPlayerPosition(predX, predY, sub.x, sub.y);
        torpedoes[i].angle = calculateAngle(sub.x, sub.y, predX, predY);
      }
      
      float angleRad = torpedoes[i].angle * PI / 180.0;
      torpedoes[i].dx = cos(angleRad) * TORPEDO_SPEED;
      torpedoes[i].dy = sin(angleRad) * TORPEDO_SPEED;
      
      sub.torpedoCount--;
      break;
    }
  }
}

void predictPlayerPosition(float &predX, float &predY, float enemyX, float enemyY) {
  float dx = player.x - enemyX;
  float dy = player.y - enemyY;
  float dist = sqrt(dx * dx + dy * dy);
  float timeToHit = dist / (TORPEDO_SPEED * 0.02);
  
  float playerAngleRad = player.angle * PI / 180.0;
  predX = player.x + cos(playerAngleRad) * player.speed * 0.02 * timeToHit;
  predY = player.y + sin(playerAngleRad) * player.speed * 0.02 * timeToHit;
  
  // Ograniči predviđanje na ekran
  if (predX < 10) predX = 10;
  if (predX > SCREEN_WIDTH - 10) predX = SCREEN_WIDTH - 10;
  if (predY < UI_HEIGHT + 10) predY = UI_HEIGHT + 10;
  if (predY > SCREEN_HEIGHT - 10) predY = SCREEN_HEIGHT - 10;
}

float calculateAngle(float x1, float y1, float x2, float y2) {
  float angle = atan2(y2 - y1, x2 - x1) * 180 / PI;
  if (angle < 0) angle += 360;
  return angle;
}

void drawGame() {
  canvas.fillScreen(COLOR_WATER);
  
  // Crtaj UI
  canvas.fillRect(0, 0, SCREEN_WIDTH, UI_HEIGHT, COLOR_UI_BG);
  canvas.setTextColor(COLOR_TEXT);
  canvas.setTextSize(1);
  
  // Informacije o igraču
  canvas.setCursor(5, 5);
  canvas.printf("Brzina: %d", playerSpeedLevel);
  canvas.setCursor(5, 15);
  canvas.printf("Smjer: %.0f°", player.angle);
  canvas.setCursor(5, 25);
  canvas.printf("Dubina: %.0f", player.depth);
  
  canvas.setCursor(80, 5);
  canvas.printf("Torpeda: %d", player.torpedoCount);
  canvas.setCursor(80, 15);
  canvas.printf("Bodovi: %d", score);
  canvas.setCursor(80, 25);
  canvas.printf("Neprijatelja: %d", enemiesAlive);
  
  // Crtaj igrača
  drawSubmarine(player);
  
  // Crtaj neprijatelje
  for (int i = 0; i < MAX_ENEMIES; i++) {
    if (enemies[i].active) {
      drawSubmarine(enemies[i]);
    }
  }
  
  // Crtaj torpeda
  for (int i = 0; i < MAX_TORPEDOES; i++) {
    if (torpedoes[i].active) {
      drawTorpedo(torpedoes[i]);
    }
  }
  
  canvas.pushSprite(0, 0);
}

void drawSubmarine(Submarine &sub) {
  uint16_t color = sub.isPlayer ? COLOR_SUBMARINE : COLOR_ENEMY;
  
  if (sub.isPlayer) {
    // Crtaj igrača kao X
    canvas.drawLine(sub.x - 5, sub.y - 5, sub.x + 5, sub.y + 5, color);
    canvas.drawLine(sub.x - 5, sub.y + 5, sub.x + 5, sub.y - 5, color);
    canvas.drawLine(sub.x - 6, sub.y - 6, sub.x + 6, sub.y + 6, color);
    canvas.drawLine(sub.x - 6, sub.y + 6, sub.x + 6, sub.y - 6, color);
  } else {
    // Crtaj neprijatelje kao kružići
    canvas.fillCircle(sub.x, sub.y, 4, color);
    canvas.drawCircle(sub.x, sub.y, 5, 0xFFFF);
  }
  
  // Crtaj indikator smjera
  float angleRad = sub.angle * PI / 180.0;
  float endX = sub.x + cos(angleRad) * 8;
  float endY = sub.y + sin(angleRad) * 8;
  canvas.drawLine(sub.x, sub.y, endX, endY, 0xFFFF);
}

void drawTorpedo(Torpedo &torp) {
  uint16_t color = torp.isPlayer ? COLOR_TORPEDO : COLOR_ENEMY;
  
  // Crtaj torpedo kao crtu duljine 4 piksela
  float angleRad = torp.angle * PI / 180.0;
  float endX = torp.x + cos(angleRad) * 4;
  float endY = torp.y + sin(angleRad) * 4;
  
  canvas.drawLine(torp.x, torp.y, endX, endY, color);
  canvas.drawLine(torp.x - 1, torp.y, endX - 1, endY, color);
}

void showGameOver() {
  canvas.fillScreen(0x0000);
  canvas.setTextColor(COLOR_TEXT);
  canvas.setTextSize(2);
  
  if (enemiesAlive == 0) {
    canvas.setCursor(110, 100);
    canvas.print("POBJEDA!");
  } else {
    canvas.setCursor(90, 100);
    canvas.print("GAME OVER");
  }
  
  canvas.setTextSize(1);
  canvas.setCursor(100, 140);
  canvas.printf("Konačni rezultat: %d", score);
  canvas.setCursor(80, 160);
  canvas.print("Pritisnite restart za novu igru");
  
  canvas.pushSprite(0, 0);
  
  if (digitalRead(BTN_RESTART) == LOW) {
    delay(200);
    initGame();
  }
}

void playSound(int freq, int duration) {
  ledcWriteTone(LEDC_CHANNEL, freq);
  delay(duration);
  ledcWriteTone(LEDC_CHANNEL, 0);
}

void loop() {
  if (!gameOver) {
    updatePlayer();
    updateEnemies();
    updateTorpedoes();
    checkCollisions();
    drawGame();
    delay(20); // ~50 FPS
  } else {
    showGameOver();
    delay(100);
  }
}
```

**Objašnjenje koda:**

**Upravljanje:**

- **GPIO34/35** - Analogne vrijednosti za smjer i dubinu
- **GPIO0/13** - Mijenjanje brzine (5 razina)
- **GPIO32** - Ispaljivanje torpeda
- **GPIO27/39** - Upravljanje torpedom lijevo/desno

**Značajke igre:**

- **Igrač** se prikazuje kao **X** na karti
- **Neprijatelje** prikazuje kao **kružiće**
- **Torpeda** su **crte duljine 4 piksela**
- **Brzina** ima 5 stupnjeva (unazad, stop, 3 naprijed)
- **Zaranjanje/izranjanje** je postupno
- **Neprijatelji** mogu predvidjeti vašu putanju
- **Kolizije** između svih objekata su implementirane
- **Samo jedan neprijatelj** puca u isto vrijeme

**UI elementi:**

- Brzina, smjer i dubina se prikazuju
- Broj torpeda i bodovi
- Broj preživjelih neprijatelja

Kod koristi **LovyanGFX** biblioteku za grafiku kao u vašem primjeru, implementira **analogno čitanje gumbova** na GPIO34/35, te sadrži kompletnu logiku igre prema vašim specifikacijama.
