# 03 Bivariate Analysis in KNIME: guida

Questo workflow traduce in KNIME il notebook `03_bivariate_analysis.ipynb`, usando solo nodi nativi (nessun nodo Python). Legge lo stesso file prodotto dal workflow di cleaning `01_Data_Cleaning_flow`, cioè `data/SBA_clean_knime.csv` (478.956 prestiti, 17 colonne).

File: `KnimeWorkflow/03_Bivariate_Analysis_flow.knwf`

## Struttura

Il workflow ha 4 nodi di preparazione e 6 metanodi, nello stesso ordine del notebook.

| Nodo / metanodo | Cosa fa | Nodi principali |
|---|---|---|
| Preparazione | Legge il CSV; crea `LogAmount = log10(GrossApproval)`; trasforma i flag 0/1 in testo; crea `Default_cat`, una copia testuale del target | CSV Reader, Math Formula, Number to String, Rule Engine |
| 1_Spearman_numeric | Correlazione di Spearman tra le variabili numeriche e con Default (sezioni 1 e 5 del notebook) | Column Filter, Rank Correlation (Spearman's rho) |
| 2_CramersV_categorical | V di Cramér per tutte le coppie di categoriche, con matrice e p-value del chi-quadro; il ramo in basso tiene solo le coppie con Default (sezioni 2 e 4) | Linear Correlation, Row Filter, Sorter, Bar Chart, Table View |
| 3_Eta_categorical_numeric | Rapporto di correlazione η tra categoriche e numeriche, calcolato con nodi nativi (sezione 3) | Unpivot, GroupBy, Math Formula, Table View |
| 3b_Boxplots_eta_pairs | Box plot delle 9 coppie con η ≥ 0,3 | Box Plot |
| 4_Default_vs_categorical | Tasso di default per categoria (grafico) e test del chi-quadro per ognuna delle 10 categoriche | GroupBy, Bar Chart, Crosstab |
| 5_Default_vs_numeric | Tasso di default per fascia di tasso d'interesse e di importo | Rule Engine, GroupBy, Bar Chart |

**Come si calcola η senza Python.** Unpivot mette in colonna le 10 categoriche. GroupBy calcola numerosità, media e deviazione standard per categoria. Alcuni Math Formula calcolano la somma dei quadrati tra gruppi (B) e dentro i gruppi (W). Infine η = √(B / (B + W)). Abbiamo verificato in Python che questo calcolo riproduce il notebook (differenza < 10⁻¹²).

## Come usarlo

1. **Importa il workflow.** Trascina `03_Bivariate_Analysis_flow.knwf` nello spazio di KNIME (pannello Space Explorer), oppure usa *File → Import KNIME Workflow*.
2. **Metti il CSV nella cartella di KNIME (una volta sola).** Il CSV Reader non usa il percorso del computer di qualcuno: legge `knime://knime.workflow/../SBA_clean_knime.csv`, cioè `SBA_clean_knime.csv` nella **cartella del workspace KNIME** (di solito `~/knime-workspace`), accanto alla cartella del workflow importato. Copia lì il tuo `SBA_clean_knime.csv` (quello prodotto da 01). In Finder: Vai → Vai alla cartella… → `~/knime-workspace`. Attenzione: se in KNIME elimini un workflow, controlla di non aver eliminato anche il CSV.
3. **Esegui tutto.** Seleziona tutti i nodi (Cmd+A) ed esegui con *Execute all* (Shift+F7 o il doppio triangolo verde). Il nodo più lento è Unpivot nel metanodo 3: crea circa 4,8 milioni di righe e impiega alcune decine di secondi.
4. **Spearman (metanodo 1).** Il nodo Rank Correlation è già incluso e configurato (Spearman's rho, 5 colonne): Open View sulla porta *Correlation matrix*.
5. **Leggi i risultati.** Tasto destro sui nodi finali (Bar Chart, Table View, Box Plot, Linear Correlation) → *Open View*. Il chi-quadro di ogni Crosstab è sulla **seconda** porta di uscita: tasto destro → *Chi-square statistics*.

## Risultati attesi (dal notebook), già verificati in KNIME il 25/11: coincidono

| Misura | Valore atteso |
|---|---|
| Righe lette | 478.956; tasso di default 5,76% |
| Spearman LogAmount – InitialInterestRate / GuaranteedPct; Rate – ApprovalFY | −0,42 / 0,52; 0,26 |
| Spearman con Default: Rate, LogAmount, GuaranteedPct, ApprovalFY | +0,110, −0,075, +0,020, +0,012 |
| V di Cramér ChannelGroup – IsRevolver / HasCollateral / LenderType | 0,64 / 0,37 / 0,36 |
| V con Default: ChannelGroup, HasCollateral, BorrState, Sector, Fixed/Var, LenderType | 0,067, 0,063, 0,055, 0,050, 0,039, 0,038 |
| η ChannelGroup – GuaranteedPct / LogAmount / Rate | 0,96 / 0,73 / 0,35 |
| η IsRevolver – GuaranteedPct / LogAmount; HasCollateral – LogAmount | 0,59 / 0,47; 0,43 |
| Default per canale: Mission, Small-loan, Express, Standard, Delegated, Specialty | 13,0%, 7,2%, 5,9%, 5,0%, 4,9%, 3,5% |
| Collaterale no / sì; Non-bank; tasso variabile / fisso | 8,1% / 4,9%; 10,0%; 6,2% / 4,0% |
| Fasce di tasso (≤5, 5–6, 6–7, 7–8, >8%) | 2,2%, 4,4%, 6,1%, 8,1%, 9,9% |
| Fasce di importo (≤50k, 50–150k, 150–500k, >500k) | 7,4%, 6,9%, 5,0%, 2,5% |

## Se un nodo è giallo o rosso

Il workflow è stato generato fuori da KNIME a partire dalle impostazioni dei nodi dei workflow 01 e 02 e degli esercizi del corso. Se un nodo non si configura da solo:

1. Doppio clic sul nodo.
2. Controlla che le colonne indicate nel nome del nodo siano selezionate.
3. *OK*, poi *Execute*.

I nomi delle colonne attese sono scritti sotto ogni nodo.

## Collegamento con gli altri workflow

- `01_Data_Cleaning_flow` (Luca) produce `data/SBA_clean_knime.csv`.
- `02_Univariate_Analysis_flow` (Edo) e questo `03` leggono entrambi quel file, quindi non serve collegarli con frecce. Dopo un nuovo cleaning, resetta ed esegui di nuovo.
- Per condividerlo: *File → Export Workflow*, con "Reset workflow before export" attivo, poi salva in `KnimeWorkflow/`.
