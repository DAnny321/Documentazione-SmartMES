# AGIC Smart MES – Fase 1 AI
## Funzionalità: Suggeritore Tempi Ciclo con Copilot
### Manuale Funzionale Dettagliato

**Versione:** 1.0  
**Data:** 2026-07-01  
**Applicazione:** AGIC Smart MES – BC 27+  
**Destinatari:** Responsabili di produzione, utenti master, amministratori di sistema

---

## 1. Introduzione e Obiettivo

La funzionalità **"Suggerisci tempi con Copilot"** integra Microsoft Copilot (powered by Azure OpenAI GPT-4.1) direttamente nelle pagine di gestione cicli di Business Central, per supportare chi compila e manutiene i cicli di lavorazione nella determinazione dei tempi ottimali.

### 1.1 Problema risolto

Nella pratica produttiva, i tempi di un ciclo di lavorazione (attrezzaggio, lavorazione, attesa, spostamento) vengono tradizionalmente fissati sulla base dell'esperienza dei pianificatori. Questo approccio:

- È soggettivo e difficile da standardizzare
- Non sfrutta i dati effettivi raccolti durante la produzione
- Può generare cicli non realistici, con impatto sulla pianificazione MRP

### 1.2 Soluzione implementata

Copilot analizza il **Registro Attività MES** (`ATC_MES_ActivityLog`) – che raccoglie i tempi effettivi di attrezzaggio e lavorazione di ogni operazione eseguita dagli operatori – e, combinandolo con il contesto (area di produzione, articolo, operazione), genera suggerimenti motivati per i quattro campi tempo della riga ciclo.

---

## 2. Architettura degli oggetti AL

### 2.1 Struttura cartelle

```
src/
  ai/
    enum/
      Enum85275.ATCMESCopilotCapability.al    ← Registrazione capability Copilot
    codeunit/
      Cod85275.ATCMESCopilotSetup.al          ← Install: registra la capability al deploy
      Cod85276.ATCMESRoutingTimeSuggester.al  ← Logica core: storico + AI + parse risposta
      Cod85277.ATCMESSuggestTimesFunc.al      ← Implementa interfaccia "AOAI Function" (schema JSON)
    page/
      Pag85275.ATCMESRoutingTimeProposal.al   ← PromptDialog: interfaccia utente Copilot
      Pag85278.ATCMESAIKeySetup.al            ← Dialog inserimento chiave API (campo mascherato)
  pageextension/
    Pag-Ext85252.ATCRoutingLinesMES.al        ← [MODIFICATO] Action "Suggerisci tempi" su cicli master
    Pag-Ext85253.ATCProdOrderRoutingMES.al   ← [MODIFICATO] Action "Suggerisci tempi" su cicli ordine
table/
  Tab85250.ATCMESGenSetup.al                 ← [MODIFICATO] Aggiunti campi AI (UseCustomAOAI, Endpoint, Deployment)
page/
  Pag85250.ATCMESGenSetup.al                 ← [MODIFICATO] Aggiunta sezione "AI Copilot Configuration"
```

### 2.2 Oggetti e relative responsabilità

| ID | Tipo | Nome | Responsabilità |
|---|---|---|---|
| 85275 | `enumextension` | `ATC_MES_CopilotCapability` | Estende l'enum `Copilot Capability` con il valore `ATC MES Routing Time Suggester`. Rende la capability visibile nella pagina amministrativa **Copilot e funzionalità agente**. |
| 85275 | `codeunit` (Install) | `ATC_MES_CopilotSetup` | Si esegue all'installazione dell'app. Registra la capability con `BillingType = Custom Billed` solo in ambienti SaaS. |
| 85276 | `codeunit` | `ATC_MES_RoutingTimeSuggester` | Logica core AI: interroga `ATC_MES_ActivityLog`, costruisce il prompt, chiama Azure OpenAI via Function Calling, parsifica la risposta JSON, gestisce errori. Espone le procedure pubbliche `TestConnection()`, `SetCustomAPIKey()`, `IsCustomAPIKeyConfigured()`, `SetDevCredentials()`. |
| 85277 | `codeunit` | `ATC_MES_SuggestTimesFunc` | Implementa l'interfaccia BC `"AOAI Function"`. Fornisce il nome e lo schema JSON completo (parametri e tipi) della funzione `suggest_routing_times` che il modello AI deve richiamare. |
| 85275 | `page` (PromptDialog) | `ATC_MES_RoutingTimeProposal` | Interfaccia Copilot con area Prompt (contesto operazione + dati storici) e area Content (tempi suggeriti modificabili + feedback AI). Supporta cicli master (`SetMasterRoutingContext`) e cicli ordine di produzione (`SetProdOrderRoutingContext`). |
| 85278 | `page` (StandardDialog) | `ATC_MES_AIKeySetup` | Titolo: **AI Copilot – API Key Setup**. Dialog per inserire o aggiornare la chiave API Azure OpenAI. Il campo è mascherato (`ExtendedDatatype = Masked`); al salvataggio la chiave viene cifrata in `IsolatedStorage`. |

---

## 3. Dove appare la funzionalità

### 3.1 Pagina Righe Ciclo Master

**Percorso:** Produzione → Impostazione → Cicli → (apri un ciclo) → Righe  
**Nome pagina BC:** *Routing Lines*

Un pulsante **"Suggerisci tempi con Copilot"** con icona sparkle ( ✦ ) appare nella barra azioni della pagina, nel gruppo "Copilot". Il pulsante è visibile solo quando il modulo Smart MES è attivo (`ActiveSmartMES = true` in Setup generale MES).

### 3.2 Pagina Righe Ciclo Ordine di Produzione

**Percorso:** Ordine di Produzione Rilasciato → (apri ordine) → Ciclo  
**Nome pagina BC:** *Prod. Order Routing*

Stessa logica: il pulsante appare nella barra azioni del ciclo dell'ordine di produzione rilasciato. In questo caso Copilot aggiorna i tempi della riga ciclo specifica dell'ordine (non il ciclo master), consentendo aggiustamenti one-time per un ordine specifico.

---

## 4. Flusso d'uso – passo per passo

### 4.1 Prerequisiti

Verificare che siano soddisfatte le seguenti condizioni prima dell'uso:

1. **Modulo Smart MES attivo:** Nella pagina *Setup generale Smart MES*, il campo "Attiva modulo" deve essere spuntato.
2. **Copilot abilitato nel tenant:** L'amministratore BC deve aver abilitato la capability "MES Routing Time Suggester" nella pagina **Copilot e funzionalità agente** (Impostazioni → Copilot e funzionalità agente). La capability è abilitata per default dopo l'installazione dell'app.
3. **Dati nel Registro MES (consigliato ma non obbligatorio):** Più operazioni di produzione sono state eseguite e registrate tramite l'app MES, migliore sarà la qualità del suggerimento. Il sistema funziona anche senza dati storici, usando standard industriali, ma la confidenza sarà "Bassa".

### 4.2 Apertura del suggeritore (da Righe Ciclo Master)

1. Aprire un ciclo di lavorazione: **Produzione → Impostazione → Cicli** → scegliere un ciclo → aprire il record.
2. Nella schermata del ciclo, scorrere alle righe.
3. Selezionare (cliccare) la riga ciclo per cui si vogliono suggerire i tempi.
4. Nella barra azioni in alto, cercare il gruppo **Copilot** e cliccare **"Suggerisci tempi con Copilot"**.

> ⚠️ **Nota:** Selezionare prima la riga ciclo corretta prima di cliccare il pulsante. Il suggeritore opera sulla riga attualmente selezionata.

### 4.3 Schermata Prompt (Contesto)

Dopo aver cliccato il pulsante, si apre la finestra Copilot in **Prompt mode** (fase di revisione contesto). La schermata mostra:

#### Sezione "Contesto operazione" (sola lettura)

| Campo | Contenuto |
|---|---|
| **Area produzione** | Codice e nome del Work Center della riga ciclo selezionata |
| **Centro di lavoro** | Codice e nome del Machine Center (visibile solo se presente) |
| **Nr. operazione** | Il codice operazione della riga ciclo (es. "010", "020") |
| **Articolo** | Codice e descrizione del primo articolo che utilizza questo ciclo |

#### Sezione "Dati storici rilevati (Registro MES)" (sola lettura)

Mostra il riepilogo dei tempi effettivi già rilevati:

```
Attrezzaggio: 15 esecuzioni, media 22.50 min.
Lavorazione: 12 esecuzioni, media 38.75 min.
```

oppure, in assenza di dati:

```
Attrezzaggio: nessun dato storico disponibile.
Lavorazione: nessun dato storico disponibile.
```

Questi dati vengono inviati al modello AI come contesto. Verificare che corrispondano all'operazione selezionata prima di procedere.

#### Azione: pulsante "Genera"

Cliccare **"Genera"** per inviare il contesto a Copilot e ricevere i tempi suggeriti. Breve attesa durante la generazione (indicatore di caricamento automatico BC).

### 4.4 Schermata Content (Proposta Copilot)

Dopo la generazione, la finestra passa in **Content mode** mostrando:

#### Sezione "Tempi suggeriti (minuti)"

| Campo | Significato | Modificabile |
|---|---|---|
| **Tempo attrezzaggio** | Tempo di setup suggerito (minuti) | ✅ Sì |
| **Tempo lavorazione (per pezzo)** | Tempo di run per unità prodotta (minuti) | ✅ Sì |
| **Tempo attesa** | Tempo di attesa (es. asciugatura, raffreddamento) – `0` se non applicabile | ✅ Sì |
| **Tempo spostamento** | Tempo di trasferimento tra reparti – `0` se non applicabile | ✅ Sì |

> ✏️ **Tutti i campi sono modificabili.** Il pianificatore può aggiustare i valori suggeriti da Copilot prima di applicarli. Questo è il comportamento inteso: Copilot è un assistente, la decisione finale spetta all'utente.

#### Sezione "Feedback Copilot"

| Campo | Significato |
|---|---|
| **Confidenza** | `Alta` = dati storici abbondanti; `Media` = dati parziali; `Bassa` = stima industriale, dati insufficienti |
| **Motivazione** | Spiegazione in italiano del ragionamento usato da Copilot |

Il campo Confidenza è colorato:
- 🟢 **Alta** → sfondo verde (Favorable)
- 🟡 **Media** → sfondo ambra (Ambiguous)
- 🔴 **Bassa** → sfondo arancione (Attention)

#### Nota unità di misura

Nella parte inferiore appare una nota:

> *"Nota: i valori sono espressi in minuti. Verificare che l'unità di misura della riga ciclo sia impostata di conseguenza prima di applicare."*

Questo è importante: i tempi sulle righe ciclo BC sono associati a un'unità di misura capacità (minuti, ore, ecc.). Se la riga ciclo usa ore, occorre convertire i valori prima o aggiornare l'unità di misura.

### 4.5 Applicare il suggerimento

Dopo aver verificato e/o modificato i valori:

- **"Applica"** → scrive i valori suggeriti nei campi `Setup Time`, `Run Time`, `Wait Time`, `Move Time` della riga ciclo selezionata. La finestra si chiude e la pagina Righe Ciclo si aggiorna automaticamente.
- **"Rigenera"** → invia nuovamente la richiesta a Copilot per ottenere un secondo suggerimento.
- **"Annulla"** → chiude la finestra senza modificare nulla.

### 4.6 Verifica del risultato

Dopo aver applicato, la riga ciclo nel BC mostrerà i campi `Setup Time`, `Run Time`, `Wait Time`, `Move Time` aggiornati con i valori approvati.

---

## 5. Come funziona internamente (per il team tecnico)

### 5.1 Dati estratti dal Registro MES

Il codeunit `ATC_MES_RoutingTimeSuggester` (85276) interroga la tabella `ATC_MES_ActivityLog` (85254) con i seguenti filtri:

| Condizione | Logica |
|---|---|
| Se `MachineCenterNo` valorizzato | Filtra per `MachineCenterNo` |
| Se solo `WorkCenterNo` valorizzato | Filtra per `WorkCenterNo` |
| `OperationNo` valorizzato | Aggiunge filtro su `OperationNo` |

Vengono aggregati i campi:
- `PostedSetupTime` (ms) → conta e somma per calcolare media, min, max
- `PostedRunTime` (ms) → conta e somma per calcolare media, min, max

I tempi vengono convertiti in **minuti** (divisione per 60.000).

> ⚠️ Vengono considerati solo i movimenti con `PostedSetupTime > 0` (rispettivamente `PostedRunTime > 0`). I movimenti non ancora registrati o con tempo zero vengono esclusi.

### 5.2 Struttura del prompt inviato all'AI

Il prompt inviato ha la seguente struttura (esempio reale):

```
RICHIESTA: Suggerisci i tempi di ciclo per la seguente operazione di produzione.

CONTESTO OPERAZIONE:
- Articolo: ART-001 - Flangia in acciaio inox
- Categoria articolo: METALLI
- Area produzione (Work Center): WC-TORN – Torneria
- Centro di lavoro (Machine Center): MC-CNC5 – CNC 5 Assi
- Nr. operazione: 010

DATI STORICI (Registro Attività MES - tempi effettivi rilevati):
- Attrezzaggio: 18 esecuzioni, media 24.50 min, min 12.00 min, max 42.00 min
- Lavorazione: 14 esecuzioni, media 55.30 min, min 38.00 min, max 78.00 min

Suggerisci i tempi appropriati motivando la risposta in italiano.
```

### 5.3 Metaprompt (system message)

Il modello riceve come primary system message:

```
Sei un assistente AI specializzato nella pianificazione della produzione manifatturiera.
Analizzi dati storici MES per suggerire tempi di ciclo ottimali.
Devi SEMPRE rispondere chiamando la funzione suggest_routing_times con valori numerici precisi in minuti.
Se i dati storici sono insufficienti (meno di 3 osservazioni), applica standard industriali e imposta confidenza "Bassa".
Il campo wait_time_minutes serve per processi che richiedono attesa fisica: asciugatura, raffreddamento, indurimento, polimerizzazione. Altrimenti usa 0.
Il campo move_time_minutes serve quando la merce deve essere fisicamente spostata tra reparti. Altrimenti usa 0.
I tempi di lavorazione (run_time) si riferiscono al tempo per produrre 1 pezzo.
Non inventare dati: basa i suggerimenti sui dati storici forniti o su standard industriali.
Scrivi la rationale in italiano, in modo conciso (massimo 200 caratteri).
```

### 5.4 Function Calling (output strutturato)

La chiamata AI usa la tecnica **Function Calling** di Azure OpenAI, che garantisce una risposta JSON strutturata e parsificabile. La funzione definita è `suggest_routing_times` con i seguenti parametri obbligatori:

| Parametro | Tipo | Descrizione |
|---|---|---|
| `setup_time_minutes` | number | Tempo attrezzaggio in minuti |
| `run_time_minutes` | number | Tempo lavorazione per pezzo in minuti |
| `wait_time_minutes` | number | Tempo attesa in minuti (0 se non applicabile) |
| `move_time_minutes` | number | Tempo spostamento in minuti (0 se non applicabile) |
| `confidence_level` | string (enum) | "Alta", "Media" o "Bassa" |
| `rationale` | string | Motivazione in italiano (max 200 caratteri) |

Esempio di risposta JSON raw ricevuta da OpenAI:

```json
{
  "tool_calls": [
    {
      "type": "function",
      "function": {
        "name": "suggest_routing_times",
        "arguments": "{\"setup_time_minutes\":22,\"run_time_minutes\":52,\"wait_time_minutes\":0,\"move_time_minutes\":5,\"confidence_level\":\"Alta\",\"rationale\":\"Basato su 18 esecuzioni storiche. Setup medio 24.5 min (outlier alto rimosso). Run stabilmente intorno a 52-55 min.\"}"
      }
    }
  ]
}
```

### 5.5 Modello AI, connessione e fatturazione

L'app utilizza esclusivamente il modello **Custom Billed**: le chiamate API passano **sempre** attraverso la subscription Azure OpenAI del partner/cliente (BYO AOAI). L'uso delle risorse gestite da Microsoft (`SetManagedResourceAuthorization`) è intenzionalmente **escluso** dal codice, poiché non compatibile con il tipo di fatturazione Custom Billed (vincolo MS: *Partner | Custom Billed | Business Central AI resources → Not Allowed*).

| Scenario | Connessione | Autenticazione | Fatturazione |
|---|---|---|---|
| **Produzione SaaS** (`ATC_AI_UseCustomAOAI = true`) | Endpoint Azure OpenAI AGIC/cliente | `SetAuthorization` con credenziali da GenSetup + IsolatedStorage | A carico del partner (subscription Azure propria) |
| **Sandbox / Sviluppo** (`ATC_AI_UseCustomAOAI = false`) | Endpoint Azure OpenAI del developer | `SetAuthorization` con credenziali da IsolatedStorage (`SetDevCredentials`) | A carico del developer |

> ⚠️ **Non esiste un percorso "Microsoft Billed"** in questa app. Per ogni ambiente, occorre configurare le credenziali Azure OpenAI prima di usare il suggeritore.

---

## 6. Configurazione iniziale e prerequisiti amministrativi

> ⚠️ **La configurazione Azure OpenAI è obbligatoria in ogni ambiente** (produzione e sandbox). L'app usa sempre il proprio servizio Azure OpenAI (Custom Billed).

---

### 6.1 Passo 1 – Attivazione modulo Smart MES

**Pagina BC:** `Smart MES General Setup`  
**Come trovarla:** barra di ricerca BC (lente o ALT+Q) → digitare **Smart MES General Setup**

| Campo | Azione |
|---|---|
| **Active Smart MES** | Spuntare la checkbox per attivare il modulo |

---

### 6.2 Passo 2 – Registrazione capability Copilot (solo primo avvio)

**Pagina BC:** `Copilot e funzionalità agente` (o in inglese: `Copilot & agent capabilities`)  
**Come trovarla:** barra di ricerca → digitare **Copilot**

1. Cercare la riga con **Publisher = AGIC** e **Capability = MES Routing Time Suggester**.
2. Verificare che lo stato sia **Attivo**. Se non lo è, cliccarci sopra e abilitarla.

> La capability viene registrata automaticamente all'installazione dell'app. Se non appare, reinstallare l'extension.

---

### 6.3 Passo 3 – Configurazione servizio Azure OpenAI

**Pagina BC:** `Smart MES General Setup`  
Sezione: **AI Copilot Configuration** (visibile nella parte inferiore della pagina)

#### Campi da compilare

| Campo BC | Valore da inserire | Dove trovarlo |
|---|---|---|
| **Use Custom Azure OpenAI** | Spuntare | — |
| **Azure OpenAI Endpoint** | `https://[nome-risorsa].openai.azure.com/` | Portale Azure → Risorsa Azure OpenAI → **Chiavi ed endpoint** |
| **Azure OpenAI Deployment** | Nome del deployment, es. `gpt-4o` | Azure AI Studio (ai.azure.com) → Risorsa → **Deployments** |

#### Impostazione chiave API

4. Cliccare l'action **"AI Copilot – API Key Setup"** nella barra azioni (gruppo *Processing*).
5. Si apre la finestra **AI Copilot – API Key Setup**:
   - Il gruppo **Current Status** mostra se una chiave è già configurata (✓ verde / ✗ arancione).
   - Nel gruppo **New API Key**, incollare la chiave nel campo **API Key** (il testo è mascherato mentre si digita).
   - Cliccare **OK** → la chiave viene salvata cifrata. Il messaggio “API Key saved successfully” conferma il salvataggio.
6. Il campo **API Key Status** nella pagina principale ora mostra ✓ verde.

> 🔐 **Dove trovare la chiave API:** Portale Azure (portal.azure.com) → Risorsa Azure OpenAI → sezione **Chiavi ed endpoint** → copiare **KEY 1** o **KEY 2**.

#### Verifica connessione

7. Cliccare l'action **"Test Azure OpenAI Connection"** (gruppo *Processing*).
8. BC esegue una chiamata di test minimale (5 token). Appare un messaggio:
   - **Successo:** "Connection to Azure OpenAI successful! ..." → la configurazione è corretta.
   - **Errore specifico:** un messaggio indica il codice e la causa (es. chiave errata, deployment non trovato).

---

### 6.4 Passo 4 (opzionale) – Configurazione sandbox / sviluppo

In ambiente sandbox, la configurazione tramite pagina (Passo 3) funziona esattamente come in produzione. In alternativa, è possibile impostare le credenziali via codice:

```al
// Eseguire UNA SOLA VOLTA in sandbox da debug console o test codeunit
var
    Suggester: Codeunit "ATC_MES_RoutingTimeSuggester";
    ApiKey: SecretText;
begin
    ApiKey := 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx';  // KEY 1 o KEY 2 da Azure Portal
    Suggester.SetDevCredentials(
        'https://[nome-risorsa].openai.azure.com/',  // Endpoint
        'gpt-4o',                                     // Nome deployment
        ApiKey);
end;
```

> 🔐 Non committare mai le credenziali nel repository. Usare `SetDevCredentials()` solo via console o test codeunit temporaneo.

---

### 6.5 Riepilogo rapido – nomi esatti delle pagine BC

| Nome pagina in BC | Cosa si fa |
|---|---|
| **Smart MES General Setup** | Attivare il modulo MES, configurare endpoint/deployment, aprire il dialog chiave API, testare la connessione |
| **AI Copilot – API Key Setup** | Dialog (non navigabile direttamente) aperto dall'action nella pagina precedente. Inserisce e salva la chiave API cifrata |
| **Copilot e funzionalità agente** | Verificare che la capability "MES Routing Time Suggester" sia attiva |
| **Routing Lines** | Pagina righe ciclo master: contiene l'action **"Suggerisci tempi con Copilot"** |
| **Prod. Order Routing** | Pagina ciclo ordine di produzione: contiene la stessa action per suggerire tempi su un ordine specifico |

---

## 7. Limitazioni note (Fase 1)

| Limitazione | Spiegazione |
|---|---|
| **Tempi in minuti fissi** | Copilot suggerisce sempre in minuti. Se la riga ciclo usa un'unità di misura diversa (ore, secondi), l'utente deve convertire i valori manualmente o aggiornare l'unità di misura prima di applicare. |
| **No embeddings semantici** | La ricerca di "articoli simili" si basa su WorkCenter + OperationNo nello storico (query AL diretta). Gli embeddings semantici richiederebbero una subscription Azure separata con modello embedding — non incluso in Fase 1. Se l'operazione è nuova, Copilot usa standard industriali (confidenza Bassa). |
| **Nessuna validazione MRP** | Applicare i tempi suggeriti aggiorna la riga ciclo senza ricalcolare automaticamente la pianificazione. Eseguire manualmente il ricalcolo se necessario. |
| **Singolo scarto codice** | La Fase 1 non analizza i codici scarto (trattati nella Fase 2). |
| **Max 500 caratteri per motivazione** | La motivazione Copilot è troncata a 500 caratteri se la risposta del modello è più lunga. |
| **Credenziali obbligatorie** | La funzionalità richiede sempre credenziali Azure OpenAI configurate (endpoint + deployment + API key). Senza configurazione, il pulsante Copilot genera un messaggio di errore guidato. |

---

## 8. Gestione degli errori frequenti

| Codice / Messaggio | Causa | Soluzione |
|---|---|---|
| `401` / `403` – Non autorizzato | Chiave API errata o scaduta | Aprire Setup → "Configure Azure OpenAI API Key" e reinserire la chiave. Verificare che il deployment sia attivo in Azure AI Studio. |
| `404` – Not Found | Endpoint o deployment non trovato | Verificare che il campo "Azure OpenAI Endpoint" sia l'URL completo (`https://[nome].openai.azure.com/`) e che il deployment esista con il nome esatto indicato. |
| `429` – Troppe richieste | Rate limiting: richieste troppo frequenti o quota esaurita | Attendere qualche secondo e cliccare nuovamente "Genera" o "Rigenera". Se persistente, verificare i limiti di quota del deployment in Azure AI Studio. |
| `503` – Servizio non disponibile | Backend Azure temporaneamente occupato | Riprovare dopo qualche minuto. |
| *"Custom Azure OpenAI not fully configured"* | Endpoint, deployment o chiave API mancanti | Completare la configurazione in Setup generale MES → sezione AI Copilot Configuration. Usare l'azione "Test Azure OpenAI Connection" per verificare. |
| Capability non registrata | App installata senza eseguire il trigger Install (raro) | Reinstallare l'extension o eseguire manualmente `ATC_MES_CopilotSetup.RegisterCapabilities()`. |

---

## 9. Domande frequenti (FAQ)

**D: I valori suggeriti vengono applicati automaticamente?**  
R: No. Copilot suggerisce e l'utente deve cliccare esplicitamente "Applica". Prima di applicare, i valori sono editabili nella finestra Copilot.

**D: Cosa succede se la riga ciclo non ha mai avuto esecuzioni nel Registro MES?**  
R: Copilot genera comunque un suggerimento usando standard industriali manifatturieri, con confidenza "Bassa". La motivazione indicherà che non ci sono dati storici.

**D: Posso usare Copilot su più righe ciclo contemporaneamente?**  
R: No, la Fase 1 opera su una riga per volta. Selezionare la riga, aprire Copilot, applicare, poi passare alla riga successiva.

**D: I valori suggeriti da Copilot vengono tracciati o loggati?**  
R: No, nella Fase 1 non esiste un log dei suggerimenti AI. Verrà valutato nelle fasi successive.

**D: Copilot vede i dati di altri clienti?**  
R: No. I dati MES (Registro Attività) sono tenant-isolated in BC SaaS. Azure OpenAI non usa i dati per addestrare i modelli (come da contratto Microsoft Commercial).

**D: Funziona su tablet/mobile?**  
R: Il PromptDialog è ottimizzato per il web client BC. Su tablet in modalità browser (non app mobile) funziona correttamente. Non è previsto l'uso da operatori a bordo macchina (il PromptDialog non è nelle pagine operative MES).

**D: Come viene calcolato il costo AI?**  
R: Il costo dipende dalla subscription Azure OpenAI configurata (Custom Billed). Ogni chiamata al suggeritore consuma indicativamente **800–1200 token** (input + output), inclusi il metaprompt, il contesto storico e la risposta JSON strutturata. Il costo per token dipende dal modello scelto nel deployment (es. GPT-4.1, GPT-4o, ecc.) secondo le tariffe Azure OpenAI correnti. La chiamata di "Test Connessione" consuma circa 20 token.

---

## 10. Esempio completo end-to-end

### Scenario: ciclo verniciatura, prima configurazione

**Contesto:** Un nuovo articolo "Telaio saldato 400x300" viene aggiunto al catalogo. Il responsabile deve compilare un ciclo di lavorazione con fase "Verniciatura a polvere" sul Work Center "WC-VERN – Verniciatura".

**Dati nel Registro MES:** 8 esecuzioni precedenti di verniciatura su WC-VERN, con media attrezzaggio 35 min e media lavorazione 28 min per pezzo. Nessun dato per questo specifico articolo, ma il Work Center ha storico.

1. Aprire il ciclo appena creato → righe ciclo → selezionare la riga "010 – WC-VERN".
2. Cliccare **"Suggerisci tempi con Copilot"**.
3. La finestra mostra:
   - *Area produzione:* WC-VERN – Verniciatura
   - *Nr. operazione:* 010
   - *Articolo:* TELAI-400 – Telaio saldato 400x300
   - *Dati storici:* Attrezzaggio: 8 esecuzioni, media 35.00 min. Lavorazione: 8 esecuzioni, media 28.00 min.
4. Cliccare **"Genera"**.
5. Copilot risponde con:
   - Tempo attrezzaggio: `33`
   - Tempo lavorazione: `28`
   - Tempo attesa: `45` (polimerizzazione vernice a polvere in forno)
   - Tempo spostamento: `5`
   - Confidenza: **Alta**
   - Motivazione: "Basato su 8 esecuzioni storiche WC-VERN. Attesa 45 min per ciclo forno standard verniciatura a polvere."
6. Il responsabile valuta: accetta attrezzaggio e lavorazione, modifica attesa a `60` (forno più lento del modello). Lascia spostamento a `5`.
7. Clicca **"Applica"**.
8. La riga ciclo in BC aggiorna: Setup Time = 33, Run Time = 28, Wait Time = 60, Move Time = 5.

---

*Fine documento – AGIC Smart MES AI Fase 1*
