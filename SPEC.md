# SPEC.md — Workout Monitor v2

## 1. Concept & Vision

**Workout Monitor** è un'app web minimalista per tracciare gli allenamenti in palestra e ricevere suggerimenti intelligenti basati sugli obiettivi dell'utente.

L'esperienza è pulita, motivante, senza distrazioni. Come un diario cartaceo della palestra, ma più smart.

**v2 Focus:** Bug fixes + Usability improvements (edit workout, date pre-fill, exercise name normalization) + remaining feature tickets (progress photos, history modal, rest timer UI, streak timezone fix, API key pre-fill from profile).

---

## 2. Tech Stack

| Layer | Choice |
|-------|--------|
| Frontend | Single HTML file — vanilla JS + Tailwind CSS (CDN) |
| Storage | localStorage (JSON) — no backend required |
| API suggerimenti | MiniMax API |
| Deploy | GitHub Pages |
| CDN assets | cdnjs, unpkg, Tailwind CDN |

**No changes from v1** — the stack works well.

---

## 3. Layout & Struttura

### View Structure
1. **Home / Dashboard** — riepilogo settimanale, ultime sessioni, obiettivo attivo
2. **Log Workout** — form per registrare/modificare un allenamento con progress photos
3. **Storico** — lista cronologica delle sessioni, filtri per muscolo, tap per history modal full-screen
4. **Suggerimenti** — consigli basati su obiettivi, generati da AI

### Navigazione
- Bottom tab bar (mobile-first): Home | Log | Storico | AI
- Nessuna sidebar — tutto accessibile in 1 tap

### Responsive
- Mobile-first, max-width 480px centered
- Desktop: stessa UX, centrato con bordo laterale

---

## 4. Features

### 4.1 Log Allenamento (v2)
- **Date picker** con data pre-popolata a oggi
- Seleziona muscolo target (pettorali, schiena, gambe, spalle, braccia, core)
- Aggiungi esercizi: nome, serie, reps, peso (kg)
- **Nome esercizio normalizzato** (lowercase, trim, capitalize first letter)
- Note libere opzionali
- Durata opzionale
- **Progress photos** — acquisizione da camera (capture=environment), salvataggio base64 in localStorage, grid 3 colonne, possibilità di rimuovere. Risoluzione max 800px, JPEG 80%. Limite 4.5MB totale per foto.
- Data modificabile manualmente
- Salvataggio in localStorage

### 4.2 Edit Workout (v2)
- Dalla vista storico, tap su workout → apre in modalità edit (log view)
- Tutti i campi modificabili (data, muscolo, esercizi, note, foto)
- Possibilità di eliminare workout
- Salva modifiche o annulla

### 4.3 History Modal / Full-screen (v2)
- Tap su workout nello storico apre modal full-screen
- Mostra: muscolo badge, data, durata, volume totale, serie totali, note, esercizi dettagliati
- Pulsante "Modifica" per tornare al form in modalità edit
- Pulsante "Chiudi" per tornare allo storico

### 4.4 Rest Timer UI Countdown (v2)
- Badge fisso sopra la tab bar (non intrusivo)
- Countdown visibile (MM:SS) con font grande lime
- Configurabile da impostazioni (30-300 secondi, step 10s)
- Skip button per chiudere immediatamente
- Auto-dismiss alla fine con messaggio "✓ Recupero!" per 3 secondi

### 4.5 Rest Timer Configurazione (v2)
- Campo numerico nelle impostazioni profilo (`recoveryTimerSeconds`)
- Valore default: 90 secondi
- Salvato nel profilo utente

### 4.6 Streak Timezone-Aware (v2)
- Calcolo streak basato su date strings YYYY-MM-DD (no Date objects che risentono di timezone)
- Funzione `localDateISO()` per generare stringhe senza parsing timezone
- Rileva correttamente il giorno anche in timezone CEST/+0200
- Permette workout di ieri come inizio streak se oggi non ancora allenato

### 4.7 API Key Precompilata dal Profilo (v2)
- API key MiniMax salvata nel profilo utente (`miniMaxApiKey` field)
- Precompilata automaticamente nella sezione AI e nelle impostazioni
- Mai inviata a server diversi da MiniMax
- Salvata in localStorage sotto chiave `workout_monitor:apikey`

### 4.8 Storico
- Lista cronologica (più recente prima)
- Filtro per muscolo
- Tap su sessione → apre history modal full-screen
- Dettaglio espandibile inline per ogni sessione
- Pulsante edit per ogni workout card
- Totale volume (serie × reps × peso) per sessione

### 4.9 Dashboard
- Sessioni questa settimana (count)
- Volume totale settimanale
- Ultimo workout (data + muscolo)
- Streak giorni consecutivi con workout (corretto per timezone)

### 4.10 Suggerimenti AI
- Input: obiettivo (forza / ipertrofia / definizione) + muscolo target
- API key precompilata dal profilo
- Output: 4-5 esercizi suggeriti con reps/sets/rest/notes
- Chiamata MiniMax API con prompt strutturato
- Cache locale dei suggerimenti (stesso obiettivo = stesso output per 24h)
- Fallback locale se API non disponibile

---

## 5. Data Model

### Workout Session (v2)
```json
{
  "id": "uuid",
  "date": "2026-04-15",
  "muscle": "pettorali",
  "exercises": [
    {
      "id": "uuid",
      "name": "Panca piana",
      "sets": 4,
      "reps": 8,
      "weight": 60
    }
  ],
  "notes": "Optional notes",
  "goal": "ipertrofia",
  "duration": 60,
  "photoIds": ["uuid1", "uuid2"],
  "totalVolume": 1920,
  "totalSets": 12,
  "totalReps": 36,
  "createdAt": "2026-04-15T10:30:00Z",
  "updatedAt": "2026-04-15T10:30:00Z"
}
```

**Note:** Exercise names are **normalized to lowercase** before storage, then capitalized on display.

### User Profile (localStorage)
```json
{
  "id": "user_profile",
  "name": "Nico",
  "goal": "ipertrofia",
  "level": "intermedio",
  "preferredUnits": "kg",
  "theme": "dark",
  "trackDuration": true,
  "trackRPE": false,
  "recoveryTimerSeconds": 90,
  "miniMaxApiKey": "sk-cp-...",
  "cachedSuggestions": []
}
```

### Photos (localStorage, separate key)
```json
{
  "photo_uuid": {
    "data": "data:image/jpeg;base64,...",
    "createdAt": "2026-04-15T10:30:00Z"
  }
}
```

---

## 6. Componenti UI

### BottomTabBar
- 4 tab: Home, Log, Storico, AI
- Icone emoji: 🏠 📝 📊 🤖
- Active state: highlighted background con padding rounded

### WorkoutCard
- Data + muscolo in header
- Lista esercizi compatta
- **Pulsante edit (matita)** per aprire modal modifica
- Tap per espandere dettaglio inline
- Volume totale in footer

### HistoryModal (v2)
- Full-screen overlay (z-index 500)
- Header sticky con pulsanti Chiudi e Modifica
- Contenuto scrollabile
- Badge muscolo colorato, data grande, stats grid
- Lista esercizi con dettaglio peso/serie/reps

### RestTimerBadge (v2)
- Fixed badge sopra tab bar (bottom: 76px)
- Background #1a1a1a, border 1px solid lime
- Countdown grande (18px) color lime
- Done state: border color green, text "✓ Recupero!"

### ExerciseRow
- Inline editing dei campi
- + button per aggiungere esercizio
- Exercise name auto-normalized on blur

### AIModal
- Slide-up modal
- Select per muscolo e obiettivo
- API key field precompilato dal profilo
- Loading spinner durante generazione
- Risultato come lista scheda con suggerimenti

### PhotoGrid (v2)
- Grid 3 colonne, gap 8px
- Thumbnail quadrato, object-fit cover
- X button per eliminare in overlay top-right
- Aggiunta via input file (capture=environment)
- Risoluzione max 800px, JPEG 80%
- Storage limit: ~4.5MB total per le foto

---

## 7. Bug Fixes Detail (v2)

### 7.1 todayISO() Typo Bug (FIXED)
**Problema:** Codice usava `todayISO()` invece di funzione corretta.
**Soluzione:** Rimuovere helper buggy, usare `localDateISO()` che genera stringa YYYY-MM-DD senza parsing timezone.

### 7.2 Date Not Pre-filled (FIXED)
**Problema:** Il form workout non aveva campo data pre-popolato.
**Soluzione:** Aggiungere `<input type="date">` nel form con `value` pre-popolato via `localDateISO()`.

### 7.3 No Edit Workout (FIXED)
**Problema:** Non era possibile modificare un workout esistente.
**Soluzione:** Edit via log view (switch to log view with pre-populated form).

### 7.4 Exercise Name Normalization (FIXED)
**Problema:** "Panca Piana", "panca piana", "PANCA PIANA" erano trattati come esercizi diversi.
**Soluzione:** Normalizzare all lowercase prima di salvare, capitalize on display.

### 7.5 Streak Timezone Bug (FIXED v2)
**Problema:** Date objects create from date strings con `T00:00:00` sono interpretate come UTC, causando errori in timezone positivi (CEST/+2).
**Soluzione:** Usare solo stringhe YYYY-MM-DD per confronto date, generare stringhe con `localDateISO()` che non fa parsing.

---

## 8. Architecture Decisions (Piotr)

- **No build step** — single HTML file, CDN deps
- **localStorage only** — no backend, no database
- **Tailwind via CDN** — rapid prototyping
- **MiniMax API** — only for suggestions
- **Progressive enhancement** — works without JS (graceful degrade for read)
- **Edit via Log view** — non blocking, mobile-friendly
- **History via Modal** — full-screen per dettaglio completo sessione
- **Rest Timer as Badge** — non intrusivo, fixed above tab bar
- **Photos as base64 in separate key** — no server upload, privacy-safe, limitato a ~4.5MB

---

## 9. Linear Sprint Board (v2)

| Issue | Titolo | Priority | Stato |
|-------|--------|----------|-------|
| WM-50 | Fix todayISO() typo bug | P0 | Done |
| WM-51 | Add date picker with today pre-filled | P0 | Done |
| WM-52 | Add edit workout functionality | P0 | Done |
| WM-53 | Exercise name normalization | P1 | Done |
| WM-54 | Add WorkoutModal component | P0 | Done |
| WM-55 | Update SPEC.md for v2 | Done | Done |
| WM-56 | Progress photos (base64 localStorage) | P1 | Done |
| WM-57 | History view modal/full-screen | P1 | Done |
| WM-58 | Rest timer UI countdown | P1 | Done |
| WM-59 | Rest timer configuration | P2 | Done |
| WM-60 | Streak timezone-aware fix | P1 | Done |
| WM-61 | API Key pre-filled from profile | P2 | Done |

---

## 10. Done Criteria (v2)

1. ✅ Date picker pre-popolato con today
2. ✅ Edit workout funzionante (modifica + salva)
3. ✅ Delete workout funzionante
4. ✅ Exercise names salvati lowercase normalizzati
5. ✅ `todayISO()` rimosso / fix
6. ✅ Progress photos — acquisizione da camera, salvataggio base64, display grid, eliminazione, resize a 800px max
7. ✅ History modal full-screen con dettaglio sessione + edit button
8. ✅ Rest timer — badge fisso con countdown, skip, auto-dismiss con "✓ Recupero!"
9. ✅ Rest timer configurabile da settings (30-300s, step 10)
10. ✅ Streak calcolato con date strings (no timezone)
11. ✅ API key precompilata dal profilo utente in AI view e settings

---

## 11. File Changes

### index.html (modified v2)
- Aggiungere `<input type="date">` nel form workout
- Aggiungere photo capture UI (input file + grid) con resize client-side
- Normalizzazione nome esercizio in `saveWorkout()`
- Logica update vs create in submit handler
- Delete workout button
- History modal full-screen
- Rest timer badge fisso con countdown e beep
- Settings per recovery timer e API key
- Profile fields: recoveryTimerSeconds, miniMaxApiKey
- Photo storage in localStorage separata (KEYS.PHOTOS) come mapping photoId → base64 data