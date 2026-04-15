# SPEC.md — Workout Monitor

## 1. Concept & Vision

**Workout Monitor** è un'app web minimalista per tracciare gli allenamenti in palestra e ricevere suggerimenti intelligenti basati sugli obiettivi dell'utente.

L'esperienza è pulita, motivante, senza distrazioni. Come un diario cartaceo della palestra, ma più smart.

---

## 2. Tech Stack

| Layer | Choice |
|-------|--------|
| Frontend | Single HTML file — vanilla JS + Tailwind CSS (CDN) |
| Storage | localStorage (JSON) — no backend required |
| API suggerimenti | MiniMax API (sk-cp-...) |
| Deploy | GitHub Pages |
| CDN assets | cdnjs, unpkg |

---

## 3. Layout & Struttura

### Pagine / View

1. **Home / Dashboard** — riepilogo settimanale, ultime sessioni, obiettivo attivo
2. **Log Workout** — form per registrare un allenamento (esercizi, serie, reps, peso)
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

### 4.1 Log Allenamento
- Seleziona muscolo target (pettorali, schiena, gambe, spalle, braccia, core)
- Aggiungi esercizi: nome, serie, reps, peso (kg)
- Note libere opzionali
- Data/ora automatica (o modifica manuale)
- Salvataggio in localStorage

### 4.2 Storico
- Lista cronologica (più recente prima)
- Filtro per muscolo
- Dettaglio espandibile per ogni sessione
- Totale volume (serie × reps × peso) per sessione

### 4.3 Dashboard
- Sessioni questa settimana (count)
- Volume totale settimanale
- Ultimo workout (data + muscolo)
- Streak giorni consecutivi con workout

### 4.4 Suggerimenti AI
- Input: obiettivo (forza / ipertrofia / definizione) + muscolo target
- Output: 3-5 esercizi suggeriti con reps/sets range
- Chiamata MiniMax API con prompt strutturato
- Cache locale dei suggerimenti (stesso obiettivo = stesso output per 24h)

---

---

## 5. Data Model

### 5.1 Schema Design Principles

- **Denormalizzazione controllata** — ogni entità è autosufficiente, riferimenti incrociati minimi
- **Validazione lato client** — tutti i campi obbligatori enforced via JS prima del salvataggio
- **Versioning implicito** — nessun campo `schema_version`, evoluzione via migration funzioni

### 5.2 Workout Session Schema

```typescript
interface WorkoutSession {
  // Identificazione
  id: string;                    // UUID v4
  
  // Timestamp
  createdAt: string;              // ISO 8601: "2026-04-15T10:30:00Z"
  updatedAt: string;              // ISO 8601, aggiornato su ogni edit
  
  // Contesto workout
  date: string;                   // ISO 8601 — data reale dell'allenamento (può differire da createdAt per workout backdated)
  muscle: MuscleGroup;             // enum: pettorali | schiena | gambe | spalle | braccia | core
  goal: GoalType;                 // enum: forza | ipertrofia | definizione
  
  // Esercizi
  exercises: Exercise[];          // min 1, max 50
  
  // Metadati opzionali
  notes?: string;                 // max 500 caratteri
  duration?: number;              // minuti, 1-300
  
  // Metriche calcolate (pre-computed per performance)
  totalVolume: number;            // kg totali = sum(exercises[].sets * exercises[].reps * exercises[].weight)
  totalSets: number;              // sum(exercises[].sets)
  totalReps: number;              // sum(exercises[].sets * exercises[].reps)
}

interface Exercise {
  id: string;                     // UUID v4 — per edit inline senza toccare l'intero array
  name: string;                   // max 100 caratteri
  sets: number;                   // 1-20
  reps: number;                   // 1-50
  weight: number;                 // kg, 0.5-999, supporta decimali (0.5 step)
  rpe?: number;                   // Rating of Perceived Exertion, 1-10
  notes?: string;                 // max 200 caratteri
}

type MuscleGroup = 'pettorali' | 'schiena' | 'gambe' | 'spalle' | 'braccia' | 'core';
type GoalType = 'forza' | 'ipertrofia' | 'definizione';
```

**Regole di validazione:**
- `id` — generato lato client con `crypto.randomUUID()`
- `muscle` — valore da dropdown predefinito, mai libero
- `exercises[].weight` — multipli di 0.5kg
- `exercises[].name` — uppercase first letter normalizzato
- Storage key: `workout_monitor:sessions`

### 5.3 User Profile Schema

```typescript
interface UserProfile {
  // Identificazione
  id: string;                     // sempre "user_profile" — singleton
  
  // Anagrafica
  name: string;                   // 1-50 caratteri, display name
  
  // Preferenze workout
  goal: GoalType;                 // obiettivo principale
  level: ExperienceLevel;        // principiante | intermedio | avanzato
  preferredUnits: 'kg' | 'lb';   // default 'kg'
  
  // Personalizzazione UI
  theme: 'light' | 'dark' | 'auto';  // default 'auto'
  
  // Tracking preferenze
  trackDuration: boolean;         // default true
  trackRPE: boolean;              // default false
  
  // Metadati
  createdAt: string;              // ISO 8601
  updatedAt: string;              // ISO 8601
  
  // Cache suggerimenti AI
  cachedSuggestions: AICacheEntry[];  // max 20 entries, TTL 24h
}

type ExperienceLevel = 'principiante' | 'intermedio' | 'avanzato';

interface AICacheEntry {
  key: string;                    // hash di "goal:muscle"
  response: AISuggestion[];
  generatedAt: string;            // ISO 8601
  expiresAt: string;              // ISO 8601, generatedAt + 24h
}

interface AISuggestion {
  exercise: string;
  sets: string;                   // es: "3-4"
  reps: string;                  // es: "8-12"
  rest: string;                   // es: "60-90 sec"
  notes?: string;
}
```

**Regole di validazione:**
- Singleton: sempre una sola istanza in localStorage
- Storage key: `workout_monitor:profile`
- Inizializzazione: al primo accesso, crea con valori default + name="Nico"

### 5.4 localStorage Keys

| Key | Type | Description |
|-----|------|-------------|
| `workout_monitor:sessions` | `WorkoutSession[]` | Array di tutte le sessioni |
| `workout_monitor:profile` | `UserProfile` | Singleton profilo utente |

### 5.5 Data Migration Strategy

Versione attuale schema: **1**

```javascript
const SCHEMA_VERSION = 1;

function migrateIfNeeded() {
  const profile = getProfile();
  if (!profile.schemaVersion || profile.schemaVersion < SCHEMA_VERSION) {
    // Future migrations here
    profile.schemaVersion = SCHEMA_VERSION;
    saveProfile(profile);
  }
}
```

### 5.6 Performance Considerations

- `totalVolume` e metriche derivate pre-calculate all save
- Session list: max 365 entries in localStorage (auto-archive oldest if >365)
- Lazy loading storico: carica solo le ultime 50 sessioni, "load more" per le precedenti
- Indice `byMuscle` costruito on-the-fly per filtri (nessun indice pre-mantenuto)

---

## 6. Design System

### 6.1 Design Principles

- **Fitness-focused** — colori energici ma non aggressivi, legibilità su schermi piccoli
- **Dark-first** — la maggioranza degli utenti gym guarda il telefono tra una serie, luce bassa
- **Touch-friendly** — tutti i tap targets min 44×44px
- **Micro-feedback** — ogni interazione confermata visivamente

### 6.2 Color Palette

```css
/* === Dark Theme (Default) === */
:root {
  /* Backgrounds */
  --bg-primary: #0f0f0f;         /* Main background — quasi nero */
  --bg-secondary: #1a1a1a;       /* Cards, elevated surfaces */
  --bg-tertiary: #262626;        /* Input fields, hover states */
  
  /* Text */
  --text-primary: #f5f5f5;        /* Main text — quasi bianco */
  --text-secondary: #a3a3a3;      /* Secondary, labels */
  --text-muted: #666666;         /* Placeholder, disabled */
  
  /* Accent — Electric Lime */
  --accent-primary: #d4ff00;     /* CTA buttons, active states */
  --accent-secondary: #b8e600;   /* Hover su accent */
  --accent-muted: rgba(212, 255, 0, 0.15); /* Accent backgrounds */
  
  /* Semantic */
  --success: #22c55e;           /* Green — workout completed, streak */
  --warning: #f59e0b;           /* Orange — RPE alto, attenzione */
  --danger: #ef4444;            /* Red — delete, error */
  
  /* Muscle Group Colors (per tag/badge) */
  --muscle-pettorali: #3b82f6;   /* Blue */
  --muscle-schiena: #8b5cf6;     /* Purple */
  --muscle-gambe: #22c55e;       /* Green */
  --muscle-spalle: #f59e0b;      /* Orange */
  --muscle-braccia: #ec4899;     /* Pink */
  --muscle-core: #06b6d4;        /* Cyan */
}

/* === Light Theme === */
.theme-light {
  --bg-primary: #ffffff;
  --bg-secondary: #f5f5f5;
  --bg-tertiary: #e5e5e5;
  
  --text-primary: #0f0f0f;
  --text-secondary: #525252;
  --text-muted: #a3a3a3;
  
  --accent-primary: #65a30d;    /* Più scuro per contrasto su sfondo chiaro */
  --accent-secondary: #4d7c0f;
  --accent-muted: rgba(101, 163, 13, 0.15);
  
  --success: #16a34a;
  --warning: #d97706;
  --danger: #dc2626;
}
```

### 6.3 Typography

```css
/* Font Stack */
font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;

/* Scale */
--text-xs: 0.75rem;      /* 12px — etichette piccole */
--text-sm: 0.875rem;     /* 14px — secondary text */
--text-base: 1rem;       /* 16px — body, default */
--text-lg: 1.125rem;     /* 18px — titoli secondari */
--text-xl: 1.25rem;      /* 20px — titoli principali */
--text-2xl: 1.5rem;      /* 24px — numeri grandi (stats) */
--text-3xl: 2rem;        /* 32px — hero numbers */

/* Weights */
--font-normal: 400;
--font-medium: 500;
--font-semibold: 600;
--font-bold: 700;

/* Letter spacing */
--tracking-tight: -0.025em;   /* Titoli */
--tracking-normal: 0;          /* Body */
--tracking-wide: 0.05em;      /* Labels uppercase */
```

**Font:** Inter (Google Fonts) — scelta per:
- Leggibilità ottima a tutti i pesi
- Numeri tabulari per stats allineati
- Ottimizzato per UI

### 6.4 Spacing System

```css
/* Base unit: 4px */
--space-1: 0.25rem;   /* 4px */
--space-2: 0.5rem;    /* 8px */
--space-3: 0.75rem;   /* 12px */
--space-4: 1rem;      /* 16px */
--space-5: 1.25rem;   /* 20px */
--space-6: 1.5rem;    /* 24px */
--space-8: 2rem;      /* 32px */
--space-10: 2.5rem;   /* 40px */
--space-12: 3rem;     /* 48px */
--space-16: 4rem;     /* 64px */

/* Component-specific */
--radius-sm: 0.375rem;   /* 6px — input, piccoli elementi */
--radius-md: 0.5rem;     /* 8px — cards */
--radius-lg: 0.75rem;    /* 12px — modali, CTA */
--radius-full: 9999px;   /* Pill buttons */

/* Touch targets */
--tap-target-min: 44px;  /* Minimo per accessibilità mobile */
```

### 6.5 Shadows & Elevation

```css
--shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.3);
--shadow-md: 0 4px 6px rgba(0, 0, 0, 0.4);
--shadow-lg: 0 10px 15px rgba(0, 0, 0, 0.5);
--shadow-glow: 0 0 20px rgba(212, 255, 0, 0.3);  /* Accent glow per focus */
```

### 6.6 Motion & Animation

```css
/* Durations */
--duration-fast: 150ms;     /* Hover, micro-interactions */
--duration-normal: 250ms;   /* Modali, expand/collapse */
--duration-slow: 400ms;     /* Page transitions */

/* Easings */
--ease-out: cubic-bezier(0.16, 1, 0.3, 1);     /* Entrata elementi */
--ease-in-out: cubic-bezier(0.45, 0, 0.55, 1); /* Transizioni morbide */
--ease-bounce: cubic-bezier(0.34, 1.56, 0.64, 1); /* Feedback positivo */

/* Animation use cases */
/* - Bottone premuto: scale(0.97) in 100ms */
/* - Card appear: translateY(8px) + opacity, 250ms ease-out */
/* - Modal slide-up: translateY(100%) → 0, 300ms ease-out */
/* - Numero stats: counting animation 600ms */
```

### 6.7 Component Tokens

```css
/* Bottom Tab Bar */
--tab-height: 64px;
--tab-icon-size: 24px;
--tab-active-bg: var(--accent-muted);
--tab-active-icon: var(--accent-primary);
--tab-inactive-icon: var(--text-muted);

/* Cards */
--card-padding: var(--space-4);
--card-radius: var(--radius-md);
--card-bg: var(--bg-secondary);

/* Buttons */
--btn-height: 48px;         /* Primary CTA */
--btn-height-sm: 36px;      /* Secondary, inline */
--btn-radius: var(--radius-lg);
--btn-primary-bg: var(--accent-primary);
--btn-primary-text: #0f0f0f; /* Dark text su accent chiaro */

/* Inputs */
--input-height: 48px;
--input-radius: var(--radius-sm);
--input-bg: var(--bg-tertiary);
--input-border: transparent;
--input-focus-border: var(--accent-primary);
```

### 6.8 Icon System

- Emoji native per semplicità (🏋️ 💪 📅 ⏱️ 🏠 📝 📊 🤖)
- No external icon library — zero dipendenze aggiuntive
- 2x resolution emoji per display Retina

### 6.9 Responsive Breakpoints

```css
/* Mobile-first */
--breakpoint-sm: 480px;   /* Max width container */
--breakpoint-md: 768px;   /* Tablet */
--breakpoint-lg: 1024px;  /* Desktop */

/* Container */
.container {
  max-width: var(--breakpoint-sm);
  margin: 0 auto;
  padding: 0 var(--space-4);
}
```

---

## 7. Linear Sprint Board

| Issue | Titolo | Assignee | Stato |
|-------|--------|----------|-------|
| CLA-40 | Setup: Repo GitHub + Tech Stack | Piotr | Backlog |
| CLA-41 | Database: Schema Workout & Exercises | Piotr | Backlog |
| CLA-42 | UI: Workout Logger View | Thomas | Backlog |
| CLA-43 | UI: Workout History & Stats | Thomas | Backlog |
| CLA-44 | UX: Design System — colori, tipografia, spacing | Piotr | Backlog |
| CLA-45 | UX: Bottom Tab Navigation | Thomas | Backlog |
| CLA-46 | Feature: Dashboard con stats | Thomas | Backlog |
| CLA-47 | Feature: Suggerimenti AI con MiniMax | Thomas | Backlog |
| CLA-48 | [BLOCKER] Enable GitHub Pages per workout-monitor | Goksu | Backlog |
| CLA-49 | Deploy: GitHub Pages setup + custom domain check | Thomas | Backlog |

---

## 8. Architecture Decisions (Piotr)

- **No build step** — single HTML file, CDN deps
- **localStorage only** — no backend, no database
- **Tailwind via CDN** — rapid prototyping
- **MiniMax API** — only for suggestions, all other logic client-side
- **Progressive enhancement** — works without JS (graceful degrade for read)

---

## 9. Done Criteria

1. ✅ Log workout completato e salvato in localStorage
2. ✅ Storico visibile con filtro muscolo
3. ✅ Dashboard mostra stats settimanali
4. ✅ Suggerimenti AI funzionanti (almeno 3 esercizi restituiti)
5. ✅ Deploy su https://niccolocoppo88.github.io/workout-monitor/
6. ✅ GitHub Pages enabled sul repo
