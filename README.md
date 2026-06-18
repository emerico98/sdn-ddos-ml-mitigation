
# SDN DDoS Detection and Mitigation

Rilevamento e mitigazione automatica di attacchi DoS/DDoS in ambiente SDN
(Software Defined Networking) emulato, mediante Mininet, controller Ryu,
OpenFlow 1.3 e un modello di Machine Learning (Random Forest).

---

## 1. Descrizione

Il progetto realizza una rete SDN emulata in Mininet in cui un controller Ryu
monitora le statistiche dei flussi OpenFlow, classifica il traffico diretto a un
server protetto (h4, 10.0.0.4) tramite un modello Random Forest e installa
automaticamente regole DROP selettive verso le sole sorgenti riconosciute come
sospette.

A differenza di un approccio a soglia fissa, la decisione di blocco è affidata interamente al
modello, addestrato su feature di flusso (rate istantanei, dimensione media dei
pacchetti) e su feature aggregate verso la destinazione (numero di sorgenti, rate
totale, flussi attivi). Il blocco non interessa l'intera porta dello switch ma la
singola coppia `sorgente -> vittima`, evitando l'over-blocking del traffico
legittimo.

---

## 2. Struttura del progetto


```

DDOS_corretto_finale/
├── topology/
│   ├── network_core_access.py           # topologia Mininet core/access (demo manuale)
│   └── auto_collect.py                  # topologia automatica per la raccolta dataset
├── controller/
│   └── ml_detector_controller_final.py  # controller finale: detection + mitigazione reattiva
├── scripts/
│   ├── dataset_collector_auto.py        # collector per la raccolta automatica del dataset
│   └── udp_realistic_sender.py          # generatore UDP con rate/payload/burst variabili
├── dataset/
│   └── dataset_v8_training.csv          # dataset flussi raccolto dalla rete emulata
├── ml/
│   └── best_model.joblib                # modello Random Forest serializzato e ottimizzato
├── results/                             # report delle metriche di addestramento e log
├── train_model.py                       # script per il training del modello e calcolo soglia ottimale
└── run_collect.sh                       # pipeline completa: pulizia + raccolta automatica + training

```

---

## 3. Prerequisiti

Ambiente di riferimento: Ubuntu 22.04.

Componenti software: Open vSwitch, Mininet, controller Ryu, Python 3 con
scikit-learn, pandas, numpy, joblib.

```bash
sudo apt update
sudo apt install -y git vim python3-pip openvswitch-switch mininet iperf3
pip3 install --break-system-packages scikit-learn pandas numpy joblib

```

### Avvio di Ryu in locale

In questo progetto il controller Ryu viene eseguito da una copia locale presente
nella cartella `ryu/`, anziché dall'installazione di sistema. Il comando base è:

```bash
PYTHONPATH=./ryu python3 ryu/bin/ryu-manager <file_controller>

```

Tutti i comandi che seguono usano questa forma. Eseguire sempre dalla cartella
radice del progetto (`DDOS_corretto_finale`).

---

## 4. Guida all'esecuzione

Assicurarsi sempre di essere posizionati all'interno della cartella principale prima di iniziare:

```bash
cd ~/percorso/della/tua/cartella/DDOS_corretto_finale

```

### 4.1 Raccolta del dataset e addestramento (un solo comando)

Lo script `run_collect.sh` esegue l'intera pipeline automatica: esegue la pulizia preliminare dei processi appesi, avvia il `dataset_collector_auto.py` in background, lancia la topologia automatica che genera sequenzialmente le tre fasi di traffico (normale, attacco distribuito, attacco singolo) per il numero di round desiderati, arresta l'ambiente e infine lancia `train_model.py`.

```bash
chmod +x run_collect.sh
sudo ./run_collect.sh

```

Parametri opzionali configurabili a monte (numero di round e durata di ogni fase in secondi):

```bash
ROUNDS=3 PHASE_SECONDS=60 sudo -E ./run_collect.sh

```

Al termine, il dataset si troverà in `dataset/dataset_v8_training.csv` e il modello ottimizzato in `ml/best_model.joblib`. Per eseguire nuovamente il solo addestramento senza ripetere la simulazione su Mininet:

```bash
python3 train_model.py

```

### 4.2 Demo: rilevamento e mitigazione in tempo reale

La verifica in tempo reale del funzionamento del modello sul traffico richiede l'apertura di due terminali distinti (entrambi posizionati dentro la cartella `DDOS_corretto_finale`).

Prima di procedere, eseguire una pulizia globale dei link e dei processi zombie:

```bash
sudo mn -c

```

**Terminale 1 — Avvio del controller Ryu con difesa ML attiva:**

```bash
PYTHONPATH=./ryu python3 ryu/bin/ryu-manager controller/ml_detector_controller_final.py

```

**Terminale 2 — Avvio della rete emulata in Mininet:**

```bash
sudo python3 topology/network_core_access.py

```

*Nota: Una volta avviata la CLI di Mininet, l'iniezione del traffico interattivo può essere effettuata lanciando i comandi di invio personalizzati o tramite gli strumenti nativi di test (es. iperf3).*

### 4.3 Verifica della mitigazione

Dopo alcune rilevazioni consecutive del traffico malevolo, all'interno della CLI di Mininet (Terminale 2), ispezionare le tabelle di flusso degli switch per validare l'intervento del controllore:

```bash
mininet> sh ovs-ofctl -O OpenFlow13 dump-flows s0
mininet> sh ovs-ofctl -O OpenFlow13 dump-flows s1
mininet> sh ovs-ofctl -O OpenFlow13 dump-flows s2

```

La presenza di regole OpenFlow con `priority=200`, match `ip,nw_src=<host_attaccante>,nw_dst=10.0.0.4` e istruzione `actions=drop` conferma che la mitigazione selettiva è stata installata con successo dal Machine Learning salvaguardando il resto della topologia.

---

## 5. Parametri principali del controller

| Parametro | Valore | Ruolo |
| --- | --- | --- |
| `PROTECTED_SERVER_IP` | 10.0.0.4 | Server protetto (vittima h4) |
| `MIN_FLOW_PPS_FOR_ML` | 1.0 | Filtro pps minimo per evitare l'interrogazione del modello su flussi dormienti |
| `SUSPICIOUS_LIMIT` | 4 | Rilevazioni consecutive classificate come attacco prima dell'installazione del blocco |
| `BLOCK_TIMEOUT` | 300 s | Tempo di permanenza (Hard Timeout) della regola DROP sullo switch |

Le metriche aggregate totali dirette al server protetto (`10.0.0.4`) vengono estratte e registrate nei log esclusivamente a scopi informativi; la decisione finale di blocco (`final_pred`) è determinata in via esclusiva dal superamento della soglia probabilistica ottimale del modello Random Forest.

