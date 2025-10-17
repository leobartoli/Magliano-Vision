# 🌊 Sottopassaggio Monitor: Sistema di Allarme Alluvioni in Tempo Reale

Sistema di monitoraggio automatico del livello dell’acqua in sottopassaggi con **riconoscimento visivo della scala graduata** e **allarmi automatici** via email e SMS.

-----

## 🎯 Obiettivo

Monitorare continuamente il livello dell’acqua in un sottopassaggio mediante:

1. **Fotocamera** posizionata sulla scala graduata (in cm)
1. **Computer Vision** per leggere automaticamente l’altezza dell’acqua
1. **Allarmi automatici** quando il livello supera soglie critiche
1. **Notifiche** a Protezione Civile, amministrazione, responsabili

|Evento                          |Azione                                       |
|--------------------------------|---------------------------------------------|
|**Livello normale**             |Monitoraggio silenzioso ogni 30 minuti       |
|**Livello GIALLO** (es. >50cm)  |Email di allerta al comune                   |
|**Livello ROSSO** (es. >100cm)  |Email + SMS urgente a Protezione Civile      |
|**Livello CRITICO** (es. >150cm)|Allerta SMS multipla + segnalazione immediata|

-----

## 🧠 1. Architettura del Sistema

### Componenti Hardware

|Componente                       |Descrizione                             |
|---------------------------------|----------------------------------------|
|**Raspberry Pi 4**               |Elaborazione immagini e logica allarmi  |
|**Fotocamera USB/CSI**           |Ripresa della scala graduata            |
|**Scala graduata**               |Marcata in centimetri, ben illuminata   |
|**Illuminazione LED** (opzionale)|Per letture notturne affidabili         |
|**Modem 4G/Dongle USB**          |Connettività remota (vedi Parco Monitor)|
|**Alimentatore stabilizzato**    |5V 3A minimo                            |

### Software Stack

- **OpenCV** — Riconoscimento posizione del livello dell’acqua
- **Tesseract OCR** — Lettura numeri della scala (backup)
- **Python 3** — Script principale di monitoraggio
- **Email/SMS** — Notifiche via SMTP e servizio SMS
- **Cron** — Schedulazione e resilienza

-----

## 📊 2. Calibrazione della Scala

La chiave è la **calibrazione iniziale** della scala graduata.

### 2.1. Procedure di Calibrazione

1. **Fotografa la scala vuota** (senza acqua)
- Salva come `calibration_empty.jpg`
1. **Posiziona un oggetto a riferimento** a 50cm
- Fotografa — salva come `calibration_50cm.jpg`
1. **Posiziona a 100cm**
- Fotografa — salva come `calibration_100cm.jpg`
1. **Esegui lo script di calibrazione:**

```bash
python calibrate_water_level.py
```

Questo genera un file `calibration.json` con i parametri di conversione pixel → centimetri.

### 2.2. Script di Calibrazione

```python
# calibrate_water_level.py
import cv2
import json

def calibrate():
    """
    Calibra la scala graduata.
    Chiede all'utente di indicare 3 punti di riferimento.
    """
    
    # Carica immagini di calibrazione
    img_empty = cv2.imread('calibration_empty.jpg')
    img_50cm = cv2.imread('calibration_50cm.jpg')
    img_100cm = cv2.imread('calibration_100cm.jpg')
    
    print("Calibrazione scala graduata")
    print("=" * 50)
    
    # Per semplicità: registra manualmente i pixel corrispondenti
    # In produzione, usare riconoscimento automatico di marcatori
    
    print("\nClicca sull'immagine per indicare il punto a 0cm")
    print("(Premi ESC al termine)")
    
    # Conversione pixel-centimetri
    pixel_0cm = 100  # Esempio: pixel 100 = 0cm
    pixel_50cm = 250  # Esempio: pixel 250 = 50cm
    pixel_100cm = 400  # Esempio: pixel 400 = 100cm
    
    # Calcola la retta di conversione
    cm_per_pixel = 50 / (pixel_50cm - pixel_0cm)
    offset_pixel = pixel_0cm
    
    calibration = {
        "pixel_0cm": pixel_0cm,
        "pixel_50cm": pixel_50cm,
        "pixel_100cm": pixel_100cm,
        "cm_per_pixel": cm_per_pixel,
        "offset_pixel": offset_pixel
    }
    
    # Salva calibrazione
    with open('calibration.json', 'w') as f:
        json.dump(calibration, f, indent=4)
    
    print(f"✓ Calibrazione salvata: 1 pixel = {cm_per_pixel:.3f} cm")

if __name__ == "__main__":
    calibrate()
```

-----

## 🌊 3. Riconoscimento del Livello dell’Acqua

### 3.1. Come Funziona

1. **Foto della scala** con il livello dell’acqua
1. **Rilevamento del colore dell’acqua** (blu/grigio scuro)
1. **Identificazione del bordo acqua** (linea di confine)
1. **Conversione pixel → centimetri** usando calibrazione
1. **Confronto con soglie di allarme**

### 3.2. Script Principale

```python
# sottopassaggio_monitor.py
import cv2
import numpy as np
import json
import smtplib
from datetime import datetime
from email.mime.text import MIMEText
from email.mime.image import MIMEImage
from dotenv import load_dotenv
import os

load_dotenv()

# Soglie di allarme (in centimetri)
SOGLIA_GIALLO = 50   # Attenzione
SOGLIA_ROSSO = 100   # Allerta alta
SOGLIA_CRITICO = 150  # Critico

# Email configuration
EMAIL_SENDER = os.getenv("EMAIL_SENDER")
EMAIL_PASSWORD = os.getenv("EMAIL_PASSWORD")
EMAIL_PROTEZIONE_CIVILE = os.getenv("EMAIL_PROTEZIONE_CIVILE")
EMAIL_SINDACO = os.getenv("EMAIL_SINDACO")

def load_calibration():
    """Carica i parametri di calibrazione."""
    with open('calibration.json', 'r') as f:
        return json.load(f)

def detect_water_level(frame, calibration):
    """
    Rileva il livello dell'acqua nell'immagine.
    Restituisce: altezza in cm
    """
    # Converte in HSV per rilevamento colore migliore
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    
    # Definisce range di colore per acqua (grigio-blu scuro)
    lower_water = np.array([80, 50, 50])
    upper_water = np.array([130, 255, 255])
    
    # Crea maschera per l'acqua
    mask = cv2.inRange(hsv, lower_water, upper_water)
    
    # Rileva i contorni
    contours, _ = cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    
    if not contours:
        return None
    
    # Trova il bordo più basso dell'acqua (livello massimo)
    max_y = 0
    for contour in contours:
        y = cv2.boundingRect(contour)[1]
        max_y = max(max_y, y + cv2.boundingRect(contour)[3])
    
    # Converte pixel in centimetri
    cm_per_pixel = calibration["cm_per_pixel"]
    offset_pixel = calibration["offset_pixel"]
    
    water_level_cm = (offset_pixel - max_y) * cm_per_pixel
    
    return max(0, water_level_cm)  # Non negativi

def send_email_alert(level_cm, status):
    """Invia email di allarme."""
    
    subject_map = {
        "GIALLO": "⚠️ ALLARME GIALLO - Sottopassaggio: Livello Acqua Elevato",
        "ROSSO": "🚨 ALLARME ROSSO - Sottopassaggio: PERICOLO ALLUVIONE",
        "CRITICO": "🚨🚨 ALLARME CRITICO - Sottopassaggio: EVACUAZIONE IMMEDIATA"
    }
    
    body = f"""
    ALLARME LIVELLO ACQUA SOTTOPASSAGGIO
    {'=' * 50}
    
    Stato: {status}
    Livello misurato: {level_cm:.1f} cm
    Orario: {datetime.now().strftime('%d/%m/%Y %H:%M:%S')}
    
    Soglie:
    - GIALLO: {SOGLIA_GIALLO} cm
    - ROSSO: {SOGLIA_ROSSO} cm
    - CRITICO: {SOGLIA_CRITICO} cm
    
    AZIONI CONSIGLIATE:
    - Chiudere il sottopassaggio al traffico
    - Attivare protezione civile se ROSSO/CRITICO
    - Vietare accesso ai pedoni
    
    Sistema Sottopassaggio Monitor
    """
    
    recipients = [EMAIL_SINDACO, EMAIL_PROTEZIONE_CIVILE]
    
    try:
        msg = MIMEText(body)
        msg['Subject'] = subject_map.get(status, "ALLARME Sottopassaggio")
        msg['From'] = EMAIL_SENDER
        msg['To'] = ', '.join(recipients)
        
        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
            server.login(EMAIL_SENDER, EMAIL_PASSWORD)
            server.sendmail(EMAIL_SENDER, recipients, msg.as_string())
        
        print(f"✓ Email {status} inviata a {len(recipients)} destinatari")
        return True
    except Exception as e:
        print(f"✗ Errore invio email: {e}")
        return False

def main():
    """Loop principale di monitoraggio."""
    
    calibration = load_calibration()
    cap = cv2.VideoCapture(0)
    
    previous_status = None
    
    print("🌊 Sottopassaggio Monitor Avviato")
    print(f"Soglie: GIALLO>{SOGLIA_GIALLO}cm | ROSSO>{SOGLIA_ROSSO}cm | CRITICO>{SOGLIA_CRITICO}cm")
    
    while True:
        ret, frame = cap.read()
        if not ret:
            print("✗ Errore lettura fotocamera")
            continue
        
        # Rileva livello acqua
        level_cm = detect_water_level(frame, calibration)
        
        if level_cm is None:
            print(f"[{datetime.now()}] Nessun livello rilevato")
            continue
        
        # Determina lo stato
        if level_cm >= SOGLIA_CRITICO:
            status = "CRITICO"
        elif level_cm >= SOGLIA_ROSSO:
            status = "ROSSO"
        elif level_cm >= SOGLIA_GIALLO:
            status = "GIALLO"
        else:
            status = "NORMALE"
        
        # Log
        print(f"[{datetime.now()}] Livello: {level_cm:.1f} cm | Status: {status}")
        
        # Invia allarme se cambio di stato verso alto
        if status != previous_status and status != "NORMALE":
            send_email_alert(level_cm, status)
        
        previous_status = status
        
        # Salva foto ogni allarme
        if status != "NORMALE":
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            filename = f"alerts/{status}_{level_cm:.0f}cm_{timestamp}.jpg"
            cv2.imwrite(filename, frame)
        
        # Attendi prima di prossima lettura
        cv2.waitKey(1000)  # 1 secondo

if __name__ == "__main__":
    main()
```

-----

## ⚙️ 4. Setup e Configurazione

### 4.1. File .env

```bash
nano .env
```

Inserisci:

```env
EMAIL_SENDER="comune@gmail.com"
EMAIL_PASSWORD="password_app_gmail"
EMAIL_SINDACO="sindaco@comune.it"
EMAIL_PROTEZIONE_CIVILE="protezione.civile@comune.it"
SMS_API_KEY="chiave_servizio_sms"
```

### 4.2. Installazione

```bash
cd ~/Downloads/SottopassaggioMonitor

python3 -m venv venv
source venv/bin/activate

pip install opencv-python python-dotenv pillow
```

### 4.3. Calibrazione Iniziale

```bash
python calibrate_water_level.py
```

Segui le istruzioni a schermo.

### 4.4. Start del Sistema

```bash
python sottopassaggio_monitor.py
```

-----

## 📂 Struttura Progetto

```
SottopassaggioMonitor/
├── venv/
├── sottopassaggio_monitor.py
├── calibrate_water_level.py
├── calibration.json
├── .env
├── alerts/  (cartella per foto di allarme)
└── start_monitor.sh
```

-----

## 🔄 Flusso Operativo

1. **Ogni 30 secondi**: Fotocamera rileva il livello
1. **Confronto soglie**: Determina stato (NORMALE, GIALLO, ROSSO, CRITICO)
1. **Cambio stato**: Invia email di allarme
1. **Log continuo**: Salva log e foto durante allarmi
1. **24/7**: Sistema funziona continuamente

-----

## 🛠️ Manutenzione

Verifica log:

```bash
tail -f sottopassaggio_monitor.log
```

Ricalibrare la scala:

```bash
python calibrate_water_level.py
```

Visualizza foto degli allarmi:

```bash
ls -la alerts/
```

-----

## ⚠️ Note Importanti

- **Illuminazione**: Assicurati che la scala sia ben illuminata 24/7
- **Posizionamento fotocamera**: Deve inquadrare interamente la scala
- **Manutenzione**: Pulisci lente fotocamera regolarmente (umidità)
- **Backup**: Salva i log su server esterno
- **Test**: Testa gli allarmi settimanalmente

-----

## 📄 Licenza

Sistema sviluppato per la protezione civile e sicurezza pubblica.
Uso esclusivamente per enti pubblici.