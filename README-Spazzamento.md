# 🧹 Spazzamento Monitor

Sistema automatico di verifica della presenza di operatori durante i turni di spazzamento stradale. Utilizza Computer Vision per rilevare operatori con casacca arancione e invia un report dettagliato via email al termine del monitoraggio.

## Cos’è

Un programma che monitora un’area stradale per un periodo di tempo definito (es. 90 minuti) e rileva automaticamente la presenza di operatori identificati dal colore della loro casacca. Il sistema registra ogni rilevamento in un log e invia il report completo via email al termine del turno.

## Come Funziona

1. **Accensione**: Cron avvia il programma all’orario del turno (es. martedì 08:00)
1. **Monitoraggio**: Cattura video dalla fotocamera e analizza ogni frame ogni 5 secondi
1. **Rilevamento**: Identifica i corpi umani e verifica se indossano una casacca arancione
1. **Registrazione**: Scrive nel log il numero di operatori rilevati e l’orario
1. **Conclusione**: Al termine del turno, invia il log via email e si ferma

## Requisiti

- Raspberry Pi 4 con Raspberry Pi OS
- Fotocamera USB o CSI
- Python 3.x
- Account Gmail (per invio email)

## Installazione

### 1. Dipendenze

```bash
cd ~/Downloads/Vision
source venv/bin/activate
pip install opencv-python python-dotenv
```

### 2. Modello di Rilevamento

```bash
wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_fullbody.xml -O haarcascade_fullbody.xml
```

### 3. File di Configurazione

Crea `.env` nella directory del progetto:

```bash
nano .env
```

Inserisci le credenziali:

```env
EMAIL_SENDER="tua_email@gmail.com"
EMAIL_PASSWORD="password_app_gmail"
EMAIL_RECIPIENT="destinatario@dominio.it"
MONITORING_DURATION_MINUTES=90
```

Proteggi il file:

```bash
chmod 600 .env
```

## Script Principale

Crea il file `spazzamento_monitor.py`:

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

# Configurazione
DURATION = int(os.getenv("MONITORING_DURATION_MINUTES", 90))
LOWER_HSV = (5, 100, 100)      # Arancione - Hue min
UPPER_HSV = (25, 255, 255)     # Arancione - Hue max

# Email
EMAIL_SENDER = os.getenv("EMAIL_SENDER")
EMAIL_PASSWORD = os.getenv("EMAIL_PASSWORD")
EMAIL_RECIPIENT = os.getenv("EMAIL_RECIPIENT")

# Modelli
CASCADE = cv2.CascadeClassifier("haarcascade_fullbody.xml")
LOG_FILE = "spazzamento_log.txt"

def detect_operators(frame):
    """
    Rileva operatori con casacca arancione nel frame.
    
    Procedura:
    1. Converte frame in scala di grigi per rilevamento forme
    2. Converte frame in HSV per analisi colore
    3. Cerca corpi umani con Haar Cascade
    4. Per ogni corpo trovato, estrae il torso (dove è la casacca)
    5. Applica filtro HSV per cercare colore arancione
    6. Se >25% dei pixel del torso è arancione, conta come operatore
    
    Restituisce: numero di operatori rilevati
    """
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    
    # Rileva tutti i corpi umani nel frame
    bodies = CASCADE.detectMultiScale(gray, 1.1, 5, minSize=(50, 100))
    count = 0
    
    # Per ogni corpo rilevato
    for (x, y, w, h) in bodies:
        # Estrae il torso (circa dal 10% al 50% dell'altezza del corpo)
        roi_y1 = y + int(h * 0.1)
        roi_y2 = y + int(h * 0.5)
        roi = hsv[roi_y1:roi_y2, x:x+w]
        
        # Crea maschera per il colore arancione
        mask = cv2.inRange(roi, LOWER_HSV, UPPER_HSV)
        
        # Se almeno il 25% del torso è arancione, è un operatore
        if roi.size > 0 and cv2.countNonZero(mask) / roi.size > 0.25:
            count += 1
    
    return count

def write_log(msg):
    """Scrive un messaggio nel file di log con timestamp."""
    ts = datetime.now().strftime("%H:%M:%S")
    line = f"[{ts}] {msg}"
    
    with open(LOG_FILE, 'a') as f:
        f.write(line + "\n")
    
    print(line)

def send_email():
    """Legge il log e lo invia via email."""
    try:
        with open(LOG_FILE, 'r') as f:
            log = f.read()
        
        # Crea messaggio email
        msg = MIMEText(f"Report Spazzamento:\n\n{log}")
        msg['Subject'] = f"Report Spazzamento - {datetime.now().strftime('%d/%m/%Y')}"
        msg['From'] = EMAIL_SENDER
        msg['To'] = EMAIL_RECIPIENT
        
        # Invia via Gmail
        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
            server.login(EMAIL_SENDER, EMAIL_PASSWORD)
            server.sendmail(EMAIL_SENDER, EMAIL_RECIPIENT, msg.as_string())
        
        print("✓ Email inviata con successo")
    except Exception as e:
        print(f"✗ Errore invio email: {e}")

def main():
    """Loop principale di monitoraggio."""
    print(f"🧹 Spazzamento Monitor - Durata: {DURATION} minuti")
    
    # Inizializza file di log
    with open(LOG_FILE, 'w') as f:
        f.write(f"Inizio monitoraggio: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n")
        f.write(f"Durata: {DURATION} minuti\n")
        f.write("-" * 50 + "\n\n")
    
    # Apri fotocamera
    cap = cv2.VideoCapture(0)
    
    # Calcola tempo di fine
    start_time = datetime.now()
    end_time = start_time + timedelta(minutes=DURATION)
    last_detection = start_time
    
    # Loop di monitoraggio
    while datetime.now() < end_time:
        ret, frame = cap.read()
        
        if not ret:
            print("⚠️ Errore lettura fotocamera")
            time.sleep(1)
            continue
        
        # Rileva ogni 5 secondi
        now = datetime.now()
        if (now - last_detection).seconds >= 5:
            count = detect_operators(frame)
            
            if count > 0:
                write_log(f"Operatori rilevati: {count}")
            
            last_detection = now
        
        cv2.waitKey(100)
    
    # Ferma la fotocamera
    cap.release()
    cv2.destroyAllWindows()
    
    # Termina monitoraggio
    write_log("Monitoraggio terminato")
    print("\nInvio report via email...")
    send_email()
    
    # Elimina file di log locale
    if os.path.exists(LOG_FILE):
        os.remove(LOG_FILE)
    
    print("✓ Spazzamento Monitor terminato")

if __name__ == "__main__":
    main()
```

## Esecuzione

### Manuale

Esegui il programma per testarlo:

```bash
python spazzamento_monitor.py
```

Il programma funzionerà per 90 minuti (configurabile in `.env`), monitorerà continuamente la fotocamera, e al termine invierà il report via email.

### Con Cron (Automatico)

Crea uno script di avvio `start_spazzamento.sh`:

```bash
#!/bin/bash

cd /home/settoretecnico/Downloads/Vision
source venv/bin/activate
python spazzamento_monitor.py >> spazzamento_output.log 2>&1
deactivate
```

Rendi eseguibile:

```bash
chmod +x start_spazzamento.sh
```

Configura cron per avviare automaticamente:

```bash
crontab -e
```

Aggiungi la seguente riga (avvia martedì alle 08:00):

```cron
0 8 * * 2 /home/settoretecnico/Downloads/Vision/start_spazzamento.sh
```

## Struttura del Progetto

```
Vision/
├── spazzamento_monitor.py        # Script principale
├── haarcascade_fullbody.xml      # Modello di rilevamento
├── .env                          # Configurazione (credenziali)
├── start_spazzamento.sh          # Script per cron
├── spazzamento_output.log        # Log di output
└── spazzamento_log.txt           # Log inviato via email (generato a runtime)
```

## Output

Il programma genera un log come questo:

```
Inizio monitoraggio: 2024-01-15 08:00:05
Durata: 90 minuti
--------------------------------------------------

[08:00:10] Operatori rilevati: 2
[08:00:15] Operatori rilevati: 1
[08:05:20] Operatori rilevati: 3
[08:10:25] Operatori rilevati: 2
[09:29:50] Operatori rilevati: 1
[09:30:05] Monitoraggio terminato
```

L’email ricevuta contiene il log completo.

## Note Importanti

- **Privacy**: Il sistema NON salva alcuna immagine, solo dati aggregati nel log
- **Gmail**: Usa una “Password per App” se hai 2FA attivo (non la password del tuo account)
- **Illuminazione**: Assicura buona illuminazione durante i turni per rilevamento accurato
- **Casacca arancione**: Il sistema rileva specificamente il colore arancione (HSV 5-25)
- **Durata**: Configurabile nel file `.env`

## Troubleshooting

Se non rileva operatori:

- Verifica che la fotocamera funziona: `ls /dev/video*`
- Prova ad aumentare l’illuminazione
- Verifica il colore esatto della casacca

Se l’email non viene inviata:

- Controlla credenziali in `.env`
- Per Gmail, usa Password per App, non la password normale
- Verifica che il Raspberry Pi ha connessione internet