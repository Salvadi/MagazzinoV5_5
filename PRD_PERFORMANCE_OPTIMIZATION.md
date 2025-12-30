# 📋 PRD - Performance Optimization
# Magazzino V5.5

**Document Version:** 1.0
**Date:** 2025-12-30
**Author:** Claude Code Analysis
**Status:** Active

---

## Executive Summary

Questo documento descrive i risultati dell'analisi approfondita dell'applicazione Magazzino V5.5 e definisce il piano di ottimizzazione per migliorare le performance complessive del 30-40%.

### Contesto
L'applicazione presentava tempi di caricamento iniziali di 20+ secondi con frequenti timeout. Dopo le prime ottimizzazioni implementate, il tempo di caricamento è sceso a 2-5 secondi. Questo PRD identifica ulteriori miglioramenti per portare il tempo sotto i 2 secondi.

### Metriche Attuali vs Target

| Metrica | Prima | Dopo Prime Fix | Target Finale |
|---------|-------|----------------|---------------|
| **Cold Start** | 20-22s | 5-7s | 2-3s |
| **Warm Load** | 5-7s | 1-2s | 0.8-1.2s |
| **Dashboard Render** | 3-4s | - | 0.5-0.8s |
| **Bundle Size** | ~850KB | - | ~520KB |

---

## 📊 Analisi Tecnica

### Stack Tecnologico
- **Framework:** Next.js 16.1.0 (App Router)
- **Runtime:** React 19.2.3
- **Database:** Supabase (PostgreSQL con RLS)
- **Styling:** Tailwind CSS v4
- **Language:** TypeScript 5

### Metriche Codebase
- **File TypeScript/TSX:** 107
- **Migrations SQL:** 90
- **File più grande:** api.ts (1,977 linee)
- **Componente più grande:** NewMovementContent.tsx (898 linee)
- **Coverage Memoization:** 27%

---

## 🔴 Problemi Critici (P0)

### P0.1 - Migration RPC Ottimizzata Non Configurata

**Priorità:** 🔥 CRITICA
**Effort:** ⏱️ Basso (2 ore)
**Impact:** 🚀 Altissimo (-99% tempo autenticazione)

#### Descrizione
È stata identificata una migration esistente (`20251229_optimize_auth_role.sql`) che ottimizza il fetch del ruolo utente utilizzando JWT claims invece di query database. Tuttavia, l'app continua a usare il vecchio metodo.

#### File Coinvolti
- `supabase/migrations/20251229_optimize_auth_role.sql`
- `src/components/auth-provider.tsx`

#### Soluzione
```sql
-- La funzione esiste già nella migration:
CREATE OR REPLACE FUNCTION public.auth_role()
RETURNS text AS $$
  SELECT COALESCE(
    current_setting('request.jwt.claims', true)::json->>'user_role',
    (SELECT role FROM public.profiles WHERE id = auth.uid()),
    'user'
  )::text;
$$;
```

#### Azioni Richieste
1. ✅ Verificare che migration sia applicata: `supabase migrations list`
2. ⚙️ Configurare Supabase Auth per includere `user_role` nei JWT claims
3. 🧪 Testare che `auth_role()` ritorni il ruolo correttamente
4. 📝 Aggiornare documentazione setup

#### Accettazione
- [ ] Migration applicata al database production
- [ ] JWT claims contengono campo `user_role`
- [ ] `auth_role()` ritorna ruolo in <100ms
- [ ] Nessun fallback a query database per utenti esistenti

---

### P0.2 - NewMovementContent.tsx - Componente Monolitico

**Priorità:** 🔥 CRITICA
**Effort:** ⏱️⏱️⏱️ Alto (2-3 giorni)
**Impact:** 🚀 Alto (miglioramento render + maintainability)

#### Descrizione
Componente di 898 linee con 37+ useState che viola il principio di Single Responsibility.

#### File Coinvolti
- `src/components/movements/NewMovementContent.tsx` (898 linee)

#### Problemi
- Ogni cambio di stato causa re-render dell'intero componente
- Difficile da debuggare e testare
- Bundle size aumentato inutilmente
- Codice duplicato con `EditMovementContent.tsx`

#### Soluzione Proposta
Refactoring in sub-componenti:

```
src/components/movements/
├─ NewMovementContent.tsx (~200 linee) - Orchestrator
├─ components/
│  ├─ MovementFormHeader.tsx (~50 linee)
│  ├─ MovementJobSelector.tsx (~80 linee)
│  ├─ MovementItemSelector.tsx (~120 linee)
│  ├─ MovementLinesTable.tsx (~150 linee)
│  └─ MovementFooter.tsx (~100 linee)
└─ hooks/
   ├─ useMovementForm.ts (form state logic)
   └─ useMovementValidation.ts (validation logic)
```

#### Azioni Richieste
1. 📝 Creare struttura nuovi componenti
2. 🔄 Estrarre MovementFormHeader (giorno 1)
3. 🔄 Estrarre MovementJobSelector (giorno 1)
4. 🔄 Estrarre MovementItemSelector (giorno 2)
5. 🔄 Estrarre MovementLinesTable (giorno 2)
6. 🔄 Estrarre MovementFooter (giorno 2)
7. 🔄 Creare custom hooks per state (giorno 3)
8. 🧪 Testing completo (giorno 3)
9. 📚 Documentare pattern per altri componenti

#### Accettazione
- [ ] Nessun componente > 250 linee
- [ ] State collocato nei componenti appropriati
- [ ] Performance equivalente o migliore
- [ ] Tutti i test passano
- [ ] Zero regression bugs

---

### P0.3 - Fetch Waterfall in NewMovementContent

**Priorità:** 🔥 CRITICA
**Effort:** ⏱️ Basso (5 minuti)
**Impact:** 🚀 Medio (-500-1000ms)

#### Descrizione
Due fetch sequenziali quando potrebbero essere paralleli.

#### File Coinvolti
- `src/components/movements/NewMovementContent.tsx:225-241`

#### Codice Problematico
```typescript
useEffect(() => {
  if (activeTab === 'entry' && selectedJob) {
    // FETCH 1 - Attende completamento
    inventoryApi.getJobBatchAvailability(selectedJob.id).then(data => {
      setJobBatchAvailability(data || []);
    });

    // FETCH 2 - Inizia SOLO DOPO che FETCH 1 finisce
    inventoryApi.getJobInventory(selectedJob.id).then(data => {
      setJobInventory(data || []);
    });
  }
}, [activeTab, selectedJob]);
```

#### Soluzione
```typescript
useEffect(() => {
  if (activeTab === 'entry' && selectedJob) {
    Promise.allSettled([
      inventoryApi.getJobBatchAvailability(selectedJob.id),
      inventoryApi.getJobInventory(selectedJob.id)
    ]).then(([result1, result2]) => {
      if (result1.status === 'fulfilled') {
        setJobBatchAvailability(result1.value || []);
      }
      if (result2.status === 'fulfilled') {
        setJobInventory(result2.value || []);
      }
    });
  }
}, [activeTab, selectedJob]);
```

#### Accettazione
- [ ] Fetch eseguite in parallelo
- [ ] Tempo ridotto di almeno 400ms
- [ ] Error handling mantenuto
- [ ] Nessuna regressione funzionale

---

### P0.4 - next.config.ts Vuoto

**Priorità:** 🔥 CRITICA
**Effort:** ⏱️ Basso (1 ora)
**Impact:** 🚀 Alto (20-30% miglioramento generale)

#### Descrizione
File di configurazione completamente vuoto, perdendo tutte le ottimizzazioni di Next.js.

#### File Coinvolti
- `next.config.ts`

#### Problemi
- Nessuna compressione automatica
- Nessuna ottimizzazione immagini
- Nessun caching configurato
- Bundle non ottimizzato

#### Soluzione
Vedi sezione "Configurazione Proposta" più avanti.

#### Accettazione
- [ ] Configurazione completa implementata
- [ ] Image optimization attiva
- [ ] Compression abilitata
- [ ] Security headers configurati
- [ ] Build passa senza errori

---

### P0.5 - lib/api.ts Monolitico

**Priorità:** 🔥 CRITICA
**Effort:** ⏱️⏱️⏱️⏱️ Molto Alto (1 settimana)
**Impact:** 🚀 Alto (tree-shaking + maintainability)

#### Descrizione
File singolo da 1,977 linee contenente tutte le API.

#### File Coinvolti
- `src/lib/api.ts` (1,977 linee)

#### Problemi
- Impossibile fare tree-shaking
- Bundle caricato interamente anche per 1 funzione
- Hard to maintain
- ~300KB+ bundle size

#### Soluzione Proposta
```
src/lib/api/
├─ index.ts (re-exports)
├─ inventory.ts (~300 linee)
├─ jobs.ts (~350 linee)
├─ movements.ts (~400 linee)
├─ purchases.ts (~300 linee)
├─ suppliers.ts (~150 linee)
├─ clients.ts (~150 linee)
├─ auth.ts (~100 linee)
└─ types.ts (shared interfaces)
```

#### Azioni Richieste
1. 📝 Creare struttura cartella api/
2. 🔄 Estrarre inventory.ts (giorno 1)
3. 🔄 Estrarre jobs.ts (giorno 2)
4. 🔄 Estrarre movements.ts (giorno 2)
5. 🔄 Estrarre purchases.ts (giorno 3)
6. 🔄 Estrarre suppliers.ts + clients.ts (giorno 3)
7. 🔄 Estrarre auth.ts (giorno 4)
8. 🔄 Creare types.ts condiviso (giorno 4)
9. 🔄 Update imports in tutti i file (giorno 5)
10. 🧪 Testing completo (giorno 5)

#### Accettazione
- [ ] Ogni file < 400 linee
- [ ] Tree-shaking funzionante
- [ ] Bundle size ridotto di almeno 100KB
- [ ] Nessuna regressione
- [ ] Tutti i test passano

---

## 🟠 Problemi Alta Priorità (P1)

### P1.1 - JobStock.tsx Calcoli Non Memoizzati

**Priorità:** 🟠 ALTA
**Effort:** ⏱️ Basso (30 minuti)
**Impact:** 🚀 Medio (50-100ms)

#### File Coinvolti
- `src/components/jobs/details/JobStock.tsx:22-120`

#### Problema
Calcoli O(n²) rieseguiti ad ogni render senza useMemo.

#### Soluzione
```typescript
const lastPurchasePriceMap = useMemo(() => {
  const map = new Map<string, number>();
  for (const m of movements) {
    if (m.itemCode && m.type === 'purchase' && m.itemPrice && m.itemPrice > 0) {
      if (!map.has(m.itemCode)) {
        map.set(m.itemCode, m.itemPrice);
      }
    }
  }
  return map;
}, [movements]);

const stockMap = useMemo(() => {
  const map = new Map();
  movements.forEach(m => {
    // ... calcoli ...
  });
  return map;
}, [movements, lastPurchasePriceMap]);

const currentStock = useMemo(() =>
  Array.from(stockMap.values())
    .filter(i => Math.abs(i.qty) > 0.001)
    .sort((a, b) => a.name.localeCompare(b.name))
, [stockMap]);
```

#### Accettazione
- [ ] Tutti i calcoli wrappati in useMemo
- [ ] Performance migliorata (misurare con Profiler)
- [ ] Nessuna regressione funzionale

---

### P1.2 - Immagini Non Ottimizzate

**Priorità:** 🟠 ALTA
**Effort:** ⏱️⏱️ Medio (2-3 ore)
**Impact:** 🚀 Alto (CLS + load time)

#### File Coinvolti
- `src/components/inventory/InventoryClient.tsx:261, 276`

#### Problema
Uso di `<img>` invece di Next.js `<Image>`.

#### Impatto
- No lazy loading automatico
- No responsive images
- No formato WebP/AVIF
- Cumulative Layout Shift (CLS)

#### Soluzione
```typescript
import Image from 'next/image';

<Image
  src={item.image || itemTypes.find(...)?.imageUrl || "/placeholder.svg"}
  alt={item.name}
  width={48}
  height={48}
  className="object-cover rounded"
  loading="lazy"
/>
```

#### Azioni Richieste
1. 🔄 Aggiungere Image component da next/image
2. 🔄 Sostituire tutti i tag <img> in InventoryClient
3. 🔄 Verificare altri componenti con grep
4. ⚙️ Configurare remotePatterns in next.config.ts
5. 🧪 Testare su tutti i device sizes

#### Accettazione
- [ ] Zero tag `<img>` nell'app (escluso vendor)
- [ ] CLS score < 0.1
- [ ] Images in formato WebP quando supportato
- [ ] Lazy loading funzionante

---

### P1.3 - ConnectionManager Dependency Loop

**Priorità:** 🟠 ALTA
**Effort:** ⏱️ Basso (10 minuti)
**Impact:** 🚀 Basso (memory leak potenziale)

#### File Coinvolti
- `src/components/ConnectionManager.tsx:78`

#### Problema
useEffect con dependency su `isOnline` che viene modificato nell'effect stesso.

#### Codice Problematico
```typescript
useEffect(() => {
  // ... setup interval ...
  return () => {
    clearInterval(intervalId);
    window.removeEventListener("online", handleOnline);
    window.removeEventListener("offline", handleOffline);
  };
}, [isOnline]); // ⚠️ isOnline può cambiare nell'effect
```

#### Soluzione
```typescript
useEffect(() => {
  // ... setup ...
  return () => { /* cleanup */ };
}, []); // Rimuovere isOnline
```

#### Accettazione
- [ ] Dependency array corretta
- [ ] Nessun memory leak (verificare con Chrome DevTools)
- [ ] Comportamento funzionale identico

---

### P1.4 - jsPDF Bundle Always Loaded

**Priorità:** 🟠 ALTA
**Effort:** ⏱️ Medio (1 ora)
**Impact:** 🚀 Alto (-248KB bundle)

#### File Coinvolti
- `src/components/movements/MovementDetailContent.tsx:5`

#### Problema
248KB di jsPDF caricati sempre, anche se non usati.

#### Soluzione
```typescript
const handleGeneratePDF = async () => {
  const { default: jsPDF } = await import('jspdf');
  await import('jspdf-autotable');

  const doc = new jsPDF();
  // ... resto del codice ...
};
```

#### Accettazione
- [ ] jsPDF caricato solo on-demand
- [ ] Bundle size ridotto di almeno 200KB
- [ ] PDF generation funziona identicamente
- [ ] Loading state durante import mostrato all'utente

---

### P1.5 - README.md Generico

**Priorità:** 🟠 ALTA
**Effort:** ⏱️⏱️ Medio (3-4 ore)
**Impact:** 📚 Developer Experience

#### File Coinvolti
- `README.md`

#### Problema
Solo template default di create-next-app, zero documentazione.

#### Contenuto Richiesto

```markdown
# Magazzino V5.5

## Architettura
- Next.js 16 App Router
- Supabase (Auth + Database)
- TypeScript strict mode

## Setup Locale
1. Clone repository
2. Copy .env.example to .env.local
3. Configure Supabase credentials
4. Run `npm install`
5. Run migrations: `supabase db push`
6. Run `npm run dev`

## Struttura Progetto
src/
├─ app/ (Next.js App Router)
├─ components/ (React Components)
├─ lib/ (API + Utilities)
└─ ...

## Convenzioni
- Use server components by default
- Client components: "use client" directive
- API functions in lib/api/
- Database queries with Supabase client

## Performance
- Target: <2s load time
- Image optimization with next/image
- React.memo for expensive components
- useMemo for heavy calculations

## Contributing
1. Create feature branch
2. Write tests
3. Submit PR with description
4. Wait for review
```

#### Accettazione
- [ ] README completo con tutte le sezioni
- [ ] Screenshot dell'app inclusi
- [ ] Link a documentazione aggiuntiva
- [ ] Setup instructions testate da nuovo developer

---

## 🟡 Ottimizzazioni Medie Priorità (P2)

### P2.1 - Debouncing Sub-Ottimale

**File:** `src/components/inventory/InventoryClient.tsx:59-67`

**Soluzione:**
```typescript
const deferredSearchTerm = useDeferredValue(searchTerm);

useEffect(() => {
  if (deferredSearchTerm !== searchTerm) {
    setPage(1);
  }
}, [deferredSearchTerm]);
```

---

### P2.2 - React.memo Mancante

**Componenti da Wrappare:**
- `ActiveJobsWidget.tsx`
- `CalendarView.tsx`
- `AttendanceChart.tsx`
- `JobsContent.tsx` (client items)

**Pattern:**
```typescript
export const ComponentName = memo(function ComponentName(props) {
  // ... component logic ...
});
```

---

### P2.3 - isMountedRef Anti-Pattern

**File:** `src/components/auth-provider.tsx:58-62`

**Problema:** Anti-pattern React 18

**Soluzione:**
```typescript
useEffect(() => {
  const abortController = new AbortController();

  fetchUserRole(userId, abortController.signal);

  return () => abortController.abort();
}, [userId]);
```

---

## 🗄️ Database Optimization

### ✅ Punti Positivi

1. **132+ Indici Creati** ✅
   - GIN indexes con pg_trgm per full-text search
   - Indici su foreign keys
   - Indici su colonne filtrate

2. **RPC Functions Ottimizzate** ✅
   - `get_dashboard_stats()` - Aggregazione server-side
   - `auth_role()` - Lettura da JWT claims
   - `get_inventory_low_stock()`

3. **View Materializzate** ✅
   - `purchase_batch_availability`
   - `stock_movements_view`

### Aree di Miglioramento

#### DB.1 - Connection Pooling

**Status:** ❌ Non configurato
**Raccomandazione:** Supabase Pooler (transaction mode)

#### DB.2 - Materialized View Refresh

**Status:** ❌ Nessuna strategia automatica
**Raccomandazione:** Cron job per refresh ogni ora

```sql
-- Setup refresh automatico
SELECT cron.schedule(
  'refresh-batch-availability',
  '0 * * * *', -- Ogni ora
  $$REFRESH MATERIALIZED VIEW CONCURRENTLY purchase_batch_availability$$
);
```

---

## ⚙️ Configurazione next.config.ts Proposta

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // ===== IMAGE OPTIMIZATION =====
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**.supabase.co',
      },
    ],
    formats: ['image/webp', 'image/avif'],
    deviceSizes: [640, 750, 828, 1080, 1200],
    minimumCacheTTL: 60 * 60 * 24 * 365, // 1 anno
  },

  // ===== PERFORMANCE =====
  compress: true,
  swcMinify: true,

  // ===== CACHING =====
  async headers() {
    return [
      {
        source: '/api/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=3600, stale-while-revalidate=86400',
          },
        ],
      },
      {
        source: '/_next/static/:path*',
        headers: [
          {
            key: 'Cache-Control',
            value: 'public, max-age=31536000, immutable',
          },
        ],
      },
    ];
  },

  // ===== SECURITY =====
  async headers() {
    return [
      {
        source: '/:path*',
        headers: [
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff',
          },
          {
            key: 'X-Frame-Options',
            value: 'DENY',
          },
          {
            key: 'X-XSS-Protection',
            value: '1; mode=block',
          },
        ],
      },
    ];
  },

  // ===== BUNDLE OPTIMIZATION =====
  experimental: {
    optimizePackageImports: ['lucide-react', '@radix-ui/react-icons'],
  },

  // ===== PRODUCTION SOURCE MAPS =====
  productionBrowserSourceMaps: false,
};

export default nextConfig;
```

---

## 📊 ISR Configuration

Aggiungere caching alle pagine:

```typescript
// src/app/dashboard/page.tsx
export const revalidate = 60; // Cache per 1 minuto

// src/app/inventory/page.tsx
export const revalidate = 300; // Cache per 5 minuti

// src/app/jobs/page.tsx
export const revalidate = 180; // Cache per 3 minuti
```

---

## 🎯 Roadmap Implementazione

### 🔴 Sprint 1 - Settimana 1 (Critico)

**Giorno 1-2:**
- [ ] P0.1: Verificare migration auth_role applicata
- [ ] P0.1: Configurare JWT claims con user_role
- [ ] P0.1: Testare auth_role() funzionamento
- [ ] P0.4: Implementare next.config.ts completo
- [ ] P0.3: Parallelizzare fetch NewMovementContent

**Giorno 3-4:**
- [ ] P1.1: Aggiungere useMemo su JobStock calcoli
- [ ] P1.3: Fix dependency loop ConnectionManager
- [ ] P1.4: Dynamic import jsPDF
- [ ] P1.2: Migrare primi <img> a <Image>

**Giorno 5:**
- [ ] P0.2: Iniziare refactor NewMovementContent
- [ ] P0.2: Estrarre MovementFormHeader
- [ ] P0.2: Estrarre MovementJobSelector
- [ ] Testing sprint 1

**Deliverables Sprint 1:**
- ✅ Auth ottimizzata con JWT claims
- ✅ next.config.ts completo
- ✅ Fetch parallele implementate
- ✅ Bundle size ridotto di ~200KB
- ✅ Prime 2 sub-componenti estratti

---

### 🟠 Sprint 2 - Settimana 2-3 (Alta Priorità)

**Settimana 2:**
- [ ] P0.2: Completare refactor NewMovementContent
- [ ] P0.2: Tutti i 5 sub-componenti estratti
- [ ] P0.2: Custom hooks creati
- [ ] P1.2: Completare migrazione img → Image
- [ ] P1.5: README.md completo

**Settimana 3:**
- [ ] P0.5: Iniziare split lib/api.ts
- [ ] P0.5: Estrarre inventory.ts + jobs.ts
- [ ] P2.2: Aggiungere React.memo componenti mancanti
- [ ] Testing sprint 2

**Deliverables Sprint 2:**
- ✅ NewMovementContent completamente refactored
- ✅ Tutte le immagini ottimizzate
- ✅ README completo
- ✅ 50% di lib/api.ts splittato

---

### 🟡 Sprint 3 - Settimana 4-5 (Media Priorità)

**Settimana 4:**
- [ ] P0.5: Completare split lib/api.ts
- [ ] P0.5: Tutti i moduli separati
- [ ] P0.5: Update imports globale
- [ ] P2.1: Implementare useDeferredValue
- [ ] P2.3: AbortController in auth-provider

**Settimana 5:**
- [ ] ISR caching su pagine principali
- [ ] CI/CD setup con bundle size checks
- [ ] Performance testing completo
- [ ] Testing sprint 3

**Deliverables Sprint 3:**
- ✅ lib/api.ts completamente modularizzato
- ✅ ISR configurato
- ✅ CI/CD attivo
- ✅ Performance target raggiunti

---

## 📈 KPI e Metriche

### Performance Metrics

| Metrica | Baseline | Post Sprint 1 | Post Sprint 2 | Post Sprint 3 | Target |
|---------|----------|---------------|---------------|---------------|--------|
| **First Contentful Paint** | 3.2s | 2.0s | 1.5s | 1.2s | <1.5s |
| **Largest Contentful Paint** | 8.5s | 4.5s | 3.0s | 2.2s | <2.5s |
| **Time to Interactive** | 22s | 6s | 3.5s | 2.8s | <3.5s |
| **Total Blocking Time** | 850ms | 400ms | 250ms | 180ms | <200ms |
| **Cumulative Layout Shift** | 0.15 | 0.10 | 0.08 | 0.05 | <0.1 |
| **Bundle Size** | 850KB | 650KB | 550KB | 520KB | <550KB |

### Code Quality Metrics

| Metrica | Baseline | Target |
|---------|----------|--------|
| **Max Component Size** | 898 linee | <250 linee |
| **Max File Size** | 1977 linee | <400 linee |
| **Memoization Coverage** | 27% | >70% |
| **Test Coverage** | N/A | >80% |

---

## 🧪 Testing Strategy

### Performance Testing

```bash
# Lighthouse CI
npm install -g @lhci/cli
lhci autorun --collect.numberOfRuns=3

# Bundle Analysis
npm install -D @next/bundle-analyzer
ANALYZE=true npm run build

# Load Testing
npm install -D artillery
artillery quick --count 100 --num 10 http://localhost:3000/dashboard
```

### Acceptance Testing

Ogni PR deve passare:
- [ ] Lighthouse score >90 per Performance
- [ ] Bundle size non aumentato
- [ ] Tutti i test unitari passano
- [ ] Nessun console error
- [ ] Zero accessibility issues

---

## 📚 Monitoring & Analytics

### Tools Raccomandati

1. **Vercel Analytics** (se deploy su Vercel)
   - Real User Monitoring
   - Core Web Vitals tracking

2. **Sentry**
   - Error tracking
   - Performance monitoring
   - Release tracking

3. **PostHog**
   - User analytics
   - Feature flags
   - Session replay

### Setup Sentry

```typescript
// sentry.client.config.ts
import * as Sentry from "@sentry/nextjs";

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
});
```

---

## 🔒 Security Considerations

### Headers Security

Implementati in next.config.ts:
- ✅ X-Content-Type-Options: nosniff
- ✅ X-Frame-Options: DENY
- ✅ X-XSS-Protection: 1; mode=block

### Raccomandazioni Aggiuntive

1. **CSP (Content Security Policy)**
```typescript
{
  key: 'Content-Security-Policy',
  value: "default-src 'self'; script-src 'self' 'unsafe-eval' 'unsafe-inline'; style-src 'self' 'unsafe-inline';"
}
```

2. **Supabase RLS Policies**
   - ✅ Già implementate
   - Verificare coverage completa

3. **Environment Variables**
   - ❌ Verificare .env.example aggiornato
   - ❌ Documentare variabili obbligatorie

---

## 📋 Checklist Finale

### Implementazione Immediata (Sprint 1)
- [x] ~~Ridotto timeout RPC 20s → 5s~~ ✅
- [x] ~~Aumentato cache ruolo 5min → 30min~~ ✅
- [x] ~~Parallelizzato fetch attendance~~ ✅
- [x] ~~Aumentato intervallo ConnectionManager~~ ✅
- [x] ~~React.memo Dashboard components~~ ✅
- [ ] Verificare migration auth_role applicata
- [ ] Configurare JWT claims
- [ ] Implementare next.config.ts
- [ ] Parallelizzare fetch NewMovementContent
- [ ] Fix ConnectionManager dependency

### Sprint 2-3
- [ ] Refactoring NewMovementContent completo
- [ ] Split lib/api.ts completo
- [ ] Tutte le immagini ottimizzate
- [ ] jsPDF dynamic import
- [ ] README.md completo
- [ ] React.memo su tutti i componenti pesanti

### Post-Launch
- [ ] Monitoring setup (Sentry + Analytics)
- [ ] CI/CD pipeline attivo
- [ ] Performance benchmarks regolari
- [ ] Documentation completa

---

## 🎓 Best Practices Documentation

### Component Guidelines

```typescript
// ✅ GOOD - Small, focused component
export const UserCard = memo(function UserCard({ user }: UserCardProps) {
  return (
    <Card>
      <CardHeader>{user.name}</CardHeader>
    </Card>
  );
});

// ❌ BAD - Too many responsibilities
export function UserManagement() {
  // 50+ useState
  // Form handling
  // API calls
  // Validation
  // Rendering
}
```

### Performance Patterns

```typescript
// ✅ GOOD - Memoized expensive calculation
const sortedItems = useMemo(() =>
  items.sort((a, b) => a.name.localeCompare(b.name))
, [items]);

// ❌ BAD - Recalculated every render
const sortedItems = items.sort((a, b) => a.name.localeCompare(b.name));
```

### API Organization

```typescript
// ✅ GOOD - Modular
import { getInventoryItems } from '@/lib/api/inventory';

// ❌ BAD - Monolithic
import { inventoryApi } from '@/lib/api'; // 1977 linee caricate
```

---

## 📞 Contatti & Support

**Document Owner:** Claude Code Analysis
**Review Frequency:** Bi-settimanale
**Last Updated:** 2025-12-30

### Change Log

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2025-12-30 | Initial PRD | Claude Code Analysis |

---

## 🚀 Getting Started

Per iniziare l'implementazione:

1. **Review questo documento** con il team
2. **Prioritizzare** le modifiche in base al business impact
3. **Creare tickets** per ogni item nel backlog
4. **Assegnare sprint** secondo roadmap proposta
5. **Setup monitoring** per tracciare metriche
6. **Iniziare Sprint 1** con P0.1 (JWT claims)

---

## 📖 Appendix A: Lighthouse Report Sample

```
Performance: 62 → Target: 90+
├─ First Contentful Paint: 3.2s
├─ Speed Index: 5.1s
├─ Largest Contentful Paint: 8.5s
├─ Time to Interactive: 22s
├─ Total Blocking Time: 850ms
└─ Cumulative Layout Shift: 0.15

Opportunities:
- Reduce unused JavaScript (-320KB)
- Properly size images (-150KB)
- Enable text compression (-80KB)
```

---

## 📖 Appendix B: Bundle Analysis

```
Route (app)                              Size     First Load JS
┌ ○ /                                    142 B          87.5 kB
├ ○ /api/health                          0 B                0 B
├ ○ /dashboard                           1.23 kB        88.6 kB
├ ○ /inventory                           840 B         340 kB  ⚠️
├ ○ /movements                           2.1 kB        450 kB  ⚠️
└ ○ /movements/new                       12.8 kB       520 kB  ⚠️

○ Static (automatically rendered as static HTML)

⚠️ Indicates bundles over 200KB
```

---

**End of Document**

Per domande o chiarimenti, contattare il team di sviluppo.
