# Data Model CE – Diagramma ER e Analisi

## Diagramma ER

> Renderizzabile in qualsiasi tool che supporti Mermaid (GitHub, Notion, Obsidian, VS Code con estensione).

```mermaid
erDiagram

    CONTO_ECONOMICO {
        int id_CE PK
        string tipo_CE
        string versione
        date data_creazione
        date data_ultimo_aggiornamento
        string azienda
        string tipologia_contratto
        string cliente_finale
        string cliente_di_fatturazione
        string condizioni_pagamento_attivo FK
        string numero_protocollo
        string ID_offerta
        string argomento
        bool accordo_quadro
        string numero_accordo_quadro
        string codice_gara_CIG
        string gruppo_di_progetti
        int id_manager_CE FK
        int id_manager_vendite FK
        date data_inizio
        date data_fine
        string BU
        string mercato_BU FK
        string CdR FK
        string service_line_prevalente
        string linea_prodotto_prevalente
    }

    UTENTE {
        int id_utente PK
        string nome
        string cognome
        string email
        string ruolo
    }

    GERARCHIA {
        int id_utente PK_FK
        string BU PK
        int livello_gerarchia
        string step_approvazione
    }

    APPROVAZIONE {
        int id_approvazione PK
        int id_CE FK
        int id_utente FK
        string esito
        timestamp timestamp_esito
        string note
    }

    CATEGORIA {
        int id_categoria PK
        int id_CE FK
        int numero_categoria
        string descrizione
        date mese_inizio
        date mese_fine
        string condizione_pag_passivo FK
        string tipo_costo
        string service_line
        string linea_prodotto
    }

    VOCE_CE {
        int id_voce_CE PK
        int id_categoria FK
        string delivery_company
        string vendor
        string SKU_part_number
        string descrizione
        string LE_job_descr FK
        decimal quantita
        string valuta
        decimal cost_unit
        decimal percentuale_SC
        decimal prezzo_unit_euro
        bool is_subscription_derivata
        decimal percentuale_derivata
    }

    PROFILI {
        string LE_job_descr PK
        string LE_code
        string legal_entity
        string job_profile_code
        decimal costo_orario
        decimal hourly_std_cost
        string CDR_type
        string legal_ID
    }

    MARGINI_MINIMI {
        string mercato PK
        string CDC PK
        decimal soglia_mercato
        decimal soglia_limite
    }

    CONDIZIONI_PAGAMENTO {
        string condizione_pagamento PK
        int numero_mesi
    }

    CAMBIO_VALUTA {
        string valuta PK
        decimal tasso_euro
        date data_aggiornamento
    }

    CONTO_ECONOMICO }o--|| UTENTE              : "manager_CE (id_manager_CE)"
    CONTO_ECONOMICO }o--|| UTENTE              : "manager_vendite (id_manager_vendite)"
    CONTO_ECONOMICO ||--o{ CATEGORIA           : "contiene"
    CONTO_ECONOMICO ||--o{ APPROVAZIONE        : "riceve"
    CONTO_ECONOMICO }o--|| CONDIZIONI_PAGAMENTO : "condizioni pagamento attivo"
    CONTO_ECONOMICO }o--|| MARGINI_MINIMI      : "valutato su (mercato_BU + CdR)"
    UTENTE          ||--o{ GERARCHIA           : "ha ruolo in BU"
    UTENTE          ||--o{ APPROVAZIONE        : "registra"
    CATEGORIA       ||--o{ VOCE_CE             : "raggruppa"
    CATEGORIA       }o--|| CONDIZIONI_PAGAMENTO : "pagamento passivo"
    VOCE_CE         }o--o| PROFILI             : "solo Figure Professionali"
    VOCE_CE         }o--o| CAMBIO_VALUTA       : "tasso di cambio"
```

---

## Considerazioni e differenze rispetto alla bozza

### 1. Campi ridondanti rimossi da VOCE_CE
Nel documento originale `id_CE` e `tipologia` erano presenti anche in VOCE_CE, ma entrambi sono già raggiungibili tramite `id_categoria → CATEGORIA → id_CE`. Rimossi. La nota nel documento già segnalava la ridondanza, ma non era stata applicata.

### 2. manager_CE e manager_vendite come FK a UTENTE
Nel documento, `manager_CE` è un campo stringa libero con `id_manager_ce` come FK aggiuntiva "per casistiche nomi duplicati". È più pulito avere **solo** `id_manager_CE` (FK) e derivare il nome dall'entità UTENTE. Stessa logica applicata a `manager_vendite` (attualmente solo stringa senza FK, es. "Luca Bagalini").

### 3. UTENTE – aggiunto campo `cognome` ed `email`
Il login e il riconoscimento univoco richiedono almeno nome, cognome e email. Il ruolo rimane (determina permessi), ma una separazione ruolo/permessi in una tabella dedicata potrebbe essere necessaria in futuro.

### 4. APPROVAZIONE – aggiunto campo `note`
In fase di rifiuto l'approvatore dovrà quasi certamente motivare la decisione. Il campo note è necessario anche operativamente.

### 5. Logica Subscription (85%/15%) — come gestirla nel data model
L'Excel gestisce la suddivisione creando due righe derivate visibili nel CE. Nel data model, ho aggiunto a VOCE_CE i campi:
- `is_subscription_derivata` (bool): distingue le righe auto-generate da quelle manuali
- `percentuale_derivata` (decimal): 0.85 o 0.15

In alternativa si potrebbe creare un'entità `VOCE_CE_SUBSCRIPTION` separata con il riferimento alla riga principale, ma aumenterebbe la complessità delle query. La soluzione con flag sulla stessa tabella è più semplice e funzionale per questo caso.

### 6. CAMBIO_VALUTA – entità mancante nella bozza
Il cambio $/€ è usato in ogni voce del CE ma nella bozza non esiste alcuna entità per gestirlo. Attualmente nell'Excel è un singolo valore fisso (es. 1.15). Nella webapp serve almeno una tabella con il tasso corrente per valuta, aggiornabile manualmente o da API esterna.

### 7. condizioni_di_pagamento in CONTO_ECONOMICO
Nella bozza `condizioni_di_pagamento` è un campo stringa libero nel CE (es. "30gg"), mentre in CATEGORIA `pag_passivo` è già una FK a CONDIZIONI_PAGAMENTO. Inconsistenza: anche in CONTO_ECONOMICO dovrebbe essere una FK alla stessa tabella.

### 8. CATEGORIA – aggiunti campi service_line, linea_prodotto, numero_categoria, descrizione
Dall'Excel si vede che ogni categoria ha: numero (es. Cat.1, Cat.2), descrizione, service line e linea di prodotto. La bozza non li include tutti. La `descrizione` categoria potrebbe essere auto-generata ma è utile averla persistita.

### 9. PROFILI – normalizzazione colonne Legal Entity
Nel foglio Excel PROFILI ci sono colonne separate per ogni Legal Entity (VIS_profili, DGS_profili, INN_profili, ecc.). Queste probabilmente rappresentano tariffe o flag specifici per entità. Il data model le ha semplificate (vengono dall'ERP). Bisogna chiarire cosa rappresentano esattamente prima di decidere se mantenerle o normalizzarle in una tabella separata `PROFILO_LEGAL_ENTITY`.

### 10. Stato del CE – campo derivato vs. persistito
Lo stato visibile in homepage ("In Lavorazione", "Approvato", "Rifiutato") è derivabile da APPROVAZIONE. Il documento lo gestisce correttamente come dato calcolato, ma potrebbe essere utile avere uno `stato` persistito in CONTO_ECONOMICO per semplificare le query della homepage e le notifiche, aggiornato a ogni evento di approvazione.

---

## Cosa manca per rifinire il data model

### Entità / lookup table mancanti

| Cosa manca | Perché serve | Note |
|---|---|---|
| **AZIENDA** | `delivery_company` e `azienda` del CE sono stringhe libere; servono come lookup per uniformità | Fonte: CRM o tabella locale (lista presente in sheet Tabelle) |
| **TIPO_COSTO** | Mappa tipo costo → codice INT/EXT; usato in CATEGORIA per il calcolo avanzamento costi | Presente in sheet Tabelle; non è nel data model |
| **LINEA_PRODOTTO** | Lookup con valori fissi (Consulenza, Subscription, Prodotti HW, ecc.) | Presente in sheet Tabelle |
| **SERVICE_LINE** | Lista molto lunga (~80 valori); da gestire come lookup | Presente in sheet Tabelle + CdR_SL |
| **BU** | Enum o lookup (3 valori: Cyber, Digital, Advisory) | Semplice ma formale |
| **MERCATO_BU** | Ogni BU ha i suoi mercati; serve il mapping BU → Mercati validi | Presente in sheet CdR_SL |
| **CdR** | Lookup dei CdR per mercato (es. DSPR001C001–C015 per EnergyUtilities) | Presente in sheet CdR_SL e CdR per mercati |
| **TIPOLOGIA_CONTRATTO** | Lookup (Gara, Subappalto, Trattativa privata, ecc.) | Presente in sheet Tabelle |
| **TIPOLOGIA_COMMESSA** | Lookup (Produttive, Manutenzione, Pre_sales, ecc.) | Presente in sheet Tabelle |
| **GRUPPO_PROGETTO** | Lookup (Time&Material, Canone, ecc.) — opzionale | Presente in sheet Tabelle |

### Aspetti logici non risolti

1. **Versioning del CE**: la bozza menziona un possibile `id_CE_padre` ma non lo formalizza. Bisogna decidere: un CE Offerta può diventare una Revisione P&L? Si crea un nuovo record con `id_CE_padre` che punta all'originale, oppure si usa un campo `versione` sullo stesso record? La scelta impatta la struttura della homepage e la storia delle modifiche.

2. **Flusso notifiche approvazione**: il documento descrive il flusso operativo (notifica → visione sola lettura → Approva/Rifiuta) ma non c'è un'entità o campo che tracci lo stato corrente dell'iter (es. "in attesa di approvazione livello 2"). Serve un campo `step_corrente` in CONTO_ECONOMICO o una logica applicativa robusta che lo derivi da APPROVAZIONE + GERARCHIA.

3. **Permessi per sezione**: Delivery compila Costi, Commerciale compila Ricavi. Questo richiede un sistema di permessi granulare per sezione del CE, non solo per ruolo generico. Il data model non lo modella.

4. **Gestione BDM (Business Development Manager)**: nell'Excel c'è una riga speciale "BDM" con extra costi percentuali. Non è rappresentata nel data model.

5. **Calcolo previsione incasso/pagamento**: dipende da una lookup "mesi" (tabella con date progressive, 60 mesi). Non è nel data model. Potrebbe essere generata dinamicamente, ma va deciso.

6. **Integrazione CRM (Dynamics 365)**: diversi campi del CE provengono dal CRM. Non è chiaro quali siano replicati localmente e quali vengano letti on-the-fly. Serve definire una strategia (ID offerta CRM come FK esterna? Sincronizzazione periodica?).

7. **Integrazione ERP (Profili)**: stessa domanda — i dati di PROFILI sono stati importati staticamente o vengono letti dall'ERP in tempo reale? Serve una data di aggiornamento e una strategia di sync.

8. **Soglia limite in MARGINI_MINIMI**: attualmente calcolata come `soglia_mercato * (1 - 7%)`. La percentuale di riduzione (7%) è un parametro di sistema — andrebbe esternalizzata in una tabella `PARAMETRI_SISTEMA` invece di essere hardcoded.
