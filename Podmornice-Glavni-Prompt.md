# PROMPT: Za igru Podmornice
```C++
Razmisli o detaljnim uputama za izradu igre za VIDI X mikroračunalo na ESP32 (C++ i Arduino IDE), a zatim napiši kompletan kod za tu igru kako bi ga mogao copy paste u editor koda.
Na donjem dijelu ekrana nalaze se gumbi za upravljanje podmornicom (Zaroni, Izroni) GPIO35, (Lijevo, Denso) GPIO34, te brzina sa tri razine naprijed i jednom razinom unazad - dakle 5 stupnjeva jer imamo i STOP) Brzinu mijenjamo sa GPIO0 i GPIO13. Pored toga se nalazi gumb za ispaljivanje torpeda GPIO32. Torpedo također ima vlastite gumbe za upravljanje ali samo Lijevo (GPIO27) i Denso(GPIO39) jer je njegova brzina konstantna (200). Maksimalna brzina podmornice je 100. Klikom na gumbe lijevo ili desno podmornica se zakreće za 5°. Zaranjanje i izranjanje je relativno sporo. Sa strane je ispisana brzina i smjer podmornice u stupnjevima. Na ekranu se nalazi grafički prikaz, poput mape, mora, gdje X označava vašu podmornicu. Crta duljine 4 pixela označava ispaljeni torpedo. Postoji i 10 neprijateljskih podmornica koje se proizvoljno kreću po mapi maksimalnom brzinom od 100. Cilj je pogoditi neprijateljsku podmornicu torpedom. One također mogu ispucavati torpeda na vas ali njihova torpeda nisu upravljiva nego se kreću pravocrtno. Neprijatelj može pretpostaviti vašu putanju gdje ćete biti u trenutku kada njihov torpedo dođe do vas te ukoliko niste promijenili smjer ili brzinu kretanja pogodak je neizbježan. Treba paziti da ne ispale sve podmornice torpedo istovremeno jer igrač neće moći preživjeti. Sve podmornice imaju po 10 torpeda i kada ih potroše nemaju ih više.
Ovo je primjer koda za neku drugu igru iz kojega treba iskoristiti dio za prikaz grafike na ekranu te dio za upravljanje, a posebno upravljanje gumbima na GPIO34 i GPIO35 jer su na svakom po dva gumba spojena otpornikom pa očitavamo analogne vrijednosti. 
#define LGFX_USE_V1
#include <LovyanGFX.hpp>
#include <FastLED.h>

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

#define LED_PIN 26
#define NUM_LEDS 5
CRGB leds[NUM_LEDS];

#define SPEAKER_PIN 25
#define LEDC_CHANNEL 0
#define LEDC_RESOLUTION 8
#define LEDC_BASE_FREQ 2000

#define SCREEN_WIDTH 320
#define SCREEN_HEIGHT 240
#define UI_HEIGHT 35
#define PLAYER_SIZE 24
#define MAX_METEORS 8
#define MAX_CELLS 3

#define BTN_LR   34
#define BTN_UD   35
#define BTN_A    32
#define BTN_RESTART 0

#define MOVE_STEP 8

#define BULLET_SPEED 14
#define BULLET_MAX 8

// -- Boje --
#define COLOR_UI_BG 0x18E3
#define COLOR_TEXT  0xFFFF

struct Meteor {
  float x, y;
  float dx, dy;
  float speed;
  int size;
  bool active;
  float angle;
  float rotationSpeed;
  uint8_t points;
  float offsets[16];
  uint16_t color;
  bool isSmall;
  int impact; // za sudare
};

struct Bullet {
  float x, y, dx, dy;
  bool active;
};

struct EnergyCell {
  float x, y;
  bool collected;
};

struct Player {
  float x, y;
  int health;
  int energy;
  int score;
};

Player player;
Meteor meteors[MAX_METEORS];
EnergyCell cells[MAX_CELLS];
Bullet bullets[BULLET_MAX];

unsigned long lastSpawn = 0;
bool gameOver = false;

// --- Topovi animacija
bool cannonFired = false;
unsigned long lastCannonFlash = 0;
#define CANNON_FLASH_MS 70

// --- Prototipovi ---
void initGame();
void generateMeteor();
void generateEnergyCell();
void updateGame();
void updatePlayerMovement();
void drawGame();
void drawMeteor(Meteor &m);
void playCollectSound();
void playCollisionSound();
void showGameOver();
void fireCannon();
void updateBullets();
void drawBullets();
void splitMeteor(Meteor &m);

// --- Setup ---
void setup() {
  Serial.begin(115200);

  lcd.init();
  lcd.setRotation(0);
  lcd.fillScreen(0x0000);
  canvas.setPsram(true);
  bool ok = canvas.createSprite(SCREEN_WIDTH, SCREEN_HEIGHT);
  Serial.println(ok ? "Canvas created!" : "Canvas FAIL!");
  if (!ok) while(1) delay(1000);

  FastLED.addLeds<WS2812B, LED_PIN, GRB>(leds, NUM_LEDS);
  ledcSetup(LEDC_CHANNEL, LEDC_BASE_FREQ, LEDC_RESOLUTION);
  ledcAttachPin(SPEAKER_PIN, LEDC_CHANNEL);

  pinMode(BTN_LR, INPUT);
  pinMode(BTN_UD, INPUT);
  pinMode(BTN_A, INPUT_PULLUP);
  pinMode(BTN_RESTART, INPUT_PULLUP);

  initGame();
}

void initGame() {
  player.x = SCREEN_WIDTH/2;
  player.y = SCREEN_HEIGHT - 50;
  player.health = 100;
  player.energy = 0;
  player.score = 0;
  for(int i=0; i<MAX_METEORS; i++) meteors[i].active = false;
  for(int i=0; i<MAX_CELLS; i++) cells[i].collected = true;
  for(int i=0; i<BULLET_MAX; i++) bullets[i].active = false;
  gameOver = false;
}

// --- Meteoriti ---
void generateMeteor() {
  for(int i=0; i<MAX_METEORS; i++) {
    if(!meteors[i].active) {
      meteors[i].x = random(40, SCREEN_WIDTH-40);
      meteors[i].y = -30;
      meteors[i].speed = random(2, 5);
      meteors[i].size = random(36, 48);
      meteors[i].active = true;
      meteors[i].angle = random(0, 360) * 0.01745;
      meteors[i].rotationSpeed = (random(-8,9)) * 0.008;
      meteors[i].points = random(8, 13);
      for(uint8_t j=0; j<meteors[i].points; j++)
        meteors[i].offsets[j] = meteors[i].size/2 * (0.7 + random(0, 31) * 0.01);
      int tint = random(3);
      if (tint == 0)
        meteors[i].color = 0x7BEF + random(-500, 500);
      else if (tint == 1)
        meteors[i].color = 0x6DEF + random(-600, 300);
      else
        meteors[i].color = 0xFEA0 + random(-400, 200);
      meteors[i].dx = 0;
      meteors[i].dy = meteors[i].speed;
      meteors[i].isSmall = false;
      meteors[i].impact = meteors[i].size / 4 + 3;
      break;
    }
  }
}

// --- Energijske ćelije ---
void generateEnergyCell() {
  for(int i=0; i<MAX_CELLS; i++) {
    if(cells[i].collected) {
      cells[i].x = random(50, SCREEN_WIDTH-50);
      cells[i].y = random(100, SCREEN_HEIGHT-100);
      cells[i].collected = false;
      break;
    }
  }
}

// --- Kretanje igrača s analogRead (VIDI X joystick stil) ---
void updatePlayerMovement() {
  int analogLR = analogRead(BTN_LR);
  int analogUD = analogRead(BTN_UD);

  // Lijevo
  if (analogLR > 4000) player.x -= MOVE_STEP;
  // Desno
  if (analogLR > 1800 && analogLR < 2200) player.x += MOVE_STEP;
  // Gore
  if (analogUD > 4000) player.y -= MOVE_STEP;
  // Dolje
  if (analogUD > 1800 && analogUD < 2200) player.y += MOVE_STEP;

  // Clamp
  if (player.x < PLAYER_SIZE/2) player.x = PLAYER_SIZE/2;
  if (player.x > SCREEN_WIDTH-PLAYER_SIZE/2) player.x = SCREEN_WIDTH-PLAYER_SIZE/2;
  if (player.y < UI_HEIGHT + PLAYER_SIZE/2) player.y = UI_HEIGHT + PLAYER_SIZE/2;
  if (player.y > SCREEN_HEIGHT-PLAYER_SIZE/2) player.y = SCREEN_HEIGHT-PLAYER_SIZE/2;
}

// --- Metak (bullet) ---
void fireCannon() {
  for (int i=0; i<BULLET_MAX; i++) {
    if (!bullets[i].active) {
      bullets[i].x = player.x;
      bullets[i].y = player.y - PLAYER_SIZE/2 - 2;
      bullets[i].dx = 0;
      bullets[i].dy = -BULLET_SPEED;
      bullets[i].active = true;
      cannonFired = true;
      lastCannonFlash = millis();
      break;
    }
  }
}

// --- Update metka ---
void updateBullets() {
  for (int i=0; i<BULLET_MAX; i++) {
    if (bullets[i].active) {
      bullets[i].x += bullets[i].dx;
      bullets[i].y += bullets[i].dy;
      if (bullets[i].y < 0) bullets[i].active = false;
    }
  }
}

void drawBullets() {
  for (int i=0; i<BULLET_MAX; i++) {
    if (bullets[i].active) {
      canvas.fillCircle(bullets[i].x, bullets[i].y, 3, 0xFFE0);
      canvas.drawCircle(bullets[i].x, bullets[i].y, 3, 0xFFFF);
    }
  }
}

// --- Meteoriti “split” i sudari ---
void splitMeteor(Meteor &m) {
  m.active = false;
  int smallCount = random(2, 4);
  for (int s=0; s<smallCount; s++) {
    for (int i=0; i<MAX_METEORS; i++) {
      if (!meteors[i].active) {
        meteors[i] = m;
        meteors[i].active = true;
        meteors[i].size = m.size / 2 + random(-3, 3);
        meteors[i].isSmall = true;
        meteors[i].impact = meteors[i].size / 5 + 1;
        float angle = random(0, 360) * 0.01745;
        float spd = random(3, 7);
        meteors[i].dx = cos(angle) * spd;
        meteors[i].dy = sin(angle) * spd;
        meteors[i].angle += random(-30, 30) * 0.01745;
        meteors[i].rotationSpeed = m.rotationSpeed * (0.7 + random(-10,10)*0.01);
        meteors[i].color += random(-300, 300);
        break;
      }
    }
  }
}

// --- Glavni update ---
void updateGame() {
  updatePlayerMovement();

  // Meteoriti i ćelije
  if(millis() - lastSpawn > 1100) {
    generateMeteor();
    if(random(100) < 30) generateEnergyCell();
    lastSpawn = millis();
  }


for(int i=0;i<MAX_METEORS;i++) {
  if(meteors[i].active) {

  meteors[i].y += 1; // BRUTALNO ubrzaj pad da vidiš efekt!

    //Serial.printf("Meteor #%d: x=%.1f y=%.1f size=%d\n", i, meteors[i].x, meteors[i].y, meteors[i].size);
  }
}

  // Meteoriti “fizika”
  for(int i=0; i<MAX_METEORS; i++) {
    if(meteors[i].active) {
      meteors[i].x += meteors[i].dx;
      meteors[i].y += meteors[i].dy;
      meteors[i].angle += meteors[i].rotationSpeed;
      // Odbijanje od rubova
      if (meteors[i].x < meteors[i].size/2 || meteors[i].x > SCREEN_WIDTH-meteors[i].size/2)
        meteors[i].dx = -meteors[i].dx;
      //if (meteors[i].y > SCREEN_HEIGHT - meteors[i].size/2) {
      //  meteors[i].dy = -abs(meteors[i].dy); // odbij se samo od dna prema gore
      //}
      if (meteors[i].x < -50 || meteors[i].x > SCREEN_WIDTH+50 ||
          meteors[i].y < -50 || meteors[i].y > SCREEN_HEIGHT+50)
        meteors[i].active = false;

      // Meteor sudari s drugim meteoritom
      for(int j=i+1; j<MAX_METEORS; j++) {
        if (meteors[j].active) {
          float dx = meteors[j].x - meteors[i].x;
          float dy = meteors[j].y - meteors[i].y;
          float dist = sqrt(dx*dx+dy*dy);
          float minDist = meteors[i].size/2 + meteors[j].size/2;
          if (dist < minDist && dist > 1) {
            float nx = dx/dist, ny = dy/dist;
            float dvx = meteors[i].dx - meteors[j].dx;
            float dvy = meteors[i].dy - meteors[j].dy;
            float impact = dvx*nx + dvy*ny;
            if (impact < 0) {
              float force = (meteors[i].impact + meteors[j].impact) / 2.0;
              meteors[i].dx -= nx * force / meteors[i].impact;
              meteors[i].dy -= ny * force / meteors[i].impact;
              meteors[j].dx += nx * force / meteors[j].impact;
              meteors[j].dy += ny * force / meteors[j].impact;
            }
          }
        }
      }
      // Van ekrana
      if (meteors[i].x < -50 || meteors[i].x > SCREEN_WIDTH+50 ||
          meteors[i].y < -50 || meteors[i].y > SCREEN_HEIGHT+50)
        meteors[i].active = false;
    }
  }

  // Meteoriti i igrač sudar
  for(int i=0; i<MAX_METEORS; i++) {
    if(meteors[i].active) {
      float dx = player.x - meteors[i].x;
      float dy = player.y - meteors[i].y;
      float dist = sqrt(dx*dx + dy*dy);
      if(dist < PLAYER_SIZE/2 + meteors[i].size/2 * 0.85) {
        player.health -= meteors[i].isSmall ? 8 : 20;
        meteors[i].active = false;
        playCollisionSound();
      }
    }
  }

  // Energijske ćelije
  for(int i=0; i<MAX_CELLS; i++) {
    if(!cells[i].collected) {
      float dx = player.x - cells[i].x;
      float dy = player.y - cells[i].y;
      if (sqrt(dx*dx + dy*dy) < 28) {
        player.energy += 25;
        player.score += 100;
        cells[i].collected = true;
        playCollectSound();
      }
    }
  }

  updateBullets();

  // Metak i meteorit
  for (int b=0; b<BULLET_MAX; b++) {
    if (bullets[b].active) {
      for (int m=0; m<MAX_METEORS; m++) {
        if (meteors[m].active) {
          float dx = bullets[b].x - meteors[m].x;
          float dy = bullets[b].y - meteors[m].y;
          float d = sqrt(dx*dx + dy*dy);
          if (d < meteors[m].size/2) {
            bullets[b].active = false;
            if (!meteors[m].isSmall && meteors[m].size > 20) {
              splitMeteor(meteors[m]);
            } else {
              meteors[m].active = false;
            }
            player.score += meteors[m].isSmall ? 30 : 100;
            break;
          }
        }
      }
    }
  }

  if(player.health <= 0) gameOver = true;
}

// --- Crtanje meteora ---
void drawMeteor(Meteor &m) {
  float cx = m.x, cy = m.y;
  uint8_t pts = m.points;
  float a = m.angle;
  int16_t px[16], py[16];
  for(uint8_t j=0; j<pts; j++) {
    float theta = a + j * (2*PI/pts);
    float r = m.offsets[j];
    px[j] = cx + cos(theta) * r;
    py[j] = cy + sin(theta) * r;
  }
  for (uint8_t j = 0; j < pts; j++) {
    uint8_t next = (j + 1) % pts;
    canvas.fillTriangle(cx, cy, px[j], py[j], px[next], py[next], m.color);
    canvas.drawLine(px[j], py[j], px[next], py[next], 0xFFFF);
  }
}

// --- Player ship, refleksije, sjaj, topovi ---
void drawPlayerShip(float x, float y) {
  int shipW = PLAYER_SIZE;
  int shipH = PLAYER_SIZE+8;
  int canopyH = 10;
  int motorW = 6;
  int motorH = 14;
  int fireLen = 16;

  // Tijelo broda
  canvas.fillTriangle(
    x - shipW/2, y + shipH/2,
    x + shipW/2, y + shipH/2,
    x,           y - shipH/2 + 4,
    0xC618
  );
  canvas.drawTriangle(
    x - shipW/2, y + shipH/2,
    x + shipW/2, y + shipH/2,
    x,           y - shipH/2 + 4,
    0x528A
  );

  // Kupola + refleksija
  canvas.fillEllipse(x, y - shipH/4, shipW/4, canopyH/2, 0xA145); // Smeđa
  canvas.drawEllipse(x, y - shipH/4, shipW/4, canopyH/2, 0x630C);
  canvas.fillEllipse(x, y - shipH/4, shipW/7, canopyH/4, 0x033F); // plavi sjaj
  canvas.drawLine(x - shipW/12, y - shipH/4 - 1, x, y - shipH/4 - 3, 0xFFFF); // window reflection

  // Motori
  int mx1 = x - shipW/2 + motorW/2;
  int mx2 = x + shipW/2 - motorW/2;
  int my  = y + shipH/2 - motorH/2;
  canvas.fillRect(mx1 - motorW/2, my, motorW, motorH, 0x8410);
  canvas.fillRect(mx2 - motorW/2, my, motorW, motorH, 0x8410);
  canvas.drawRect(mx1 - motorW/2, my, motorW, motorH, 0xFFFF);
  canvas.drawRect(mx2 - motorW/2, my, motorW, motorH, 0xFFFF);

  // Vatra iz motora
  for (int f=0; f<3; f++) {
    int fireOff = random(-2,3);
    uint16_t fireColor = (random(2)==0) ? 0xF800 : 0xFFE0;
    canvas.drawLine(mx1, my+motorH, mx1 + fireOff, my+motorH+fireLen + random(4), fireColor);
    fireColor = (random(2)==0) ? 0xF800 : 0xFFE0;
    canvas.drawLine(mx2, my+motorH, mx2 + fireOff, my+motorH+fireLen + random(4), fireColor);
  }

  // Topovi (treptanje kad puca)
  int noseY = y - shipH/2 + 4;
  int noseX = x;
  uint16_t cannonColor = (cannonFired && millis() - lastCannonFlash < CANNON_FLASH_MS) ? 0xFFE0 : 0x528A;
  canvas.drawLine(noseX-6, noseY+4, noseX-6, noseY-8, cannonColor);
  canvas.drawLine(noseX+6, noseY+4, noseX+6, noseY-8, cannonColor);
  if (cannonFired && millis() - lastCannonFlash < CANNON_FLASH_MS) {
    canvas.drawPixel(noseX-6, noseY-8, 0xFFFF);
    canvas.drawPixel(noseX+6, noseY-8, 0xFFFF);
  }
}

// --- Crtanje igre ---
void drawGame() {
  canvas.fillScreen(0x0000);
  canvas.fillRect(0, 0, SCREEN_WIDTH, UI_HEIGHT, COLOR_UI_BG);

  // Meteoriti
  for(int i=0; i<MAX_METEORS; i++) if(meteors[i].active) drawMeteor(meteors[i]);
  // Energijske ćelije
  for(int i=0; i<MAX_CELLS; i++) {
    if(!cells[i].collected) {
      canvas.fillCircle(cells[i].x, cells[i].y, 11, 0x07E0);
      canvas.drawCircle(cells[i].x, cells[i].y, 14, 0xFFFF);
    }
  }
  // Player brod
  drawPlayerShip(player.x, player.y);
  // Metci
  drawBullets();

  // UI
  canvas.setTextColor(COLOR_TEXT);
  canvas.setTextSize(1);
  canvas.setCursor(10,10);  canvas.printf("Health: %d", player.health);
  canvas.setCursor(SCREEN_WIDTH-100,10);  canvas.printf("Energy: %d", player.energy);
  canvas.setCursor(SCREEN_WIDTH/2-30,10); canvas.printf("Score: %d", player.score);

  canvas.pushSprite(0,0);

  if (cannonFired && millis() - lastCannonFlash > CANNON_FLASH_MS)
    cannonFired = false;
}

// --- Zvuk ---
void playCollectSound() {
  ledcWriteTone(LEDC_CHANNEL, 1040);
  delay(36);
  ledcWriteTone(LEDC_CHANNEL, 0);
}

void playCollisionSound() {
  for(int i=320; i>=70; i-=80) {
    ledcWriteTone(LEDC_CHANNEL, i);
    delay(18);
  }
  ledcWriteTone(LEDC_CHANNEL, 0);
}

// --- Game over ekran ---
void showGameOver() {
  canvas.fillScreen(0x0000);
  canvas.setTextSize(2);
  canvas.setCursor(90,100);
  canvas.print("GAME OVER");
  canvas.setCursor(68,140);
  canvas.printf("Final Score: %d", player.score);
  canvas.setTextSize(1);
  canvas.setCursor(90, 180);
  canvas.print("Drzi tipku za restart");
  canvas.pushSprite(0,0);
  if(digitalRead(BTN_RESTART) == LOW) {
    initGame();
    delay(400);
  }
}

// --- Loop ---
void loop() {
  static bool lastFire = HIGH;
  bool nowFire = digitalRead(BTN_A);
  if (lastFire == HIGH && nowFire == LOW) {
    fireCannon();
  }
  lastFire = nowFire;

  if(!gameOver) {
    updateGame();
    drawGame();
    delay(16); // ~60 FPS
  } else {
    showGameOver();
  }
}

Pripazi: Gumbi GPIO34 i GPIO35 imaju ovakvu strukturu koda gdje se za jedan gumb traže vrijednosti od oko 2000 (1800 do 2200), a za drugi gumb 4095 ali koristimo > 4000 radi preciznosti.
  // Lijevo
  if (analogLR > 4000) player.x -= MOVE_STEP;
  // Desno
  if (analogLR > 1800 && analogLR < 2200) player.x += MOVE_STEP;
  // Gore
  if (analogUD > 4000) player.y -= MOVE_STEP;
  // Dolje
  if (analogUD > 1800 && analogUD < 2200) player.y += MOVE_STEP;

Pripazi: Ne smiju sve podmornice pucati istovremeno. Neka puca jedna po jedna podmornica.
Pripazi: Ukoliko neprijateljski torpedo pogodi neprijateljsku podmornicu ona također umire. Ukoliko neprijateljski torpedo pogodi bilo koji drugi neprijateljski torpedo oba torpeda umiru.
Pripazi: Ukoliko neprijateljska podmornica dotakne drugu neprijateljsku podmornicu obje umiru.

Bitno je: Nemoj biti lijen i napiši kompletan kod za igru kako bi ga mogao copy paste u editor koda.
```
