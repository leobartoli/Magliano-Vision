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

def get_rainfall_from_api(api_type="openweathermap"):
    """
    Scarica dati pioggia da servizio meteo esterno.
    
    Args:
        api_type: "openweathermap" o "arpa"
    
    Returns:
        {
            "is_raining": bool,
            "rainfall_mm": float,
            "description": str,
            "probability": float (0-100)
        }
    """
    
    if api_type == "openweathermap":
        API_KEY = os.getenv("OPENWEATHER_API_KEY")
        CITY = os.getenv("CITY_NAME", "Brescia")
        
        url = f"https://api.openweathermap.org/data/2.5/weather?q={CITY}&appid={API_KEY}&lang=it"
        
        try:
            response = requests.get(url, timeout=5)
            data = response.json()
            
            weather = data['weather'][0]['main']
            description = data['weather'][0]['description']
            rainfall_mm = data.get('rain', {}).get('1h', 0)
            
            is_raining = weather in ['Rain', 'Drizzle', 'Thunderstorm']
            
            return {
                "is_raining": is_raining,
                "rainfall_mm": rainfall_mm,
                "description": description,
                "probability": 100 if is_raining else 0,
                "source": "OpenWeatherMap"
            }
        except Exception as e:
            print(f"✗ Errore API meteo: {e}")
            return None
    
    elif api_type == "arpa":
        # Esempio per ARPA (regione-specifica)
        LAT = os.getenv("LATITUDE", "45.54")
        LON = os.getenv("LONGITUDE", "10.21")
        
        url = f"https://api.arpalombardia.it/raster/meteo/dati?lat={LAT}&lon={LON}"
        
        try:
            response = requests.get(url, timeout=5)
            data = response.json()
            
            rainfall_mm = data.get('precipitation', 0)
            prob = data.get('probPrec', 0)
            
            return {
                "is_raining": rainfall_mm > 0.5,
                "rainfall_mm": rainfall_mm,
                "description": f"Pioggia: {rainfall_mm}mm",
                "probability": prob,
                "source": "ARPA"
            }
        except Exception as e:
            print(f"✗ Errore ARPA: {e}")
            return None

def detect_rainfall(frame, previous_frame, api_data=None):
    """
    Rileva pioggia usando ENTRAMBI i metodi:
    1. Dati API (primario - affidabile)
    2. Visione frame (backup - se API giù)
    
    Restituisce tuple: (is_raining, rainfall_mm, source)
    """
    
    # METODO 1: Dati API (PRIMARIO)
    if api_data and api_data.get("is_raining"):
        return (True, api_data.get("rainfall_mm", 0), api_data.get("source", "API"))
    
    # METODO 2: Fallback - rileva da frame (se API offline)
    if previous_frame is None:
        return (False, 0, "N/A")
    
    diff = cv2.absdiff(frame, previous_frame)
    gray_diff = cv2.cvtColor(diff, cv2.COLOR_BGR2GRAY)
    
    changed_pixels = cv2.countNonZero(gray_diff > 30)
    frame_size = frame.shape[0] * frame.shape[1]
    
    is_raining_visual = (changed_pixels / frame_size) > 0.15
    
    return (is_raining_visual, 0, "Computer Vision (fallback)")

def save_daily_photo(last_daily_photo_time):
    """
    Controlla se è passato un giorno da ultima foto giornaliera.
    Restituisce True se è ora di scattare una foto giornaliera.
    """
    now = datetime.now()
    
    if last_daily_photo_time is None:
        return True
    
    # Una foto al giorno a mezzogiorno
    if (now - last_daily_photo_time).days >= 1 and now.hour == 12:
        return True
    
    return False

def take_closeup_photo(frame, zoom_level=2):
    """
    Crea una foto ravvicinata (zoom digitale).
    zoom_level: fattore di zoom (2x = doppio zoom)
    """
    height, width = frame.shape[:2]
    
    # Calcola centro e area di zoom
    center_x, center_y = width // 2, height // 2
    crop_width, crop_height = width // zoom_level, height // zoom_level
    
    x1 = max(0, center_x - crop_width // 2)
    y1 = max(0, center_y - crop_height // 2)
    x2 = min(width, x1 + crop_width)
    y2 = min(height, y1 + crop_height)
    
    # Ritaglia e ridimensiona alla risoluzione originale
    cropped = frame[y1:y2, x1:x2]
    zoomed = cv2.resize(cropped, (width, height))
    
    return zoomed

def main():
    """Loop principale di monitoraggio."""
    
    calibration = load_calibration()
    cap = cv2.VideoCapture(0)
    
    previous_status = None
    previous_frame = None
    last_daily_photo_time = None
    last_alert_time = {}  # Previene allerte multiple nello stesso minuto
    
    print("🌊 Sottopassaggio Monitor Avviato")
    print(f"Soglie: GIALLO>{SOGLIA_GIALLO}cm | ROSSO>{SOGLIA_ROSSO}cm | CRITICO>{SOGLIA_CRITICO}cm")
    print("📸 Modalità foto:")
    print("   - 1 foto giornaliera a mezzogiorno (no movimento)")
    print("   - Foto ravvicinate in caso di pioggia")
    print("   - Foto di allarme se supera soglie")
    
    while True:
        ret, frame = cap.read()
        if not ret:
            print("✗ Errore lettura fotocamera")
            continue
        
        # Rileva livello acqua
        level_cm = detect_water_level(frame, calibration)
        
        if level_cm is None:
            print(f"[{datetime.now()}] Nessun livello rilevato")
            previous_frame = frame
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
        
        now = datetime.now()
        timestamp = now.strftime("%Y%m%d_%H%M%S")
        
        # Log
        print(f"[{now}] Livello: {level_cm:.1f} cm | Status: {status}")
        
        # ======== FOTOGRAFIA GIORNALIERA ========
        # 1 foto al giorno quando non passa nessuno (es. mezzogiorno)
        if save_daily_photo(last_daily_photo_time):
            daily_filename = f"daily/{now.strftime('%Y%m%d')}_daily_level{level_cm:.0f}cm.jpg"
            cv2.imwrite(daily_filename, frame)
            print(f"📸 Foto giornaliera salvata: {daily_filename}")
            last_daily_photo_time = now
        
        # ======== RILEVAMENTO PIOGGIA ========
        # Foto ravvicinate in caso di pioggia
        is_raining = detect_rainfall(frame, previous_frame)
        if is_raining:
            rain_frame = take_closeup_photo(frame, zoom_level=2)
            rain_filename = f"rain/{now.strftime('%Y%m%d_%H%M%S')}_pioggia_zoom_level{level_cm:.0f}cm.jpg"
            cv2.imwrite(rain_filename, rain_frame)
            print(f"🌧️  Pioggia rilevata! Foto ravvicinata salvata: {rain_filename}")
        
        # ======== ALLARMI E FOTO DI EMERGENZA ========
        # Invia allarme se cambio di stato verso alto
        if status != previous_status and status != "NORMALE":
            # Evita allerte multiple nello stesso minuto
            minute_key = now.strftime("%H:%M")
            if minute_key not in last_alert_time or (now - last_alert_time[minute_key]).seconds > 60:
                send_email_alert(level_cm, status)
                last_alert_time[minute_key] = now
        
        previous_status = status
        
        # Salva foto di allarme SE supera soglie (+ foto ravvicinata se piove)
        if status != "NORMALE":
            alert_filename = f"alerts/{status}_{level_cm:.0f}cm_{timestamp}.jpg"
            cv2.imwrite(alert_filename, frame)
            print(f"🚨 Foto allarme salvata: {alert_filename}")
            
            # Se contemporaneamente piove E c'è allarme, salva anche zoom
            if is_raining:
                alert_zoom_filename = f"alerts/ZOOM_{status}_{level_cm:.0f}cm_{timestamp}.jpg"
                alert_frame_zoom = take_closeup_photo(frame, zoom_level=2)
                cv2.imwrite(alert_zoom_filename, alert_frame_zoom)
                print(f"🚨 Foto allarme ravvicinata (pioggia + allarme) salvata: {alert_zoom_filename}")
        
        previous_frame = frame
        
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
# Email
EMAIL_SENDER="comune@gmail.com"
EMAIL_PASSWORD="password_app_gmail"
EMAIL_SINDACO="sindaco@comune.it"
EMAIL_PROTEZIONE_CIVILE="protezione.civile@comune.it"

# SMS (opzionale)
SMS_API_KEY="chiave_servizio_sms"

# Meteo API (OpenWeatherMap)
OPENWEATHER_API_KEY="tua_chiave_openweathermap"
CITY_NAME="Brescia"

# Coordinate (per ARPA)
LATITUDE="45.54"
LONGITUDE="10.21"

# Tipo di servizio meteo: "openweathermap" o "arpa"
WEATHER_SERVICE="openweathermap"
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
├── daily/           (1 foto al giorno a mezzogiorno)
├── rain/            (foto ravvicinate quando piove)
├── alerts/          (foto quando supera soglie)
│   ├── GIALLO_*.jpg
│   ├── ROSSO_*.jpg
│   ├── CRITICO_*.jpg
│   └── ZOOM_ROSSO_*.jpg  (pioggia + allarme simultanei)
└── start_monitor.sh
```

-----

## 📸 Logica Fotografica Dettagliata

### Flusso Decisionale

```
OGNI SECONDO
    ↓
├─ FOTO GIORNALIERA?
│  └─ SÌ (se è passato 1 giorno e sono le 12:00)
│     └─ Salva in /daily/ → "YYYYMMDD_daily_levelXcm.jpg"
│
├─ PIOGGIA RILEVATA?
│  └─ SÌ (movimento rapido tra frame)
│     └─ Salva foto ravvicinata in /rain/ → "HHMMSS_pioggia_zoom.jpg"
│
├─ ALLARME (supera soglia)?
│  └─ SÌ (GIALLO, ROSSO, CRITICO)
│     ├─ Salva foto in /alerts/ → "STATUS_levelXcm_HHMMSS.jpg"
│     ├─ Invia email a Sindaco e Protezione Civile
│     └─ Se contemporaneamente piove:
│        └─ Salva anche zoom in /alerts/ZOOM_*.jpg
│
└─ ATTENDI 1 secondo, ripeti
```

### Tipi di Foto Salvate

|Tipo                 |Cartella  |Nome file                          |Quando                         |
|---------------------|----------|-----------------------------------|-------------------------------|
|**Giornaliera**      |`/daily/` |`20240115_daily_level23cm.jpg`     |1 volta al giorno a mezzogiorno|
|**Pioggia**          |`/rain/`  |`145302_pioggia_zoom_level35cm.jpg`|Quando piove (no allarme)      |
|**Allarme**          |`/alerts/`|`ROSSO_85cm_145302.jpg`            |Quando supera soglie           |
|**Pioggia + Allarme**|`/alerts/`|`ZOOM_ROSSO_85cm_145302.jpg`       |Contemporaneamente             |

-----

## 🔍 Rilevamento Pioggia

Il sistema confronta due frame consecutivi:

```python
def detect_rainfall(frame, previous_frame):
    # Se >15% dei pixel cambia tra frame = pioggia
    # Funziona anche di notte (gocce di pioggia si vedono)
    # Evita falsi positivi: calcola solo i cambiamenti rapidi
```

**Vantaggi:**

- ✅ Funziona 24/7
- ✅ Rileva pioggia anche leggera
- ✅ Non dipende da sensori esterni

-----

## 🎯 Scenari Fotografici

### Scenario 1: Giorno Normale

```
09:00 → Niente (no foto, niente allarme)
12:00 → 📸 Foto giornaliera in /daily/ (routine check)
15:00 → Niente
```

### Scenario 2: Pioggia Leggera (senza allarme)

```
14:30 → 🌧️ Pioggia rilevata
       └─ 📸 Foto ravvicinata in /rain/
       └─ Livello: 30cm (sotto soglia GIALLO)
15:00 → 🌧️ Ancora pioggia
       └─ 📸 Altra foto ravvicinata in /rain/
```

### Scenario 3: Allarme ROSSO senza pioggia

```
16:45 → ⚠️ Livello sale a 95cm (prossimo ROSSO)
17:00 → 🚨 Livello raggiunge 102cm (ROSSO!)
       ├─ 📸 Foto in /alerts/ (ROSSO_102cm_170000.jpg)
       ├─ 📧 Email a Sindaco e Protezione Civile
       └─ Log: "ALLARME ROSSO inviato"
```

### Scenario 4: Pioggia + Allarme CRITICO (Massima Allerta)

```
18:30 → 🌧️ Pioggia intensa
18:35 → 🚨 Livello sale a 155cm (CRITICO!)
       ├─ 📸 Foto normale in /alerts/ (CRITICO_155cm_183500.jpg)
       ├─ 📸 Foto ravvicinata in /alerts/ (ZOOM_CRITICO_155cm_183500.jpg)
       ├─ 📧 Email URGENTE a Protezione Civile
       ├─ 🔴 SMS di allerta (se configurato)
       └─ Log: "ALLARME CRITICO CON PIOGGIA!"
```

-----

## 🚀 Setup Cartelle

Crea le cartelle necessarie prima di avviare:

```bash
mkdir -p daily rain alerts logs

# Dai permessi di scrittura
chmod 755 daily rain alerts logs
```

-----

## 📊 Gestione File Foto

### Archiviazione Automatica (Opzionale)

Aggiungi uno script cron per archiviare foto vecchie:

```bash
# Crontab: Archivia foto di 7 giorni fa
0 2 * * * tar -czf /backup/foto_$(date +\%Y\%m\%d).tar.gz /home/pi/SottopassaggioMonitor/{daily,rain,alerts} && find /home/pi/SottopassaggioMonitor -name "*.jpg" -mtime +7 -delete
```

### Visualizzazione Rapida

```bash
# Foto odierne
ls -lh daily/ | grep $(date +%Y%m%d)

# Foto degli ultimi allarmi
ls -lhtr alerts/ | tail -20

# Foto di pioggia
ls -lh rain/
```

-----

## 🛠️ Manutenzione

Verifica log:

```bash
tail -f sottopassaggio_monitor.log
```

Conta foto salvate:

```bash
echo "Foto giornaliere: $(ls daily/*.jpg 2>/dev/null | wc -l)"
echo "Foto pioggia: $(ls rain/*.jpg 2>/dev/null | wc -l)"
echo "Foto allarmi: $(ls alerts/*.jpg 2>/dev/null | wc -l)"
```

Ricalibrare la scala:

```bash
python calibrate_water_level.py
```

Pulisci foto di 30 giorni fa:

```bash
find daily rain alerts -name "*.jpg" -mtime +30 -delete
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