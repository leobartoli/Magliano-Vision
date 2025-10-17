# 🌳 Parco Monitor: Sistema di Rilevamento Leggero Condizionale (Raspberry Pi)

Questo progetto implementa un sistema di Computer Vision ottimizzato per Raspberry Pi 4, progettato per monitorare continuamente un parco pubblico e rilevare quando rimane vuoto.

-----

## 🎯 Obiettivo

Il sistema adotta una logica condizionale per ottimizzare le risorse e inviare notifiche automatiche:

|Fase                |Descrizione                                                                                                            |
|--------------------|-----------------------------------------------------------------------------------------------------------------------|
|**Ricerca Attiva**  |Cattura una foto ogni 10 secondi finché vengono rilevate persone.                                                      |
|**Notifica e Pausa**|Quando il parco è vuoto, salva una foto, invia una notifica email e sospende il monitoraggio fino al giorno successivo.|
|**Resilienza**      |Il servizio si riavvia automaticamente dopo interruzioni di corrente o reboot.                                         |

-----

## 🧠 1. Architettura e Motore CV

Per garantire la massima stabilità e il minimo consumo di RAM sul Raspberry Pi, viene adottato un approccio leggero.

|Componente                 |Descrizione                                                                             |
|---------------------------|----------------------------------------------------------------------------------------|
|**Motore CV**              |Classificatore a Cascata di Haar (haarcascade_fullbody.xml), più leggero del modulo DNN.|
|**Logica di Schedulazione**|Gestita interamente in Python tramite datetime (senza librerie esterne).                |
|**Resilienza**             |Riavvio automatico tramite cron (@reboot).                                              |

-----

## ⚙️ 2. Setup Iniziale e Dipendenze

Il progetto risiede nella directory:

```
~/Downloads/Vision/
```

### 2.1. Prerequisiti

- Raspberry Pi OS (64-bit consigliato)
- Python 3.x
- Fotocamera USB o CSI compatibile

### 2.2. Installazione (Ambiente Virtuale)

```bash
# Entra nella cartella del progetto
cd ~/Downloads/Vision

# 1. Crea e attiva l'ambiente virtuale
python3 -m venv venv
source venv/bin/activate

# 2. Installa le librerie necessarie
pip install opencv-python python-dotenv
```

### 2.3. Download del Modello CV

Scarica il classificatore leggero per il rilevamento dei corpi umani:

```bash
wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_fullbody.xml -O haarcascade_fullbody.xml
```

-----

## 📧 3. Configurazione E-mail e Credenziali

Le credenziali SMTP devono essere fornite tramite un file `.env` nella directory del progetto.

### 3.1. Creazione del file .env

```bash
nano .env
```

Inserisci:

```env
EMAIL_SENDER="tua_email@gmail.com"
EMAIL_PASSWORD="la_tua_password_app_o_standard"
EMAIL_RECIPIENT="destinatario_email@dominio.it"
```

⚠️ **Se utilizzi Gmail**, crea una Password per App.

Imposta permessi ristretti al file:

```bash
chmod 600 .env
```

-----

## 🚀 4. Esecuzione e Gestione del Servizio

Per garantire il funzionamento continuo (24/7) e la resilienza agli shutdown, il sistema utilizza uno script wrapper e cron.

### 4.1. Script di Avvio (start_monitor.sh)

Crea il file `start_monitor.sh`:

```bash
#!/bin/bash

# Directory di lavoro del progetto
cd /home/settoretecnico/Downloads/Vision

# Attiva l'ambiente virtuale
source venv/bin/activate

# Avvia lo script Python in background e registra l'output
nohup python parco_monitor.py > monitor_output.log 2>&1 &

# Disattiva l'ambiente virtuale
deactivate
```

Rendi eseguibile:

```bash
chmod +x start_monitor.sh
```

### 4.2. Configurazione Cron

Apri la tabella cron:

```bash
crontab -e
```

Aggiungi:

```cron
# Riavvia dopo qualsiasi reboot o interruzione di corrente
@reboot /home/settoretecnico/Downloads/Vision/start_monitor.sh

# Riavvia il monitoraggio ogni giorno alle 06:00
0 6 * * * /home/settoretecnico/Downloads/Vision/start_monitor.sh
```

-----

## 🧩 5. Logica Dettagliata dello Script

Il file `parco_monitor.py` opera in due stati principali:

|Stato           |Condizione/Azione                                                                                                               |Ritardo   |
|----------------|--------------------------------------------------------------------------------------------------------------------------------|----------|
|**STATO ATTIVO**|Se vengono rilevate persone, continua il monitoraggio. Se il parco è vuoto: salva una foto, invia email, imposta IS_PAUSED=True.|10 secondi|
|**STATO PAUSA** |Il programma attende senza catturare foto. All’orario di riattivazione (es. 06:00), riprende il monitoraggio.                   |5 minuti  |

Questo approccio minimizza l’elaborazione e il consumo energetico durante la notte o in condizioni di inattività.

-----

## 📂 Struttura del Progetto

```
Vision/
├── venv/
├── haarcascade_fullbody.xml
├── parco_monitor.py
├── start_monitor.sh
├── .env
└── monitor_output.log
```

-----

## 🔁 Flusso Operativo

1. Il Raspberry Pi avvia automaticamente il servizio all’accensione.
1. Il sistema cattura immagini ogni 10 secondi.
1. Quando non rileva persone:
- Salva una foto nel percorso locale.
- Invia un’email di notifica.
- Si sospende fino al giorno successivo.
1. Alle 06:00 riprende automaticamente il monitoraggio.

-----

## 🌐 6. Connettività 4G/LTE (Opzionale ma Consigliato)

Per installazioni in aree senza Wi-Fi stabile, è consigliato usare un **dongle USB 4G** collegato direttamente al Raspberry Pi.

### 6.1. Perché il 4G

|Vantaggio               |Descrizione                                          |
|------------------------|-----------------------------------------------------|
|**Indipendenza di rete**|Nessuna dipendenza da Wi-Fi esterna instabile        |
|**Affidabilità**        |Connessioni cellulari stabili nelle aree aperte      |
|**Semplicità**          |Soluzione economica e facile da installare           |
|**Resilienza**          |Fallback automatico in caso di perdita di connessione|

### 6.2. Componenti Necessari

- **Dongle USB 4G/LTE** (€30-80)
  - Marche consigliate: Huawei E3372, TP-Link MA260, Sierra MC7455
  - Plug & play, riconosciuto automaticamente
  - Include slot per SIM Card
- **SIM Card dati** — Sottoscritta dal comune
  - Consumo dati minimo (solo email + log)
  - Piano base 5-10 GB/mese (€10-15)
  - Operatori: Vodafone, TIM, Tre
- **Cavo USB con alimentazione** (opzionale, per dispositivi ad alto consumo)

### 6.3. Installazione Hardware

Semplicissimo:

1. **Spegni il Raspberry Pi**
1. **Inserisci la SIM card** nel dongle USB 4G
1. **Collega il dongle a una porta USB** del Pi
1. **Accendi il Pi**

Il dispositivo viene riconosciuto automaticamente come interfaccia di rete.

Verifica riconoscimento:

```bash
# Visualizza i dispositivi USB
lsusb

# Visualizza le interfacce di rete
ifconfig
# Dovresti vedere un'interfaccia ppp* o eth* nuova
```

### 6.4. Configurazione Software (Automatica)

La maggior parte dei dongle 4G moderni si connette **automaticamente** senza configurazione.

Se non si connette, installa il gestore:

```bash
# Aggiorna il sistema
sudo apt update
sudo apt upgrade -y

# Installa ModemManager (gestisce automaticamente connessioni 4G)
sudo apt install -y modem-manager
```

Riavvia e il dongle dovrebbe collegarsi da solo:

```bash
sudo reboot
```

Verifica la connessione:

```bash
# Controlla l'IP
ifconfig

# Prova a pingare
ping -c 3 8.8.8.8
```

### 6.5. Verifica dello Status

Crea uno script semplice per monitorare la connessione:

```bash
nano ~/Downloads/Vision/check_4g_connection.sh
```

Inserisci:

```bash
#!/bin/bash

LOG_FILE="/tmp/4g_status.log"

# Controlla connessione internet
if ping -c 1 8.8.8.8 &> /dev/null; then
    echo "$(date) - Connessione 4G OK" >> $LOG_FILE
else
    echo "$(date) - Connessione 4G OFFLINE" >> $LOG_FILE
    # Opzionale: riavvia ModemManager
    sudo systemctl restart modem-manager
fi
```

Rendi eseguibile:

```bash
chmod +x ~/Downloads/Vision/check_4g_connection.sh
```

Aggiungi a crontab ogni 10 minuti:

```bash
crontab -e
```

Aggiungi:

```cron
# Verifica connessione 4G ogni 10 minuti
*/10 * * * * /home/settoretecnico/Downloads/Vision/check_4g_connection.sh
```

### 6.6. Troubleshooting 4G

|Problema                       |Soluzione                                                            |
|-------------------------------|---------------------------------------------------------------------|
|**Dongle non riconosciuto**    |Controlla `lsusb`, prova una porta USB diversa                       |
|**Nessun segnale 4G**          |Verifica copertura operatore, posiziona meglio il dongle             |
|**Connessione lenta/instabile**|Posiziona il dongle in un punto elevato, lontano da oggetti metallici|
|**Consumo dati elevato**       |Riduci frequenza foto (es. 30 secondi), comprimi log                 |
|**Dongle si disconnette**      |Aggiornamento firmware del dongle (vedi sito produttore)             |

### 6.7. Accesso Remoto (Opzionale: ZeroTier)

Se hai bisogno di accedere al Pi via SSH da remoto (per debug/manutenzione):

Installa ZeroTier:

```bash
curl https://install.zerotier.com | sudo bash
sudo systemctl enable zerotier-one
sudo systemctl start zerotier-one
```

Unisciti a una rete privata:

```bash
sudo zerotier-cli join [ID_RETE_16_CARATTERI]
```

Poi potrai accedere da remoto:

```bash
ssh pi@[IP_ZEROTIER]
```

-----

## 🛠️ 7. Manutenzione

Verifica log:

```bash
tail -f ~/Downloads/Vision/monitor_output.log
```

Verifica status 4G (se installato):

```bash
sudo zerotier-cli listnetworks
tail -f /tmp/4g_health.log
```

Riavvio manuale del servizio:

```bash
./start_monitor.sh
```

Aggiornamento librerie:

```bash
source venv/bin/activate
pip install --upgrade opencv-python python-dotenv
deactivate
```

-----

## 📄 Licenza

Questo progetto è distribuito sotto licenza MIT.  
Creato per il monitoraggio ambientale leggero e automatizzato con Raspberry Pi.