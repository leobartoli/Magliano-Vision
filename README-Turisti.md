Perfetto ✅
Ecco la versione finale, formattata professionalmente del tuo README.md — completa di:
	•	codice Python integrato e formattato,
	•	descrizione del tipo di telecamera consigliata,
	•	integrazione del modulo USB 4G per connettività remota,
	•	struttura pronta per GitHub o pubblicazione ufficiale interna del Comune.

⸻

🏛️ Monitoraggio Flusso Turisti con Raspberry Pi e Computer Vision

Progetto istituzionale — Comune di Magliano in Toscana

Sistema automatico e non invasivo per il conteggio degli ingressi e delle uscite di visitatori in aree specifiche di un bene culturale.
Utilizza OpenCV e Centroid Tracking su Raspberry Pi 4, con possibilità di connessione remota tramite modulo USB 4G.

⸻

📖 Concetto e Funzionamento: Flusso (Ingressi/Uscite)

Il sistema analizza il flusso video per tracciare il movimento delle persone e conta quante attraversano una linea virtuale (Tripwire) predefinita, registrando la direzione (Ingresso ↘ o Uscita ↖).

🔧 Tecnologie
	•	Hardware: Raspberry Pi 4 Model B (4GB o 8GB RAM)
	•	Librerie: OpenCV, NumPy, SciPy, dotenv
	•	Algoritmi: Haar Cascade (rilevamento) + Centroid Tracking (tracciamento)
	•	Output: Conteggi numerici (anonimi)
	•	Comunicazione: Email automatica via SMTP o connessione remota 4G

⸻

📷 Tipo di Telecamera Consigliata

Tipo	Modello esempio	Caratteristiche richieste	Note
USB Camera (1080p)	Logitech C920 / C922	Autofocus, 30fps, 1920×1080	Installazione semplice, plug-and-play
Modulo CSI Camera	Raspberry Pi Camera Module 3	Sensore Sony IMX708, HDR, autofocus	Alta qualità e bassa latenza
Telecamera IR (opzionale)	Arducam IR-Cut 1080p	Visione notturna	Ideale per siti poco illuminati

💡 Suggerimento:
Posiziona la telecamera in modo perpendicolare alla linea di passaggio (vista dall’alto o laterale a 45°) per la massima precisione del conteggio.

⸻

📡 Connettività Remota (Modulo USB 4G)

Per siti senza connessione Wi-Fi o LAN, il sistema può essere dotato di un modulo 4G USB compatibile con Raspberry Pi.
Consigliati:

Modulo	Chipset	SIM	Note
Huawei E8372h-320	HiLink	Micro-SIM	Compatibile plug-and-play (interfaccia wwan0)
ZTE MF79U	Qualcomm	Micro-SIM	Supporta connessione automatica all’avvio
Quectel EC25 + HAT USB	Industrial Grade	Nano-SIM	Per installazioni stabili e continue

Configurazione automatica connessione 4G:

sudo nmcli connection add type gsm ifname '*' con-name 4G autoconnect yes apn web.omnitel.it
sudo nmcli connection up 4G


⸻

🧭 Flusso Operativo
	1.	Avvio (Cron): il sistema si avvia automaticamente all’orario di apertura.
	2.	Rilevamento: individua persone tramite Haar Cascade.
	3.	Tracciamento: segue i centroidi nel tempo.
	4.	Conteggio: calcola passaggi oltre la linea virtuale.
	5.	Report: invia il riepilogo via email e rimuove il log locale.

⸻

📋 Requisiti Tecnici

Componente	Requisito	Note
Hardware	Raspberry Pi 4 (4GB/8GB)	Necessario per performance
OS	Raspberry Pi OS (Bullseye o successivo)	Con Python 3.9+
Telecamera	USB / CSI	Posizionamento stabile
Modulo 4G (opzionale)	Huawei E8372h-320 o simili	Per invio email via rete mobile
Email	Gmail	Necessaria “Password per App” se 2FA attiva


⸻

⚙️ Installazione e Setup

Assumiamo che la directory del progetto sia:

~/Downloads/Vision

🧩 Passo 1 — Ambiente Virtuale e Dipendenze

cd ~/Downloads/Vision
python3 -m venv venv
source venv/bin/activate
pip install opencv-python python-dotenv numpy scipy

🧠 Passo 2 — Modello di Rilevamento

wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_fullbody.xml -O haarcascade_fullbody.xml

🔑 Passo 3 — File .env

EMAIL_SENDER="tua_email@gmail.com"
EMAIL_PASSWORD="password_app_gmail"
EMAIL_RECIPIENT="direttore@beneculturale.it"
MONITORING_DURATION_MINUTES=600
COUNTING_LINE_PERCENTAGE=70

Proteggi le credenziali:

chmod 600 .env


⸻

🐍 Script Principale — presenze_flusso.py

import cv2
import numpy as np
import os
from datetime import datetime, timedelta
from dotenv import load_dotenv
import smtplib
from email.mime.text import MIMEText
import time
from scipy.spatial import distance as dist
from collections import OrderedDict

# --- CLASSE CENTROID TRACKER (Leggera per Raspberry Pi) ---
class CentroidTracker:
    def __init__(self, maxDisappeared=30, maxDistance=50):
        self.nextObjectID = 0
        self.objects = OrderedDict()
        self.disappeared = OrderedDict()
        self.trajectories = OrderedDict()
        self.maxDisappeared = maxDisappeared
        self.maxDistance = maxDistance

    def register(self, centroid):
        self.objects[self.nextObjectID] = centroid
        self.disappeared[self.nextObjectID] = 0
        self.trajectories[self.nextObjectID] = [centroid]
        self.nextObjectID += 1

    def deregister(self, objectID):
        del self.objects[objectID]
        del self.disappeared[objectID]
        del self.trajectories[objectID]

    def update(self, rects):
        if len(rects) == 0:
            for objectID in list(self.disappeared.keys()):
                self.disappeared[objectID] += 1
                if self.disappeared[objectID] > self.maxDisappeared:
                    self.deregister(objectID)
            return self.objects

        inputCentroids = np.zeros((len(rects), 2), dtype="int")
        for (i, (x, y, w, h)) in enumerate(rects):
            inputCentroids[i] = (int(x + w / 2), int(y + h / 2))

        if len(self.objects) == 0:
            for i in range(len(inputCentroids)):
                self.register(inputCentroids[i])
        else:
            objectIDs = list(self.objects.keys())
            objectCentroids = list(self.objects.values())
            D = dist.cdist(np.array(objectCentroids), inputCentroids)
            rows = D.min(axis=1).argsort()
            cols = D.argmin(axis=1)[rows]
            usedRows, usedCols = set(), set()
            for (row, col) in zip(rows, cols):
                if row in usedRows or col in usedCols:
                    continue
                if D[row, col] > self.maxDistance:
                    continue
                objectID = objectIDs[row]
                self.objects[objectID] = inputCentroids[col]
                self.disappeared[objectID] = 0
                self.trajectories[objectID].append(inputCentroids[col])
                if len(self.trajectories[objectID]) > 10:
                    self.trajectories[objectID] = self.trajectories[objectID][-10:]
                usedRows.add(row)
                usedCols.add(col)
            unusedRows = set(range(0, D.shape[0])) - usedRows
            unusedCols = set(range(0, D.shape[1])) - usedCols
            if D.shape[0] >= D.shape[1]:
                for row in unusedRows:
                    objectID = objectIDs[row]
                    self.disappeared[objectID] += 1
                    if self.disappeared[objectID] > self.maxDisappeared:
                        self.deregister(objectID)
            else:
                for col in unusedCols:
                    self.register(inputCentroids[col])
        return self.objects
# --- FINE CLASSE ---

load_dotenv()
DURATION = int(os.getenv("MONITORING_DURATION_MINUTES", 600))
LINE_PERCENTAGE = int(os.getenv("COUNTING_LINE_PERCENTAGE", 70))
EMAIL_SENDER = os.getenv("EMAIL_SENDER")
EMAIL_PASSWORD = os.getenv("EMAIL_PASSWORD")
EMAIL_RECIPIENT = os.getenv("EMAIL_RECIPIENT")
CASCADE = cv2.CascadeClassifier("haarcascade_fullbody.xml")
LOG_FILE = "flusso_log.txt"
TRACKER = CentroidTracker(maxDisappeared=30, maxDistance=50)
TOTAL_IN = 0
TOTAL_OUT = 0
OBJECT_COUNTED = {}

def detect_and_track(frame, line_y):
    global TOTAL_IN, TOTAL_OUT
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    rects = CASCADE.detectMultiScale(gray, scaleFactor=1.1, minNeighbors=5, minSize=(30, 60))
    objects = TRACKER.update(rects)
    for (objectID, centroid) in objects.items():
        if len(TRACKER.trajectories[objectID]) < 2:
            continue
        p1_y = TRACKER.trajectories[objectID][-2][1]
        p2_y = centroid[1]
        if objectID not in OBJECT_COUNTED:
            OBJECT_COUNTED[objectID] = False
        if not OBJECT_COUNTED[objectID]:
            if p1_y < line_y and p2_y >= line_y:
                TOTAL_IN += 1
                OBJECT_COUNTED[objectID] = True
            elif p1_y > line_y and p2_y <= line_y:
                TOTAL_OUT += 1
                OBJECT_COUNTED[objectID] = True

def write_log(msg):
    ts = datetime.now().strftime("%H:%M:%S")
    with open(LOG_FILE, 'a') as f:
        f.write(f"[{ts}] {msg}\n")

def send_email():
    if not all([EMAIL_SENDER, EMAIL_PASSWORD, EMAIL_RECIPIENT]):
        print("✗ Credenziali email mancanti.")
        return
    with open(LOG_FILE, 'r') as f:
        log = f.read()
    msg = MIMEText(f"Report Flusso Turisti:\n\n{log}\n\nTOTALE INGRESSI: {TOTAL_IN}\nTOTALE USCITE: {TOTAL_OUT}")
    msg['Subject'] = f"Report Flusso Affluenza - {datetime.now().strftime('%d/%m/%Y')}"
    msg['From'] = EMAIL_SENDER
    msg['To'] = EMAIL_RECIPIENT
    with smtplib.SMTP_SSL('smtp.gmail.com', 465) as server:
        server.login(EMAIL_SENDER, EMAIL_PASSWORD)
        server.sendmail(EMAIL_SENDER, EMAIL_RECIPIENT, msg.as_string())
    print("✓ Email inviata con successo")

def main():
    global TOTAL_IN, TOTAL_OUT
    with open(LOG_FILE, 'w') as f:
        f.write(f"Inizio monitoraggio: {datetime.now()}\nDurata: {DURATION} min\n{'-'*40}\n")
    cap = cv2.VideoCapture(0)
    if not cap.isOpened():
        write_log("Errore: fotocamera non disponibile.")
        send_email()
        return
    ret, frame = cap.read()
    FRAME_HEIGHT = frame.shape[0]
    LINE_Y = int(FRAME_HEIGHT * (LINE_PERCENTAGE / 100))
    write_log(f"Linea Virtuale Y: {LINE_Y} ({LINE_PERCENTAGE}%)")
    end_time = datetime.now() + timedelta(minutes=DURATION)
    while datetime.now() < end_time:
        ret, frame = cap.read()
        if not ret:
            time.sleep(1)
            continue
        detect_and_track(frame, LINE_Y)
        cv2.waitKey(1)
    cap.release()
    cv2.destroyAllWindows()
    write_log(f"TOTALE INGRESSI: {TOTAL_IN}")
    write_log(f"TOTALE USCITE: {TOTAL_OUT}")
    send_email()
    os.remove(LOG_FILE)
    print("✓ Monitoraggio completato.")

if __name__ == "__main__":
    main()


⸻

⚙️ Script di Avvio — start_flusso_monitor.sh

#!/bin/bash
PROJECT_DIR="/home/settoretecnico/Downloads/Vision"
cd "$PROJECT_DIR" || exit 1
source venv/bin/activate
python presenze_flusso.py >> flusso_monitoraggio_output.log 2>&1
deactivate


⸻

⏰ Automazione (Cron)

crontab -e

Aggiungi:

0 8 * * * /home/settoretecnico/Downloads/Vision/start_flusso_monitor.sh


⸻

📁 Struttura Directory

~/Downloads/Vision/
├── presenze_flusso.py
├── start_flusso_monitor.sh
├── haarcascade_fullbody.xml
├── .env
├── venv/
├── flusso_monitoraggio_output.log
└── README.md


⸻

🧾 Licenza e Crediti

Progetto per il Comune di Magliano in Toscana
Settore Tecnico — Servizio Innovazione Digitale
Basato su Python + OpenCV
Versione: 1.0 (2025)

⸻

Vuoi che ti generi anche la versione PDF del README formattato (con intestazione istituzionale e logo del Comune in alto)? Sarebbe perfetta per allegarla come documentazione tecnica ufficiale.