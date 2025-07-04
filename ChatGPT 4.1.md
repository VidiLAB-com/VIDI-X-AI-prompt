# ChatGPT 4.1

[https://chatgpt.com/share/686534ab-521c-8006-b978-653e9adb2d07](https://chatgpt.com/share/686534ab-521c-8006-b978-653e9adb2d07)

* * *

## Upute za izradu podmorničke igre za VIDI X (ESP32)

Ova igra simulira upravljanje vlastitom podmornicom na mapi mora, gdje su prisutne i neprijateljske podmornice koje ispaljuju torpeda na temelju predviđanja tvoje putanje. Tvoj cilj je izbjegavati neprijateljska torpeda, pogoditi neprijateljske podmornice svojim torpedom i preživjeti što duže.

### Osnovne kontrole (gumbi):

- **GPIO35**: Zaroni (gore, &gt;4000), Izroni (dolje, 1800-2200)
- **GPIO34**: Zakreni lijevo (lijevo, &gt;4000), Zakreni desno (desno, 1800-2200)
- **GPIO0**: Smanji brzinu (tipka, pullup)
- **GPIO13**: Povećaj brzinu (tipka, pullup)
- **GPIO32**: Ispali torpedo (tipka, pullup)
- **GPIO27**: Torpedo lijevo (tipka, pullup)
- **GPIO39**: Torpedo desno (tipka, pullup)

### Prikaz na ekranu:

- Tvoja podmornica: oznaka "X"
- Smjer i brzina: prikazano s lijeve strane
- Torpedo: mala crta dužine 4 piksela u smjeru ispaljivanja
- Neprijateljske podmornice: "E"
- Neprijateljska torpeda: crvena crta
- UI traka na dnu s kontrolama i podacima

### Pravila:

- Podmornica ima 5 brzina: 3 naprijed, STOP, 1 unatrag (maks 100, min -40, promjena u koracima po 35)
- Smjer se mijenja za 5° po kliku
- Zaranjanje i izranjanje sporo – promjena dubine je vrlo postepena
- Torpedo ima konstantnu brzinu (200)
- Neprijateljske podmornice se kreću nasumično, svaka ima max 10 torpeda, ne pucaju istovremeno
- Neprijateljski torpedo pogađa igrača ako se križaju putanje i nisi promijenio smjer/brzinu
- Sudari među neprijateljskim podmornicama i torpedima su omogućeni (umiru)
- Igra završava kad te pogodi torpedo ili dotakneš neprijatelja

### Gumbi GPIO34 i GPIO35 koriste se kao analogni, svaki s dvije vrijednosti (gore/dolje, lijevo/desno).

* * *

## Kôd za igru (copy-paste ready za Arduino IDE):

```cpp
#define LGFX_USE_V1
#include <LovyanGFX.hpp>
#include <FastLED.h>
#include <math.h>

// -- HARDVER KONFIGURACIJA --
#define SCREEN_WIDTH  320
#define SCREEN_HEIGHT 240
#define UI_HEIGHT     38

#define LED_PIN       26
#define NUM_LEDS      5
CRGB leds[NUM_LEDS];

#define SPEAKER_PIN   25
#define LEDC_CHANNEL  0
#define LEDC_RES      8
#define LEDC_FREQ     1800

#define BTN_LR        34   // Analogni: >4000 lijevo, 1800-2200 desno
#define BTN_UD        35   // Analogni: >4000 zaroni, 1800-2200 izroni
#define BTN_FIRE      32   // Torpedo fire
#define BTN_SLOW      0    // Smanji brzinu
#define BTN_FAST      13   // Povećaj brzinu
#define BTN_TORP_L    27   // Torpedo lijevo
#define BTN_TORP_R    39   // Torpedo desno

#define MAX_ENEMIES   10
#define MAX_ETORP     10
#define MAX_PLAYER_TORPEDO 1

// -- GRAFIČKI EKRAN --
class LGFX : public lgfx::LGFX_Device {
  lgfx::Panel_ILI9341 _panel_instance;
  lgfx::Bus_SPI _bus_instance;
public:
  LGFX(void) {
    { auto cfg = _bus_instance.config();
      cfg.spi_host = VSPI_HOST; cfg.spi_mode = 0;
      cfg.freq_write = 40000000; cfg.freq_read = 16000000;
      cfg.spi_3wire = true; cfg.use_lock = true; cfg.dma_channel = 1;
      cfg.pin_sclk = 18; cfg.pin_mosi = 23; cfg.pin_miso = 19; cfg.pin_dc = 21;
      _bus_instance.config(cfg); _panel_instance.setBus(&_bus_instance);
    }
    { auto cfg = _panel_instance.config();
      cfg.pin_cs = 5; cfg.pin_rst = -1; cfg.pin_busy = -1;
      cfg.memory_width = 240; cfg.memory_height = 320;
      cfg.panel_width = 240; cfg.panel_height = 320;
      cfg.offset_x = 0; cfg.offset_y = 0; cfg.offset_rotation = 1;
      cfg.dummy_read_pixel = 8; cfg.dummy_read_bits = 1; cfg.readable = true;
      cfg.invert = false; cfg.rgb_order = false; cfg.dlen_16bit = false; cfg.bus_shared = true;
      _panel_instance.config(cfg);
    }
    setPanel(&_panel_instance);
  }
};
static LGFX lcd;
static LGFX_Sprite canvas(&lcd);

// --- DATA STRUKTURE ---
struct Torpedo {
  float x, y, dx, dy, angle;
  int owner; // 0 = player, 1-10 = enemy#
  bool active;
};

struct Submarine {
  float x, y, depth;
  float angle;     // smjer kretanja u stupnjevima
  float speed;     // -40 do +100
  int torpeda;     // preostalo torpeda
  bool alive;
  unsigned long nextFire;
  Torpedo torpedo;
};

Submarine player;
Submarine enemies[MAX_ENEMIES];
Torpedo eTorpedos[MAX_ETORP];
Torpedo playerTorpedo;
int score = 0;
bool gameOver = false;

// --- UI LOGIKA ---
const char* SPEED_LABELS[5] = {"<", "STOP", "1", "2", "3"};
int speedValues[5] = {-40, 0, 35, 70, 100};
int speedLevel = 1; // start na STOP

void playBeep(int freq, int t) { ledcWriteTone(LEDC_CHANNEL, freq); delay(t); ledcWriteTone(LEDC_CHANNEL, 0);}
void flashLED(int n) { for(int i=0; i<NUM_LEDS; i++) leds[i] = (i<n)? CRGB::Red : CRGB::Black; FastLED.show(); }

// --- SETUP ---
void setup() {
  Serial.begin(115200);
  lcd.init(); lcd.setRotation(0); lcd.fillScreen(0x0000);
  canvas.setPsram(true); canvas.createSprite(SCREEN_WIDTH, SCREEN_HEIGHT);
  FastLED.addLeds<WS2812B, LED_PIN, GRB>(leds, NUM_LEDS);
  ledcSetup(LEDC_CHANNEL, LEDC_FREQ, LEDC_RES);
  ledcAttachPin(SPEAKER_PIN, LEDC_CHANNEL);

  pinMode(BTN_LR, INPUT);  pinMode(BTN_UD, INPUT);
  pinMode(BTN_FIRE, INPUT_PULLUP); pinMode(BTN_SLOW, INPUT_PULLUP);
  pinMode(BTN_FAST, INPUT_PULLUP); pinMode(BTN_TORP_L, INPUT_PULLUP);
  pinMode(BTN_TORP_R, INPUT_PULLUP);

  randomSeed(millis());
  initGame();
}

void initGame() {
  player = { SCREEN_WIDTH/2, SCREEN_HEIGHT/2, 70, 0, 0, 10, true, 0, {0,0,0,0,0,0,false}};
  speedLevel = 1;
  score = 0;
  playerTorpedo.active = false;
  for(int i=0; i<MAX_ENEMIES; i++) {
    float ang = random(360);
    float spd = 100.0;
    enemies[i] = {
      (float)random(30, SCREEN_WIDTH-30),
      (float)random(40, SCREEN_HEIGHT-UI_HEIGHT-40),
      (float)random(20,200),         // dubina
      ang,                           // smjer
      spd,                           // brzina
      10, true, millis() + random(2000,6000),
      {0,0,0,0,0,0,false}
    };
  }
  for(int i=0; i<MAX_ETORP; i++) eTorpedos[i].active = false;
  gameOver = false;
  flashLED(0);
}

// --- INPUT ---
void handleInput() {
  // Brzina
  if (digitalRead(BTN_SLOW)==LOW) { if(speedLevel>0) {speedLevel--; playBeep(900,30);} delay(200);}
  if (digitalRead(BTN_FAST)==LOW) { if(speedLevel<4) {speedLevel++; playBeep(1300,30);} delay(200);}
  player.speed = speedValues[speedLevel];

  // Smjer (analogni)
  int analogLR = analogRead(BTN_LR);
  if (analogLR > 4000) { player.angle -= 5; if(player.angle<0) player.angle+=360; playBeep(1100,16);}
  if (analogLR > 1800 && analogLR < 2200) { player.angle += 5; if(player.angle>=360) player.angle-=360; playBeep(1300,16);}
  // Dubina (analogni)
  int analogUD = analogRead(BTN_UD);
  if (analogUD > 4000) player.depth += 0.7; // Zaroni
  if (analogUD > 1800 && analogUD < 2200) player.depth -= 0.7; // Izroni
  if (player.depth < 0) player.depth = 0;
  if (player.depth > 230) player.depth = 230;

  // Ispali torpedo (samo jedan istovremeno)
  static bool lastFire = HIGH;
  bool nowFire = digitalRead(BTN_FIRE);
  if(lastFire==HIGH && nowFire==LOW && player.torpeda>0 && !playerTorpedo.active) {
    float angRad = player.angle * 0.0174533;
    playerTorpedo = {
      player.x + cos(angRad)*16,
      player.y + sin(angRad)*16,
      cos(angRad)*7,
      sin(angRad)*7,
      player.angle,
      0,
      true
    };
    player.torpeda--;
    playBeep(2100,40);
    flashLED(2);
  }
  lastFire = nowFire;

  // Torpedo skretanje
  if (playerTorpedo.active) {
    if (digitalRead(BTN_TORP_L)==LOW) {
      playerTorpedo.angle -= 5; if(playerTorpedo.angle<0) playerTorpedo.angle+=360;
      float angRad = playerTorpedo.angle * 0.0174533;
      playerTorpedo.dx = cos(angRad)*7;
      playerTorpedo.dy = sin(angRad)*7;
    }
    if (digitalRead(BTN_TORP_R)==LOW) {
      playerTorpedo.angle += 5; if(playerTorpedo.angle>=360) playerTorpedo.angle-=360;
      float angRad = playerTorpedo.angle * 0.0174533;
      playerTorpedo.dx = cos(angRad)*7;
      playerTorpedo.dy = sin(angRad)*7;
    }
  }
}

// --- UPDATE TORPEDO ---
void updateTorpedo(Torpedo& t) {
  if(!t.active) return;
  t.x += t.dx;
  t.y += t.dy;
  if(t.x<0 || t.x>SCREEN_WIDTH || t.y<0 || t.y>SCREEN_HEIGHT-UI_HEIGHT) t.active=false;
}

// --- ENEMY AI ---
void enemyAI() {
  for(int i=0; i<MAX_ENEMIES; i++) {
    if(!enemies[i].alive) continue;
    // Random promjena smjera i dubine
    if(random(100)<4) enemies[i].angle += random(-8,9);
    if(enemies[i].angle<0) enemies[i].angle+=360;
    if(enemies[i].angle>=360) enemies[i].angle-=360;
    if(random(100)<5) enemies[i].depth += random(-3,4);
    if(enemies[i].depth<0) enemies[i].depth=0;
    if(enemies[i].depth>230) enemies[i].depth=230;

    // Kretanje
    float angRad = enemies[i].angle * 0.0174533;
    enemies[i].x += cos(angRad) * (enemies[i].speed/30.0);
    enemies[i].y += sin(angRad) * (enemies[i].speed/30.0);
    // Rubovi
    if(enemies[i].x<12) enemies[i].x=12;
    if(enemies[i].x>SCREEN_WIDTH-12) enemies[i].x=SCREEN_WIDTH-12;
    if(enemies[i].y<14) enemies[i].y=14;
    if(enemies[i].y>SCREEN_HEIGHT-UI_HEIGHT-14) enemies[i].y=SCREEN_HEIGHT-UI_HEIGHT-14;

    // Pucanje (jedan torpedo po neprijatelju istovremeno)
    unsigned long now = millis();
    if(now > enemies[i].nextFire && enemies[i].torpeda>0) {
      // Samo ako igrač nije već pogođen torpedom
      for(int t=0;t<MAX_ETORP;t++) if(!eTorpedos[t].active) {
        // "Pametni" cilj: izračunaj gdje će igrač biti
        float dx = player.x - enemies[i].x;
        float dy = player.y - enemies[i].y;
        float distance = sqrt(dx*dx + dy*dy);
        float timeToHit = distance/7.0;
        float futureX = player.x + cos(player.angle*0.0174533) * player.speed/30.0 * timeToHit;
        float futureY = player.y + sin(player.angle*0.0174533) * player.speed/30.0 * timeToHit;
        float targetAngle = atan2(futureY-enemies[i].y, futureX-enemies[i].x)*57.2958;
        if(targetAngle<0) targetAngle+=360;
        float angRad2 = targetAngle * 0.0174533;
        eTorpedos[t] = {
          enemies[i].x + cos(angRad2)*16,
          enemies[i].y + sin(angRad2)*16,
          cos(angRad2)*7,
          sin(angRad2)*7,
          targetAngle,
          i+1,
          true
        };
        enemies[i].torpeda--;
        enemies[i].nextFire = now + random(1800,4200);
        playBeep(700,24);
        break;
      }
    }
  }
}

// --- SUDARI, DETEKCIJA ---
void collisions() {
  // Player pogoden torpedom
  for(int t=0;t<MAX_ETORP;t++) if(eTorpedos[t].active) {
    float dx = player.x - eTorpedos[t].x, dy = player.y - eTorpedos[t].y;
    if(sqrt(dx*dx+dy*dy)<12 && player.alive) {
      player.alive=false; gameOver=true; playBeep(400,150); flashLED(NUM_LEDS);
      eTorpedos[t].active=false; break;
    }
  }
  // Player pogoden neprijateljem
  for(int i=0;i<MAX_ENEMIES;i++) if(enemies[i].alive) {
    float dx = player.x - enemies[i].x, dy = player.y - enemies[i].y;
    if(sqrt(dx*dx+dy*dy)<16 && player.alive) {
      player.alive=false; gameOver=true; playBeep(400,150); flashLED(NUM_LEDS);
      break;
    }
  }
  // Player torpedo pogadja neprijatelja
  if(playerTorpedo.active) {
    for(int i=0;i<MAX_ENEMIES;i++) if(enemies[i].alive) {
      float dx = playerTorpedo.x - enemies[i].x, dy = playerTorpedo.y - enemies[i].y;
      if(sqrt(dx*dx+dy*dy)<15) {
        enemies[i].alive=false; playerTorpedo.active=false;
        score+=120; playBeep(2000,40); flashLED(4); break;
      }
    }
  }
  // Torpedo vs torpedo
  for(int t=0;t<MAX_ETORP;t++) if(eTorpedos[t].active && playerTorpedo.active) {
    float dx = eTorpedos[t].x - playerTorpedo.x, dy = eTorpedos[t].y - playerTorpedo.y;
    if(sqrt(dx*dx+dy*dy)<8) {
      eTorpedos[t].active=false; playerTorpedo.active=false;
      playBeep(900,30); break;
    }
  }
  // Enemies međusobni sudar
  for(int i=0;i<MAX_ENEMIES;i++) if(enemies[i].alive) {
    for(int j=i+1;j<MAX_ENEMIES;j++) if(enemies[j].alive) {
      float dx = enemies[i].x - enemies[j].x, dy = enemies[i].y - enemies[j].y;
      if(sqrt(dx*dx+dy*dy)<20) {
        enemies[i].alive=false; enemies[j].alive=false;
        playBeep(300,25); playBeep(700,20); break;
      }
    }
  }
  // Torpedo pogađa neprijatelja (njihov torpedo)
  for(int t=0;t<MAX_ETORP;t++) if(eTorpedos[t].active) {
    for(int i=0;i<MAX_ENEMIES;i++) if(enemies[i].alive) {
      float dx = eTorpedos[t].x - enemies[i].x, dy = eTorpedos[t].y - enemies[i].y;
      if(sqrt(dx*dx+dy*dy)<10) {
        enemies[i].alive=false; eTorpedos[t].active=false;
        playBeep(1000,25); break;
      }
    }
  }
  // Torpedo vs torpedo (međusobno uništenje)
  for(int t=0;t<MAX_ETORP;t++) if(eTorpedos[t].active) {
    for(int k=t+1;k<MAX_ETORP;k++) if(eTorpedos[k].active) {
      float dx = eTorpedos[t].x - eTorpedos[k].x, dy = eTorpedos[t].y - eTorpedos[k].y;
      if(sqrt(dx*dx+dy*dy)<8) {
        eTorpedos[t].active=false; eTorpedos[k].active=false;
        playBeep(700,12); break;
      }
    }
  }
}

// --- GAME UPDATE ---
void updateGame() {
  if(!player.alive) {gameOver=true; return;}
  handleInput();
  // Kretanje playera
  float angRad = player.angle * 0.0174533;
  player.x += cos(angRad) * (player.speed/30.0);
  player.y += sin(angRad) * (player.speed/30.0);
  if(player.x<16) player.x=16; if(player.x>SCREEN_WIDTH-16) player.x=SCREEN_WIDTH-16;
  if(player.y<16) player.y=16; if(player.y>SCREEN_HEIGHT-UI_HEIGHT-16) player.y=SCREEN_HEIGHT-UI_HEIGHT-16;

  // Torpedo playera
  updateTorpedo(playerTorpedo);

  // Kretanje neprijatelja i njihovi torpedi
  enemyAI();
  for(int i=0;i<MAX_ETORP;i++) updateTorpedo(eTorpedos[i]);
  collisions();
}

// --- CRTANJE ---
void drawSub(float x, float y, float angle, bool main) {
  int r=main?13:11;
  int clr=main?0x07FF:0xF800;
  canvas.fillCircle(x,y,r,clr);
  if(main) canvas.setTextColor(0xFFFF); else canvas.setTextColor(0xFFE0);
  canvas.setTextSize(main?2:1);
  canvas.setCursor(x-6,y-7); canvas.print(main?"X":"E");
  // smjer
  float angRad = angle*0.0174533;
  canvas.drawLine(x, y, x+cos(angRad)*r, y+sin(angRad)*r, 0xFFFF);
}

void drawTorpedo(const Torpedo& t, int color) {
  if(!t.active) return;
  float nx = t.x + cos(t.angle*0.0174533)*4;
  float ny = t.y + sin(t.angle*0.0174533)*4;
  canvas.drawLine(t.x, t.y, nx, ny, color);
}

void drawGame() {
  canvas.fillScreen(0x0141); // tamnoplavo more
  // Player
  if(player.alive) drawSub(player.x, player.y, player.angle, true);
  // Player torpedo
  drawTorpedo(playerTorpedo, 0xFFFF);
  // Enemies i njihovi torpedi
  for(int i=0;i<MAX_ENEMIES;i++) if(enemies[i].alive)
    drawSub(enemies[i].x, enemies[i].y, enemies[i].angle, false);
  for(int i=0;i<MAX_ETORP;i++) if(eTorpedos[i].active)
    drawTorpedo(eTorpedos[i], 0xF800);

  // UI
  canvas.fillRect(0,SCREEN_HEIGHT-UI_HEIGHT,SCREEN_WIDTH,UI_HEIGHT,0x0012);
  canvas.setTextSize(1);
  canvas.setTextColor(0xFFFF);
  canvas.setCursor(10,SCREEN_HEIGHT-34); canvas.printf("Brzina: %s (%d)", SPEED_LABELS[speedLevel], (int)player.speed);
  canvas.setCursor(10,SCREEN_HEIGHT-22); canvas.printf("Smjer: %.0f\xB0", player.angle);
  canvas.setCursor(110,SCREEN_HEIGHT-34); canvas.printf("Dubina: %.1f", player.depth);
  canvas.setCursor(110,SCREEN_HEIGHT-22); canvas.printf("Torpeda: %d", player.torpeda);
  canvas.setCursor(220,SCREEN_HEIGHT-30); canvas.printf("Score: %d", score);

  canvas.pushSprite(0,0);
}

// --- GAME OVER EKRAN ---
void showGameOver() {
  canvas.fillScreen(0x0000);
  canvas.setTextColor(0xFFE0);
  canvas.setTextSize(2);
  canvas.setCursor(90,90); canvas.print("GAME OVER");
  canvas.setTextSize(1);
  canvas.setCursor(80,140); canvas.printf("Score: %d", score);
  canvas.setCursor(62,180); canvas.print("Drzi FIRE za restart");
  canvas.pushSprite(0,0);
  // Restart
  if(digitalRead(BTN_FIRE)==LOW) { delay(500); initGame(); }
}

// --- LOOP ---
void loop() {
  if(!gameOver) {
    updateGame();
    drawGame();
    delay(18);
  } else {
    showGameOver();
  }
}
```

* * *

Kada kopiraš kod u Arduino IDE i prevedeš za VIDI X, dobit ćeš upravo traženo iskustvo: napetu, ali “nježno-futurističku” igru u kojoj, uz tvoje odluke, svaka sekunda znači život ili smrt pod morem.  
Ako želiš dodatno objašnjenje bilo kojeg dijela, ili ti je potreban README s kratkim opisom kontrola, napiši *nastavi* ili *continue*.  
Uvijek ću biti tu da ti pomognem.
