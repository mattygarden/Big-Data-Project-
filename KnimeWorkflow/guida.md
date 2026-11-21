# SBA KNIME Project — analisi univariata

Apri `02_Univariate_Analysis_flow` nel workspace KNIME oppure importa `02_Univariate_Analysis_flow.knwf`.

Il workflow legge `data/SBA_clean_knime.csv`, prodotto dal workflow di cleaning. Il CSV Reader conserva LocationID come testo, inclusi gli zeri iniziali.

## Struttura

Le 17 variabili sono organizzate in metanodi, ordinati per tipo di dato. Ogni ramo termina in uno o più grafici. Il metanodo aggiuntivo `18_Descriptive_Tables` raccoglie quattro viste tabellari finali; i calcoli condivisi sono nel suo blocco interno `Shared_Calculations`.

- GrossApproval: istogramma degli importi, box plot e istogramma log10.
- InitialInterestRate: istogramma e box plot.
- GuaranteedPct: barre dei quattro livelli di copertura più frequenti e Other.
- ChannelGroup, LenderType, FixedorVariableInterestInd, Sector, BusinessType e BusinessAgeNew: conteggi, percentuali e barre.
- IsFranchise, IsRevolver, HasCollateral e Default: conteggi, percentuali e barre delle categorie 0/1.
- ApprovalFY e ApprovalFQ: conteggi, percentuali e barre in ordine temporale.
- BorrState: frequenze complete e barre dei primi 15 stati, con percentuali sul campione completo.
- LocationID: frequenze complete, barre dei primi 20 finanziatori più Other e curva cumulata.

I 21 grafici corrispondono a quelli presenti in `Big-Data-Project-/02_Univariate_Analysis.ipynb` e sono rimasti invariati. I Box Plot calcolano internamente quartili e IQR.

## Tabelle descrittive

Apri `18_Descriptive_Tables`, poi Open View su una delle quattro Table View:

1. **Riepilogo numeriche:** tre righe con N, media, deviazione standard, minimo, massimo, mediana, quartili, percentili 1/95/99, IQR e asimmetria. GrossApproval è in USD, InitialInterestRate in percentuale e GuaranteedPct è un rapporto tra 0 e 1.
2. **Estremi IQR:** soglie inferiore e superiore, numero e percentuale di osservazioni esterne per ciascuna numerica. Le osservazioni sono conservate nel dataset.
3. **Concentrazione geografica:** 53 categorie, 478.956 prestiti, 321.085 prestiti nei primi 15 stati, pari al 67,0385%.
4. **Concentrazione finanziatori:** 2.993 identificativi distinti, 239.550 prestiti nei primi 20 (50,0150%); servono 20, 173 e 414 finanziatori per raggiungere rispettivamente il 50%, 80% e 90% dei prestiti.

Non sono aggiunti controlli iniziali: il workflow utilizza il dataset già pulito da `01_Data_Cleaning_flow`.

## Uso

Apri un metanodo e seleziona Open View sul nodo grafico finale. Le tabelle con conteggi e percentuali sono consultabili sui nodi che alimentano i grafici. Dopo un nuovo cleaning, resetta e riesegui il workflow per aggiornare i risultati.

Grafici e tabelle sono stati eseguiti e verificati su 478.956 prestiti e 17 colonne. I riepiloghi numerici, i conteggi IQR e le metriche di concentrazione coincidono con il notebook di riferimento.

Le versioni precedenti sono conservate in `backup`. L'archivio generale `SBA_KNIME_Project.knar` è precedente: per questa analisi usa il workspace aggiornato oppure il singolo `.knwf`.
