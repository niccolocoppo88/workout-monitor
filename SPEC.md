# SPEC.md — Workout Monitor v2

## 1. Concept & Vision

**Workout Monitor** è un'app web minimalista per tracciare gli allenamenti in palestra e ricevere suggerimenti intelligenti basati sugli obiettivi dell'utente.

L'esperienza è pulita, motivante, senza distrazioni. Come un diario cartaceo della palestra, ma più smart.

**v2 Focus:** Bug fixes + Usability improvements (edit workout, date pre-fill, exercise name normalization).

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
2. **Log Workout** — form per registrare/modificare un allenamento
3. **Storico** — lista cronologica delle sessioni, filtri per data/muscolo
4. **Suggerimenti** — consigli basati su obiettivi, generati da AI

### Navigazione
- Bottom tab bar (mobile-first): Home | Log | Storico | AI
- Nessuna sidebar — tutto accessibile in 1 tap

### Responsive
- Mobile-first, max-width 480px centered
- Desktop: stessa UX, centrato con bordo laterale

---

## 4. Features

### 4.1 Log Allenamento (FIXED in v2)

- **Date picker** con data pre-popolata a oggi (BUG FIX: `todayISO()` typo)
- Seleziona muscolo target (pettorali, schiena, gambe, spalle, braccia, core)
- Aggiungi esercizi: nome, serie, reps, peso (kg)
- **Nome esercizio normalizzato** (lowercase, trim, mapping varianti)
- Note libere opzionali
- Data/ora modificabile manualmente
- Salvataggio in localStorage

### 4.2 Edit Workout (NEW in v2)

- Dalla vista storico, tap su workout → apre in modalità edit
- Tutti i campi modificabili (data, muscolo, esercizi, note)
- Possibilità di eliminare workout
- Salva modifiche o annulla

### 4.3 Storico
- Lista cronologica (più recente prima)
- Filtro per muscolo
- Dettaglio espandibile per ogni sessione
- **Pulsante edit** per ogni workout card
- Totale volume (serie × reps × peso) per sessione

### 4.4 Dashboard
- Sessioni questa settimana (count)
- Volume totale settimanale
- Ultimo workout (data + muscolo)
- Streak giorni consecutivi con workout

### 4.5 Suggerimenti AI
- Input: obiettivo (forza / ipertrofia / definizione) + muscolo target
- Output: 3-5 esercizi suggeriti con reps/sets range
- Chiamata MiniMax API con prompt strutturato
- Cache locale dei suggerimenti (stesso obiettivo = stesso output per 24h)

---

## 5. Data Model

### Workout Session (v2)
```json
{
  "id": "uuid",
  "date": "2026-04-15T10:30:00Z",
  "muscle": "pettorali",
  "exercises": [
    {
      "name": "panca piana",
      "sets": 4,
      "reps": 8,
      "weight": 60
    }
  ],
  "notes": "Optional notes",
  "goal": "ipertrofia"
}
```

**Note:** Exercise names are **normalized to lowercase** before storage.

### User Profile (localStorage)
```json
{
  "name": "Nico",
  "goal": "ipertrofia",
  "level": "intermedio"
}
```

---

## 6. Componenti UI

### BottomTabBar
- 4 tab: Home, Log, Storico, AI
- Icone emoji: 🏠 📝 📊 🤖
- Active state: highlighted background

### WorkoutCard
- Data + muscolo in header
- Lista esercizi compatta
- **Pulsante edit (matita)** per aprire modal modifica
- Tap per espandere dettaglio
- Volume totale in footer

### WorkoutModal (NEW in v2)
- Slide-up modal per create/edit workout
- Campi: date picker, muscle select, goal select, exercises list, notes
- Date picker pre-popolato con today
- Save / Cancel / Delete buttons

### ExerciseRow
- Inline editing dei campi
- + button per aggiungere esercizio
- Exercise name auto-normalized on blur

### AIModal
- Slide-up modal
- Textarea per obiettivo
- Loading spinner durante generazione
- Risultato come lista scheda

---

## 7. Bug Fixes Detail (v2)

### 7.1 todayISO() Typo Bug
**Problema:** Codice usava `todayISO()` invece di `new Date().toISOString()` o funzione corretta.
**Soluzione:** Rimuovere helper buggy, usare direttamente `new Date().toISOString()` con timezone locale corretto. Aggiungere helper `formatDateForInput()` che restituisce `YYYY-MM-DD` per il date picker HTML5.

### 7.2 Date Not Pre-filled
**Problema:** Il form workout non aveva campo data, solo timestamp automatico.
**Soluzione:** Aggiungere `<input type="date">` al form con `value` pre-popolato a today via `formatDateForInput(new Date())`.

### 7.3 No Edit Workout
**Problema:** Non era possibile modificare un workout esistente.
**Soluzione:** 
- Aggiungere WorkoutModal per create/edit
- Dal bottone edit sulla WorkoutCard, aprire modal con dati pre-popolati
- In submit, distinguere create vs update (se `workout.id` esiste → update)

### 7.4 Exercise Name Normalization
**Problema:** "Panca Piana", "panca piana", "PANCA PIANA" erano trattati come esercizi diversi.
**Soluzione:** 
- Normalizzare all lowercase prima di salvare
- Trim whitespace
- Mantenere mapping opzionale per varianti comuni (opzionale in v2)

---

## 8. Architecture Decisions (Piotr)

- **No build step** — single HTML file, CDN deps
- **localStorage only** — no backend, no database
- **Tailwind via CDN** — rapid prototyping
- **MiniMax API** — only for suggestions
- **Progressive enhancement** — works without JS (graceful degrade for read)
- **Edit via Modal** — non blocking, mobile-friendly

---

## 9. Linear Sprint Board (v2)

| Issue | Titolo | Priority | Stato |
|-------|--------|----------|-------|
| WM-50 | Fix todayISO() typo bug | P0 | Todo |
| WM-51 | Add date picker with today pre-filled | P0 | Todo |
| WM-52 | Add edit workout functionality | P0 | Todo |
| WM-53 | Exercise name normalization | P1 | Todo |
| WM-54 | Add WorkoutModal component | P0 | Todo |
| WM-55 | Update SPEC.md for v2 | Done | Done |

---

## 10. Done Criteria

1. ✅ Date picker pre-popolato con today
2. ✅ Edit workout funzionante (modifica + salva)
3. ✅ Delete workout funzionante
4. ✅ Exercise names salvati lowercase normalizzati
5. ✅ `todayISO()` rimosso / fix

---

## 11. File Changes

### index.html (modified)
- Aggiungere `<input type="date">` nel form workout
- Aggiungere WorkoutModal per edit
- Normalizzazione nome esercizio in `collectExercises()`
- Funzione `formatDateForInput()` per pre-popolare date picker
- Logica update vs create in submit handler
- Delete workout button nel modal
