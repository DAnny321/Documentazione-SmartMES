# AGIC Smart MES – Routing Copilot
## Manuale Funzionale e Tecnico – Versione 2.0

**Prodotto:** AGIC Smart MES – Routing Copilot
**Versione app:** 1.0.0.0
**Piattaforma:** Microsoft Dynamics 365 Business Central 27+ (SaaS)
**Runtime AL:** 16.0
**Data:** 2026-07-02
**Autore:** AGIC
**Destinatari:** Responsabili di produzione · Utenti master · Sviluppatori AL · Amministratori BC

---

## Indice

1. [Introduzione e obiettivi](#1-introduzione-e-obiettivi)
2. [Architettura funzionale](#2-architettura-funzionale)
3. [Architettura tecnica e oggetti AL](#3-architettura-tecnica-e-oggetti-al)
4. [Prerequisiti e configurazione amministrativa](#4-prerequisiti-e-configurazione-amministrativa)
5. [Guida operativa – Utente funzionale](#5-guida-operativa--utente-funzionale)
6. [Come funziona internamente – Tecnico](#6-come-funziona-internamente--tecnico)
7. [Strumenti AI disponibili (Tools)](#7-strumenti-ai-disponibili-tools)
8. [Pagine di impostazione](#8-pagine-di-impostazione)
9. [Gestione errori e troubleshooting](#9-gestione-errori-e-troubleshooting)
10. [Sicurezza e dati](#10-sicurezza-e-dati)
11. [Limitazioni e roadmap](#11-limitazioni-e-roadmap)

---

## 1. Introduzione e obiettivi

### 1.1 Contesto

In un contesto manifatturiero che usa Business Central, la creazione di un ciclo di lavorazione (Routing) richiede competenza tecnica: l'utente deve indicare manualmente le fasi operative, i centri di lavoro, i tempi di attrezzaggio e i tempi di lavorazione. Questi tempi vengono oggi determinati empiricamente, spesso senza consultare i dati storici disponibili nel sistema.

**AGIC Routing Copilot** è un assistente AI conversazionale integrato in Business Central che aiuta i pianificatori a:
- Creare cicli di lavorazione completi tramite dialogo in linguaggio naturale
- Recuperare automaticamente i tempi storici dai Movimenti Contabili di Capacità (tabella 5832)
- Copiare la struttura di cicli esistenti come template per nuovi articoli simili
- Trovare articoli con caratteristiche simili (per categoria, dimensione) per basare i suggerimenti su dati reali

### 1.2 Principi di progettazione

| Principio | Applicazione |
|---|---|
| **Assistente, non automatismo** | Copilot suggerisce e chiede conferma — non crea dati senza approvazione esplicita dell'utente |
| **Dati reali prima degli standard industriali** | Le stime partono dai Movimenti Contabili di Capacità effettivi; solo se insufficienti si usano standard industriali |
| **Sicurezza dei segreti** | La chiave API Azure OpenAI è salvata cifrata in IsolatedStorage, mai esposta in log o messaggi |
| **Resilienza agli errori** | Il pattern `[TryFunction]` garantisce che qualsiasi eccezione appaia come messaggio nel chat, non come crash |
| **Fatturazione Custom Billed** | Il sistema usa sempre la subscription Azure OpenAI del partner/cliente. Non usa risorse gestite da Microsoft |

---

## 2. Architettura funzionale

### 2.1 Dove si trova la funzionalità

La chat **AGIC Routing Copilot** è accessibile da tre punti in Business Central:

| Punto di accesso | Come aprirlo | Contesto passato automaticamente |
|---|---|---|
| **Ricerca globale BC** | ALT+Q → digita "Smart MES – Routing Copilot" | Nessun contesto |
| **Righe Ciclo Master** | Produzione → Impostazione → Cicli → (apri ciclo) → Righe → azione **"Routing Copilot"** | Numero ciclo + versione corrente |
| **Righe Ciclo Ordine di Produzione** | Ordine Produzione Rilasciato → (apri ordine) → Ciclo → azione **"Routing Copilot"** | Nessun contesto specifico |

### 2.2 Flusso conversazionale tipo

```
1. Utente apre il Routing Copilot
   └─ BC carica il chat con header AGIC arancione, welcome message e suggestion chips

2. Utente scrive o clicca un suggerimento
   └─ Es: "Crea un ciclo seriale per articoli categoria LEGNO"

3. Copilot identifica che ha bisogno di dati
   └─ Chiede con quale criterio cercare gli articoli simili

4. Utente risponde: "Usa categoria LEGNO"
   └─ Copilot esegue lo strumento find_items_by_criteria
   └─ Trova articoli categoria LEGNO con cicli esistenti

5. Copilot analizza i movimenti contabili di capacità
   └─ Esegue get_capacity_stats per i centri di lavoro usati da quegli articoli
   └─ Calcola tempi medi effettivi per operazione

6. Copilot presenta la proposta e chiede conferma
   └─ "Ho trovato 3 operazioni comuni. Vuoi che crei il ciclo LEGNO25?"

7. Utente conferma
   └─ Copilot esegue create_routing_header e poi create_routing_line per ogni riga

8. Copilot mostra il riepilogo del ciclo creato
```

---

## 3. Architettura tecnica e oggetti AL

### 3.1 Struttura cartelle

```
AGIC Smart MES/
├── app.json                           ← resourceExposurePolicy: allowDebugging=true (OBBLIGATORIO)
├── src/
│   ├── ai/
│   │   ├── controladdin/
│   │   │   ├── ATC_MES_ChatAddin.al  ← Definizione ControlAddin BC
│   │   │   ├── scripts/
│   │   │   │   └── chat.js           ← UI chatbot (IIFE singola, AGIC branded)
│   │   │   └── styles/
│   │   │       └── chat.css          ← Stile AGIC #E84819
│   │   ├── codeunit/
│   │   │   ├── Cod85275.ATCMESCopilotSetup.al       ← Install: registra capability
│   │   │   ├── Cod85276.ATCMESRoutingTimeSuggester.al ← Auth + credential manager
│   │   │   ├── Cod85280.ATCMESRoutingCopilotMgr.al  ← Orchestratore conversazione
│   │   │   ├── Cod85281.ATCMESFuncFindItems.al      ← Tool: trova articoli
│   │   │   ├── Cod85282.ATCMESFuncGetCapStats.al    ← Tool: tempi storici (tab.5832)
│   │   │   ├── Cod85283.ATCMESFuncGetRoutingTemplate.al ← Tool: legge ciclo esistente
│   │   │   ├── Cod85284.ATCMESFuncCreateRouting.al  ← Tool: crea testata ciclo
│   │   │   └── Cod85285.ATCMESFuncCreateRoutingLine.al ← Tool: crea riga ciclo
│   │   ├── enum/
│   │   │   └── Enum85275.ATCMESCopilotCapability.al ← Estende enum Copilot Capability
│   │   └── page/
│   │       ├── Pag85278.ATCMESAIKeySetup.al         ← Dialog inserimento API key
│   │       └── Pag85279.ATCMESRoutingAssistant.al   ← Pagina chat (Card + ControlAddin)
│   ├── pageextension/
│   │   ├── Pag-Ext85252.ATCRoutingLinesMES.al       ← [MOD] Aggiunta action Routing Copilot
│   │   └── Pag-Ext85253.ATCProdOrderRoutingMES.al   ← [MOD] Aggiunta action Routing Copilot
│   └── table/
│       └── Tab85250.ATCMESGenSetup.al               ← [MOD] Campi AI: endpoint, deployment, flag
```

### 3.2 Oggetti AL — responsabilità

| ID | Tipo | Nome | Responsabilità |
|---|---|---|---|
| 85275 | `enumextension` | `ATC_MES_CopilotCapability` | Aggiunge valore `ATC MES Routing Time Suggester` all'enum `Copilot Capability` di BC |
| 85275 | `codeunit` (Install) | `ATC_MES_CopilotSetup` | Eseguito all'installazione: registra la capability con `BillingType = Custom Billed` |
| 85276 | `codeunit` | `ATC_MES_RoutingTimeSuggester` | Gestione credenziali Azure OpenAI: `ApplyAuthorization`, `SetCustomAPIKey`, `IsCustomAPIKeyConfigured`, `SetDevCredentials`, `TestConnection` |
| 85280 | `codeunit` | `ATC_MES_RoutingCopilotMgr` | Orchestratore del chatbot: mantiene la storia conversazione (AOAI Chat Messages), implementa il loop ReAct, chiama i tool |
| 85281 | `codeunit` | `ATC_MES_FuncFindItems` | Tool AI: interroga `Item` (tab.27) per categoria o dimensione |
| 85282 | `codeunit` | `ATC_MES_FuncGetCapStats` | Tool AI: aggrega `Quantity` da `Capacity Ledger Entry` (tab.5832) per work center + lista articoli |
| 85283 | `codeunit` | `ATC_MES_FuncGetRoutingTemplate` | Tool AI: legge righe ciclo (tab.99000764) di un articolo di riferimento inclusi campi MES |
| 85284 | `codeunit` | `ATC_MES_FuncCreateRouting` | Tool AI: crea `Routing Header` (Serial/Parallel) |
| 85285 | `codeunit` | `ATC_MES_FuncCreateRoutingLine` | Tool AI: crea `Routing Line` con tempi e campi MES extension |
| — | `controladdin` | `ATC_MES_ChatAddin` | ControlAddin BC: definisce procedure AL→JS e eventi JS→AL |
| 85278 | `page` (StandardDialog) | `ATC_MES_AIKeySetup` | Dialog "AI Copilot – API Key Setup": inserisce e salva la chiave API cifrata |
| 85279 | `page` (Card) | `ATC_MES_RoutingAssistant` | Pagina chatbot: ospita il ControlAddin, gestisce `ControlAddInReady` e `OnUserMessage` |

### 3.3 ControlAddin — contratto AL/JS

```al
// ATC_MES_ChatAddin.al
controladdin "ATC_MES_ChatAddin"
{
    Scripts     = 'src/ai/controladdin/scripts/chat.js';
    StyleSheets = 'src/ai/controladdin/styles/chat.css';

    // AL → JS (chiamati da BC verso il browser)
    procedure Initialize(WelcomeHtml: Text);   // Inizializza chat e mostra chips
    procedure AddBotMessage(Html: Text);       // Aggiunge risposta AI formattata
    procedure ResetChat(WelcomeHtml: Text);    // Azzera e reinizializza

    // JS → AL (eventi dal browser verso BC)
    event ControlAddInReady();                 // DOM pronto, JS caricato
    event OnUserMessage(UserText: Text);       // Utente ha inviato un messaggio
}
```

> **CRITICO per il funzionamento**: `app.json` deve avere `"allowDebugging": true` nella `resourceExposurePolicy`. Senza questa impostazione BC non serve i file JS/CSS al browser e la pagina appare completamente bianca.

---

## 4. Prerequisiti e configurazione amministrativa

### 4.1 Requisiti di sistema

| Requisito | Dettaglio |
|---|---|
| Business Central | Versione 27+ (SaaS) |
| Subscription Azure | Account Azure con accesso a Azure OpenAI Service |
| Modello AI | Qualsiasi modello supportato da Azure OpenAI Chat Completions (es. GPT-4o, GPT-4.1) |
| Licenza BC | La funzionalità Copilot deve essere abilitata nel tenant BC |
| Smart MES | Modulo AGIC Smart MES installato e attivato |

### 4.2 Passo 1 — Attivare il modulo Smart MES

**Pagina BC:** `Smart MES General Setup`
**Come trovarla:** Barra di ricerca BC (ALT+Q) → digita **Smart MES General Setup**

Nel gruppo **General**:
- Campo **Active Smart MES** → spuntare

### 4.3 Passo 2 — Verificare la capability Copilot

**Pagina BC:** `Copilot e funzionalità agente`
**Come trovarla:** ALT+Q → digita **Copilot**

Cercare la riga:
- **Funzionalità**: `MES Routing Time Suggester`
- **Editore**: `AGIC`
- **Stato**: deve essere **Attivo**
- **Tipo di fatturazione**: `Fatturazione...` (Custom Billed)

Se la riga non è presente: l'extension non è stata pubblicata con il trigger Install, eseguire un reinstall completo.

### 4.4 Passo 3 — Configurare Azure OpenAI

**Pagina BC:** `Smart MES General Setup` → sezione **AI Copilot Configuration**

| Campo | Descrizione | Esempio |
|---|---|---|
| **Use Custom Azure OpenAI** | Abilita la connessione al proprio servizio Azure | ☑ |
| **Azure OpenAI Endpoint** | URL della risorsa Azure OpenAI | `https://[nome].openai.azure.com/` |
| **Azure OpenAI Deployment** | Nome del deployment del modello | `gpt-4o` |
| **API Key Status** | Indicatore verde (✓) se la chiave è salvata | — |

**Dove trovare queste informazioni in Azure Portal:**
1. Accedere a [portal.azure.com](https://portal.azure.com)
2. Aprire la risorsa Azure OpenAI
3. Menu **Chiavi ed endpoint** → copiare **KEY 1** (o KEY 2) e l'**Endpoint URL**
4. Per il nome deployment: Azure AI Studio → risorsa → **Deployments**

### 4.5 Passo 4 — Salvare la chiave API

Nella pagina **Smart MES General Setup**, cliccare l'action:
**"AI Copilot – API Key Setup"** (gruppo Processing)

Si apre il dialog:
1. **Current Status**: indica se una chiave è già configurata (✓ verde o ✗ arancione)
2. Campo **API Key**: incollare la chiave (il testo è mascherato durante la digitazione)
3. Cliccare **OK** → la chiave viene salvata **cifrata** in `IsolatedStorage`

> La chiave non viene mai salvata in tabelle BC, non appare nei log, non è visibile nel debugger.

### 4.6 Passo 5 — Testare la connessione

Cliccare l'action **"Test Azure OpenAI Connection"** (gruppo Processing).

BC esegue una chiamata di test minimale (5 token). Risultato:
- **Successo**: "Connection to Azure OpenAI successful! Endpoint, deployment and API key are configured correctly."
- **Errore**: messaggio specifico (401 = chiave errata, 404 = endpoint/deployment non trovato, 429 = rate limit)

---

## 5. Guida operativa – Utente funzionale

### 5.1 Apertura del Routing Copilot

**Da Righe Ciclo Master:**
1. Aprire un ciclo: Produzione → Impostazione → Cicli → selezionare il ciclo
2. Nelle righe, cliccare **"Routing Copilot"** nella barra azioni (gruppo Copilot)

**Da ricerca globale:**
1. ALT+Q → digita "Smart MES – Routing Copilot"
2. La chat si apre senza contesto predefinito

### 5.2 Interfaccia chat

All'apertura, la chat mostra:
- **Header arancione AGIC** con logo e titolo "AGIC Routing Copilot"
- **Messaggio di benvenuto** con le 4 aree di intervento
- **5 suggestion chips** cliccabili come punti di partenza rapidi

**Suggestion chips predefinite:**
| Chip | Cosa fa |
|---|---|
| Crea ciclo seriale per categoria LEGNO | Avvia il flusso di creazione ciclo |
| Tempi medi su WC-FRESA per articoli METALLI | Interroga lo storico capacità |
| Struttura ciclo dell'articolo ART-001 | Legge il ciclo di un articolo esistente |
| Quali articoli hanno categoria ACCIAIO? | Cerca articoli per categoria |
| Crea ciclo TEST-01 di tipo Seriale | Crea solo la testata ciclo |

### 5.3 Esempi di richieste e risposte attese

#### Esempio A — Creare un ciclo completo

```
Utente: Crea ciclo seriale per categoria LEGNO

Copilot: Ho capito. Per suggerire le righe del ciclo ho bisogno di
         trovare articoli simili. Ho cercato nella categoria LEGNO e
         trovato 8 articoli. Vuoi che analizzi i tempi storici per
         questi articoli oppure vuoi usare un articolo specifico
         come riferimento?

Utente: Analizza tutti gli articoli categoria LEGNO

Copilot: Ho analizzato i Movimenti Contabili di Capacità per gli
         8 articoli categoria LEGNO. Ho trovato 3 operazioni comuni:
         
         • WC-TAGLIO: 22 esecuzioni, tempo medio 28 min/pezzo
         • WC-LEVIGAT: 18 esecuzioni, tempo medio 15 min/pezzo
         • WC-FINITURA: 22 esecuzioni, tempo medio 12 min/pezzo
         
         Vuoi che crei il ciclo LEGNO-01 seriale con queste 3 righe?

Utente: Sì, crea il ciclo con codice LEGNO25

Copilot: Creo la testata ciclo LEGNO25...
         ✓ Testata LEGNO25 (Seriale) creata.
         Creo le righe:
         ✓ Riga 010 – WC-TAGLIO, setup 5 min, run 28 min
         ✓ Riga 020 – WC-LEVIGAT, setup 3 min, run 15 min  
         ✓ Riga 030 – WC-FINITURA, setup 2 min, run 12 min
         
         Ciclo LEGNO25 creato con 3 righe. Vuoi apportare modifiche?
```

#### Esempio B — Copiare struttura da articolo esistente

```
Utente: Struttura ciclo dell'articolo ART-001

Copilot: L'articolo ART-001 (Flangia Acciaio Inox) ha il ciclo FLANGIA-B
         con 4 operazioni:
         
         010 – WC-TORN: Setup 20 min, Run 45 min, Attesa 0, Spost. 5
         020 – WC-FRES: Setup 15 min, Run 30 min
         030 – WC-RETT: Setup 10 min, Run 20 min
         040 – WC-CTRL: Setup 5 min, Run 8 min
         
         Vuoi creare un nuovo ciclo basato su questa struttura,
         oppure hai bisogno di altre informazioni?
```

### 5.4 Regole di comportamento del Copilot

- **Chiede sempre conferma** prima di creare testata o righe ciclo
- **Usa i dati storici reali** quando disponibili (almeno 3 osservazioni)
- **Una riga alla volta**: crea le righe in sequenza, non in batch
- **Risponde in italiano** con linguaggio professionale manifatturiero
- **Segnala problemi chiaramente**: ciclo già esistente, ciclo certificato non modificabile, centro di lavoro bloccato

### 5.5 Azione "Nuova conversazione"

Il pulsante **"Nuova conversazione"** nel ribbon azzera la storia della chat e ricarica il messaggio di benvenuto con i chip di suggerimento. Utile per iniziare un nuovo progetto di ciclo.

---

## 6. Come funziona internamente – Tecnico

### 6.1 Pattern ReAct (Reasoning + Acting)

Il Routing Copilot usa il pattern **ReAct** (Reasoning and Acting) per orchestrare le chiamate AI e le azioni su BC. A differenza del native OpenAI Function Calling, il ReAct è implementato a livello di prompt, il che lo rende più compatibile con ogni versione di BC.

**Come funziona il ciclo ReAct:**

```
1. Utente invia messaggio
   │
2. AddUserMessage(UserText) → AOAIChatMessages
   │
3. GenerateChatCompletion()
   │
4. Risposta AI: contiene {"action":"TOOL_NAME","params":{...}}?
   ├─ SÌ  → DispatchAction() → esegue il tool → AddUserMessage("[Risultato strumento]: ...")
   │         → torna al punto 3 (max 5 iterazioni)
   └─ NO  → restituisce testo come risposta finale all'utente
```

**Prompt di sistema (metaprompt)** estratto da `GetMetaprompt()` in `Cod85280`:
```
Sei un assistente esperto nella creazione e gestione dei cicli di lavorazione
in Microsoft Dynamics 365 Business Central Manufacturing.

HAI ACCESSO AI SEGUENTI STRUMENTI. Quando hai bisogno di dati o vuoi eseguire
un'azione, rispondi SOLO con un JSON nel formato esatto:
{"action": "TOOL_NAME", "params": {...}}

STRUMENTI DISPONIBILI:
1. find_items_by_criteria  – Params: {"category_code":"LEGNO", ...}
2. get_capacity_stats       – Params: {"work_center_no":"WC-001", "item_nos":[...], ...}
3. get_routing_template     – Params: {"item_no":"ART001"}
4. create_routing_header    – Params: {"routing_no":"LEGNO25", "type":"Serial", ...}
5. create_routing_line      – Params: {"routing_no":"LEGNO25", "operation_no":"010", ...}

REGOLE FONDAMENTALI:
- Usa SEMPRE uno strumento alla volta
- Prima di create_*, chiedi conferma esplicita all'utente
- Rispondi sempre in italiano, in modo conciso e professionale
```

### 6.2 Gestione della conversazione

`AOAIChatMessages` è una variabile **module-level** di `Cod85280`. Poiché `Cod85280` è dichiarato come `var` nella pagina `Pag85279`, la storia della conversazione **persiste per tutta la durata della sessione della pagina**.

```al
// In Pag85279 — var section
var
    CopilotMgr: Codeunit "ATC_MES_RoutingCopilotMgr";  // ← persiste tra messaggi
```

Ogni chiamata a `CopilotMgr.TryProcessMessage()` usa la stessa istanza di `AOAIChatMessages`, che include:
- Il metaprompt (impostato una sola volta al primo messaggio)
- Tutti i messaggi utente e risposta AI precedenti
- I risultati degli strumenti passati come messaggi utente

### 6.3 Ciclo di vita di una sessione

```
OnOpenPage() → SetRoutingContext (se c'è contesto)
               │
ControlAddInReady() → Initialize(BuildWelcomeHtml())
               │       └─ JS mostra header AGIC + welcome + chips
               │
[Utente invia messaggio]
               │
OnUserMessage() → CopilotMgr.TryProcessMessage()
               │   ├─ Se primo messaggio: ApplyAuthorization + SetCapabilityIfRegistered
               │   │                      + SetPrimarySystemMessage(GetMetaprompt())
               │   ├─ AddUserMessage(UserText) → AOAIChatMessages
               │   └─ Loop ReAct (max 5 iterazioni)
               │         ├─ GenerateChatCompletion
               │         ├─ TryParseActionRequest → DispatchAction → tool
               │         └─ Oppure: risposta finale testo
               │
CurrPage.ChatAddin.AddBotMessage(Response)
               └─ JS: escape HTML, converti **bold**, newlines → <br>, visualizza bolla
```

### 6.4 Autorizzazione — Custom Billed

Il sistema usa **esclusivamente** la subscription Azure OpenAI del partner/cliente (Custom Billed). Non usa mai le risorse gestite da Microsoft (`SetManagedResourceAuthorization`).

```al
// Cod85276 — ApplyAuthorization
procedure ApplyAuthorization(var AzureOpenAI: Codeunit "Azure OpenAI"; ...): Boolean
begin
    if MESSetup.ATC_AI_UseCustomAOAI then
        exit(SetCustomAuthorization(AzureOpenAI, MESSetup))  // usa tab.85250 + IsolatedStorage
    else
        exit(SetDevAuthorization(AzureOpenAI));              // usa IsolatedStorage solo (sandbox)
end;
```

**Livelli di credenziali:**

| Scenario | Dove si leggono le credenziali | Metodo |
|---|---|---|
| Produzione (UseCustomAOAI = true) | `Tab85250` (endpoint, deployment) + `IsolatedStorage` (API key cifrata) | `SetCustomAuthorization` |
| Sandbox/sviluppo (UseCustomAOAI = false) | `IsolatedStorage` con chiavi `ATC_MES_AI_*` | `SetDevAuthorization` |

### 6.5 Dati storici — Capacity Ledger Entry (5832)

`Cod85282.ATCMESFuncGetCapStats` interroga la tabella standard BC **5832 "Capacity Ledger Entry"** usando la chiave **Key5 (Type, "No.", "Item No.", "Posting Date")**.

```al
CapLedgEntry.SetRange(Type, CapLedgEntry.Type::"Work Center");
CapLedgEntry.SetRange("No.", WorkCenterNo);
CapLedgEntry.SetFilter("Item No.", ItemNoFilter);       // lista articoli categoria
CapLedgEntry.SetFilter("Posting Date", '>=%1', DateFrom);
CapLedgEntry.SetRange(Reversed, false);                 // esclude movimenti annullati
```

**`Quantity`** = tempo capacità consumato nell'unità definita da `Cap. Unit of Measure Code`. La conversione a minuti:

| Cap. Unit of Measure Code | Conversione |
|---|---|
| `HOURS`, `HOUR`, `H`, `ORA`, `ORE` | × 60 |
| `MIN`, `MINUTES`, `MINUTE`, `M`, `MINUTI` | × 1 (già in minuti) |
| `SEC`, `SECONDS`, `S`, `SECONDI` | ÷ 60 |
| Altro | presupposto minuti |

### 6.6 SetCopilotCapability — gestione condizionale

BC richiede che `AzureOpenAI.SetCopilotCapability()` venga chiamato prima di `GenerateChatCompletion()`. La chiamata è resa condizionale per evitare crash in ambienti dove la capability non è ancora registrata:

```al
local procedure SetCapabilityIfRegistered(var AzureOpenAI: Codeunit "Azure OpenAI")
var
    CopilotCapabilityMgr: Codeunit "Copilot Capability";
begin
    if CopilotCapabilityMgr.IsCapabilityRegistered(
        Enum::"Copilot Capability"::"ATC MES Routing Time Suggester") then
        AzureOpenAI.SetCopilotCapability(
            Enum::"Copilot Capability"::"ATC MES Routing Time Suggester");
end;
```

---

## 7. Strumenti AI disponibili (Tools)

### Tool 1: `find_items_by_criteria` — Cod85281

Interroga `Item` (tab.27) filtrato per categoria articolo o dimensione globale 1.

**Parametri JSON:**
```json
{
  "category_code": "LEGNO",     // Facoltativo: Item Category Code
  "dimension1_value": "",       // Facoltativo: Global Dimension 1 Code
  "item_no": "ART-001"          // Facoltativo: singolo articolo specifico
}
```

**Output JSON:**
```json
{
  "items": [
    {"no": "ART-001", "description": "Pannello MDF", "category": "LEGNO", "routing_no": "LEGNO-A"},
    ...
  ],
  "count": 8
}
```

**Limitazioni:** max 30 articoli restituiti.

---

### Tool 2: `get_capacity_stats` — Cod85282

Aggrega `Quantity` da `Capacity Ledger Entry` (5832) per work center e lista articoli. Restituisce la media dei tempi operativi.

**Parametri JSON:**
```json
{
  "work_center_no": "WC-FRESA",
  "item_nos": ["ART-001", "ART-002", "ART-003"],
  "date_from": "2024-01-01"
}
```

**Output JSON:**
```json
{
  "work_center_no": "WC-FRESA",
  "items_filter": "ART-001|ART-002|ART-003",
  "entry_count": 22,
  "avg_capacity_time_minutes": 28.5,
  "total_capacity_time_minutes": 627.0,
  "note": "avg_capacity_time represents total operation time per entry (setup+run combined from standard BC posting)"
}
```

---

### Tool 3: `get_routing_template` — Cod85283

Legge le righe del ciclo di lavorazione associato a un articolo di riferimento. Include i campi MES custom (messaggi, soglie).

**Parametri JSON:**
```json
{
  "item_no": "ART-001"
}
```

**Output JSON:**
```json
{
  "item_no": "ART-001",
  "item_description": "Pannello MDF",
  "routing_no": "LEGNO-A",
  "routing_description": "Ciclo legno standard",
  "routing_type": "Serial",
  "routing_status": "Certified",
  "lines": [
    {
      "operation_no": "010",
      "description": "Taglio",
      "type": "Work Center",
      "no": "WC-TAGLIO",
      "work_center_no": "WC-TAGLIO",
      "setup_time": 5.0,
      "setup_time_uom": "MINUTI",
      "run_time": 28.0,
      "run_time_uom": "MINUTI",
      "wait_time": 0.0,
      "move_time": 5.0,
      "mes_setup_message": "Verifica posizionamento sega",
      "mes_run_message": "",
      "mes_stop_message": ""
    }
  ],
  "line_count": 4
}
```

---

### Tool 4: `create_routing_header` — Cod85284

Crea una nuova testata ciclo (`Routing Header`). Lo stato iniziale è `New`.

**Parametri JSON:**
```json
{
  "routing_no": "LEGNO25",
  "description": "Ciclo legno 2025",
  "type": "Serial"
}
```

**Output JSON (successo):**
```json
{
  "success": true,
  "routing_no": "LEGNO25",
  "description": "Ciclo legno 2025",
  "type": "Serial",
  "status": "New"
}
```

**Validazioni:**
- Il ciclo non deve già esistere
- `type` accetta: `Serial`, `Parallel`, `SERIALE`, `PARALLELO`

---

### Tool 5: `create_routing_line` — Cod85285

Crea una riga del ciclo di lavorazione (`Routing Line`) con tempi standard e campi MES.

**Parametri JSON:**
```json
{
  "routing_no": "LEGNO25",
  "operation_no": "010",
  "work_center_no": "WC-TAGLIO",
  "type": "Work Center",
  "description": "Taglio",
  "setup_time": 5.0,
  "run_time": 28.0,
  "wait_time": 0.0,
  "move_time": 5.0,
  "mes_setup_message": "Verifica posizionamento",
  "mes_run_message": "",
  "mes_stop_message": ""
}
```

**Output JSON (successo):**
```json
{
  "success": true,
  "routing_no": "LEGNO25",
  "operation_no": "010",
  "work_center_no": "WC-TAGLIO",
  "description": "Taglio",
  "setup_time": 5.0,
  "run_time": 28.0,
  "wait_time": 0.0,
  "move_time": 5.0
}
```

**Validazioni:**
- Il ciclo deve esistere
- Il ciclo non deve essere in stato `Certified` (i cicli certificati non sono modificabili)
- Il centro di lavoro non deve essere bloccato (`Blocked = false`)
- I tempi sono in minuti (unità di misura della riga ciclo)

---

## 8. Pagine di impostazione

### 8.1 Smart MES General Setup — Sezione AI Copilot Configuration

**Pagina:** `Smart MES General Setup` | **ObjectType:** Page 85250

| Campo | Tipo | Descrizione |
|---|---|---|
| **Use Custom Azure OpenAI** | Boolean | Abilita connessione BYO Azure OpenAI |
| **Azure OpenAI Endpoint** | Text[250] | URL risorsa Azure, es. `https://[nome].openai.azure.com/` |
| **Azure OpenAI Deployment** | Text[100] | Nome deployment, es. `gpt-4o` |
| **API Key Status** | Computed | ✓ verde se chiave configurata, ✗ arancione se mancante |

**Action disponibili (gruppo Processing):**
- **"AI Copilot – API Key Setup"**: apre il dialog per inserire/aggiornare la chiave API
- **"Test Azure OpenAI Connection"**: esegue una chiamata test (5 token) e mostra l'esito
- **"Create Job Queue for Time Check"**: crea la coda processi per il controllo soglie MES

### 8.2 AI Copilot – API Key Setup

**Pagina:** `ATC_MES_AIKeySetup` | **ObjectType:** Page 85278 (StandardDialog)

Dialog modale aperto dall'action sopra. Mostra:
- **Current Status**: ✓/✗ con indicatore colore
- **New API Key**: campo mascherato (ExtendedDatatype = Masked)
- **Info**: testo guida dove trovare la chiave in Azure Portal

Al click su **OK**: la chiave viene assegnata a una variabile `SecretText` (isolata da ogni operazione UI) e salvata con `IsolatedStorage.SetEncrypted()`.

> **Nota tecnica sicurezza**: la chiave è isolata in `SaveApiKey()` locale procedure separata per evitare il blocco BC "L'operazione è stata interrotta per impedire l'esposizione di un valore segreto" (causato da `Message()` in scope con `SecretText`).

---

## 9. Gestione errori e troubleshooting

### 9.1 Errori nel chat (visibili all'utente)

Tutti gli errori sono gestiti dal pattern `[TryFunction]` in `Cod85280.TryProcessMessage` e appaiono come bolle nel chat con prefisso `⚠ Errore:`.

| Messaggio nel chat | Causa | Soluzione |
|---|---|---|
| "Impossibile connettersi ad Azure OpenAI. Verifica la configurazione..." | `ApplyAuthorization` restituisce false | Verificare Endpoint, Deployment e API Key in Smart MES General Setup |
| "⚠ Errore: La funzionalità di Copilot non è stata impostata." | `SetCopilotCapability` fallisce | Verificare che la capability "MES Routing Time Suggester" sia **Attiva** nella pagina Copilot e funzionalità agente |
| "Errore di autenticazione Azure OpenAI (401/403)..." | Chiave API non valida o scaduta | Inserire nuova chiave in "AI Copilot – API Key Setup" |
| "Endpoint o deployment Azure OpenAI non trovato (404)..." | URL errato o deployment rimosso | Verificare Endpoint e Deployment in Smart MES General Setup |
| "Troppe richieste ad Azure OpenAI (429)..." | Rate limit raggiunto | Attendere e riprovare; considerare aumento quota in Azure AI Studio |
| "Troppe iterazioni consecutive..." | Loop ReAct superato (max 5) | Riformulare la richiesta in modo più specifico |
| "Routing X is Certified and cannot be modified" | Tentativo di modifica ciclo certificato | Aprire il ciclo, portarlo in stato "New" o "Under Development" prima di procedere |

### 9.2 Pagina completamente bianca

**Causa**: `resourceExposurePolicy.allowDebugging` è `false` in `app.json`.

**Soluzione:**
```json
// app.json
"resourceExposurePolicy": {
    "allowDebugging": true,
    "allowDownloadingSource": true,
    "includeSourceInSymbolFile": true
}
```
Rebuild + Republish + Hard refresh browser (Ctrl+Shift+Del → cancella cache immagini/file).

### 9.3 Chat visibile ma messaggi non arrivano

**Cause e soluzioni:**

1. **Credenziali non configurate**: Verificare API Key Status in Smart MES General Setup
2. **Capability non registrata**: Verificare pagina "Copilot e funzionalità agente"
3. **Ciclo di vita pagina**: La storia conversazione è in memoria. Se la pagina viene ricaricata, Copilot dimentica la conversazione precedente. Usare "Nuova conversazione" per azzerare esplicitamente

### 9.4 Capability non visibile nella pagina Copilot & Agent

La capability viene registrata dal trigger `OnInstallAppPerDatabase` in `Cod85275`. Se non compare:
- Verificare che l'environment sia SaaS (`IsSaaSInfrastructure() = true`)
- Eseguire un reinstall completo dell'extension

---

## 10. Sicurezza e dati

### 10.1 Dati inviati ad Azure OpenAI

Ogni chiamata AI include:
- Il metaprompt di sistema (istruzioni ruolo, strumenti disponibili)
- Il testo dei messaggi dell'utente
- I risultati degli strumenti (nomi e codici articoli, centri di lavoro, tempi aggregati, numeri di operazione)

**Non vengono mai inviati**: dati personali operatori, quantità di scarto, fermo macchina, dati finanziari.

### 10.2 Storage delle credenziali

| Dato | Storage | Accessibile |
|---|---|---|
| Azure OpenAI Endpoint | `Tab85250` (campo Text[250]) | Qualsiasi utente con accesso al setup |
| Azure OpenAI Deployment | `Tab85250` (campo Text[100]) | Qualsiasi utente con accesso al setup |
| API Key | `IsolatedStorage.SetEncrypted(scope=Module)` | Solo tramite `SetAuthorization`; non leggibile come testo |

### 10.3 Privacy e GDPR

Azure OpenAI non usa i dati inviati per addestrare i modelli (contratto Microsoft Commercial). I nomi degli operatori MES e i dati di tracciabilità non vengono mai inclusi nelle chiamate AI.

---

## 11. Limitazioni e roadmap

### 11.1 Limitazioni della versione attuale

| Limitazione | Spiegazione |
|---|---|
| **Tempi in minuti** | Gli strumenti create_routing_line usano minuti. Verificare che l'unità di misura della riga ciclo sia configurata in minuti |
| **Max 30 articoli** per ricerca | `find_items_by_criteria` restituisce al massimo 30 risultati |
| **Max 5 iterazioni ReAct** | Per evitare loop infiniti, il ciclo si ferma dopo 5 chiamate AI consecutive |
| **Nessun embedding semantico** | La ricerca articoli usa filtri esatti (categoria, dimensione) non similarità semantica |
| **Cicli Certified non modificabili** | BC impedisce la modifica. Portare il ciclo in stato New/Under Development prima |
| **No persistenza tra sessioni** | La storia conversazione è in memoria. Ricaricando la pagina si azzera |
| **Solo BC SaaS** | `resourceExposurePolicy.allowDebugging` funziona solo in SaaS con publishing standard |

### 11.2 Roadmap futura (Fasi 2 e 3)

| Fase | Feature | Valore |
|---|---|---|
| **Fase 2** | Analisi scarti con AI (ScrapCode + ScrapQuantity da ActivityLog) | Riduzione scarti in produzione |
| **Fase 2** | Analisi fermi macchina con AI (StopCode + StopTime) | Manutenzione predittiva |
| **Fase 3** | Embedding semantico articoli per similarità fuzzy | Ricerca "sedie di legno" trova anche "legno massello" |
| **Fase 3** | Persistenza storia conversazione (cache su table custom) | Continuare la conversazione tra sessioni |
| **Fase 4** | Agente autonomo con approvazione workflow BC | Creazione cicli con doppia approvazione manager |

---

*Fine documento — AGIC Smart MES Routing Copilot v2.0*
*Ultimo aggiornamento: 2026-07-02*
