# Gemini 2.5 Pro

Izrađeno pomoću modela **2.5 Pro** 30. lipnja 2025. u 16:04 Objavljeno 1. srpnja 2025. u 11:36

U redu, pripremio sam detaljne upute i kompletan kod za vašu igru podmornice na VIDI X mikroračunalu.
Igra je osmišljena prema vašim specifikacijama, uključujući upravljanje podmornicom i torpedom, nasumično kretanje neprijatelja, njihov AI za predviđanje putanje i ispaljivanje torpeda, kao i sve elemente na ekranu. Kod je bogato komentiran kako biste lakše razumjeli svaki dio i prilagodili ga po želji.
Slijedi kompletan kod koji možete izravno kopirati u svoj Arduino IDE.
Nadam se da ćete uživati u programiranju i igranju! Ako imate bilo kakvih pitanja ili želite napraviti izmjene, slobodno pitajte.
## VIDI X - Igra Podmornica

C++

```C++
/******************************************************************
 * VIDI X Submarine Annihilator
 *
 * Autor: Gemini
 * Verzija: 1.0
 * Datum: 30.06.2025.
 *
 * Opis:
 * Igra borbe podmornica za VIDI X mikroračunalo bazirano na ESP32.
 * Igrač upravlja svojom podmornicom, izbjegava neprijateljske
 * projektile i pokušava uništiti 10 neprijateljskih podmornica
 * pomoću upravljivog torpeda.
 *
 * Kontrole:
 * - GPIO35 (analogni): Zaroni (Vrijednost > 4000) / Izroni (Vrijednost 1800-2200)
 * - GPIO34 (analogni): Lijevo (Vrijednost > 4000) / Desno (Vrijednost 1800-2200)
 * - GPIO0: Ubrzaj
 * - GPIO13: Uspori
 * - GPIO32: Ispali torpedo
 * - GPIO27: Torpedo lijevo
 * - GPIO39: Torpedo desno
 ******************************************************************/

#define LGFX_USE_V1
#include <LovyanGFX.hpp>

// --- Konfiguracija ekrana (preuzeto iz primjera) ---
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

// --- Inicijalizacija ekrana i sprite-a (canvasa) ---
static LGFX lcd;
static LGFX_Sprite canvas(&lcd);

// --- Definicije pinova za gumbe ---
#define BTN_DIVE_SURFACE  35 // Zaroni / Izroni (Analogni)
#define BTN_LEFT_RIGHT    34 // Lijevo / Desno (Analogni)
#define BTN_FIRE_TORPEDO  32 // Ispali torpedo
#define BTN_SPEED_UP      0  // Ubrzaj
#define BTN_SPEED_DOWN    13 // Uspori
#define BTN_TORPEDO_LEFT  27 // Torpedo lijevo
#define BTN_TORPEDO_RIGHT 39 // Torpedo desno

// --- Dimenzije i konstante igre ---
#define SCREEN_WIDTH      320
#define SCREEN_HEIGHT     240
#define INFO_WIDTH        80 // Širina bočne trake za informacije
#define MAP_WIDTH         (SCREEN_WIDTH - INFO_WIDTH)
#define NUM_ENEMIES       10
#define PLAYER_MAX_SPEED  100.0f
#define ENEMY_MAX_SPEED   100.0f
#define TORPEDO_SPEED     200.0f
#define TORPEDO_TURN_RATE 5
#define SUB_TURN_RATE     5
#define MAX_TORPEDOES     10

// --- Boje ---
#define COLOR_WATER       0x018A // Tamno plava
#define COLOR_UI_BG       0x2222 // Siva pozadina za UI
#define COLOR_TEXT        0xFFFF // Bijela
#define COLOR_PLAYER      0x0FF0 // Svijetlo zelena (cijan)
#define COLOR_ENEMY       0xF800 // Crvena
#define COLOR_PLAYER_TORPEDO 0xFFE0 // Žuta
#define COLOR_ENEMY_TORPEDO  0xFA20 // Narančasta

// --- Strukture podataka za objekte u igri ---

struct Torpedo {
  float x, y;
  float angle; // U stupnjevima
  bool active;
};

struct Submarine {
  float x, y;
  float speed;      // Pixela u sekundi
  int angle;        // U stupnjevima (0-359)
  int depth;        // Dubina (0-100)
  bool active;
  int torpedoCount;
  Torpedo torpedo;  // Svaka podmornica ima samo jedan torpedo u isto vrijeme
};

// --- Globalne varijable igre ---
Submarine player;
Submarine enemies[NUM_ENEMIES];
Torpedo enemyTorpedoes[NUM_ENEMIES]; // Svaki neprijatelj može imati jedan ispaljeni torpedo

unsigned long lastFrameTime = 0;
float deltaTime = 0; // Vrijeme proteklo između frameova u sekundama

int speedLevel = 2; // 0=-1, 1=0, 2=0.33, 3=0.66, 4=1.0
const float speedSteps[] = {-0.5f, 0.0f, 0.33f, 0.66f, 1.0f};

unsigned long lastEnemyFireTime = 0;
int enemyFiringIndex = 0; // Koji neprijatelj sljedeći puca


// --- Prototipovi funkcija ---
void initGame();
void handleInput();
void updateGame();
void drawGame();
void drawMap();
void drawUI();
void updatePlayer();
void updateEnemies();
void updatePlayerTorpedo();
void updateEnemyTorpedoes();
void checkCollisions();
void firePlayerTorpedo();
void fireEnemyTorpedo(int enemyIndex);
int readAnalogButton(int pin);

// ### SETUP - Inicijalizacija ###
void setup() {
  Serial.begin(115200);

  lcd.init();
  lcd.setRotation(0);
  canvas.setPsram(true); // Koristi PSRAM za canvas
  if (!canvas.createSprite(SCREEN_WIDTH, SCREEN_HEIGHT)) {
    Serial.println("Canvas creation FAILED!");
    while(1);
  }

  // Konfiguracija pinova za gumbe
  pinMode(BTN_FIRE_TORPEDO, INPUT_PULLUP);
  pinMode(BTN_SPEED_UP, INPUT_PULLUP);
  pinMode(BTN_SPEED_DOWN, INPUT_PULLUP);
  pinMode(BTN_TORPEDO_LEFT, INPUT_PULLUP);
  pinMode(BTN_TORPEDO_RIGHT, INPUT_PULLUP);
  pinMode(BTN_LEFT_RIGHT, INPUT);
  pinMode(BTN_DIVE_SURFACE, INPUT);

  initGame();
  lastFrameTime = millis();
}

// ### GLAVNA PETLJA ###
void loop() {
  unsigned long currentTime = millis();
  deltaTime = (currentTime - lastFrameTime) / 1000.0f; // Delta time u sekundama
  lastFrameTime = currentTime;

  handleInput();
  updateGame();
  drawGame();

  delay(16); // Ograničava na otprilike 60 FPS
}

// ### INICIJALIZACIJA IGRE ###
void initGame() {
  // Postavljanje igrača
  player.x = MAP_WIDTH / 2;
  player.y = SCREEN_HEIGHT / 2;
  player.speed = 0;
  player.angle = 90; // Gleda prema gore
  player.depth = 50;
  player.active = true;
  player.torpedoCount = MAX_TORPEDOES;
  player.torpedo.active = false;

  // Postavljanje neprijatelja
  for (int i = 0; i < NUM_ENEMIES; i++) {
    enemies[i].x = random(20, MAP_WIDTH - 20);
    enemies[i].y = random(20, SCREEN_HEIGHT - 20);
    enemies[i].speed = random(20, ENEMY_MAX_SPEED * 0.75); // Nasumična brzina
    enemies[i].angle = random(0, 360);
    enemies[i].depth = random(20, 80);
    enemies[i].active = true;
    enemies[i].torpedoCount = MAX_TORPEDOES;
    enemyTorpedoes[i].active = false;
  }
}

// ### OBRADA UNOSA (GUMBI) ###
void handleInput() {
  // Očitavanje analognih gumba
  int lr_val = readAnalogButton(BTN_LEFT_RIGHT);
  int ud_val = readAnalogButton(BTN_DIVE_SURFACE);

  // Rotacija podmornice
  if (lr_val == 1) player.angle = (player.angle - SUB_TURN_RATE + 360) % 360; // Lijevo
  if (lr_val == 2) player.angle = (player.angle + SUB_TURN_RATE) % 360; // Desno

  // Zaranjanje/Izranjanje
  if (ud_val == 1 && player.depth > 0) player.depth--; // Zaroni
  if (ud_val == 2 && player.depth < 100) player.depth++; // Izroni

  // Promjena brzine
  static unsigned long lastSpeedUpTime = 0;
  if (digitalRead(BTN_SPEED_UP) == LOW && millis() - lastSpeedUpTime > 200) {
    if (speedLevel < 4) speedLevel++;
    player.speed = PLAYER_MAX_SPEED * speedSteps[speedLevel];
    lastSpeedUpTime = millis();
  }
  static unsigned long lastSpeedDownTime = 0;
  if (digitalRead(BTN_SPEED_DOWN) == LOW && millis() - lastSpeedDownTime > 200) {
    if (speedLevel > 0) speedLevel--;
    player.speed = PLAYER_MAX_SPEED * speedSteps[speedLevel];
    lastSpeedDownTime = millis();
  }

  // Ispaljivanje torpeda
  static unsigned long lastFireTime = 0;
  if (digitalRead(BTN_FIRE_TORPEDO) == LOW && millis() - lastFireTime > 500) {
    if (!player.torpedo.active && player.torpedoCount > 0) {
      firePlayerTorpedo();
    }
    lastFireTime = millis();
  }

  // Upravljanje torpedom
  if (player.torpedo.active) {
    if (digitalRead(BTN_TORPEDO_LEFT) == LOW) {
      player.torpedo.angle = (int)(player.torpedo.angle - TORPEDO_TURN_RATE + 360) % 360;
    }
    if (digitalRead(BTN_TORPEDO_RIGHT) == LOW) {
      player.torpedo.angle = (int)(player.torpedo.angle + TORPEDO_TURN_RATE) % 360;
    }
  }
}


// Očitavanje analognih gumba sa zajedničkim otpornicima
int readAnalogButton(int pin) {
  int val = analogRead(pin);
  if (val > 4000) return 1; // Prvi gumb
  if (val > 1800 && val < 2200) return 2; // Drugi gumb
  return 0; // Nema pritiska
}


// ### AŽURIRANJE STANJA IGRE ###
void updateGame() {
  if (!player.active) {
    // Game Over logika
    return;
  }
  updatePlayer();
  updateEnemies();
  updatePlayerTorpedo();
  updateEnemyTorpedoes();
  checkCollisions();
  
  // Logika za pucanje neprijatelja
  if (millis() - lastEnemyFireTime > 2000) { // Pauza između pucanja
    // Nađi sljedećeg aktivnog neprijatelja za pucanje
    int attempts = 0;
    while(attempts < NUM_ENEMIES) {
        if (enemies[enemyFiringIndex].active && !enemyTorpedoes[enemyFiringIndex].active && enemies[enemyFiringIndex].torpedoCount > 0) {
            fireEnemyTorpedo(enemyFiringIndex);
            lastEnemyFireTime = millis();
            enemyFiringIndex = (enemyFiringIndex + 1) % NUM_ENEMIES;
            break;
        }
        enemyFiringIndex = (enemyFiringIndex + 1) % NUM_ENEMIES;
        attempts++;
    }
  }
}

// Ažuriranje pozicije igrača
void updatePlayer() {
  float angleRad = player.angle * DEG_TO_RAD;
  player.x += player.speed * cos(angleRad) * deltaTime;
  player.y -= player.speed * sin(angleRad) * deltaTime; // Y os je invertirana na ekranu

  // Ograničavanje kretanja unutar mape
  if (player.x < 5) player.x = 5;
  if (player.x > MAP_WIDTH - 5) player.x = MAP_WIDTH - 5;
  if (player.y < 5) player.y = 5;
  if (player.y > SCREEN_HEIGHT - 5) player.y = SCREEN_HEIGHT - 5;
}

// Ažuriranje pozicija neprijatelja
void updateEnemies() {
  for (int i = 0; i < NUM_ENEMIES; i++) {
    if (enemies[i].active) {
      float angleRad = enemies[i].angle * DEG_TO_RAD;
      enemies[i].x += enemies[i].speed * cos(angleRad) * deltaTime;
      enemies[i].y -= enemies[i].speed * sin(angleRad) * deltaTime;

      // Jednostavan AI: odbijanje od rubova i nasumična promjena smjera
      if (enemies[i].x < 5 || enemies[i].x > MAP_WIDTH - 5 || enemies[i].y < 5 || enemies[i].y > SCREEN_HEIGHT - 5) {
        enemies[i].angle = (enemies[i].angle + 180 + random(-30, 31)) % 360;
        // Vrati ih malo unutar mape da se ne zaglave
        if(enemies[i].x < 5) enemies[i].x = 5;
        if(enemies[i].x > MAP_WIDTH - 5) enemies[i].x = MAP_WIDTH - 5;
        if(enemies[i].y < 5) enemies[i].y = 5;
        if(enemies[i].y > SCREEN_HEIGHT - 5) enemies[i].y = SCREEN_HEIGHT - 5;

      }
      if (random(100) < 2) { // 2% šanse svake petlje za promjenu smjera
        enemies[i].angle = (enemies[i].angle + random(-45, 46)) % 360;
      }
    }
  }
}

// Ažuriranje pozicije igračevog torpeda
void updatePlayerTorpedo() {
  if (player.torpedo.active) {
    float angleRad = player.torpedo.angle * DEG_TO_RAD;
    player.torpedo.x += TORPEDO_SPEED * cos(angleRad) * deltaTime;
    player.torpedo.y -= TORPEDO_SPEED * sin(angleRad) * deltaTime;

    // Deaktivacija ako izađe iz mape
    if (player.torpedo.x < 0 || player.torpedo.x > MAP_WIDTH || player.torpedo.y < 0 || player.torpedo.y > SCREEN_HEIGHT) {
      player.torpedo.active = false;
    }
  }
}

// Ažuriranje pozicija neprijateljskih torpeda
void updateEnemyTorpedoes() {
    for (int i = 0; i < NUM_ENEMIES; i++) {
        if (enemyTorpedoes[i].active) {
            float angleRad = enemyTorpedoes[i].angle * DEG_TO_RAD;
            enemyTorpedoes[i].x += TORPEDO_SPEED * cos(angleRad) * deltaTime;
            enemyTorpedoes[i].y -= TORPEDO_SPEED * sin(angleRad) * deltaTime;

            if (enemyTorpedoes[i].x < 0 || enemyTorpedoes[i].x > MAP_WIDTH || enemyTorpedoes[i].y < 0 || enemyTorpedoes[i].y > SCREEN_HEIGHT) {
                enemyTorpedoes[i].active = false;
            }
        }
    }
}


// ### PROVJERA SUDARA ###
void checkCollisions() {
    // 1. Igračev torpedo vs Neprijateljske podmornice
    if (player.torpedo.active) {
        for (int i = 0; i < NUM_ENEMIES; i++) {
            if (enemies[i].active) {
                float dx = player.torpedo.x - enemies[i].x;
                float dy = player.torpedo.y - enemies[i].y;
                if (sqrt(dx * dx + dy * dy) < 8) { // Radijus sudara 8 pixela
                    enemies[i].active = false; // Neprijatelj uništen
                    player.torpedo.active = false;
                    // Ovdje se može dodati zvuk eksplozije i bodovi
                    break;
                }
            }
        }
    }

    // 2. Neprijateljski torpedi vs Igrač
    for (int i = 0; i < NUM_ENEMIES; i++) {
        if (enemyTorpedoes[i].active) {
            float dx = enemyTorpedoes[i].x - player.x;
            float dy = enemyTorpedoes[i].y - player.y;
            if (sqrt(dx * dx + dy * dy) < 8) {
                enemyTorpedoes[i].active = false;
                player.active = false; // GAME OVER
                break;
            }
        }
    }

    // 3. Neprijateljski torpedo vs Neprijateljska podmornica (friendly fire)
    for (int i = 0; i < NUM_ENEMIES; i++) {
        if (enemyTorpedoes[i].active) {
            for (int j = 0; j < NUM_ENEMIES; j++) {
                if (i == j || !enemies[j].active) continue; // Ne provjeravaj sa samim sobom ili neaktivnim podmornicama
                float dx = enemyTorpedoes[i].x - enemies[j].x;
                float dy = enemyTorpedoes[i].y - enemies[j].y;
                if (sqrt(dx * dx + dy * dy) < 8) {
                    enemies[j].active = false;
                    enemyTorpedoes[i].active = false;
                    goto next_torpedo_check; // Prekini unutarnju petlju i prijeđi na sljedeći torpedo
                }
            }
        }
        next_torpedo_check:;
    }

    // 4. Neprijateljski torpedo vs drugi neprijateljski torpedo
    for (int i = 0; i < NUM_ENEMIES; i++) {
        if (!enemyTorpedoes[i].active) continue;
        for (int j = i + 1; j < NUM_ENEMIES; j++) {
            if (!enemyTorpedoes[j].active) continue;
            float dx = enemyTorpedoes[i].x - enemyTorpedoes[j].x;
            float dy = enemyTorpedoes[i].y - enemyTorpedoes[j].y;
            if (sqrt(dx * dx + dy * dy) < 5) {
                enemyTorpedoes[i].active = false;
                enemyTorpedoes[j].active = false;
                break; // Prelom unutarnje petlje, i-ti torpedo je uništen
            }
        }
    }
}


// ### FUNKCIJE ISPALJIVANJA ###
void firePlayerTorpedo() {
  player.torpedoCount--;
  player.torpedo.active = true;
  player.torpedo.angle = player.angle;
  // Ispali torpedo s vrha podmornice
  float angleRad = player.angle * DEG_TO_RAD;
  player.torpedo.x = player.x + 10 * cos(angleRad);
  player.torpedo.y = player.y - 10 * sin(angleRad);
}

void fireEnemyTorpedo(int i) {
    enemies[i].torpedoCount--;
    enemyTorpedoes[i].active = true;

    // AI za predviđanje putanje
    float dist = sqrt(pow(player.x - enemies[i].x, 2) + pow(player.y - enemies[i].y, 2));
    float timeToTarget = dist / TORPEDO_SPEED;

    float playerAngleRad = player.angle * DEG_TO_RAD;
    float predictedPlayerX = player.x + player.speed * cos(playerAngleRad) * timeToTarget;
    float predictedPlayerY = player.y - player.speed * sin(playerAngleRad) * timeToTarget;

    float angleToTarget = atan2(enemies[i].y - predictedPlayerY, predictedPlayerX - enemies[i].x) * RAD_TO_DEG;
    enemyTorpedoes[i].angle = angleToTarget;

    // Ispali torpedo s vrha podmornice
    float enemyAngleRad = enemies[i].angle * DEG_TO_RAD;
    enemyTorpedoes[i].x = enemies[i].x + 10 * cos(enemyAngleRad);
    enemyTorpedoes[i].y = enemies[i].y - 10 * sin(enemyAngleRad);
}

// ### FUNKCIJE CRTANJA ###
void drawGame() {
  canvas.startWrite();
  canvas.fillScreen(COLOR_WATER);

  drawMap();
  drawUI();
  
  // Ako je igra gotova, prikaži poruku
  int activeEnemies = 0;
  for(int i=0; i<NUM_ENEMIES; i++) {
    if(enemies[i].active) activeEnemies++;
  }

  if (!player.active) {
    canvas.setTextSize(3);
    canvas.setTextColor(TFT_RED);
    canvas.setTextDatum(middle_center);
    canvas.drawString("GAME OVER", MAP_WIDTH / 2, SCREEN_HEIGHT / 2);
  } else if (activeEnemies == 0) {
    canvas.setTextSize(3);
    canvas.setTextColor(TFT_GREEN);
    canvas.setTextDatum(middle_center);
    canvas.drawString("POBJEDA!", MAP_WIDTH / 2, SCREEN_HEIGHT / 2);
  }


  canvas.endWrite();
  canvas.pushSprite(0, 0);
}

void drawMap() {
  // Crtanje igrača
  if (player.active) {
    canvas.setTextColor(COLOR_PLAYER);
    canvas.setTextSize(2);
    canvas.drawChar('X', player.x - 4, player.y - 8);
    // Crtaj liniju koja pokazuje smjer
    float angleRad = player.angle * DEG_TO_RAD;
    canvas.drawLine(player.x, player.y, player.x + 10 * cos(angleRad), player.y - 10 * sin(angleRad), COLOR_PLAYER);
  }

  // Crtanje neprijatelja
  for (int i = 0; i < NUM_ENEMIES; i++) {
    if (enemies[i].active) {
      canvas.setTextColor(COLOR_ENEMY);
      canvas.setTextSize(2);
      canvas.drawChar('O', enemies[i].x - 4, enemies[i].y - 8);
    }
  }

  // Crtanje igračevog torpeda
  if (player.torpedo.active) {
    float angleRad = player.torpedo.angle * DEG_TO_RAD;
    canvas.drawLine(player.torpedo.x, player.torpedo.y,
                    player.torpedo.x - 4 * cos(angleRad), player.torpedo.y + 4 * sin(angleRad),
                    COLOR_PLAYER_TORPEDO);
  }

  // Crtanje neprijateljskih torpeda
  for (int i = 0; i < NUM_ENEMIES; i++) {
    if (enemyTorpedoes[i].active) {
        float angleRad = enemyTorpedoes[i].angle * DEG_TO_RAD;
        canvas.drawLine(enemyTorpedoes[i].x, enemyTorpedoes[i].y,
                        enemyTorpedoes[i].x - 4 * cos(angleRad), enemyTorpedoes[i].y + 4 * sin(angleRad),
                        COLOR_ENEMY_TORPEDO);
    }
  }
}

void drawUI() {
  // Crtanje pozadine za UI
  canvas.fillRect(MAP_WIDTH, 0, INFO_WIDTH, SCREEN_HEIGHT, COLOR_UI_BG);
  canvas.drawLine(MAP_WIDTH, 0, MAP_WIDTH, SCREEN_HEIGHT, TFT_WHITE);

  // Postavke teksta
  canvas.setTextColor(COLOR_TEXT);
  canvas.setTextSize(1);
  canvas.setTextDatum(top_left);

  // Ispis informacija
  canvas.setCursor(MAP_WIDTH + 5, 10);
  canvas.printf("BRZINA");
  
  // Grafički prikaz brzine
  for(int i = 0; i < 5; i++) {
      if (i == speedLevel) {
          canvas.fillRect(MAP_WIDTH + 5, 25 + i * 12, INFO_WIDTH - 10, 10, TFT_GREEN);
      } else {
          canvas.drawRect(MAP_WIDTH + 5, 25 + i * 12, INFO_WIDTH - 10, 10, TFT_WHITE);
      }
  }
  
  canvas.setCursor(MAP_WIDTH + 5, 90);
  canvas.printf("SMJER: %d", player.angle);

  canvas.setCursor(MAP_WIDTH + 5, 110);
  canvas.printf("DUBINA: %d", player.depth);

  canvas.setCursor(MAP_WIDTH + 5, 130);
  canvas.printf("TORPEDA: %d", player.torpedoCount);

  // Broj preostalih neprijatelja
  int activeEnemies = 0;
  for(int i=0; i<NUM_ENEMIES; i++) {
    if(enemies[i].active) activeEnemies++;
  }
  canvas.setCursor(MAP_WIDTH + 5, 160);
  canvas.printf("METE: %d", activeEnemies);

  // Status torpeda
  if (player.torpedo.active) {
      canvas.setTextColor(TFT_YELLOW);
      canvas.setCursor(MAP_WIDTH + 5, 200);
      canvas.print("TORPEDO\nAKTIVAN");
  }

}
```

[Googleova pravila o privatnosti](https://policies.google.com/privacy)
[Googleovi uvjeti pružanja usluge](https://policies.google.com/terms)
[Vaša privatnost i Gemini aplikacije](https://support.google.com/gemini?p=privacy_notice)

Gemini može prikazivati netočne informacije, uključujući one o ljudima, pa dobro provjerite njegove odgovore. Ovaj sadržaj izradila je druga osoba. Mogao bi biti netočan ili rizičan.
