# I Due Sistemi di Memoria LEXE

> Architettura, differenze e punti di forza dei due canali di retrieval
> controllati dagli switch nel pannello destro della chat.
>
> Ultimo aggiornamento: 2026-02-14

---

## Panoramica

LEXE dispone di **due sistemi di memoria indipendenti**, ognuno con scopo,
storage e ciclo di vita diversi. L'utente li controlla tramite due switch
nel PreviewPanel della webchat:

| Switch | ID | Default | Backend field | Stato |
|--------|----|---------|---------------|-------|
| **CONVERSAZIONALE** | `memory` | ON | `use_memory_retrieval` | Attivo |
| **SEMANTICA** | `ragV3` | OFF | `use_rag_v3_retrieval` | Coming Soon |

Entrambi i toggle sono persistiti in `localStorage` (sopravvivono al
reload) e trasmessi al backend come campi di `CustomerStreamRequest`.

---

## 1. Memoria CONVERSAZIONALE (lexe-memory L0-L4)

### Cos'e'

Un sistema di **memoria personale dell'utente** a 5 livelli, gestito dal
microservizio `lexe-memory` (porta 8103). Memorizza fatti, preferenze e
contesto dell'utente e li richiama automaticamente per personalizzare le
risposte dell'AI.

### Come funziona (end-to-end)

```
Utente invia messaggio (use_memory_retrieval=true)
    |
    v
[RETRIEVE] customer_router.py chiama lexe-memory POST /memory/search
    |  query = messaggio utente
    |  similarity threshold = 0.3, limit = 5
    |  layer = L3 (semantic, pgvector HNSW)
    |
    v
Risultati iniettati nel system prompt:
    "## Memoria dell'utente
     Queste sono informazioni che conosci sull'utente: ..."
    |
    v
LLM genera risposta usando contesto + memoria
    |
    v
[STORE] store_memory_facts() estrae fatti dal dialogo
    |  Pattern regex: "mi chiamo X", "sono X", "chiamami X"
    |  Salva in L3 via POST /memory/store
    |
    v
done event SSE include memory_status: "retrieved" | "empty" | "skipped"
```

### I 5 livelli di memoria (L0-L4)

| Layer | Nome | Storage | TTL | Scopo |
|-------|------|---------|-----|-------|
| **L0** | Session | Valkey (Redis) | 24h | Contesto sessione corrente, volatilissimo |
| **L1** | Working | Valkey + PostgreSQL | 7 giorni | Memoria di lavoro, importanza dinamica con decay esponenziale |
| **L2** | Episodic | PostgreSQL | 90 giorni | Fatti e decisioni consolidate da L0/L1 |
| **L3** | Semantic | PostgreSQL + pgvector | Permanente | Fatti semantici con embedding vettoriale (1536 dim) |
| **L4** | Graph | Apache AGE (PostgreSQL) | Permanente | Knowledge graph con nodi ed archi (Cypher) |

### Pregi

- **Personalizzazione immediata**: L'AI ricorda il nome dell'utente, le sue
  preferenze e il contesto delle conversazioni precedenti
- **Architettura a livelli**: I fatti importanti vengono promossi da L0 a L3
  con decay esponenziale dell'importanza — i ricordi irrilevanti svaniscono
  naturalmente
- **Deduplicazione L3**: Threshold di similarita' al 95% impedisce fatti
  duplicati
- **Identity leak prevention**: Blocca fatti identitari senza `contact_id`,
  impedendo cross-contamination tra utenti
- **GDPR compliant**: Export Art. 20, soft/hard delete Art. 17, audit log
- **Multi-tenant isolation**: `tenant_id` obbligatorio in ogni operazione,
  fail-closed se mancante
- **Bassa latenza**: L0/L1 in Redis, ricerca L3 via indice HNSW in ~5ms
- **Funziona OGGI**: Integrato end-to-end nel QUICK FIX path, testato in
  produzione

### Limitazioni attuali

- **Estrazione primitiva**: Solo regex (`mi chiamo X`), nessun LLM-based
  fact extraction — cattura solo nomi, non preferenze complesse
- **Solo L3 usato dal QUICK FIX path**: L0/L1/L2/L4 esistono nel servizio ma
  `customer_router.py` interroga solo L3 semantic search
- **Router non registrati**: `delta.py`, `profile.py`, `context.py` definiti
  in lexe-memory ma non esposti in `main.py`
- **RLS policies mancanti**: Isolation applicativa, non a livello PostgreSQL

---

## 2. Memoria SEMANTICA — RAG v3 (Coming Soon)

### Cos'e'

Un sistema di **retrieval conversazionale** che indicizza i messaggi della
conversazione corrente e li recupera con ricerca ibrida (dense + sparse +
Reciprocal Rank Fusion). Progettato per dare al LLM un contesto piu' ricco
della semplice finestra di messaggi recenti.

### Come funzionera' (end-to-end)

```
Utente invia messaggio (use_rag_v3_retrieval=true)
    |
    v
[INDEX] Messaggi della conversazione indicizzati in rag.events
    |  Chunking: 1 messaggio = 1 chunk con contesto (speaker + frasi adiacenti)
    |  Embedding: qwen3-embedding (1536 dim) via LiteLLM
    |  Importanza: scoring euristico (identity +0.35, preference +0.25, ...)
    |
    v
[RETRIEVE] Ricerca ibrida sulla conversazione
    |  Dense: cosine similarity pgvector (top 100)
    |  Sparse: BM25 full-text search PostgreSQL (top 100)
    |  Fusion: RRF (k=60) + importance boost + address match boost
    |  Optional: query rewriting (LLM) + cross-encoder reranking
    |
    v
Contesto formattato per injection nel prompt LLM
    |  max_context_tokens = 4000
    |  max_chunks = 20
    |
    v
LLM genera risposta con contesto conversazionale arricchito
```

### Componenti implementati in lexe-memory

| Componente | File | LOC | Stato |
|------------|------|-----|-------|
| API Router | `rag/api/router.py` | 523 | Implementato, feature-flagged |
| Ricerca ibrida | `rag/search.py` | 518 | Implementato (RRF fusion) |
| Repository DB | `rag/repository.py` | 910 | Implementato con RLS |
| Chunking | `rag/chunking.py` | 486 | Implementato (structure-aware) |
| Embeddings | `rag/embeddings.py` | 361 | Implementato (LiteLLM batch) |
| Query rewriting | `rag/rewriter.py` | 421 | P1, gated |
| Cross-encoder reranking | `rag/reranker.py` | 369 | P1, gated |
| Importance scoring | `rag/importance.py` | 350 | Euristico + LLM (P1) |
| Conversation summary | `rag/summary.py` | 442 | P1, gated |
| Schemas Pydantic | `rag/schemas.py` | 258 | Completo |

### Pregi (rispetto a una semplice finestra di messaggi)

- **Ricerca ibrida RRF**: Combina semantica vettoriale (cattura significato)
  e lessicale BM25 (cattura termini esatti, sigle, numeri articoli) — supera
  entrambe le singole strategie
- **Chunking contestuale**: Ogni chunk include speaker + frase precedente +
  successiva (ispirato al paper Anthropic Contextual Retrieval) — il LLM vede
  il contesto anche di messaggi lontani nel thread
- **Importance scoring**: Messaggi con fatti identitari o decisioni ricevono
  boost, domande effimere vengono penalizzate — il retrieval privilegia cio'
  che conta
- **Query rewriting (P1)**: Risolve pronomi e riferimenti anaforici prima
  della ricerca — "cosa dicevi prima?" diventa una query concreta
- **Cross-encoder reranking (P1)**: Modello `mmarco-mMiniLMv2` multilingue
  ri-ordina i risultati con un secondo passaggio di precision — utile quando
  la query e' ambigua
- **Multi-tenant RLS**: Ogni operazione e' scoped a `(tenant_id,
  conversation_id)` con RLS PostgreSQL — isolation a livello database, non
  solo applicativo
- **Feature flags granulari**: 6 flag indipendenti per abilitare/disabilitare
  ogni componente in produzione senza deploy

### Perche' non e' ancora attivo

- **Router non registrato**: Il RAG router non e' montato in `main.py` di
  lexe-memory
- **Nessuna integrazione in customer_router.py**: Il QUICK FIX path non chiama
  gli endpoint RAG v3
- **Migrazione dati mancante**: Le conversazioni esistenti non sono state
  indicizzate nello schema `rag.*`
- **Performance non validata**: Nessun benchmark su conversazioni lunghe

---

## Confronto diretto

| Aspetto | CONVERSAZIONALE | SEMANTICA (RAG v3) |
|---------|-----------------|-------------------|
| **Domanda a cui risponde** | "Cosa so di questo utente?" | "Cosa e' stato detto in questa conversazione?" |
| **Scope** | Cross-conversazione (permanente) | Intra-conversazione (per thread) |
| **Dati memorizzati** | Fatti semantici (nome, preferenze) | Messaggi completi (user + assistant) |
| **Storage** | `memory.semantic_vectors` | `rag.chunks` + `rag.embeddings` |
| **Database** | lexe-postgres:5435, schema `memory` | lexe-postgres:5435, schema `rag` |
| **Embedding model** | text-embedding-3-small (1536d) | qwen3-embedding (1536d) |
| **Metodo di ricerca** | Dense only (HNSW cosine) | Ibrido RRF (dense + sparse BM25) |
| **Reranking** | No | Si (cross-encoder, P1) |
| **Query rewriting** | No | Si (rule-based + LLM, P1) |
| **Retention** | Permanente (L3) | Configurabile (default 180 giorni) |
| **Granularita'** | Fatto singolo ("si chiama Mario") | Messaggio + contesto adiacente |
| **Isolamento** | `tenant_id` + `contact_id` (applicativo) | `tenant_id` + `conversation_id` (RLS) |
| **Injection nel prompt** | `## Memoria dell'utente` | Contesto conversazionale formattato |
| **Latenza** | ~5ms (HNSW index) | ~20-50ms (hybrid + rerank) |
| **Stato** | Attivo in produzione | Implementato, non integrato |

---

## Quando usare quale

### Memoria CONVERSAZIONALE (switch ON di default)

L'utente la vuole attiva quando:
- Vuole che l'AI ricordi il suo nome e le sue preferenze tra sessioni diverse
- Vuole personalizzazione persistente ("ricordati che preferisco risposte brevi")
- Vuole che informazioni personali influenzino le risposte future

L'utente la disattiva quando:
- Sta facendo una ricerca anonima/impersonale
- Non vuole che fatti vengano estratti e memorizzati
- Sta testando il sistema senza contesto pregresso

### Memoria SEMANTICA / RAG v3 (switch OFF, coming soon)

L'utente la vorra' attiva quando:
- La conversazione e' lunga e il contesto delle ultime N messages non basta
- Ha discusso un articolo specifico 50 messaggi fa e vuole richiamarlo
- Vuole che l'AI trovi connessioni tra parti distanti della conversazione

L'utente la terra' disattivata quando:
- La conversazione e' breve (< 20 messaggi)
- Vuole risposte piu' veloci (il retrieval ibrido aggiunge latenza)
- Il contesto dei 10 messaggi recenti e' sufficiente

---

## Architettura dei servizi

```
                   lexe-webchat (frontend)
                         |
                    [toggle switch]
                    memory  ragV3
                      |       |
                      v       v
              use_memory_retrieval
              use_rag_v3_retrieval
                         |
                         v
                   lexe-core:8000
                 customer_router.py
                    /          \
                   v            v
         lexe-memory:8103    (futuro)
         POST /memory/search  POST /rag/retrieve
         POST /memory/store   POST /rag/index
              |                     |
              v                     v
          PostgreSQL            PostgreSQL
       memory.semantic_vectors  rag.chunks
       (pgvector HNSW)         rag.embeddings
              |                (pgvector + BM25)
              v
           Valkey
        L0/L1 cache
```

---

## File principali

### Memoria CONVERSAZIONALE

| File | Descrizione |
|------|-------------|
| `lexe-core/src/lexe_core/gateway/customer_router.py` | Orchestrazione retrieve + store nel QUICK FIX path |
| `lexe-memory/src/lexe_memory/service.py` | MemoryService: routing L0-L4 |
| `lexe-memory/src/lexe_memory/layers/l3_semantic.py` | L3: pgvector search + dedup |
| `lexe-memory/src/lexe_memory/api/routes.py` | API `/memory/store`, `/memory/search` |
| `lexe-memory/src/lexe_memory/models/memory.py` | Contratti V2.1, scope, retention |

### Memoria SEMANTICA (RAG v3)

| File | Descrizione |
|------|-------------|
| `lexe-memory/src/lexe_memory/rag/api/router.py` | API `/rag/index`, `/rag/retrieve`, `/rag/context` |
| `lexe-memory/src/lexe_memory/rag/search.py` | Orchestrazione ricerca ibrida RRF |
| `lexe-memory/src/lexe_memory/rag/chunking.py` | Chunking structure-aware + importance |
| `lexe-memory/src/lexe_memory/rag/repository.py` | Database layer con RLS |
| `lexe-memory/src/lexe_memory/rag/rewriter.py` | Query rewriting (P1) |
| `lexe-memory/src/lexe_memory/rag/reranker.py` | Cross-encoder reranking (P1) |

### Frontend

| File | Descrizione |
|------|-------------|
| `lexe-webchat/src/components/ui/RetrievalToggleBar.tsx` | I due switch con "Coming Soon" |
| `lexe-webchat/src/stores/configStore.ts` | Persistenza toggle in localStorage |
| `lexe-webchat/src/hooks/useStreaming.ts` | Invio toggle nel body della POST |

---

## Roadmap integrazione RAG v3

1. Registrare il RAG router in `lexe-memory/main.py`
2. Aggiungere indexing automatico in `customer_router.py` (ogni messaggio)
3. Aggiungere retrieval condizionato a `use_rag_v3_retrieval` nel QUICK FIX path
4. Migrare conversazioni esistenti nello schema `rag.*`
5. Benchmark su conversazioni con 100+ messaggi
6. Rimuovere flag `comingSoon` dal toggle frontend
