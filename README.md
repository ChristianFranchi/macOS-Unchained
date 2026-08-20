# ⛓️‍💥 macOS Unchained

> Vincoli di prestazioni nascosti di macOS: come **misurarli** e come **toglierli**.
> Ogni caso qui dentro è misurato, riproducibile, e riporta anche cosa **non** è stato capito.

---

## Caso #1 — il cursore colorato di Accessibilità può bruciare oltre un core

Se hai attivato **Impostazioni ▸ Accessibilità ▸ Monitor ▸ Puntatore ▸ colore contorno/riempimento**, alcune app possono passare da pochi punti percentuali a **oltre il 50% di CPU ciascuna**, in continuo, a mouse fermo.

**Misurato** su MacBook Pro M5, macOS 27.0:

| | contorno colorato | colori default |
|---|---|---|
| Hammerspoon | **58,7%** | **2,4%** |
| Bartender 6 | **67,2%** | **10,7%** |

Reversibile nei due sensi, tre ripetizioni. La **dimensione** del puntatore non c'entra: resta 1,5× in tutti gli stati.

### Il meccanismo

`-[NSCursor set]` ha due strade:

| | cursore standard | cursore colorato |
|---|---|---|
| cosa viene trasmesso | **nome + seed** | **8 bitmap via XPC** |
| costo | ~0 (Apple spedisce PNG già rasterizzati) | **~3 ms**: 2 PDF × 4 scale + ombra sfocata |
| cache | sì | **nessuna** |

AppKit rivaluta le tracking area a **ogni ciclo di display**: un'app che ricrea la sua `NSTrackingArea` chiama `set` **60-120 volte al secondo**. 3 ms × 120 = un core.

⇒ Il costo è un **prodotto**: `costo-per-chiamata (Apple) × frequenza (app)`.

### Fonte primaria

[Mozilla bug 1736049](https://bugzilla.mozilla.org/show_bug.cgi?id=1736049) — *«We call -[NSCursor set] on every mouse move»*, VERIFIED FIXED, ottobre 2021:

> *«On macOS Monterey, with cursor accessibility coloring enabled, this function call is expensive»*

Firefox risolse **smettendo di chiamarlo**. [LuLu](https://github.com/objective-see/LuLu/issues/884) idem (80% → 14%). [Stats](https://github.com/exelban/stats/pull/3468) aveva la patch pronta in −19 righe: **chiusa senza commento**.

Apple corresse la **perdita di memoria** in macOS 12.1 beta 4. Il **costo di CPU è ancora lì**, cinque anni dopo.

📌 La colorazione del puntatore è arrivata con **macOS 12 Monterey (2021)**. La *dimensione* esiste dal 2005 e usa il percorso rasterizzato: quella non costa.

### 🔴 Non è un problema qualunque

È la funzione di **accessibilità**. Chi colora il cursore lo fa perché ci vede poco — e paga un core senza saperlo, spesso su hardware più modesto.

---

## ⚠️ Il tranello che rende il Mac inavviabile

Riscrivere il colore da riga di comando **senza `-float` su ogni componente** salva i valori come **stringhe**. Ogni processo AppKit che legge il colore va in `SIGABRT`: il Mac diventa inutilizzabile e **il guasto sopravvive al riavvio e alla modalità sicura**.

```bash
# ⛔ MAI — i numeri diventano stringhe
defaults write com.apple.universalaccess cursorOutline -dict red 0 green 0.98 blue 0.19 alpha 1

# ✅ tipo esplicito su OGNI componente
defaults write com.apple.universalaccess cursorOutline -dict \
  red -float 0 green -float 0.98 blue -float 0.19 alpha -float 1

# ✅ e verificare SEMPRE i tipi: i numeri NON devono essere fra virgolette
plutil -p ~/Library/Preferences/com.apple.universalaccess.plist
```

`plutil -lint` **non** intercetta l'errore: il plist resta formalmente valido.

**Riparazione** (da un altro Mac, volume montato in Condivisione disco):

```bash
P="/Volumes/<volume>/Users/<utente>/Library/Preferences/com.apple.universalaccess.plist"
plutil -remove cursorFill "$P"
plutil -remove cursorOutline "$P"
plutil -replace cursorIsCustomized -bool false "$P"
```

---

## 🔬 Misurare (senza installare nulla)

```bash
./bin/cursor-cost
```

Confronta il costo con e senza colorazione e stampa il verdetto.

⚠️ **Non usare `ps` per questi confronti**: su processi attivi da giorni `%CPU` è una media smorzata e può muoversi **al contrario**. Misurato: `ps` 87% contro `top` 117% nello stesso istante. Usa `top -l 3` o `powermetrics`.

---

## 🚧 Aperto — non lo sappiamo ancora

Il costo **non è deterministico**. Stesse chiavi, stessa macchina:

| momento | stato | Hammerspoon | Bartender |
|---|---|---|---|
| dopo giorni di uptime | verde attivo | 58,7% | 67,2% |
| poco dopo un riavvio, colore riapplicato dal pannello | verde attivo | **1,7%** | **9,5%** |

Quindi «colore = costo» è **incompleto**. Ipotesi da verificare:

- il costo si accumula con l'**uptime** (coerente con altre misure: `WindowServer` 1300 ms/s dopo 6 giorni contro 290 da fresco; `MenuBarAgent` 2,5 GB contro 51 MB)
- applicare dal **pannello** segue un percorso di registrazione diverso da `defaults write` + reload del demone
- il degrado colpisce anche i **client** (`FineTune`: 258 timer sotto i 2 ms al secondo prima di un riavvio, **0** dopo)

**Contributi benvenuti**: se riproduci o falsifichi, apri una issue con `top -l 3` prima/dopo e l'uptime.

---

## Perché questo repo

Il caso #1 è emerso da una caccia durata un giorno in cui **otto ipotesi si sono rivelate sbagliate**: ogni app spenta dava −5/−10% e nessuna risolveva. Quel pattern — molti interventi mirati con effetto piccolo e nessuna soluzione — è **esso stesso il sintomo**: significa che non c'è un colpevole, c'è un moltiplicatore condiviso.

Qui finiscono i vincoli di quel tipo: quelli che non si vedono, che colpiscono tutti insieme, e che nessuna guida di «ottimizzazione» nomina.

## Licenza

MIT.
