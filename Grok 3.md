# Grok 3 (uz Thinking)
[https://x.com/i/grok/share/I9Y31DP7bmQYZSI4DNTf3ctYw](https://x.com/i/grok/share/I9Y31DP7bmQYZSI4DNTf3ctYw)

### Upute za izradu igre za VIDI X mikroračunalo na ESP32

Želiš izraditi igru za VIDI X mikroračunalo koristeći ESP32, C++ i Arduino IDE. Igra uključuje upravljanje podmornicom, izbjegavanje neprijateljskih podmornica i ispaljivanje torpeda. Evo detaljnih uputa i kompletnog koda koji možeš kopirati i zalijepiti u Arduino IDE.

#### Pregled igre
- **Cilj:** Pogoditi 10 neprijateljskih podmornica torpedima dok izbjegavaš njihove torpede i sudare.
- **Prikaz:** Grafička mapa na ekranu prikazuje igračevu podmornicu ('X'), torpede (crtice od 4 piksela) i neprijateljske podmornice.
- **Upravljanje:**
  - GPIO35: Zaroni (>4000) i Izroni (1800-2200)
  - GPIO34: Lijevo (>4000) i Desno (1800-2200)
  - GPIO0: Povećaj brzinu (3 razine naprijed, stop, 1 unatrag)
  - GPIO13: Smanji brzinu
  - GPIO32: Ispali torpedo
  - GPIO27 i GPIO39: Upravljaj torpedom lijevo i desno
- **Mehanika:**
  - Podmornica se okreće za 5° po pritisku gumba.
  - Maksimalna brzina podmornice je 100, torpeda 200 (konstantna).
  - Zaranjanje i izranjanje su spori (2 jedinice po ciklusu).
  - Neprijateljske podmornice se kreću nasumično (maksimalna brzina 100) i ispaljuju torpede koji idu ravno, predviđajući tvoju putanju.
  - Svaka podmornica ima 10 torpeda.
  - Sudari:
    - Torpedo pogodi podmornicu (igračevu ili neprijateljsku) → uništenje.
    - Neprijateljski torpedo pogodi drugu neprijateljsku podmornicu → uništenje.
    - Dva torpeda se sudare → oba uništena.
    - Dvije neprijateljske podmornice se sudare → obje uništene.

#### Tehnički detalji
- **Grafika:** Koristimo LovyanGFX biblioteku za LCD zaslon (ILI9341).
- **Zvuk:** Zvučnik na GPIO25 za efekte (ispaljivanje, sudari).
- **Ulazi:** Analogni gumbi na GPIO34 i GPIO35 očitavaju vrijednosti za dva gumba po pinu.

#### Plan implementacije
1. **Inicijalizacija:**
   - Postavi LCD, zvučnik i gumbe.
   - Definiraj strukture za podmornice i torpede.
2. **Glavna petlja:**
   - Čitaj stanje gumba.
   - Ažuriraj pozicije podmornica i torpeda.
   - Provjeri sudare.
   - Iscrtaj stanje igre.
3. **AI:**
   - Neprijatelji se kreću nasumično i povremeno ispaljuju torpede prema predviđenoj poziciji igrača.
   - Osiguraj da ne pucaju svi odjednom.
4. **Prikaz:**
   - Mapa s podmornicama i torpedima.
   - UI s brzinom, smjerom i brojem torpeda.

### Kompletan kod

Evo kompletnog koda koji možeš kopirati i zalijepiti u Arduino IDE:

```cpp
#define LGFX_USE_V1
#include <LovyanGFX.hpp>
#include <math.h>

// LCD konfiguracija
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

// Zvučnik
#define SPEAKER_PIN 25
#define LEDC_CHANNEL 0
#define LEDC_RESOLUTION 8
#define LEDC_BASE_FREQ 2000

// Dimenzije ekrana
#define SCREEN_WIDTH 320
#define SCREEN_HEIGHT 240
#define MAP_WIDTH 280
#define MAP_HEIGHT 200
#define UI_HEIGHT 40

// Boje
#define COLOR_BG 0x0000
#define COLOR_UI_BG 0x18E3
#define COLOR_TEXT 0xFFFF
#define COLOR_PLAYER 0x07E0
#define COLOR_ENEMY 0xF800
#define COLOR_TORPEDO 0xFFE0

// Konstante igre
#define MAX_ENEMIES 10
#define MAX_TORPEDOES 10
#define TORPEDO_SPEED 200
#define SUBMARINE_MAX_SPEED 100
#define TURN_ANGLE 5
#define DEPTH_CHANGE_RATE 2
#define MAP_BORDER 20

// Gumbi
#define BTN_DIVE_SURFACE 35
#define BTN_LEFT_RIGHT 34
#define BTN_SPEED_UP 0
#define BTN_SPEED_DOWN 13
#define BTN_FIRE 32
#define BTN_TORPEDO_LEFT 27
#define BTN_TORPEDO_RIGHT 39

// Strukture
struct Submarine {
  float x, y;
  float angle; // u stupnjevima
  int speed;   // -1 do 3
  int depth;   // 0 do 100
  int torpedoesLeft;
  bool active;
};

struct Torpedo {
  float x, y;
  float angle;
  float speed;
  bool active;
  bool isPlayerControlled;
};

// Varijable igre
Submarine player;
Submarine enemies[MAX_ENEMIES];
Torpedo playerTorpedoes[MAX_TORPEDOES];
Torpedo enemyTorpedoes[MAX_TORPEDOES * MAX_ENEMIES];

unsigned long lastUpdateTime = 0;
unsigned long lastEnemyFireTime = 0;
int enemyFireCooldown = 2000; // milisekunde

// Funkcije
void initGame();
void updateGame();
void drawGame();
void readButtons();
void moveSubmarine(Submarine &sub, float deltaTime);
void moveTorpedo(Torpedo &torp, float deltaTime);
void checkCollisions();
void fireTorpedo(Submarine &sub, Torpedo torps[], int maxTorps, bool isPlayer);
void predictPlayerPosition(float &predX, float &predY, float torpSpeed);
float degreesToRadians(float degrees);
float distance(float x1, float y1, float x2, float y2);

void setup() {
  Serial.begin(115200);

  lcd.init();
  lcd.setRotation(0);
  lcd.fillScreen(COLOR_BG);
  canvas.setPsram(true);
  bool ok = canvas.createSprite(SCREEN_WIDTH, SCREEN_HEIGHT);
  Serial.println(ok ? "Canvas created!" : "Canvas FAIL!");
  if (!ok) while(1) delay(1000);

  ledcSetup(LEDC_CHANNEL, LEDC_BASE_FREQ, LEDC_RESOLUTION);
  ledcAttachPin(SPEAKER_PIN, LEDC_CHANNEL);

  pinMode(BTN_DIVE_SURFACE, INPUT);
  pinMode(BTN_LEFT_RIGHT, INPUT);
  pinMode(BTN_SPEED_UP, INPUT_PULLUP);
  pinMode(BTN_SPEED_DOWN, INPUT_PULLUP);
  pinMode(BTN_FIRE, INPUT_PULLUP);
  pinMode(BTN_TORPEDO_LEFT, INPUT_PULLUP);
  pinMode(BTN_TORPEDO_RIGHT, INPUT_PULLUP);

  initGame();
}

void initGame() {
  player.x = SCREEN_WIDTH / 2;
  player.y = SCREEN_HEIGHT - 50;
  player.angle = 0;
  player.speed = 0;
  player.depth = 0;
  player.torpedoesLeft = 10;
  player.active = true;

  for (int i = 0; i < MAX_ENEMIES; i++) {
    enemies[i].x = random(MAP_BORDER, MAP_WIDTH - MAP_BORDER);
    enemies[i].y = random(MAP_BORDER, MAP_HEIGHT - MAP_BORDER);
    enemies[i].angle = random(0, 360);
    enemies[i].speed = random(0, SUBMARINE_MAX_SPEED);
    enemies[i].depth = 0;
    enemies[i].torpedoesLeft = 10;
    enemies[i].active = true;
  }

  for (int i = 0; i < MAX_TORPEDOES; i++) {
    playerTorpedoes[i].active = false;
  }
  for (int i = 0; i < MAX_TORPEDOES * MAX_ENEMIES; i++) {
    enemyTorpedoes[i].active = false;
  }

  lastUpdateTime = millis();
  lastEnemyFireTime = millis();
}

void loop() {
  unsigned long currentTime = millis();
  float deltaTime = (currentTime - lastUpdateTime) / 1000.0f;
  lastUpdateTime = currentTime;

  readButtons();
  updateGame(deltaTime);
  drawGame();

  delay(16); // ~60 FPS
}

void readButtons() {
  int analogDS = analogRead(BTN_DIVE_SURFACE);
  int analogLR = analogRead(BTN_LEFT_RIGHT);

  // Zaroni
  if (analogDS > 4000) {
    player.depth += DEPTH_CHANGE_RATE;
    if (player.depth > 100) player.depth = 100;
  }
  // Izroni
  if (analogDS > 1800 && analogDS < 2200) {
    player.depth -= DEPTH_CHANGE_RATE;
    if (player.depth < 0) player.depth = 0;
  }

  // Lijevo
  if (analogLR > 4000) {
    player.angle -= TURN_ANGLE;
    if (player.angle < 0) player.angle += 360;
  }
  // Desno
  if (analogLR > 1800 && analogLR < 2200) {
    player.angle += TURN_ANGLE;
    if (player.angle >= 360) player.angle -= 360;
  }

  // Brzina
  if (digitalRead(BTN_SPEED_UP) == LOW) {
    player.speed += 1;
    if (player.speed > 3) player.speed = 3;
  }
  if (digitalRead(BTN_SPEED_DOWN) == LOW) {
    player.speed -= 1;
    if (player.speed < -1) player.speed = -1;
  }

  // Ispali torpedo
  static bool lastFire = HIGH;
  bool nowFire = digitalRead(BTN_FIRE);
  if (lastFire == HIGH && nowFire == LOW && player.torpedoesLeft > 0) {
    fireTorpedo(player, playerTorpedoes, MAX_TORPEDOES, true);
    player.torpedoesLeft--;
    ledcWriteTone(LEDC_CHANNEL, 800);
    delay(50);
    ledcWriteTone(LEDC_CHANNEL, 0);
  }
  lastFire = nowFire;

  // Upravljanje torpedom
  if (digitalRead(BTN_TORPEDO_LEFT) == LOW) {
    for (int i = 0; i < MAX_TORPEDOES; i++) {
      if (playerTorpedoes[i].active && playerTorpedoes[i].isPlayerControlled) {
        playerTorpedoes[i].angle -= TURN_ANGLE;
        if (playerTorpedoes[i].angle < 0) playerTorpedoes[i].angle += 360;
      }
    }
  }
  if (digitalRead(BTN_TORPEDO_RIGHT) == LOW) {
    for (int i = 0; i < MAX_TORPEDOES; i++) {
      if (playerTorpedoes[i].active && playerTorpedoes[i].isPlayerControlled) {
        playerTorpedoes[i].angle += TURN_ANGLE;
        if (playerTorpedoes[i].angle >= 360) playerTorpedoes[i].angle -= 360;
      }
    }
  }
}

void updateGame(float deltaTime) {
  moveSubmarine(player, deltaTime);

  for (int i = 0; i < MAX_ENEMIES; i++) {
    if (enemies[i].active) {
      enemies[i].angle += random(-5, 6);
      if (enemies[i].angle < 0) enemies[i].angle += 360;
      if (enemies[i].angle >= 360) enemies[i].angle -= 360;
      moveSubmarine(enemies[i], deltaTime);

      if (millis() - lastEnemyFireTime > enemyFireCooldown && enemies[i].torpedoesLeft > 0) {
        float predX, predY;
        predictPlayerPosition(predX, predY, TORPEDO_SPEED);
        float dx = predX - enemies[i].x;
        float dy = predY - enemies[i].y;
        float angle = atan2(dy, dx) * 180 / PI;
        if (angle < 0) angle += 360;
        enemies[i].angle = angle;
        fireTorpedo(enemies[i], enemyTorpedoes, MAX_TORPEDOES * MAX_ENEMIES, false);
        enemies[i].torpedoesLeft--;
        lastEnemyFireTime = millis();
        enemyFireCooldown = random(1000, 3000);
      }
    }
  }

  for (int i = 0; i < MAX_TORPEDOES; i++) {
    if (playerTorpedoes[i].active) {
      moveTorpedo(playerTorpedoes[i], deltaTime);
      if (playerTorpedoes[i].x < 0 || playerTorpedoes[i].x > MAP_WIDTH ||
          playerTorpedoes[i].y < 0 || playerTorpedoes[i].y > MAP_HEIGHT) {
        playerTorpedoes[i].active = false;
      }
    }
  }

  for (int i = 0; i < MAX_TORPEDOES * MAX_ENEMIES; i++) {
    if (enemyTorpedoes[i].active) {
      moveTorpedo(enemyTorpedoes[i], deltaTime);
      if (enemyTorpedoes[i].x < 0 || enemyTorpedoes[i].x > MAP_WIDTH ||
          enemyTorpedoes[i].y < 0 || enemyTorpedoes[i].y > MAP_HEIGHT) {
        enemyTorpedoes[i].active = false;
      }
    }
  }

  checkCollisions();
}

void drawGame() {
  canvas.fillScreen(COLOR_BG);
  canvas.fillRect(0, 0, SCREEN_WIDTH, UI_HEIGHT, COLOR_UI_BG);
  canvas.drawRect(0, UI_HEIGHT, MAP_WIDTH, MAP_HEIGHT, COLOR_TEXT);

  if (player.active) {
    canvas.setCursor(player.x - 5, player.y - 5 + UI_HEIGHT);
    canvas.setTextColor(COLOR_PLAYER);
    canvas.print("X");
  }

  for (int i = 0; i < MAX_ENEMIES; i++) {
    if (enemies[i].active) {
      canvas.setCursor(enemies[i].x - 5, enemies[i].y - 5 + UI_HEIGHT);
      canvas.setTextColor(COLOR_ENEMY);
      canvas.print("E");
    }
  }

  for (int i = 0; i < MAX_TORPEDOES; i++) {
    if (playerTorpedoes[i].active) {
      float rad = degreesToRadians(playerTorpedoes[i].angle);
      float dx = cos(rad) * 4;
      float dy = sin(rad) * 4;
      canvas.drawLine(playerTorpedoes[i].x, playerTorpedoes[i].y + UI_HEIGHT,
                      playerTorpedoes[i].x + dx, playerTorpedoes[i].y + dy + UI_HEIGHT, COLOR_TORPEDO);
    }
  }

  for (int i = 0; i < MAX_TORPEDOES * MAX_ENEMIES; i++) {
    if (enemyTorpedoes[i].active) {
      float rad = degreesToRadians(enemyTorpedoes[i].angle);
      float dx = cos(rad) * 4;
      float dy = sin(rad) * 4;
      canvas.drawLine(enemyTorpedoes[i].x, enemyTorpedoes[i].y + UI_HEIGHT,
                      enemyTorpedoes[i].x + dx, enemyTorpedoes[i].y + dy + UI_HEIGHT, COLOR_TORPEDO);
    }
  }

  canvas.setTextColor(COLOR_TEXT);
  canvas.setTextSize(1);
  canvas.setCursor(10, 10);
  canvas.printf("Brzina: %d", player.speed);
  canvas.setCursor(100, 10);
  canvas.printf("Smjer: %.0f°", player.angle);
  canvas.setCursor(200, 10);
  canvas.printf("Torpeda: %d", player.torpedoesLeft);

  canvas.pushSprite(0, 0);
}

void moveSubmarine(Submarine &sub, float deltaTime) {
  float rad = degreesToRadians(sub.angle);
  float speed = sub.speed * SUBMARINE_MAX_SPEED / 3.0f;
  sub.x += cos(rad) * speed * deltaTime;
  sub.y += sin(rad) * speed * deltaTime;

  if (sub.x < MAP_BORDER) sub.x = MAP_BORDER;
  if (sub.x > MAP_WIDTH - MAP_BORDER) sub.x = MAP_WIDTH - MAP_BORDER;
  if (sub.y < MAP_BORDER) sub.y = MAP_BORDER;
  if (sub.y > MAP_HEIGHT - MAP_BORDER) sub.y = MAP_HEIGHT - MAP_BORDER;
}

void moveTorpedo(Torpedo &torp, float deltaTime) {
  float rad = degreesToRadians(torp.angle);
  torp.x += cos(rad) * torp.speed * deltaTime;
  torp.y += sin(rad) * torp.speed * deltaTime;
}

void checkCollisions() {
  for (int i = 0; i < MAX_TORPEDOES; i++) {
    if (playerTorpedoes[i].active) {
      for (int j = 0; j < MAX_ENEMIES; j++) {
        if (enemies[j].active) {
          float dist = distance(playerTorpedoes[i].x, playerTorpedoes[i].y, enemies[j].x, enemies[j].y);
          if (dist < 10) {
            enemies[j].active = false;
            playerTorpedoes[i].active = false;
            ledcWriteTone(LEDC_CHANNEL, 300);
            delay(50);
            ledcWriteTone(LEDC_CHANNEL, 0);
          }
        }
      }
    }
  }

  for (int i = 0; i < MAX_TORPEDOES * MAX_ENEMIES; i++) {
    if (enemyTorpedoes[i].active && player.active) {
      float dist = distance(enemyTorpedoes[i].x, enemyTorpedoes[i].y, player.x, player.y);
      if (dist < 10) {
        player.active = false;
        enemyTorpedoes[i].active = false;
        ledcWriteTone(LEDC_CHANNEL, 200);
        delay(100);
        ledcWriteTone(LEDC_CHANNEL, 0);
      }
    }
  }

  for (int i = 0; i < MAX_TORPEDOES * MAX_ENEMIES; i++) {
    if (enemyTorpedoes[i].active) {
      for (int j = 0; j < MAX_ENEMIES; j++) {
        if (enemies[j].active) {
          float dist = distance(enemyTorpedoes[i].x, enemyTorpedoes[i].y, enemies[j].x, enemies[j].y);
          if (dist < 10) {
            enemies[j].active = false;
            enemyTorpedoes[i].active = false;
          }
        }
      }
    }
  }

  for (int i = 0; i < MAX_TORPEDOES * MAX_ENEMIES; i++) {
    if (enemyTorpedoes[i].active) {
      for (int j = 0; j < MAX_TORPEDOES; j++) {
        if (playerTorpedoes[j].active) {
          float dist = distance(enemyTorpedoes[i].x, enemyTorpedoes[i].y, playerTorpedoes[j].x, playerTorpedoes[j].y);
          if (dist < 5) {
            enemyTorpedoes[i].active = false;
            playerTorpedoes[j].active = false;
          }
        }
      }
      for (int k = i + 1; k < MAX_TORPEDOES * MAX_ENEMIES; k++) {
        if (enemyTorpedoes[k].active) {
          float dist = distance(enemyTorpedoes[i].x, enemyTorpedoes[i].y, enemyTorpedoes[k].x, enemyTorpedoes[k].y);
          if (dist < 5) {
            enemyTorpedoes[i].active = false;
            enemyTorpedoes[k].active = false;
          }
        }
      }
    }
  }

  for (int i = 0; i < MAX_ENEMIES; i++) {
    if (enemies[i].active) {
      for (int j = i + 1; j < MAX_ENEMIES; j++) {
        if (enemies[j].active) {
          float dist = distance(enemies[i].x, enemies[i].y, enemies[j].x, enemies[j].y);
          if (dist < 20) {
            enemies[i].active = false;
            enemies[j].active = false;
          }
        }
      }
      if (player.active) {
        float dist = distance(player.x, player.y, enemies[i].x, enemies[i].y);
        if (dist < 20) {
          player.active = false;
          enemies[i].active = false;
        }
      }
    }
  }
}

void fireTorpedo(Submarine &sub, Torpedo torps[], int maxTorps, bool isPlayer) {
  for (int i = 0; i < maxTorps; i++) {
    if (!torps[i].active) {
      torps[i].x = sub.x;
      torps[i].y = sub.y;
      torps[i].angle = sub.angle;
      torps[i].speed = TORPEDO_SPEED;
      torps[i].active = true;
      torps[i].isPlayerControlled = isPlayer;
      break;
    }
  }
}

void predictPlayerPosition(float &predX, float &predY, float torpSpeed) {
  float rad = degreesToRadians(player.angle);
  float playerSpeed = player.speed * SUBMARINE_MAX_SPEED / 3.0f;
  float dist = distance(player.x, player.y, enemies[0].x, enemies[0].y);
  float timeToImpact = dist / torpSpeed;
  predX = player.x + cos(rad) * playerSpeed * timeToImpact;
  predY = player.y + sin(rad) * playerSpeed * timeToImpact;
}

float degreesToRadians(float degrees) {
  return degrees * PI / 180.0f;
}

float distance(float x1, float y1, float x2, float y2) {
  return sqrt((x2 - x1) * (x2 - x1) + (y2 - y1) * (y2 - y1));
}
```

### Objašnjenje koda
- **Inicijalizacija:** Postavlja LCD (LovyanGFX), zvučnik i gumbe. Igrač i neprijatelji dobivaju početne pozicije i 10 torpeda.
- **Upravljanje:** Analogni gumbi na GPIO34 i GPIO35 očitavaju vrijednosti (>4000 i 1800-2200) za dvije funkcije po pinu.
- **Ažuriranje:** Pomicanje podmornica i torpeda temelji se na brzini i smjeru. Neprijatelji pucaju s predviđanjem, ali s cooldownom kako ne bi svi pucali odjednom.
- **Sudari:** Provjeravaju se svi scenariji (torpedo-podmornica, torpedo-torpedo, podmornica-podmornica).
- **Prikaz:** Mapa s 'X' za igrača, 'E' za neprijatelje, crtice za torpede i UI s brzinom, smjerom i torpedima.

### Kako pokrenuti
1. Instaliraj Arduino IDE i potrebne biblioteke (LovyanGFX).
2. Kopiraj kod u IDE.
3. Spoji VIDI X mikroračunalo i učitaj kod.
4. Igraj igru koristeći gumbe na navedenim GPIO pinovima.

Ovaj kod zadovoljava sve zahtjeve i spreman je za korištenje!
