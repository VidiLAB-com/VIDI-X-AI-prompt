# VIDI-X-AI-prompt
Kompletan tekst AI promptanje za izradu igara na VIDI X mikroračunalu biti će dostupan na [Vidilab.com](https://vidilab.com) portalu.

# Gemini 2.5 Pro

Iz prvog pokušaja doista impresivan kod koji je dovoljno težak da ga pokušate zaigrati nekoliko puta ne biste li preživjeli barem malo dulje u novoj rundi. Zanimljivo, Gemini se čak i potpisao u kod te mu je upisao broj verzije.

Ipak, bilo je potrebno prilagoditi lijevu i desnu stranu za upravljanje torpedom, kao i brzinu same igre jer je barem deset puta prebrza.

Prompt i rezultat prompta pogledajte na linku: [Prompt rezultat na Gemini](https://g.co/gemini/share/dc0a1022702c)

Na dobivenom kodu prilagodili smo varijable:

```cpp
#define PLAYER_MAX_SPEED  20.0f
#define ENEMY_MAX_SPEED   10.0f
#define TORPEDO_SPEED     20.0f
```

te povećali pauzu u glavnoj `void loop()` petlji za deset puta kako bi igra bila sporija:

```cpp
delay(160); // Ograničava na otprilike 60 FPS
```

![Gemini25Pro.png](Gemini25Pro.png)
*Potpis: Potpisivanje Geminija kao autora koda zaista je jedinstveno rješenje*

![VIDI-X-Podmornice-1.png](VIDI-X-Podmornice-1.png)
*Potpis: Svi podaci (brzina, smjer, dubina, broj torpeda i neprijatelja) našli su se na ekranu. Impresivno!*

---

# Gemini 2.5 Flash

Iz prvog pokušaja dobili smo grešku kako nam fali atribut `playerSub.score`. No nakon dodavanja linija koda u kojima ga definiramo, kod je prošao. Dakle u `struct Submarine` dodajemo:

```cpp
int score;
```

Zatim u `void initGame()` dodajemo:

```cpp
playerSub.score = 0; // Initial score: 0
```

I naposljetku u `void setup()` postavljamo rotaciju ekrana:

```cpp
lcd.setRotation(0);
```

Prompt: [Prompt rezultat na Gemini Flash](https://g.co/gemini/share/abf32ada1cde)

![Gemini25Flash.png](Gemini25Flash.png)
*Potpis: Kod je prebogat komentarima koji će nam olakšati snalaženje u budućim iteracijama s kodom. Pohvalno!*

![VIDI-X-Podmornice-2.png](VIDI-X-Podmornice-2.png)
*Potpis: Zanimljivo je kako smo pored svih drugih podataka dobili i podatak o zdravlju naše podmornice. Grafika je inovativnija. Zanimljivo!*

---

# Claude Sonet 4 (putem websim.com)

Iz dva pokušaja nismo uspjeli dobiti kompletan kod korištenjem [Clude AI](https://claude.ai/) servisa. Prvi put dobili smo 650 linija koda, a drugi put 660 linija koda koji nije u potpunosti bio kompletan. Izgleda da je to njegov maksimum u besplatnoj verziji. No pokušali smo trikom nasamariti Sonet 4 na način da ga pokrenemo putem [WebSim](https://websim.com) servisa. No kako je taj servis namijenjen za izradu web stranica, na sam početak našeg prompta dodali smo: **„Kompletan programski kod ove igre neka bude napisan na HTML web stranici te mu dodaj copy gumb. Upute za izradu koda koji će biti prikazan na HTML stranici glase ovako:“** i zatim nastavili sa potpuno istim promptom kao i kod ostalih.
Kod je imao jednu grešku. Kod include mu je nedostajalo ime biblioteke te smo, kako bi kod proradio ispravno, u drugoj liniji koda trebali napisati:

```cpp
#include <LovyanGFX.hpp>
```

Kako izgleda HTML web stranica koju smo dobili, pogledajte na linku: 
[Websim Claude prompt rezultat](https://websim.com/p/9x6_6208fqie04p1xvqk)

![Igra-2.psd](Igra-2.psd)
*Potpis: Ovaj kod ima potencijala za nastavak razvoja igre.*

---

# Grok 3 (uz Thinking)

Iz drugog pokušaja, kada smo upalili **Thinking** opciju, dobili smo kod koji smo mogli zaigrati uz minimalne preinake. Bez Thinking opcije nije imao dovoljno tokena za prikaz kompletnog koda koji je generirao, niti je bio suvisao na naredbu „nastavi“ koja obično potakne nastavak ispisa s novim nizom tokena.
Na dobivenom kodu igre bilo je potrebno prilagoditi varijablu brzine. No to je nekako naša greška kada smo u promptu definirali brzinu 100, a nismo taj podatak vezali uz ništa „čvrsto“ kako bi AI mogao znati što to točno znači. Dakle, promjena linija:

```cpp
#define TORPEDO_SPEED 20
#define SUBMARINE_MAX_SPEED 10
```

učinila je kod ispravnim za pokretanje. No, uz manju grešku koja se manifestira kao nestanak neprijateljskih podmornica. Analizom otkrivamo da uslijed detekcije kolizije torpeda i podmornice biva procijenjeno kako je ispaljeni torpedo u radijusu podmornice, pa time podmornica koja je ispalila torpedo biva uništena. Srećom, to ne utječe na vašu podmornicu te vi možete pucati, ukoliko je koja neprijateljska podmornica još ostala, a da nije ispalila torpedo. Nema jasno definirane grafike koja bi pokazivala smjer u kojem je okrenuta vaša podmornica.

Prompt i kod pogledajte na linku: [Prompt rezultat na X (Grok)](https://x.com/i/grok/share/I9Y31DP7bmQYZSI4DNTf3ctYw)

![VIDI-X-Podmornice-3.png](VIDI-X-Podmornice-3.png)
*Potpis: Ubrzo svi neprijatelji nestanu. Nije nas se dojmio kod!*

---

# ChatGPT Codex

**Codex** je zapravo Agent koji je razvio OpenAI za kojega je potrebno imati pretplatu na ChatGPT. Treniran je za razumijevanje i generiranje koda, i to kroz integraciju s GitHubom. Razumije širok raspon programskih jezika. Za razliku od nekih drugih LLM-ova, Codex ne nudi bogato razumijevanje lingvistike u širem kontekstu, već se fokusira na razvojne zadatke, dopunjavanje koda i sugestije unutar programerskih alata.
Da biste ga koristili, Codex je potrebno povezati sa GitHub računom te mu dati pristup repozitorijima, ili kompletnom računu, po kojima smije raditi izmjene. Time dobiva mogućnost učenja na primjerima koji se tamo nalaze. U našem testu ograničili smo ga samo na kod koji smo davali i drugim LLM-ovima. Dobili smo funkcionalnu igru kratkoga koda, vrlo jednostavne grafike, no nedostajale su informacije o broju neprijatelja i torpeda. Piše samo brzina i smjer kretanja. Igrivost je zanimljiva jer podmornice ispaljuju torpeda dovoljno često i precizno da igra bude teška, ali igriva.

Prompt i kod pogledajte na linku repozitorija: [VIDI-X Submarine Codex](https://github.com/VidiLAB-com/VIDI-X-submarine)

![Igra-1.psd](Igra-1.psd)
*Potpis: Zanimljivo je natjerati podmornice da unište jedna drugu gađajući vas.*

---

# ChatGPT i ChatGPT 4.5

**ChatGPT 4.5** predstavlja prijelaznu fazu između GPT-4 Turbo i nadolazeće GPT-5 arhitekture. AI zajednica ga prepoznaje kao iteraciju unutar GPT-4 linije s osjetno većom brzinom, poboljšanom stabilnošću memorije i boljom sposobnošću upravljanja dugim kontekstima. Govori se kako ChatGPT 4.5 postiže ravnotežu između snage GPT-4o i responzivnosti, uz prepoznatljiv OpenAI-jev stil dijaloga i generiranja sadržaja koji je pogodan i za kreativne zadatke i za tehničku asistenciju.
U praksi, nismo dobili kompletan kod naše igre, nego nam je LLM javio kako je napravio osnovno kretanje podmornice, a za korištenje torpeda i kolizija da mu kažemo što želimo dalje, iako je sve potrebne informacije imao u prvom promptu. Pokretanjem tog malog koda uviđamo kako niti podmornica nije nacrtana na ekranu.
Slično se dogodilo i sa besplatnom inačicom ChatGPT-a koja nema oznaku verzije pri korištenju. Kod koji smo dobili javljao je razne greške. Zaključujemo kako ChatGPT 4.5 nije baš pogodan za tehničku asistenciju programiranja, iako OpenAI svakih nekoliko tjedana radi fino podešavanje modela prema naputcima korisnika (oni palci gore i palci dolje kao oznaka dobrog i lošeg rezultata) te možemo očekivati poboljšanja u skorijoj budućnosti, ili dok ga ne zamijeni GPT-5.

![ChatGPT 45.png](ChatGPT%2045.png)
*Potpis: Imamo dojam kao da su starije verzije ChatGPT-a bolje pripremljene za kodiranje.*

---

# ChatGPT 4.1

Korištenjem **ChatGPT 4.1** verzije dobili smo kod uz samo jedan dodatni klik na „Nastavi“ gumb pri kreiranju prompta, nakon čega dobivamo kod u kojem nismo mogli preživjeti duže od par sekundi radi količine torpeda koji su se kretali prema nama iz više smjerova. Zanimljivo je bilo vidjeti podmornice u obliku kružnice koje su ujedno označavale i područje koje ne smije doći u kontakt sa torpedom ili neprijateljskim podmornicama kako bismo ostali na životu.

Verziju koju smo dobili od verzije 4.1 pogledajte na linku: 
 [ChatGPT 4.1 kod](https://chatgpt.com/share/686534ab-521c-8006-b978-653e9adb2d07)

![Igra-3.psd](Igra-3.psd)
*Potpis: Podmornice kao baloni nisu nas baš oduševile.*

---

# ChatGPT 4o

**ChatGPT 4o**, također od OpenAI-ja, predstavlja trenutno najnapredniji javno dostupan model iz serije GPT-4, optimiziran za brzinu i multimodalnu komunikaciju. Verzija „4o“ (O označava „omni“) omogućuje razumijevanje i generiranje teksta, slike, ali i programskog koda, što ga čini posebno pogodnim za interaktivne i edukativne scenarije u kojima korisnik ne komunicira samo tekstom, nego i slikom. Tu je i mogućnost interakcije govorom što mu dodaje mogućnost generiranja zvuka. Značajka dubinskog istraživanja može uvelike oplemeniti generirane rezultate.
Korištenjem ChatGPT 4o modela za našu igru bilo je potrebno „nastavi“ napisati pet puta kako bismo dobili kompletan kod. Od tih pet puta imamo zapravo tri koda koja je potrebno iskombinirati kako bismo dobili kod bez greške koji se može pokrenuti i zaigrati. Dakle, treću, petu i šestu iteraciju koda smo zalijepili u jednu cjelinu. Ovdje je trebalo ipak malo pogledati kod, te se lako moglo prepoznati što je dobro, a što viška, jer prva, druga i treća iteracija koda izgledaju slično te kao nadogradnja, pa je logično da ćemo uzeti samo treću. Isto je bilo i sa četvrtom i petom kod kojih je pisao komentar:

```cpp
// --- Prethodni kod izostavljen za preglednost ---
```

Svakome tko razumije kod, jasno je kako je peta iteracija nadogradnja na četvrtu, uz izostavljen kod iz prethodnih inačica.

Spajanjem smo dobili grafički lijepu igru čiji kod je dostupan na linku: [ChatGPT 4o kod](https://chatgpt.com/share/68653a3d-c52c-8006-9315-d15102d114e1)

Nedostatak je što je nemoguće preživjeti duže od par sekundi jer po vama pucaju svi istovremeno. Također nedostaju informacije o preostalom broju torpeda i broju neprijatelja. Nije radila niti detekcija kolizije između podmornica.

![Igra-2.psd](Igra-2.psd)
*Potpis: Dobar je za nastavak razvoja igre. Vide se sličnosti sa Claude Sonet 4 kodom.*

---

# Perplexity.ai s Claude Sonet 4 Razmišljanje modelom

**Perplexity** nam nudi na odabir osam LLM modela u **PRO** inačici, te četiri načina interakcije s njima. Pretraži, Istraživanje i Laboratorij. Doplatom za MAX inačicu moguće je dobiti pristup dodatnim modelima.

Mi smo koristili **Claude Sonet 4 Razmišljanje** uz opciju **Pretraži** jer odabir samo opcije Laboratorij - koja nam ne daje mogućnost izbora LLM modela - nije proizvela kod koji se mogao kompajlirati. Neće to uvijek biti tako, pa slobodno probajte i Laboratorij u svojim projektima. Naime, te malene varijacije ovise o nekim parametrima, poput temperature i samplinga, koji su vam dostupni tek kada imate API pristup LLM modelima te se uhvatite u koštac sa programskim pristupom umjetnoj inteligenciji. Ti brojevi ili su predefinirani ili su nasumični u trenutku kada koristite web sučelje za interakciju s modelima. Tek rijetki omogućavaju pristup tim parametrima putem web sučelja, a najčešće to dozvoljavaju AI alati za kreiranje slika, videa ili zvuka.
Kod koji je proizveo Claude Sonet 4 Razmišljanje uz opciju Pretraži pogledajte na linku:

[Perplexity kod](https://www.perplexity.ai/search/razmisli-o-detaljnim-uputama-z-jufAjMoSSlGOQTA4kE2heQ)

Boljka mu je da podmornice nestaju kada ispale torpedo jer algoritam misli da se dogodio pogodak, radi načina na koji radi. Računa udaljenost torpeda od podmornice. Taj dio koda nalazi se iza komentara

```cpp
// Neprijateljski torpedo vs neprijatelj
```

te u tom dijelu možete potražiti rješavanje tog problema. Kad riješite taj, imat ćete neke druge dijelove koda za srediti. No svidjelo nam se kako su različiti događaji popraćeni različitim zvučnim efektima.

![Igra-4.jpg](Igra-4.jpg)
*Potpis: Torpedo ahead… all hands, brace for impact!*
