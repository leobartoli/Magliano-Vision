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

## 🛠️ Manutenzione

Verifica log:

```bash
tail -f ~/Downloads/Vision/monitor_output.log
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