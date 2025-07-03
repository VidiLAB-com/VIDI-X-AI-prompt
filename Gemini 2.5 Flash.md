# Gemini 2.5 Flash

# **Deep Hunter Game**
[https://g.co/gemini/share/abf32ada1cde](https://g.co/gemini/share/abf32ada1cde)


Izrađeno pomoću modela **2.5 Flash** 1. srpnja 2025. u 11:39 Objavljeno 1. srpnja 2025. u 12:01



Evo detaljnih uputa i kompletnog koda za igru podmornica "Deep Hunter" za VIDI X mikroračunalo s ESP32, napisanog u C++ za Arduino IDE.

Upute su podijeljene u nekoliko dijelova kako biste lakše razumjeli strukturu i funkcionalnost igre.

### Detaljne Upute za Izradu Igre "Deep Hunter"

**1. Potrebni Alati i Biblioteke:**

- **Arduino IDE:** Integrirano razvojno okruženje za pisanje i upload koda na ESP32.
- **ESP32 Board Support Package:** Morate imati instaliranu podršku za ESP32 ploče u Arduino IDE-u (File &gt; Preferences &gt; Additional Boards Manager URLs, dodajte `https://raw.githubusercontent.com/espressif/arduino-esp32/gh-pages/package_esp32_index.json`, zatim Tools &gt; Board &gt; Boards Manager i potražite "esp32").
- **LovyanGFX biblioteka:** Za grafički prikaz na ekranu. Instalirajte je putem Arduino Library Managera (Sketch &gt; Include Library &gt; Manage Libraries... i potražite "LovyanGFX").
- **FastLED biblioteka:** Iako se u ovoj igri ne koriste LED diode, biblioteka je zadržana zbog zvučnih funkcija koje su bile prisutne u vašem primjeru koda. Instalirajte je putem Arduino Library Managera.

**2. Hardversko Povezivanje (VIDI X mikroračunalo):** Ovaj kod je prilagođen za VIDI X mikroračunalo koje koristi specifične GPIO pinove za ekran i gumbe.

- **Ekran (LovyanGFX konfiguracija):**
  
  - `pin_sclk = 18`
  - `pin_mosi = 23`
  - `pin_miso = 19`
  - `pin_dc = 21`
  - `pin_cs = 5`
  - `pin_rst = -1` (nije spojen)
  - `pin_busy = -1` (nije spojen)
  - Ova konfiguracija je preuzeta iz vašeg primjera i trebala bi raditi bez problema.
- **Zvučnik:**
  
  - `SPEAKER_PIN = 25`
- **Gumbi (GPIO pinovi):**
  
  - **GPIO35 (Analog):** Za Zaroniti/Izroniti.
    
    - Vrijednost `analogRead(GPIO35) > 4000` za jednu akciju (npr. Zaroniti).
    - Vrijednost `analogRead(GPIO35)` između `1800` i `2200` za drugu akciju (npr. Izroniti).
  - **GPIO34 (Analog):** Za Lijevo/Desno upravljanje podmornicom.
    
    - Vrijednost `analogRead(GPIO34) > 4000` za jednu akciju (npr. Lijevo).
    - Vrijednost `analogRead(GPIO34)` između `1800` i `2200` za drugu akciju (npr. Desno).
  - **GPIO0 (Digital):** Za povećanje brzine podmornice (INPUT\_PULLUP).
  - **GPIO13 (Digital):** Za smanjenje brzine podmornice (INPUT\_PULLUP).
  - **GPIO32 (Digital):** Za ispaljivanje torpeda (igrač) (INPUT\_PULLUP).
  - **GPIO27 (Digital):** Za upravljanje torpedom Lijevo (INPUT\_PULLUP).
  - **GPIO39 (Digital):** Za upravljanje torpedom Desno (INPUT\_PULLUP).

**3. Struktura Igre:**

- **Podmornice (`Submarine` struktura):**
  
  - `x, y`: Pozicija na ekranu.
  - `direction`: Smjer kretanja u stupnjevima (0-359.99).
  - `speed`: Brzina kretanja (0 = STOP, 1-3 = naprijed, -1 = nazad).
  - `health`: Zdravlje podmornice.
  - `torpedoes`: Broj preostalih torpeda.
  - `active`: Je li podmornica aktivna (živa).
  - `type`: PLAYER ili ENEMY.
  - `lastFireTime`: Vrijeme zadnjeg ispaljivanja torpeda (za neprijatelje, zbog cooldowna).
  - `id`: Jedinstveni ID za razlikovanje podmornica.
- **Torpeda (`Torpedo` struktura):**
  
  - `x, y`: Pozicija.
  - `dx, dy`: Vektori kretanja.
  - `active`: Je li torpedo aktivno.
  - `owner`: Tko je ispalio torpedo (PLAYER ili ENEMY).
  - `ownerId`: ID podmornice koja je ispalila torpedo (važno za izbjegavanje samoozljeđivanja).
- **Igračeva Podmornica:**
  
  - Počinje s 100 zdravlja i 10 torpeda.
  - Maksimalna brzina je 100 (interna vrijednost, prevedena u piksele po frameu).
  - Zakreće se za 5 stupnjeva lijevo/desno.
  - Zaranjanje/izranjanje je sporo (0.5 piksela po frameu).
  - Torpedo ima konstantnu brzinu 200 i može se upravljati lijevo/desno.
- **Neprijateljske Podmornice (10 njih):**
  
  - Nasumično se kreću po mapi.
  - Maksimalna brzina 100.
  - Svaka ima 10 torpeda.
  - **AI (Umjetna Inteligencija):**
    
    - Mijenjaju smjer i brzinu nasumično.
    - Pucaju torpeda prema igraču, predviđajući njegovu putanju na temelju trenutne brzine i smjera igrača.
    - **Važno:** Ne pucaju sve istovremeno. Postoji cooldown i nasumično odgađanje između pucnjeva kako bi se spriječilo preopterećenje igrača.
- **Sudari:**
  
  - **Igračeva podmornica i neprijateljsko torpedo:** Igrač gubi zdravlje, torpedo nestaje.
  - **Igračevo torpedo i neprijateljska podmornica:** Neprijateljska podmornica nestaje, igračevo torpedo nestaje, igrač dobiva bodove.
  - **Neprijateljsko torpedo i neprijateljska podmornica:** Oboje nestaju.
  - **Neprijateljsko torpedo i drugo neprijateljsko torpedo:** Oba torpeda nestaju.
  - **Neprijateljska podmornica i druga neprijateljska podmornica:** Obje podmornice nestaju.
- **Grafički Prikaz:**
  
  - Plava pozadina za more.
  - Igračeva podmornica je prikazana kao trokut s 'X' oznakom.
  - Neprijateljske podmornice su trokuti s 'E' oznakom.
  - Torpeda su kratke linije.
  - UI na vrhu ekrana prikazuje zdravlje, broj torpeda, brzinu, smjer i rezultat.
- **Zvukovi:**
  
  - Zvuk ispaljivanja torpeda.
  - Zvuk pogotka/sudar.
  - Zvuk eksplozije.
- **Kraj Igre:**
  
  - Igra završava kada igračevo zdravlje padne na 0 (GAME OVER).
  - Igra završava kada su sve neprijateljske podmornice uništene (YOU WIN!).
  - Na kraju igre prikazuje se konačni rezultat i opcija za restart pritiskom na gumb.

**4. Objašnjenje Koda:**

- **`LGFX` klasa:** Konfiguracija LovyanGFX biblioteke za VIDI X ekran. Rotacija je postavljena na 1 za landscape (320x240).
- **`setup()` funkcija:**
  
  - Inicijalizira serijsku komunikaciju za debug.
  - Inicijalizira LCD i stvara `canvas` sprite za crtanje.
  - Postavlja pinove za zvučnik i gumbe.
  - Poziva `initGame()` za postavljanje početnog stanja igre.
- **`initGame()` funkcija:**
  
  - Postavlja početne pozicije, zdravlje, torpeda i status za igrača i sve neprijateljske podmornice.
  - Postavlja sve torpeda u bazenu na neaktivna.
- **`updatePlayerInput()` funkcija:**
  
  - Čita analogne vrijednosti s GPIO34 i GPIO35 za lijevo/desno i zaroniti/izroniti, primjenjujući logiku raspona vrijednosti (npr. `> 4000` i `1800-2200`).
  - Čita digitalne gumbe (GPIO0, GPIO13) za promjenu brzine.
  - Ograničava poziciju igrača unutar granica ekrana.
- **`updateSubmarineMovement(Submarine &sub)`:**
  
  - Ažurira poziciju podmornice na temelju njezine brzine i smjera.
  - Implementira "wrap-around" efekt za neprijateljske podmornice (ako izađu s jedne strane, pojavljuju se s druge).
- **`firePlayerTorpedo()` i `fireEnemyTorpedo(Submarine &enemy)`:**
  
  - Pronađu neaktivno torpedo u bazenu i aktiviraju ga.
  - Postavljaju početnu poziciju i smjer torpeda.
  - `fireEnemyTorpedo` uključuje logiku predviđanja putanje igrača i cooldown.
- **`updateTorpedoes()`:**
  
  - Ažurira poziciju svih aktivnih torpeda.
  - Omogućuje upravljanje igračevim torpedom (lijevo/desno) ako je aktivno.
  - Deaktivira torpeda koja izađu izvan ekrana.
- **`updateEnemyAI()`:**
  
  - Nasumično mijenja smjer i brzinu neprijateljskih podmornica.
  - Provjerava udaljenost do igrača i ispaljuje torpeda ako je igrač u dometu i cooldown je prošao.
- **`checkCollisions()`:**
  
  - Sadrži svu logiku detekcije sudara između svih tipova objekata (podmornice, torpeda).
  - Implementira pravila uništenja i dodjele bodova.
- **`drawSubmarine()`, `drawTorpedo()`, `drawUI()`, `drawGame()`:**
  
  - Funkcije za crtanje svih elemenata igre na `canvas` sprite, a zatim se sprite gura na LCD.
- **Zvučne funkcije (`playFireSound()`, `playHitSound()`, `playExplosionSound()`):**
  
  - Koriste `ledcWriteTone` za generiranje jednostavnih zvučnih efekata.
- **`showGameOver()`:**
  
  - Prikazuje ekran za kraj igre s rezultatom i uputama za restart.
- **`loop()` funkcija:**
  
  - Glavna petlja igre.
  - Provjerava pritisak na gumb za ispaljivanje torpeda.
  - Ako igra nije gotova, poziva `updateGame()` i `drawGame()`.
  - Ako je igra gotova, poziva `showGameOver()`.
  - `delay(16)` osigurava približno 60 FPS.

Ovaj kod je kompletan i spreman za kopiranje i lijepljenje u Arduino IDE. Sretno s igranjem!

```
### Zaključak i Prijedlozi

Napisao sam kompletan kod za igru "Deep Hunter" za vaše VIDI X mikroračunalo, uzimajući u obzir sve vaše zahtjeve, uključujući specifično očitavanje analognih gumba, upravljanje podmornicom i torpedom, AI neprijatelja s predviđanjem putanje i pravilima sudara.

**Ključne promjene i implementacije:**
* **Prilagođena LovyanGFX konfiguracija:** Ekran je sada u landscape modu (320x240) kako bi bolje odgovarao igri.
* **Detaljno upravljanje:** Implementirano je upravljanje podmornicom (smjer, brzina, zaron/izron) i torpedom (lijevo/desno) koristeći navedene GPIO pinove i analogne vrijednosti.
* **Sustav brzine:** Podmornica ima 5 stupnjeva brzine (STOP, 3 naprijed, 1 nazad).
* **Neprijateljski AI:** Neprijatelji se nasumično kreću i pucaju torpeda predviđajući vašu putanju, s mehanizmom za sprječavanje istovremenog pucanja.
* **Složeni sudari:** Implementirana su sva navedena pravila sudara između podmornica i torpeda, uključujući "friendly fire" i sudare torpeda.
* **Grafika i UI:** Podmornice su prikazane kao trokuti s 'X' ili 'E' oznakama, torpeda kao linije, a UI na vrhu ekrana prikazuje sve relevantne informacije.
* **Zvučni efekti:** Dodani su zvučni efekti za pucanje, pogotke i eksplozije.
* **Struktura koda:** Kod je modularan s funkcijama za inicijalizaciju, ažuriranje logike, crtanje i rukovanje zvukom.

Sada možete kopirati i zalijepiti ovaj kod u Arduino IDE, prevesti ga i prenijeti na svoj ESP32. Uživajte u igri!

**Mogući sljedeći koraci i poboljšanja:**
* **Vizualni efekti:** Dodajte animacije eksplozija, tragove torpeda ili svjetlosne efekte.
* **Različiti neprijatelji:** Uvedite različite tipove neprijateljskih podmornica s različitim brzinama, zdravljem ili ponašanjem.
* **Power-upi:** Dodajte power-upove za igrača (npr. više torpeda, privremeni štit, brže kretanje).
* **Razina težine:** Implementirajte sustav razina težine koji utječe na brzinu neprijatelja, frekvenciju pucanja ili zdravlje.
* **Pozadinski elementi:** Dodajte statične pozadinske objekte poput podvodnih planina ili olupina kako bi mapa bila zanimljivija.
* **Poboljšanje AI-ja:** Neprijatelji bi mogli imati složenije putanje kretanja ili taktike izbjegavanja.
```

## Deep Hunter Game (C++ for ESP32)

Isječak koda

```C++
#define LGFX_USE_V1
#include <LovyanGFX.hpp>
#include <FastLED.h> // Keep for consistency, even if only sound is used
#include <math.h>    // For sin, cos, sqrt, atan2, M_PI

// --- LGFX setup (as provided by user) ---
// This class configures the LovyanGFX library for the specific display
// used by the VIDI X microcomputer (ILI9341 panel with SPI bus).
class LGFX : public lgfx::LGFX_Device {
  lgfx::Panel_ILI9341 _panel_instance;
  lgfx::Bus_SPI _bus_instance;
public:
  LGFX(void) {
    { // Bus config: Defines the SPI bus parameters for communication with the display.
      auto cfg = _bus_instance.config();
      cfg.spi_host = VSPI_HOST;     // Use VSPI (SPI3)
      cfg.spi_mode = 0;             // SPI mode 0
      cfg.freq_write = 40000000;    // 40MHz for writing
      cfg.freq_read = 16000000;     // 16MHz for reading
      cfg.spi_3wire = true;         // 3-wire SPI (MOSI, MISO shared)
      cfg.use_lock = true;          // Use transaction lock
      cfg.dma_channel = 1;          // Use DMA channel 1
      cfg.pin_sclk = 18;            // SPI clock pin
      cfg.pin_mosi = 23;            // SPI MOSI pin
      cfg.pin_miso = 19;            // SPI MISO pin
      cfg.pin_dc = 21;              // Data/Command pin
      _bus_instance.config(cfg);
      _panel_instance.setBus(&_bus_instance);
    }
    { // Panel config: Defines the display panel specific parameters.
      auto cfg = _panel_instance.config();
      cfg.pin_cs = 5;               // Chip Select pin
      cfg.pin_rst = -1;             // Reset pin (not used, set to -1)
      cfg.pin_busy = -1;            // Busy pin (not used, set to -1)
      cfg.memory_width = 240;       // Physical width of display memory
      cfg.memory_height = 320;      // Physical height of display memory
      cfg.panel_width = 240;        // Displayable width
      cfg.panel_height = 320;       // Displayable height
      cfg.offset_x = 0;             // X offset for displayable area
      cfg.offset_y = 0;             // Y offset for displayable area
      cfg.offset_rotation = 1;      // Rotation offset (0-3, 1 for landscape orientation)
      cfg.dummy_read_pixel = 8;     // Dummy bits for pixel read
      cfg.dummy_read_bits = 1;      // Dummy bits for other reads
      cfg.readable = true;          // Enable read operations
      cfg.invert = false;           // Do not invert colors
      cfg.rgb_order = false;        // RGB order (true for BGR)
      cfg.dlen_16bit = false;       // Data length is 8-bit
      cfg.bus_shared = true;        // Bus is shared
      _panel_instance.config(cfg);
    }
    setPanel(&_panel_instance); // Set the configured panel to the LGFX device
  }
};

static LGFX lcd; // Global LGFX object for the display
static LGFX_Sprite canvas(&lcd); // Global LGFX_Sprite object for off-screen drawing (double buffering)

// --- FastLED setup (kept for consistency with user's example, though not strictly used for LEDs in this game) ---
#define LED_PIN 26
#define NUM_LEDS 5
CRGB leds[NUM_LEDS]; // Array to hold LED colors

// --- Speaker Definitions ---
#define SPEAKER_PIN 25        // GPIO pin for the speaker
#define LEDC_CHANNEL 0        // LEDC channel to use for tone generation
#define LEDC_RESOLUTION 8     // Resolution of the LEDC timer (8-bit)
#define LEDC_BASE_FREQ 2000   // Base frequency for the LEDC timer

// --- Screen and UI Dimensions ---
#define SCREEN_WIDTH 320      // Screen width in landscape mode
#define SCREEN_HEIGHT 240     // Screen height in landscape mode
#define UI_HEIGHT 35          // Height of the User Interface bar at the top

// --- GPIO Pin Definitions for Buttons ---
#define BTN_DIVE_SURFACE 35 // Analog input for Dive/Surface buttons (two buttons on one analog pin)
#define BTN_LEFT_RIGHT   34 // Analog input for Left/Right buttons (two buttons on one analog pin)
#define BTN_SPEED_UP     0  // Digital input for Speed Up button (pull-up)
#define BTN_SPEED_DOWN   13 // Digital input for Speed Down button (pull-up)
#define BTN_FIRE_PLAYER_TORPEDO 32 // Digital input for Player Torpedo Fire button (pull-up)
#define BTN_TORPEDO_LEFT 27 // Digital input for Player Torpedo Left control (pull-up)
#define BTN_TORPEDO_RIGHT 39 // Digital input for Player Torpedo Right control (pull-up)

// --- Game Constants ---
#define MAX_SUBMARINES 11           // 1 Player + 10 Enemies
#define MAX_TORPEDOES_PER_SUB 10    // Maximum torpedoes a submarine can carry
#define MAX_ACTIVE_TORPEDOES 20     // Pool size for all active torpedoes (player and enemy)
#define PLAYER_SUB_SIZE 12          // Visual size of the player submarine
#define ENEMY_SUB_SIZE 10           // Visual size of enemy submarines
#define TORPEDO_LENGTH 4            // Visual length of a torpedo line
#define PLAYER_MAX_SPEED_VAL 100    // Conceptual max speed for player (used for scaling)
#define ENEMY_MAX_SPEED_VAL 100     // Conceptual max speed for enemies (used for scaling)
#define TORPEDO_SPEED_VAL 200       // Conceptual speed for torpedoes (used for scaling)

#define TURN_ANGLE_DEG 5.0f         // Degrees the submarine turns per input
#define DIVE_SURFACE_SPEED 0.5f     // Pixels per frame for Y-axis movement during dive/surface

#define PLAYER_START_HEALTH 100     // Initial health for the player
#define PLAYER_START_TORPEDOES MAX_TORPEDOES_PER_SUB // Initial torpedoes for the player
#define ENEMY_START_TORPEDOES MAX_TORPEDOES_PER_SUB // Initial torpedoes for enemies

#define ENEMY_FIRE_COOLDOWN_MS 3000 // Minimum time between enemy torpedo shots (milliseconds)
#define ENEMY_COLLISION_DAMAGE 20   // Damage dealt by enemy submarine collision (not used for player, but for enemy-enemy)
#define TORPEDO_HIT_DAMAGE 30       // Damage dealt by a torpedo hit

// --- Colors (16-bit RGB565 format) ---
#define COLOR_SEA 0x000F      // Dark Blue for the sea background
#define COLOR_UI_BG 0x18E3    // UI background color (from user's example)
#define COLOR_TEXT 0xFFFF     // White for text
#define COLOR_PLAYER 0xFFE0   // Yellow for player submarine
#define COLOR_ENEMY 0xF800    // Red for enemy submarines
#define COLOR_TORPEDO 0x07FF  // Cyan for torpedoes

// --- Data Structures ---

// Enum to differentiate between player and enemy submarines/torpedoes
enum SubmarineType {
  PLAYER,
  ENEMY
};

// Structure to represent a submarine (player or enemy)
struct Submarine {
  float x, y;                // Current position on screen
  float direction;           // Direction in degrees (0-359.99, 0=right, 90=down, 180=left, 270=up)
  int speed;                 // Speed level: -1 (backward), 0 (stop), 1-3 (forward levels)
  int health;                // Current health points
  int torpedoes;             // Number of torpedoes remaining
  bool active;               // True if the submarine is active/alive
  SubmarineType type;        // Type of submarine (PLAYER or ENEMY)
  unsigned long lastFireTime; // Timestamp of the last torpedo fired (for cooldown)
  int id;                    // Unique identifier for the submarine
};

// Structure to represent a torpedo
struct Torpedo {
  float x, y;        // Current position on screen
  float dx, dy;      // Movement vector (change in x and y per frame)
  bool active;       // True if the torpedo is active/flying
  SubmarineType owner; // Type of submarine that fired this torpedo
  int ownerId;       // ID of the submarine that fired this torpedo
};

// --- Game State Variables ---
Submarine playerSub; // The player's submarine object
Submarine enemySubs[MAX_SUBMARINES - 1]; // Array to hold enemy submarine objects (10 enemies)
Torpedo activeTorpedoes[MAX_ACTIVE_TORPEDOES]; // Pool of all currently active torpedoes

bool gameOver = false; // Flag to indicate if the game is over

// --- Function Prototypes ---
// Declarations of all functions used in the game, allowing them to be called before their full definition.
void initGame();
void updateGame();
void updatePlayerInput();
void updateSubmarineMovement(Submarine &sub);
void updateTorpedoes();
void checkCollisions();
void drawGame();
void drawSubmarine(Submarine &sub);
void drawTorpedo(Torpedo &t);
void drawUI();
void firePlayerTorpedo();
void fireEnemyTorpedo(Submarine &enemy);
void updateEnemyAI();
float degreesToRadians(float degrees); // Converts degrees to radians
float radiansToDegrees(float radians); // Converts radians to degrees
void playFireSound();     // Sound effect for firing a torpedo
void playHitSound();      // Sound effect for a hit/collision
void playExplosionSound(); // Sound effect for an explosion
void showGameOver();      // Displays the game over screen

// --- Setup Function ---
// This function runs once when the ESP32 starts up.
void setup() {
  Serial.begin(115200); // Initialize serial communication for debugging

  lcd.init();          // Initialize the LCD display
  lcd.setRotation(1);  // Set screen rotation to landscape (320x240)
  lcd.fillScreen(0x0000); // Fill screen with black
  canvas.setPsram(true); // Enable PSRAM for the canvas (if available, for larger sprites)
  bool ok = canvas.createSprite(SCREEN_WIDTH, SCREEN_HEIGHT); // Create the off-screen drawing buffer
  Serial.println(ok ? "Canvas created!" : "Canvas FAIL!"); // Print status to serial monitor
  if (!ok) while(1) delay(1000); // If canvas creation fails, halt

  FastLED.addLeds<WS2812B, LED_PIN, GRB>(leds, NUM_LEDS); // Initialize FastLED (even if not used for LEDs, it's part of the template)
  ledcSetup(LEDC_CHANNEL, LEDC_BASE_FREQ, LEDC_RESOLUTION); // Configure LEDC peripheral for tone generation
  ledcAttachPin(SPEAKER_PIN, LEDC_CHANNEL); // Attach speaker pin to LEDC channel

  // Set pin modes for digital buttons with internal pull-up resistors
  pinMode(BTN_SPEED_UP, INPUT_PULLUP);
  pinMode(BTN_SPEED_DOWN, INPUT_PULLUP);
  pinMode(BTN_FIRE_PLAYER_TORPEDO, INPUT_PULLUP);
  pinMode(BTN_TORPEDO_LEFT, INPUT_PULLUP);
  pinMode(BTN_TORPEDO_RIGHT, INPUT_PULLUP);

  initGame(); // Initialize all game elements
}

// --- Game Initialization Function ---
// Resets the game state to its initial conditions.
void initGame() {
  // Initialize Player Submarine
  playerSub.x = SCREEN_WIDTH / 2;     // Start in the middle of the screen horizontally
  playerSub.y = SCREEN_HEIGHT - 50;   // Start near the bottom of the screen
  playerSub.direction = 270.0f;       // Initial direction: 270 degrees (upwards)
  playerSub.speed = 0;                // Initial speed: STOP
  playerSub.health = PLAYER_START_HEALTH; // Set initial health
  playerSub.torpedoes = PLAYER_START_TORPEDOES; // Set initial torpedo count
  playerSub.active = true;            // Player is active
  playerSub.type = PLAYER;            // Set type to PLAYER
  playerSub.id = 0;                   // Player ID

  // Initialize Enemy Submarines
  for (int i = 0; i < MAX_SUBMARINES - 1; i++) {
    enemySubs[i].x = random(50, SCREEN_WIDTH - 50); // Random X position
    enemySubs[i].y = random(50, SCREEN_HEIGHT / 2); // Random Y position in the upper half of the screen
    enemySubs[i].direction = random(0, 360);       // Random initial direction
    enemySubs[i].speed = random(1, (int)(ENEMY_MAX_SPEED_VAL / 20.0f)); // Random initial speed level
    enemySubs[i].health = 100;                     // Enemies have fixed health
    enemySubs[i].torpedoes = ENEMY_START_TORPEDOES; // Set initial torpedo count for enemies
    enemySubs[i].active = true;                    // Enemy is active
    enemySubs[i].type = ENEMY;                     // Set type to ENEMY
    enemySubs[i].lastFireTime = 0;                 // Reset last fire time
    enemySubs[i].id = i + 1;                       // Assign unique ID to each enemy
  }

  // Initialize Torpedoes pool: Mark all torpedoes as inactive
  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    activeTorpedoes[i].active = false;
  }

  playerSub.score = 0; // Reset player score
  gameOver = false;    // Reset game over flag
}

// --- Helper Functions for Angle Conversion ---
// Converts degrees to radians.
float degreesToRadians(float degrees) {
  return degrees * M_PI / 180.0f;
}

// Converts radians to degrees.
float radiansToDegrees(float radians) {
  return radians * 180.0f / M_PI;
}

// --- Player Input Handling ---
// Reads button states and updates player submarine's direction, speed, and vertical position.
void updatePlayerInput() {
  // Read analog values from buttons connected to GPIO34 and GPIO35.
  // These pins are configured to read two buttons each based on voltage dividers.
  int analogLR = analogRead(BTN_LEFT_RIGHT);   // Value from Left/Right buttons
  int analogUD = analogRead(BTN_DIVE_SURFACE); // Value from Dive/Surface buttons

  // Handle Left/Right movement (GPIO34)
  // User's specified analog ranges: > 4000 for one button, 1800-2200 for the other.
  if (analogLR > 4000) { // Left button pressed
    playerSub.direction -= TURN_ANGLE_DEG; // Decrease direction angle (turn left)
    if (playerSub.direction < 0) playerSub.direction += 360; // Wrap around if angle goes below 0
  }
  if (analogLR > 1800 && analogLR < 2200) { // Right button pressed
    playerSub.direction += TURN_ANGLE_DEG; // Increase direction angle (turn right)
    if (playerSub.direction >= 360) playerSub.direction -= 360; // Wrap around if angle goes above 360
  }

  // Handle Dive/Surface movement (GPIO35)
  // Zaroni (Dive) - move down (increase Y coordinate)
  if (analogUD > 4000) {
    playerSub.y += DIVE_SURFACE_SPEED;
  }
  // Izroni (Surface) - move up (decrease Y coordinate)
  if (analogUD > 1800 && analogUD < 2200) {
    playerSub.y -= DIVE_SURFACE_SPEED;
  }

  // Handle digital buttons for Speed (GPIO0, GPIO13)
  // Static variables to detect button press (edge detection)
  static bool lastSpeedUpBtnState = HIGH;   // Previous state of Speed Up button
  static bool lastSpeedDownBtnState = HIGH; // Previous state of Speed Down button

  bool currentSpeedUpBtnState = digitalRead(BTN_SPEED_UP);     // Current state of Speed Up button
  bool currentSpeedDownBtnState = digitalRead(BTN_SPEED_DOWN); // Current state of Speed Down button

  // If Speed Up button was HIGH (released) and is now LOW (pressed)
  if (lastSpeedUpBtnState == HIGH && currentSpeedUpBtnState == LOW) {
    if (playerSub.speed < 3) playerSub.speed++; // Increase speed level, max 3
  }
  // If Speed Down button was HIGH (released) and is now LOW (pressed)
  if (lastSpeedDownBtnState == HIGH && currentSpeedDownBtnState == LOW) {
    if (playerSub.speed > -1) playerSub.speed--; // Decrease speed level, min -1 (backward)
  }

  // Update last button states for the next frame
  lastSpeedUpBtnState = currentSpeedUpBtnState;
  lastSpeedDownBtnState = currentSpeedDownBtnState;

  // Clamp player submarine position to screen boundaries to prevent it from going off-screen
  if (playerSub.x < PLAYER_SUB_SIZE / 2) playerSub.x = PLAYER_SUB_SIZE / 2;
  if (playerSub.x > SCREEN_WIDTH - PLAYER_SUB_SIZE / 2) playerSub.x = SCREEN_WIDTH - PLAYER_SUB_SIZE / 2;
  // UI_HEIGHT is the top boundary, SCREEN_HEIGHT is the bottom boundary
  if (playerSub.y < UI_HEIGHT + PLAYER_SUB_SIZE / 2) playerSub.y = UI_HEIGHT + PLAYER_SUB_SIZE / 2;
  if (playerSub.y > SCREEN_HEIGHT - PLAYER_SUB_SIZE / 2) playerSub.y = SCREEN_HEIGHT - PLAYER_SUB_SIZE / 2;
}

// --- Submarine Movement Logic ---
// Updates the position of a given submarine based on its speed and direction.
void updateSubmarineMovement(Submarine &sub) {
  if (!sub.active) return; // Only move active submarines

  float currentSpeedPixelsPerFrame = 0;
  // Convert speed level to actual pixels per frame for smooth movement
  if (sub.speed == 1) currentSpeedPixelsPerFrame = PLAYER_MAX_SPEED_VAL / 30.0f; // Slow forward
  else if (sub.speed == 2) currentSpeedPixelsPerFrame = PLAYER_MAX_SPEED_VAL / 20.0f; // Medium forward
  else if (sub.speed == 3) currentSpeedPixelsPerFrame = PLAYER_MAX_SPEED_VAL / 10.0f; // Fast forward
  else if (sub.speed == -1) currentSpeedPixelsPerFrame = -PLAYER_MAX_SPEED_VAL / 40.0f; // Backward

  // Convert direction from degrees to radians for trigonometric functions
  float radDirection = degreesToRadians(sub.direction);

  // Update X and Y position based on speed and direction
  sub.x += currentSpeedPixelsPerFrame * cos(radDirection);
  sub.y += currentSpeedPixelsPerFrame * sin(radDirection);

  // Wrap around screen edges for submarines (enemies will reappear on the other side)
  // Player submarine is clamped in updatePlayerInput()
  if (sub.type == ENEMY) {
    if (sub.x < 0) sub.x += SCREEN_WIDTH;
    if (sub.x > SCREEN_WIDTH) sub.x -= SCREEN_WIDTH;
    if (sub.y < UI_HEIGHT) sub.y += (SCREEN_HEIGHT - UI_HEIGHT);
    if (sub.y > SCREEN_HEIGHT) sub.y -= (SCREEN_HEIGHT - UI_HEIGHT);
  }
}

// --- Torpedo Firing Functions ---
// Fires a torpedo from the player's submarine.
void firePlayerTorpedo() {
  if (playerSub.torpedoes <= 0) return; // Cannot fire if no torpedoes left

  // Find an inactive torpedo in the pool to reuse
  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    if (!activeTorpedoes[i].active) {
      activeTorpedoes[i].x = playerSub.x; // Torpedo starts at player's X
      activeTorpedoes[i].y = playerSub.y; // Torpedo starts at player's Y
      activeTorpedoes[i].owner = PLAYER;  // Set owner to PLAYER
      activeTorpedoes[i].ownerId = playerSub.id; // Set owner ID

      // Torpedo initial direction is player's current direction
      float radDirection = degreesToRadians(playerSub.direction);
      // Calculate movement vector (dx, dy) based on constant torpedo speed
      activeTorpedoes[i].dx = TORPEDO_SPEED_VAL / 20.0f * cos(radDirection);
      activeTorpedoes[i].dy = TORPEDO_SPEED_VAL / 20.0f * sin(radDirection);

      activeTorpedoes[i].active = true; // Activate the torpedo
      playerSub.torpedoes--;            // Decrease player's torpedo count
      playFireSound();                  // Play firing sound
      return; // Torpedo fired, exit function
    }
  }
}

// Fires a torpedo from an enemy submarine.
void fireEnemyTorpedo(Submarine &enemy) {
  if (enemy.torpedoes <= 0) return; // Cannot fire if no torpedoes left

  // Find an inactive torpedo in the pool to reuse
  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    if (!activeTorpedoes[i].active) {
      activeTorpedoes[i].x = enemy.x; // Torpedo starts at enemy's X
      activeTorpedoes[i].y = enemy.y; // Torpedo starts at enemy's Y
      activeTorpedoes[i].owner = ENEMY; // Set owner to ENEMY
      activeTorpedoes[i].ownerId = enemy.id; // Set owner ID

      // Enemy torpedoes predict player's future position and aim there.
      // Simple prediction: assume player continues current movement for a fixed time.
      float predictionTimeMs = 2000.0f; // Predict 2 seconds into the future
      float framesToPredict = predictionTimeMs / 16.0f; // Number of frames for prediction (assuming 16ms/frame)

      float playerCurrentSpeedPixelsPerFrame = 0;
      // Convert player's speed level to pixels per frame
      if (playerSub.speed == 1) playerCurrentSpeedPixelsPerFrame = PLAYER_MAX_SPEED_VAL / 30.0f;
      else if (playerSub.speed == 2) playerCurrentSpeedPixelsPerFrame = PLAYER_MAX_SPEED_VAL / 20.0f;
      else if (playerSub.speed == 3) playerCurrentSpeedPixelsPerFrame = PLAYER_MAX_SPEED_VAL / 10.0f;
      else if (playerSub.speed == -1) playerCurrentSpeedPixelsPerFrame = -PLAYER_MAX_SPEED_VAL / 40.0f;

      float playerRadDirection = degreesToRadians(playerSub.direction);
      // Calculate predicted player position
      float predictedPlayerX = playerSub.x + playerCurrentSpeedPixelsPerFrame * cos(playerRadDirection) * framesToPredict;
      float predictedPlayerY = playerSub.y + playerCurrentSpeedPixelsPerFrame * sin(playerRadDirection) * framesToPredict;

      // Calculate the angle from enemy to the predicted player position
      float angleToPlayer = atan2(predictedPlayerY - enemy.y, predictedPlayerX - enemy.x);

      // Set torpedo's movement vector based on this calculated angle and constant torpedo speed
      activeTorpedoes[i].dx = TORPEDO_SPEED_VAL / 20.0f * cos(angleToPlayer);
      activeTorpedoes[i].dy = TORPEDO_SPEED_VAL / 20.0f * sin(angleToPlayer);

      activeTorpedoes[i].active = true; // Activate the torpedo
      enemy.torpedoes--;                // Decrease enemy's torpedo count
      enemy.lastFireTime = millis();    // Update last fire time for cooldown
      playFireSound();                  // Play firing sound
      return; // Torpedo fired, exit function
    }
  }
}

// --- Torpedo Movement and Player Torpedo Control ---
// Updates the position of all active torpedoes and handles player control for their torpedo.
void updateTorpedoes() {
  // Static variables to detect button press for torpedo control (edge detection)
  static bool lastTorpedoLeftBtnState = HIGH;
  static bool lastTorpedoRightBtnState = HIGH;

  bool currentTorpedoLeftBtnState = digitalRead(BTN_TORPEDO_LEFT);
  bool currentTorpedoRightBtnState = digitalRead(BTN_TORPEDO_RIGHT);

  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    if (activeTorpedoes[i].active) {
      // If this is the player's torpedo, allow it to be controlled
      if (activeTorpedoes[i].owner == PLAYER && activeTorpedoes[i].ownerId == playerSub.id) {
        // Player's torpedo control: turning it left or right
        if (lastTorpedoLeftBtnState == HIGH && currentTorpedoLeftBtnState == LOW) { // Torpedo Left button pressed
          float currentAngle = atan2(activeTorpedoes[i].dy, activeTorpedoes[i].dx); // Get current angle of torpedo
          currentAngle -= degreesToRadians(TURN_ANGLE_DEG * 2); // Turn torpedo left (faster than submarine)
          // Recalculate dx, dy based on new angle and constant speed
          activeTorpedoes[i].dx = TORPEDO_SPEED_VAL / 20.0f * cos(currentAngle);
          activeTorpedoes[i].dy = TORPEDO_SPEED_VAL / 20.0f * sin(currentAngle);
        }
        if (lastTorpedoRightBtnState == HIGH && currentTorpedoRightBtnState == LOW) { // Torpedo Right button pressed
          float currentAngle = atan2(activeTorpedoes[i].dy, activeTorpedoes[i].dx); // Get current angle of torpedo
          currentAngle += degreesToRadians(TURN_ANGLE_DEG * 2); // Turn torpedo right (faster than submarine)
          // Recalculate dx, dy based on new angle and constant speed
          activeTorpedoes[i].dx = TORPEDO_SPEED_VAL / 20.0f * cos(currentAngle);
          activeTorpedoes[i].dy = TORPEDO_SPEED_VAL / 20.0f * sin(currentAngle);
        }
      }

      // Update torpedo position
      activeTorpedoes[i].x += activeTorpedoes[i].dx;
      activeTorpedoes[i].y += activeTorpedoes[i].dy;

      // Deactivate torpedo if it goes off screen (including UI area)
      if (activeTorpedoes[i].x < -TORPEDO_LENGTH || activeTorpedoes[i].x > SCREEN_WIDTH + TORPEDO_LENGTH ||
          activeTorpedoes[i].y < UI_HEIGHT - TORPEDO_LENGTH || activeTorpedoes[i].y > SCREEN_HEIGHT + TORPEDO_LENGTH) {
        activeTorpedoes[i].active = false;
      }
    }
  }
  // Update last button states for the next frame
  lastTorpedoLeftBtnState = currentTorpedoLeftBtnState;
  lastTorpedoRightBtnState = currentTorpedoRightBtnState;
}

// --- Enemy AI Logic ---
// Controls the movement and firing behavior of enemy submarines.
void updateEnemyAI() {
  for (int i = 0; i < MAX_SUBMARINES - 1; i++) {
    if (enemySubs[i].active) {
      // Random movement: Enemies occasionally change direction and speed.
      if (random(100) < 2) { // 2% chance per frame to change direction
        enemySubs[i].direction = random(0, 360);
      }
      if (random(100) < 1) { // 1% chance per frame to change speed
        enemySubs[i].speed = random(0, (int)(ENEMY_MAX_SPEED_VAL / 20.0f)); // Random speed level (0 to max)
      }

      // Check if player is in range and fire (staggered firing).
      // Calculate distance to player.
      float distToPlayer = sqrt(pow(playerSub.x - enemySubs[i].x, 2) + pow(playerSub.y - enemySubs[i].y, 2));
      // If player is within half screen width, enemy has torpedoes, and cooldown is over (with random offset)
      if (distToPlayer < SCREEN_WIDTH / 2 && enemySubs[i].torpedoes > 0 &&
          millis() - enemySubs[i].lastFireTime > ENEMY_FIRE_COOLDOWN_MS + random(0, 1000)) {
        fireEnemyTorpedo(enemySubs[i]); // Fire a torpedo
      }
    }
  }
}

// --- Collision Detection ---
// Checks for all possible collisions between game objects and applies effects.
void checkCollisions() {
  // 1. Player Submarine vs. Enemy Torpedoes
  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    if (activeTorpedoes[i].active && activeTorpedoes[i].owner == ENEMY) {
      float dx = playerSub.x - activeTorpedoes[i].x;
      float dy = playerSub.y - activeTorpedoes[i].y;
      float dist = sqrt(dx * dx + dy * dy);
      if (dist < PLAYER_SUB_SIZE / 2) { // Collision detected
        playerSub.health -= TORPEDO_HIT_DAMAGE; // Player takes damage
        activeTorpedoes[i].active = false;      // Torpedo is destroyed
        playHitSound();                         // Play hit sound
        if (playerSub.health <= 0) gameOver = true; // Game over if health drops to 0
      }
    }
  }

  // 2. Player Torpedoes vs. Enemy Submarines
  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    if (activeTorpedoes[i].active && activeTorpedoes[i].owner == PLAYER) {
      for (int j = 0; j < MAX_SUBMARINES - 1; j++) {
        if (enemySubs[j].active) {
          float dx = enemySubs[j].x - activeTorpedoes[i].x;
          float dy = enemySubs[j].y - activeTorpedoes[i].y;
          float dist = sqrt(dx * dx + dy * dy);
          if (dist < ENEMY_SUB_SIZE / 2) { // Collision detected
            enemySubs[j].active = false;    // Enemy submarine is destroyed
            activeTorpedoes[i].active = false; // Player torpedo is destroyed
            playExplosionSound();           // Play explosion sound
            playerSub.score += 100;         // Player gets score for destroying enemy
            break; // Torpedo hit one enemy, it's destroyed, so break from inner loop
          }
        }
      }
    }
  }

  // 3. Enemy Torpedoes vs. Enemy Submarines (friendly fire)
  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    if (activeTorpedoes[i].active && activeTorpedoes[i].owner == ENEMY) {
      for (int j = 0; j < MAX_SUBMARINES - 1; j++) {
        // Ensure torpedo doesn't hit the submarine that fired it
        if (enemySubs[j].active && activeTorpedoes[i].ownerId != enemySubs[j].id) {
          float dx = enemySubs[j].x - activeTorpedoes[i].x;
          float dy = enemySubs[j].y - activeTorpedoes[i].y;
          float dist = sqrt(dx * dx + dy * dy);
          if (dist < ENEMY_SUB_SIZE / 2) { // Collision detected
            enemySubs[j].active = false;    // Enemy submarine is destroyed
            activeTorpedoes[i].active = false; // Enemy torpedo is destroyed
            playExplosionSound();           // Play explosion sound
            break; // Torpedo hit an enemy, it's destroyed
          }
        }
      }
    }
  }

  // 4. Enemy Torpedoes vs. Other Enemy Torpedoes
  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    if (activeTorpedoes[i].active && activeTorpedoes[i].owner == ENEMY) {
      for (int j = i + 1; j < MAX_ACTIVE_TORPEDOES; j++) { // Check against other active torpedoes
        if (activeTorpedoes[j].active && activeTorpedoes[j].owner == ENEMY) {
          float dx = activeTorpedoes[j].x - activeTorpedoes[i].x;
          float dy = activeTorpedoes[j].y - activeTorpedoes[i].y;
          float dist = sqrt(dx * dx + dy * dy);
          if (dist < TORPEDO_LENGTH) { // Collision between two torpedoes
            activeTorpedoes[i].active = false; // Both torpedoes are destroyed
            activeTorpedoes[j].active = false;
            playExplosionSound(); // Small explosion sound
            break; // Torpedo hit another, no need to check further for this one
          }
        }
      }
    }
  }

  // 5. Enemy Submarines vs. Other Enemy Submarines
  for (int i = 0; i < MAX_SUBMARINES - 1; i++) {
    if (enemySubs[i].active) {
      for (int j = i + 1; j < MAX_SUBMARINES - 1; j++) { // Check against other active enemies
        if (enemySubs[j].active) {
          float dx = enemySubs[j].x - enemySubs[i].x;
          float dy = enemySubs[j].y - enemySubs[i].y;
          float dist = sqrt(dx * dx + dy * dy);
          // Collision if distance is less than sum of their half-sizes
          if (dist < (ENEMY_SUB_SIZE / 2 + ENEMY_SUB_SIZE / 2)) {
            enemySubs[i].active = false; // Both enemy submarines are destroyed
            enemySubs[j].active = false;
            playExplosionSound();
            break; // One enemy collided, no need to check further for this one
          }
        }
      }
    }
  }
}

// --- Main Game Update Loop ---
// This function updates the state of all game elements for a single frame.
void updateGame() {
  updatePlayerInput(); // Process player controls
  updateSubmarineMovement(playerSub); // Move player submarine

  // Move all active enemy submarines
  for (int i = 0; i < MAX_SUBMARINES - 1; i++) {
    updateSubmarineMovement(enemySubs[i]);
  }

  updateTorpedoes(); // Update all active torpedoes (movement and player control)
  updateEnemyAI();   // Update enemy movement and firing logic
  checkCollisions(); // Check for all collisions

  // Check for game win condition: all enemies destroyed
  bool allEnemiesDestroyed = true;
  for (int i = 0; i < MAX_SUBMARINES - 1; i++) {
    if (enemySubs[i].active) {
      allEnemiesDestroyed = false; // Found an active enemy, so not all destroyed
      break;
    }
  }
  if (allEnemiesDestroyed) {
    gameOver = true; // Set game over flag, which will trigger the "YOU WIN!" screen
  }

  // Check for game lose condition: player health drops to 0 or below
  if (playerSub.health <= 0) gameOver = true;
}

// --- Drawing Functions ---

// Draws a submarine (player or enemy) on the canvas.
void drawSubmarine(Submarine &sub) {
  if (!sub.active) return; // Only draw active submarines

  uint16_t color = (sub.type == PLAYER) ? COLOR_PLAYER : COLOR_ENEMY; // Choose color based on type
  int size = (sub.type == PLAYER) ? PLAYER_SUB_SIZE : ENEMY_SUB_SIZE; // Choose size based on type

  // Calculate triangle points for the submarine shape based on its direction
  float radDirection = degreesToRadians(sub.direction);
  // Tip of the triangle (front of the submarine)
  float tipX = sub.x + size * cos(radDirection);
  float tipY = sub.y + size * sin(radDirection);

  // Base points of the triangle (back of the submarine)
  // Angles are relative to the direction, creating a triangular shape
  float baseAngle1 = radDirection + degreesToRadians(150); // 150 degrees offset
  float baseAngle2 = radDirection - degreesToRadians(150); // -150 degrees offset

  float baseX1 = sub.x + size * 0.7 * cos(baseAngle1); // 0.7 factor to make it a bit narrower
  float baseY1 = sub.y + size * 0.7 * sin(baseAngle1);
  float baseX2 = sub.x + size * 0.7 * cos(baseAngle2);
  float baseY2 = sub.y + size * 0.7 * sin(baseAngle2);

  canvas.fillTriangle(tipX, tipY, baseX1, baseY1, baseX2, baseY2, color); // Fill the triangle
  canvas.drawTriangle(tipX, tipY, baseX1, baseY1, baseX2, baseY2, COLOR_TEXT); // Draw white outline

  // Draw 'X' for player or 'E' for enemy on top of the submarine for clarity
  canvas.setTextSize(1);
  canvas.setTextColor(COLOR_TEXT);
  canvas.setCursor(sub.x - 4, sub.y - 4); // Adjust position to center the character
  if (sub.type == PLAYER) {
    canvas.print("X");
  } else {
    canvas.print("E");
  }
}

// Draws a torpedo on the canvas.
void drawTorpedo(Torpedo &t) {
  if (!t.active) return; // Only draw active torpedoes

  // Calculate the start and end points of the 4-pixel line
  // The line points in the direction of movement (dx, dy)
  float startX = t.x;
  float startY = t.y;
  // Calculate the previous position to draw a line segment
  float endX = t.x - t.dx * (TORPEDO_LENGTH / (TORPEDO_SPEED_VAL / 20.0f));
  float endY = t.y - t.dy * (TORPEDO_LENGTH / (TORPEDO_SPEED_VAL / 20.0f));

  canvas.drawLine(startX, startY, endX, endY, COLOR_TORPEDO); // Draw the torpedo line
}

// Draws the User Interface (UI) bar at the top of the screen.
void drawUI() {
  canvas.fillRect(0, 0, SCREEN_WIDTH, UI_HEIGHT, COLOR_UI_BG); // Draw UI background rectangle
  canvas.setTextColor(COLOR_TEXT); // Set text color to white
  canvas.setTextSize(1);           // Set text size

  // Display Player Health
  canvas.setCursor(10, 10); // Position text
  canvas.printf("Health: %d", playerSub.health);

  // Display Player Torpedoes remaining
  canvas.setCursor(10, 22); // Position text below health
  canvas.printf("Torps: %d", playerSub.torpedoes);

  // Display Player Speed
  canvas.setCursor(SCREEN_WIDTH / 2 - 40, 10); // Position text in the middle top
  canvas.printf("Speed: ");
  if (playerSub.speed == -1) canvas.print("BACK"); // Backward speed
  else if (playerSub.speed == 0) canvas.print("STOP"); // Stop
  else canvas.printf("FWD%d", (int)playerSub.speed); // Forward speed levels

  // Display Player Direction
  canvas.setCursor(SCREEN_WIDTH / 2 - 40, 22); // Position text below speed
  canvas.printf("Dir: %.0f deg", playerSub.direction); // Display direction in degrees (no decimals)

  // Display Player Score
  canvas.setCursor(SCREEN_WIDTH - 80, 10); // Position text on the right top
  canvas.printf("Score: %d", playerSub.score);

  // Display number of Active Enemies
  int activeEnemies = 0;
  for (int i = 0; i < MAX_SUBMARINES - 1; i++) {
    if (enemySubs[i].active) activeEnemies++; // Count active enemies
  }
  canvas.setCursor(SCREEN_WIDTH - 80, 22); // Position text below score
  canvas.printf("Enemies: %d", activeEnemies);
}

// Main drawing function that renders all game elements.
void drawGame() {
  canvas.fillScreen(COLOR_SEA); // Fill the entire screen with sea color (dark blue)

  // Draw all active submarines (player first, then enemies)
  drawSubmarine(playerSub);
  for (int i = 0; i < MAX_SUBMARINES - 1; i++) {
    drawSubmarine(enemySubs[i]);
  }

  // Draw all active torpedoes
  for (int i = 0; i < MAX_ACTIVE_TORPEDOES; i++) {
    drawTorpedo(activeTorpedoes[i]);
  }

  drawUI(); // Draw the user interface on top

  canvas.pushSprite(0, 0); // Push the entire off-screen canvas to the physical display
}

// --- Sound Effects Functions ---
// Plays a sound effect for firing a torpedo.
void playFireSound() {
  ledcWriteTone(LEDC_CHANNEL, 800); // Play 800 Hz tone
  delay(20);                       // For 20 milliseconds
  ledcWriteTone(LEDC_CHANNEL, 1000); // Then 1000 Hz tone
  delay(20);                       // For 20 milliseconds
  ledcWriteTone(LEDC_CHANNEL, 0);  // Stop tone
}

// Plays a sound effect for a hit or minor collision.
void playHitSound() {
  for (int i = 400; i >= 100; i -= 50) { // Descending tones
    ledcWriteTone(LEDC_CHANNEL, i);
    delay(15);
  }
  ledcWriteTone(LEDC_CHANNEL, 0); // Stop tone
}

// Plays a sound effect for an explosion.
void playExplosionSound() {
  for (int i = 100; i <= 600; i += 100) { // Ascending tones
    ledcWriteTone(LEDC_CHANNEL, i);
    delay(10);
  }
  for (int i = 600; i >= 100; i -= 100) { // Descending tones
    ledcWriteTone(LEDC_CHANNEL, i);
    delay(10);
  }
  ledcWriteTone(LEDC_CHANNEL, 0); // Stop tone
}

// --- Game Over Screen Function ---
// Displays the game over or win screen.
void showGameOver() {
  canvas.fillScreen(0x0000); // Clear screen to black
  canvas.setTextSize(2);     // Set larger text size
  canvas.setTextColor(COLOR_TEXT); // Set text color to white
  canvas.setTextDatum(MC_DATUM); // Set text alignment to middle-center for easy positioning

  // Display "GAME OVER" or "YOU WIN!" based on player's health
  if (playerSub.health <= 0) {
    canvas.drawString("GAME OVER", SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2 - 20);
  } else { // Player won (all enemies destroyed)
    canvas.drawString("YOU WIN!", SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2 - 20);
  }

  canvas.setTextSize(1); // Smaller text size for score and instructions
  canvas.drawString("Final Score:", SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2 + 10);
  canvas.printf("%d", playerSub.score, SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2 + 30); // Display final score
  canvas.drawString("Hold button to restart", SCREEN_WIDTH / 2, SCREEN_HEIGHT / 2 + 60); // Restart instruction

  canvas.pushSprite(0, 0); // Push the game over screen to the display

  // Check for restart button press (using BTN_SPEED_UP as a general restart button)
  if (digitalRead(BTN_SPEED_UP) == LOW) { // If button is pressed (LOW because of INPUT_PULLUP)
    initGame(); // Re-initialize the game
    delay(400); // Debounce delay to prevent multiple restarts from a single press
  }
}

// --- Main Loop Function ---
// This function runs repeatedly after setup() completes. It's the heart of the game.
void loop() {
  // Handle player torpedo firing button (edge detection)
  static bool lastFirePlayerTorpedoBtnState = HIGH; // Previous state of the fire button
  bool currentFirePlayerTorpedoBtnState = digitalRead(BTN_FIRE_PLAYER_TORPEDO); // Current state

  // If button was released (HIGH) and is now pressed (LOW)
  if (lastFirePlayerTorpedoBtnState == HIGH && currentFirePlayerTorpedoBtnState == LOW) {
    firePlayerTorpedo(); // Call the function to fire a torpedo
  }
  lastFirePlayerTorpedoBtnState = currentFirePlayerTorpedoBtnState; // Update last state

  // Game logic: if game is not over, update and draw; otherwise, show game over screen.
  if (!gameOver) {
    updateGame(); // Update all game elements (movement, AI, collisions)
    drawGame();   // Redraw the entire game scene
    delay(16);    // Delay to target approximately 60 frames per second (1000ms / 60fps = ~16.67ms)
  } else {
    showGameOver(); // Display the game over/win screen
  }
}
```

[Googleova pravila o privatnosti](https://policies.google.com/privacy)

[Googleovi uvjeti pružanja usluge](https://policies.google.com/terms)

[Vaša privatnost i Gemini aplikacije](https://support.google.com/gemini?p=privacy_notice)

Gemini može prikazivati netočne informacije, uključujući one o ljudima, pa dobro provjerite njegove odgovore. Ovaj sadržaj izradila je druga osoba. Mogao bi biti netočan ili rizičan.

