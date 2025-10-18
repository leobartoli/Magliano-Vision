# 🧹 Spazzamento Monitor

Sistema di verifica della presenza di operatori tramite rilevamento del colore della casacca durante un turno di lavoro.

-----

## Obiettivo

Monitorare un’area stradale durante uno specifico turno (es. spazzamento martedì 08:00-09:30) per rilevare operatori identificati dal colore della casacca. Genera un log e lo invia via email al termine.

-----

## Setup

### 1. Installazione

```bash
cd ~/Downloads/Vision
source venv/bin/activate
pip install opencv-python python-dotenv
```

Scarica il modello:

```bash
wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_fullbody.xml -O haarcascade_fullbody.xml
```

### 2. Calibrazione Colore

Prima di usare il sistema, calibra il colore della casacca. Crea `calibrate_color.py`:

```python
import cv2
import numpy as np

cap = cv2.VideoCapture(0)

lower_h, lower_s, lower_v = 5, 100, 100
upper_h, upper_s, upper_v = 25, 255, 255

def nothing(x):
    pass

cv2.namedWindow('Calibration')
cv2.createTrackbar('Lower H', 'Calibration', lower_h, 180, nothing)
cv2.createTrackbar('Upper H', 'Calibration', upper_h, 180, nothing)
cv2.createTrackbar('Lower S', 'Calibration', lower_s, 255, nothing)
cv2.createTrackbar('Upper S', 'Calibration', upper_s, 255, nothing)
cv2.createTrackbar('Lower V', 'Calibration', lower_v, 255, nothing)
cv2.createTrackbar('Upper V', 'Calibration', upper_v, 255, nothing)

while True:
    ret, frame = cap.read()
    if not ret:
        break
    
    lower_h = cv2.getTrackbarPos('Lower H', 'Calibration')
    upper_h = cv2.getTrackbarPos('Upper H', 'Calibration')
    lower_s = cv2.getTrackbarPos('Lower S', 'Calibration')
    upper_s = cv2.getTrackbarPos('Upper S', 'Calibration')
    lower_v = cv2.getTrackbarPos('Lower V', 'Calibration')
    upper_v = cv2.getTrackbarPos('Upper V', 'Calibration')
    
    lower = np.array([lower_h, lower_s, lower_v])
    upper = np.array([upper_h, upper_s, upper_v])
    
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    mask = cv2.inRange(hsv, lower, upper)
    result = cv2.bitwise_and(frame, frame, mask=mask)
    
    cv2.imshow('Calibration', np.hstack([frame, result]))
    
    key = cv2.waitKey(1) & 0xFF
    if key == ord('q'):
        break
    elif key == ord('p'):
        print(f"LOWER: ({lower_h}, {lower_s}, {lower_v})")
        print(f"UPPER: ({upper_h}, {upper_s}, {upper_v})")

cap.release()
cv2.destroyAllWindows()
```

Esegui e regola i valori finché il filtro non isola bene la casacca. Premi ‘P’ per stampare i valori.

```bash
python calibrate_color.py
```

Valori comuni:

- **Arancione:** (5, 100, 100) - (25, 255, 255)
- **Giallo:** (20, 150, 100) - (35, 255, 255)
- **Verde:** (40, 50, 50) - (80, 255, 255)

### 3. Configurazione Email

Crea `.env`:

```bash
nano .env
```

Inserisci:

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

-----

## Utilizzo

### Script Principale

Crea `spazzamento_monitor.py`:

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

DURATION = int(os.getenv("MONITORING_DURATION_MINUTES", 90))
INTERVAL = 5
THRESHOLD = 0.25

LOWER_HSV = (5, 100, 100)
UPPER_HSV = (25, 255, 255)

EMAIL_SENDER = os.getenv("EMAIL_SENDER")
EMAIL_PASSWORD = os.getenv("EMAIL_PASSWORD")
EMAIL_RECIPIENT = os.getenv("EMAIL_RECIPIENT")

CASCADE = cv2.CascadeClassifier("haarcascade_fullbody.xml")
LOG_FILE = "spazzamento_log.txt"

def detect_operators(frame):
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    
    bodies = CASCADE.detectMultiScale(gray, 1.1, 5, minSize=(50, 100))
    count = 0
    
    for (x, y, w, h) in bodies:
        roi_y1 = y + int(h * 0.1)
        roi_y2 = y + int(h * 0.5)
        roi_x1 = x + int(w * 0.1)
        roi_x2 = x + int(w * 0.9)
        
        roi = hsv[roi_y1:roi_y2, roi_x1:roi_x2]
        mask = cv2.inRange(roi, LOWER_HSV, UPPER_HSV)
        
        if roi.size > 0:
            match = cv2.countNonZero(mask) / roi.size
            if match > THRESHOLD:
                count += 1
    
    return count

def write_log(msg):
    ts = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    line = f"[{ts}] {msg}\n"
    with open(LOG_FILE, 'a') as f:
        f.write(line)
    print(line.strip())

def send_email():
    try:
        with open(LOG_FILE, 'r') as f:
            log = f.read()
        
        msg = MIMEText(f"Report Spazzamento:\n\n{log}")
        msg['Subject'] = f"Report Spazzamento - {datetime.now().strftime('%d/%m/%Y')}"
        msg['From'] = EMAIL_SENDER
        msg['To'] = EMAIL_RECIPIENT
        
        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
            server.login(EMAIL_SENDER, EMAIL_PASSWORD)
            server.sendmail(EMAIL_SENDER, EMAIL_RECIPIENT, msg.as_string())
        
        print("✓ Email inviata")
        return True
    except Exception as e:
        print(f"✗ Errore: {e}")
        return False

def main():
    print(f"🧹 Spazzamento Monitor - Durata: {DURATION} minuti")
    
    with open(LOG_FILE, 'w') as f:
        f.write(f"INIZIO: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n")
        f.write(f"Durata: {DURATION} minuti\n\n")
    
    cap = cv2.VideoCapture(0)
    start = datetime.now()
    end = start + timedelta(minutes=DURATION)
    last_check = start
    total = 0
    
    while datetime.now() < end:
        ret, frame = cap.read()
        if not ret:
            time.sleep(1)
            continue
        
        now = datetime.now()
        if (now - last_check).seconds >= INTERVAL:
            count = detect_operators(frame)
            if count > 0:
                write_log(f"Operatori: {count}")
                total += count
            last_check = now
        
        cv2.waitKey(100)
    
    cap.release()
    cv2.destroyAllWindows()
    
    write_log(f"Fine. Totale: {total}")
    send_email()
    
    if os.path.exists(LOG_FILE):
        os.remove(LOG_FILE)

if __name__ == "__main__":
    main()
```

### Script di Avvio

Crea `start_spazzamento.sh`:

```bash
#!/bin/bash
cd /home/settoretecnico/Downloads/Vision
source venv/bin/activate
nohup python spazzamento_monitor.py > spazzamento_output.log 2>&1 &
deactivate
```

Rendi eseguibile:

```bash
chmod +x start_spazzamento.sh
```

### Schedulazione

```bash
crontab -e
```

Aggiungi:

```cron
@reboot sleep 30 && /home/settoretecnico/Downloads/Vision/start_spazzamento.sh

0 8 * * 2 /home/settoretecnico/Downloads/Vision/start_spazzamento.sh
```

Il primo comando riavvia il sistema dopo un reboot. Il secondo avvia il monitoraggio martedì alle 08:00.

-----

## Utilizzo Manuale

Testa il sistema:

```bash
python spazzamento_monitor.py
```

Verifica log:

```bash
tail -f spazzamento_output.log
```

-----

## Struttura Progetto

```
Vision/
├── spazzamento_monitor.py
├── calibrate_color.py
├── haarcascade_fullbody.xml
├── start_spazzamento.sh
├── .env
└── spazzamento_output.log
```

-----

## Note

- Calibra il colore della casacca prima di usare il sistema
- Assicura buona illuminazione durante i turni
- Non archivia immagini, solo log aggregato
- Il file log viene eliminato dopo l’invio email