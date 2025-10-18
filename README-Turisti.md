# 🏛️ Monitoraggio Affluenza Turisti con Raspberry Pi e Computer Vision

**Progetto:** Sistema automatico e non invasivo per il conteggio e l’analisi dell’affluenza di visitatori (turisti) in aree specifiche di un bene culturale, operativo per l’intera giornata.

## 📖 Concetto e Funzionamento

Il sistema utilizza la libreria OpenCV su un Raspberry Pi per l’analisi del flusso video (Computer Vision). Non salva alcuna immagine, ma registra solo dati numerici aggregati (conteggio e orario).

### Flusso Operativo

**Avvio (Cron):** Il sistema si avvia automaticamente all’orario di apertura (es. 08:00) tramite un job scheduler (Cron).

**Cattura Video:** Acquisisce il flusso video dalla telecamera collegata (USB o CSI).

**Rilevamento Persone:** Ogni 5 secondi, analizza un frame utilizzando il modello Haar Cascade per rilevare la presenza di corpi umani.

**Registrazione:** Scrive il conteggio delle persone rilevate in quel momento nel file di log (presenze_log.txt).

**Chiusura:** Al termine dell’orario stabilito (MONITORING_DURATION_MINUTES), invia il log completo via email e chiude il programma, eliminando il log locale.

## 📋 Requisiti Tecnici

|Componente|Requisito                             |Note                                                                                      |
|----------|--------------------------------------|------------------------------------------------------------------------------------------|
|Hardware  |Raspberry Pi 4                        |Consigliato per le prestazioni di calcolo video                                           |
|OS        |Raspberry Pi OS (o altra distro Linux)|Compatibile con Python e OpenCV                                                           |
|Telecamera|USB o CSI                             |Posizionamento cruciale per ottimizzare il campo visivo                                   |
|Email     |Account Gmail                         |È obbligatorio usare una “Password per App” se si ha l’autenticazione a due fattori attiva|

## ⚙️ Installazione e Setup

Assumi che la directory del progetto sia `~/Downloads/Vision`.

### Passo 3.1: Dipendenze Python e Ambiente

Crea e attiva l’ambiente virtuale (se non già fatto) e installa le librerie richieste.

```bash
cd ~/Downloads/Vision
# source venv/bin/activate  # Attiva l'ambiente se non già attivo
pip install opencv-python python-dotenv
```

### Passo 3.2: Modello di Rilevamento (Haar Cascade)

Scarica il file del modello pre-addestrato nella directory del progetto.

```bash
wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_fullbody.xml -O haarcascade_fullbody.xml
```

### Passo 3.3: Configurazione delle Credenziali (.env)

Crea un file chiamato `.env` nella directory principale e inserisci i dettagli di configurazione.

```bash
nano .env
```

```
EMAIL_SENDER="tua_email@gmail.com"
EMAIL_PASSWORD="password_app_gmail"
EMAIL_RECIPIENT="direttore@beneculturale.it"
MONITORING_DURATION_MINUTES=600  # Esempio: 10 ore (08:00 - 18:00)
```

Proteggi il file `.env` per la sicurezza delle credenziali:

```bash
chmod 600 .env
```

## 🐍 Script Python: presenze_turisti.py

Crea questo file e copia il codice al suo interno. È il cuore del sistema di rilevamento e logging.

```python
import cv2
import numpy as np
import os
from datetime import datetime, timedelta
from dotenv import load_dotenv
import smtplib
from email.mime.text import MIMEText
import time

load_dotenv()

# Configurazione del Monitoraggio
# Durata del monitoraggio in minuti. Default: 600 minuti (10 ore, es. 08:00 - 18:00)
DURATION = int(os.getenv("MONITORING_DURATION_MINUTES", 600))
DETECTION_INTERVAL = 5 # Secondi tra un rilevamento e il successivo

# Email Configuration
EMAIL_SENDER = os.getenv("EMAIL_SENDER")
EMAIL_PASSWORD = os.getenv("EMAIL_PASSWORD")
EMAIL_RECIPIENT = os.getenv("EMAIL_RECIPIENT")

# Modello di Rilevamento e Log
CASCADE = cv2.CascadeClassifier("haarcascade_fullbody.xml")
LOG_FILE = "presenze_log.txt"

def detect_people(frame):
    """
    Rileva i corpi umani (turisti) nel frame.
    
    Procedura:
    1. Converte il frame in scala di grigi.
    2. Cerca corpi umani utilizzando Haar Cascade.
    
    Restituisce: numero di persone rilevate
    """
    if CASCADE.empty():
        # Verifica di sicurezza
        print("✗ Errore: Modello Haar Cascade non caricato.")
        return 0

    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    
    # Rileva tutti i corpi umani nel frame
    bodies = CASCADE.detectMultiScale(
        gray, 
        scaleFactor=1.1, 
        minNeighbors=5, 
        minSize=(30, 60), # Dimensioni minime del corpo da rilevare
        flags=cv2.CASCADE_SCALE_IMAGE
    )
    
    # Restituisce semplicemente il conteggio dei corpi rilevati
    return len(bodies)

def write_log(msg):
    """Scrive un messaggio nel file di log con timestamp."""
    ts = datetime.now().strftime("%H:%M:%S")
    line = f"[{ts}] {msg}"
    
    try:
        with open(LOG_FILE, 'a') as f:
            f.write(line + "\n")
        print(line)
    except IOError as e:
        print(f"✗ Errore scrittura log: {e}")

def send_email():
    """Legge il log e lo invia via email."""
    if not all([EMAIL_SENDER, EMAIL_PASSWORD, EMAIL_RECIPIENT]):
        print("✗ Credenziali email mancanti. Report non inviato.")
        return

    try:
        with open(LOG_FILE, 'r') as f:
            log = f.read()
        
        # Crea messaggio email
        today_date = datetime.now().strftime('%d/%m/%Y')
        msg = MIMEText(f"Report Presenze Turisti:\n\n{log}")
        msg['Subject'] = f"Report Affluenza - {today_date}"
        msg['From'] = EMAIL_SENDER
        msg['To'] = EMAIL_RECIPIENT
        
        # Invia via Gmail
        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
            server.login(EMAIL_SENDER, EMAIL_PASSWORD)
            server.sendmail(EMAIL_SENDER, EMAIL_RECIPIENT, msg.as_string())
        
        print("✓ Email inviata con successo")
    except smtplib.SMTPAuthenticationError:
        print("✗ Errore invio email: Credenziali non valide. Assicurati di usare una 'Password per App' di Gmail.")
    except FileNotFoundError:
        print("✗ Errore invio email: File di log non trovato.")
    except Exception as e:
        print(f"✗ Errore invio email: {e}")

def main():
    """Loop principale di monitoraggio."""
    print(f"🏛️ Monitoraggio Turisti - Durata configurata: {DURATION} minuti")

    # Verifica la presenza del modello Haar Cascade
    if not os.path.exists(CASCADE.getFilename()):
        print(f"✗ Errore: File modello {CASCADE.getFilename()} non trovato. Assicurati che sia stato scaricato correttamente.")
        return
    
    # Inizializza file di log
    try:
        duration_hours = DURATION / 60
        with open(LOG_FILE, 'w') as f:
            f.write(f"Inizio monitoraggio: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n")
            f.write(f"Durata configurata: {DURATION} minuti ({duration_hours} ore)\n")
            f.write("-" * 50 + "\n\n")
    except IOError as e:
        print(f"✗ Impossibile creare il file di log: {e}")
        return
    
    # Apri fotocamera (indice 0 è tipico)
    cap = cv2.VideoCapture(0)
    
    if not cap.isOpened():
        write_log("⚠️ Errore: Impossibile aprire la fotocamera. Il dispositivo potrebbe non essere disponibile.")
        print("✗ Fotocamera non disponibile. Esco.")
        send_email() 
        return
    
    write_log("Fotocamera aperta con successo. Inizio loop di monitoraggio.")
    
    # Calcola tempo di fine
    start_time = datetime.now()
    end_time = start_time + timedelta(minutes=DURATION)
    last_detection = start_time
    
    # Loop di monitoraggio
    while datetime.now() < end_time:
        ret, frame = cap.read()
        
        if not ret:
            print("⚠️ Errore lettura fotocamera. Riprovo in 1 secondo.")
            time.sleep(1)
            continue
        
        # Rileva solo ogni DETECTION_INTERVAL secondi
        now = datetime.now()
        if (now - last_detection).seconds >= DETECTION_INTERVAL:
            count = detect_people(frame)
            
            # Logga solo se il conteggio è maggiore di zero
            if count > 0:
                write_log(f"Persone rilevate: {count}")
            
            # Aggiorna il tempo dell'ultima rilevazione per il prossimo check
            last_detection = now
        
        # Breve attesa per non sovraccaricare la CPU
        cv2.waitKey(100) 
    
    # Ferma la fotocamera e chiudi finestre
    cap.release()
    cv2.destroyAllWindows()
    
    # Termina monitoraggio
    write_log("Monitoraggio terminato")
    print("\nInvio report via email...")
    
    # Tentativo di invio email
    send_email()
    
    # Elimina file di log locale
    try:
        if os.path.exists(LOG_FILE):
            os.remove(LOG_FILE)
    except Exception as e:
        print(f"Avviso: Impossibile eliminare il file di log locale: {e}")

    print("✓ Monitoraggio Turisti completato.")

if __name__ == "__main__":
    main()
```

## ⚙️ Script Bash: start_turisti_monitor.sh

Crea questo script nella directory principale. Deve essere reso eseguibile e utilizzato da Cron.

```bash
#!/bin/bash

# =================================================================
# SCRIPT DI AVVIO PER IL MONITORAGGIO DEI TURISTI (Per Cron)
# =================================================================

# !!! MODIFICA QUESTO PERCORSO !!!
# Deve puntare alla posizione esatta della tua cartella Vision sul Raspberry Pi
PROJECT_DIR="/home/settoretecnico/Downloads/Vision"

# Vai alla directory del progetto
cd "$PROJECT_DIR" || { echo "Errore CRON: La directory di progetto non è stata trovata in $PROJECT_DIR"; exit 1; }

# Attiva l'ambiente virtuale
source venv/bin/activate || { echo "Errore CRON: Impossibile attivare l'ambiente virtuale venv. Verifica il percorso."; exit 1; }

# Esegui lo script Python e reindirizza l'output a un log di sistema
python presenze_turisti.py >> monitoraggio_output.log 2>&1

# Disattiva l'ambiente virtuale
deactivate

# Fine script
```

## ⏰ Automazione (Cron Setup)

Per avviare il monitoraggio automaticamente all’orario di apertura (es. 08:00) e tutti i giorni della settimana:

Rendi lo script di avvio eseguibile:

```bash
chmod +x start_turisti_monitor.sh
```

Apri l’editor Cron:

```bash
crontab -e
```

Aggiungi la seguente riga. Ricorda di verificare il percorso!

```
# Avvia lo script alle 08:00, tutti i giorni (dal lunedì alla domenica)
0 8 * * * /home/settoretecnico/Downloads/Vision/start_turisti_monitor.sh
```

## ⚠️ Note Importanti

**Verifica la Directory:** Assicurati che `PROJECT_DIR` in `start_turisti_monitor.sh` sia corretto. Un errore nel percorso impedisce l’avvio automatico.

**Log di Sistema:** Per verificare l’esecuzione e diagnosticare eventuali problemi del sistema, controlla il file `monitoraggio_output.log`.

**Privacy:** Non viene salvata alcuna immagine o video. Solo il conteggio numerico e l’orario vengono registrati.​​​​​​​​​​​​​​​​