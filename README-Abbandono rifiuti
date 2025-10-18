# 🗑️ Rifiuti Monitor: Sistema di Rilevamento Abbandono Rifiuti (Raspberry Pi)

Sistema di Computer Vision ottimizzato per Raspberry Pi 4, progettato per rilevare e segnalare l’abbandono di rifiuti nel territorio comunale.

-----

## 🎯 Obiettivo

Logica condizionale per monitoraggio efficiente:

|Fase                    |Descrizione                                                             |
|------------------------|------------------------------------------------------------------------|
|**Monitoraggio Attivo** |Cattura frame ogni 5 secondi, analizza colore/forma per rilevare rifiuti|
|**Rilevamento Anomalia**|Quando rileva rifiuti, scatta foto ad alta risoluzione e invia notifica |
|**Segnalazione e Log**  |Registra coordinate GPS (opzionale), timestamp e foto in database locale|
|**Resilienza**          |Riavvio automatico dopo interruzioni                                    |

-----

## 🧠 1. Architettura CV

|Componente             |Descrizione                                                |
|-----------------------|-----------------------------------------------------------|
|**Motore CV**          |Haar Cascade + Color Detection (HSV per rifiuti neri/grigi)|
|**Alternativa Leggera**|YOLO Nano per maggiore accuratezza (opzionale)             |
|**Database Locale**    |SQLite per log eventi e foto                               |
|**GPS**                |Modulo u-blox NEO-6M (opzionale, per coordinate)           |

-----

## ⚙️ 2. Setup Iniziale

```bash
cd ~/Downloads/RifiutiMonitor

# Ambiente virtuale
python3 -m venv venv
source venv/bin/activate

# Dipendenze
pip install opencv-python python-dotenv sqlite3 pillow

# Modello (Haar Cascade generico)
wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_frontalface_default.xml -O haarcascade_objects.xml
```

-----

## 📧 3. Configurazione .env

```bash
nano .env
```

```env
EMAIL_SENDER="comune_alert@gmail.com"
EMAIL_PASSWORD="password_app_gmail"
EMAIL_RECIPIENTS="sindaco@comune.it,vigili@comune.it,ambiente@comune.it"
EMAIL_SUBJECT="⚠️ ABBANDONO RIFIUTI RILEVATO"

# GPS (opzionale)
ENABLE_GPS=false
GPS_PORT="/dev/ttyUSB0"
GPS_BAUDRATE=9600

# Sensibilità rilevamento (0-255)
TRASH_COLOR_LOWER_HSV="0,0,0"
TRASH_COLOR_UPPER_HSV="180,50,100"

# Percorsi
PHOTOS_DIR="/home/settoretecnico/RifiutiMonitor/photos"
DB_PATH="/home/settoretecnico/RifiutiMonitor/rifiuti.db"

# Soglie
MIN_CONTOUR_AREA=500
DETECTION_COOLDOWN=300
```

```bash
chmod 600 .env
```

-----

## 🚀 4. Script di Avvio

```bash
nano start_rifiuti_monitor.sh
```

```bash
#!/bin/bash

cd /home/settoretecnico/RifiutiMonitor
source venv/bin/activate
nohup python rifiuti_monitor.py > monitor_output.log 2>&1 &
deactivate
```

```bash
chmod +x start_rifiuti_monitor.sh
```

-----

## 📋 5. Cron per Resilienza

```bash
crontab -e
```

```cron
# Riavvia dopo reboot
@reboot /home/settoretecnico/RifiutiMonitor/start_rifiuti_monitor.sh

# Health check ogni 30 minuti
*/30 * * * * /home/settoretecnico/RifiutiMonitor/health_check.sh

# Pulizia foto vecchie (> 7 giorni) ogni notte
0 3 * * * /home/settoretecnico/RifiutiMonitor/cleanup_old_photos.sh
```

-----

## 🧩 6. Logica dello Script Principale

**rifiuti_monitor.py:**

|Azione                  |Descrizione                              |Intervallo    |
|------------------------|-----------------------------------------|--------------|
|**Cattura Frame**       |Legge stream dalla fotocamera            |5 secondi     |
|**Color Detection**     |Analizza HSV per rifiuti neri/grigi      |Real-time     |
|**Contour Analysis**    |Filtra per area e forma (scarta fogliame)|Real-time     |
|**Rilevamento Positivo**|Scatta foto HD, salva GPS, invia email   |Al rilevamento|
|**Cooldown**            |Evita duplicati per 5 minuti             |300 sec       |

**Pseudocodice:**

```
while True:
    frame = camera.read()
    hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
    
    # Mascara per colori scuri (rifiuti)
    mask = cv2.inRange(hsv, lower_trash, upper_trash)
    contours = cv2.findContours(mask)
    
    for contour in contours:
        area = cv2.contourArea(contour)
        if area > MIN_CONTOUR_AREA and not in_cooldown():
            # Rilevamento positivo
            save_hd_photo()
            get_gps_coordinates()
            send_alert_email()
            log_to_database()
            start_cooldown(300 sec)
    
    time.sleep(5)
```

-----

## 📂 Struttura del Progetto

```
RifiutiMonitor/
├── venv/
├── rifiuti_monitor.py          # Script principale
├── start_rifiuti_monitor.sh    # Launcher
├── health_check.sh             # Verifica health
├── cleanup_old_photos.sh       # Pulizia automatica
├── haarcascade_objects.xml     # Modello CV (opzionale)
├── .env
├── rifiuti.db                  # Database SQLite
├── photos/                     # Archivio foto
│   └── YYYYMMDD_HHMMSS_trash.jpg
└── monitor_output.log
```

-----

## 🌐 7. Opzione GPS (NEO-6M)

### Hardware

- Modulo u-blox NEO-6M (€10-20)
- Antenna GPS esterna (migliore ricezione)
- Connessione UART al Raspberry Pi

### Installazione Software

```bash
sudo apt install -y gpsd gpsd-clients python3-gps

# Configura UART su GPIO 14/15
sudo nano /boot/firmware/config.txt
# Aggiungi: dtoverlay=uart0
# Reboot

# Avvia gpsd
sudo systemctl enable gpsd
sudo systemctl start gpsd
```

### Uso in Python

```python
import gps
session = gps.gps("localhost", "2947")
lat, lon = session.fix.lat, session.fix.lon
# Salva coordinate con foto
```

-----

## 📊 8. Dashboard Web (Opzionale)

Per visualizzare segnalazioni in tempo reale:

```bash
pip install flask
```

**app.py** (server web locale):

```python
from flask import Flask, render_template, jsonify
import sqlite3

app = Flask(__name__)

@app.route('/api/alerts')
def get_alerts():
    conn = sqlite3.connect('rifiuti.db')
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM alerts ORDER BY timestamp DESC LIMIT 100')
    data = cursor.fetchall()
    conn.close()
    return jsonify(data)

@app.route('/')
def index():
    return render_template('dashboard.html')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Accedi da browser: `http://[IP_RASPBERRY]:5000`

-----

## 🌐 9. Connettività 4G (Consigliato)

Identico al setup Parco Monitor:

- Dongle USB 4G Huawei E3372 (€40)
- SIM dati comunale (€10-15/mese)
- Installazione: plug & play
- Verifica: `ping 8.8.8.8`

-----

## 🔧 10. Manutenzione

```bash
# Visualizza log
tail -f ~/RifiutiMonitor/monitor_output.log

# Verifica database
sqlite3 rifiuti.db "SELECT COUNT(*) FROM alerts;"

# Riavvia servizio
./start_rifiuti_monitor.sh

# Pulisci foto manualmente
rm -rf photos/YYYYMMDD*

# Aggiorna dipendenze
source venv/bin/activate
pip install --upgrade opencv-python
deactivate
```

-----

## 📧 11. Template Email di Notifica

**Oggetto:**

```
⚠️ ABBANDONO RIFIUTI RILEVATO - [Data/Ora]
```

**Corpo:**

```
Rilevamento Automatico - Sistema Rifiuti Monitor

📍 Posizione: [Indirizzo/Coordinate GPS]
⏰ Data/Ora: [Timestamp]
🎥 Fotografia: [Allegato]
📊 Intensità: [Bassa/Media/Alta]

Coordinate GPS: [Lat,Lon]
URL Maps: https://maps.google.com/?q=[LAT],[LON]

Azione consigliata: Verifica sul posto e rimozione rifiuti
```

-----

## 🛡️ 12. Miglioramenti Futuri

- Integrazione con API Comune (creazione ticket automatico)
- ML fine-tuning per ridurre falsi positivi
- Heatmap di zone critiche nel comune
- Integrazione con app mobile per operatori
- Rilevamento tipo rifiuti (carta, plastica, ingombranti)

-----

## 📄 Licenza

MIT - Creato per il monitoraggio ambientale automatizzato del territorio comunale