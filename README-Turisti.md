Certamente. Ho combinato tutte le sezioni di documentazione, i requisiti e il codice Python completo aggiornato per il Conteggio del Flusso (Ingressi/Uscite) in un unico file README.md.
Questo file include tutte le istruzioni e gli script necessari per l'installazione e l'avvio del sistema.
🏛️ README.md: Monitoraggio Flusso Turisti con Raspberry Pi e Computer Vision
Progetto: Sistema automatico e non invasivo per il conteggio degli ingressi e delle uscite di visitatori (turisti) in aree specifiche di un bene culturale. Utilizza OpenCV e Centroid Tracking su Raspberry Pi.
📖 Concetto e Funzionamento: Flusso (Ingressi/Uscite)
Il sistema analizza il flusso video per tracciare il movimento delle persone e conta quante di esse attraversano una linea virtuale (Tripwire) predefinita, registrando la direzione del movimento (Ingresso o Uscita).
 * Tecnologie: Raspberry Pi 4, OpenCV, Haar Cascade (per il rilevamento) e Centroid Tracking (per il tracciamento del movimento).
 * Privacy: Non salva alcuna immagine o video. Registra solo i conteggi numerici.
 * Reportistica: I conteggi finali (Ingressi Totali, Uscite Totali) vengono inviati via email al termine del monitoraggio.
Flusso Operativo
 * Avvio (Cron): Il sistema si avvia automaticamente all’orario di apertura.
 * Detection & Tracking: Rileva i corpi umani (haarcascade_fullbody.xml) e traccia il centroide di ciascuno attraverso i frame.
 * Conteggio: Controlla se il centroide tracciato ha attraversato la Linea Virtuale in una direzione specifica (dal basso all'alto per "Uscita" o dall'alto al basso per "Ingresso", a seconda della configurazione).
 * Chiusura: Al termine, invia il report email e cancella il log locale.
📋 Requisiti Tecnici
| Componente | Requisito | Note |
|---|---|---|
| Hardware | Raspberry Pi 4 | Obbligatorio per le prestazioni di tracking richieste. |
| OS | Raspberry Pi OS | Compatibile con Python, OpenCV e Scipy. |
| Telecamera | USB o CSI | Posizionamento cruciale per definire una chiara "linea di transito". |
| Email | Account Gmail | Obbligatorio usare una "Password per App" se l'autenticazione a due fattori è attiva. |
⚙️ Installazione e Setup
Assumi che la directory del progetto sia ~/Downloads/Vision.
Passo 3.1: Dipendenze Python e Ambiente
Crea e attiva l’ambiente virtuale e installa le librerie richieste.
cd ~/Downloads/Vision
# source venv/bin/activate  # Attiva l'ambiente se non già attivo
pip install opencv-python python-dotenv numpy scipy

Passo 3.2: Modello di Rilevamento (Haar Cascade)
Scarica il file del modello pre-addestrato nella directory del progetto.
wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_fullbody.xml -O haarcascade_fullbody.xml

Passo 3.3: Configurazione delle Credenziali (.env)
Crea un file chiamato .env nella directory principale e inserisci i dettagli di configurazione, inclusa la posizione della Linea Virtuale.
nano .env

EMAIL_SENDER="tua_email@gmail.com"
EMAIL_PASSWORD="password_app_gmail"
EMAIL_RECIPIENT="direttore@beneculturale.it"
MONITORING_DURATION_MINUTES=600  # Esempio: 10 ore (08:00 - 18:00)
COUNTING_LINE_PERCENTAGE=70      # Linea Virtuale: coordinata Y al 70% dell'altezza del frame

Proteggi il file .env per la sicurezza delle credenziali:
chmod 600 .env

🐍 Script Python: presenze_flusso.py (Codice Completo)
Crea questo file e copia il codice al suo interno. È il cuore del sistema di tracciamento e conteggio.
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

# --- CLASSE CENTROID TRACKER (Leggera e customizzata per Raspberry Pi) ---
class CentroidTracker:
    def __init__(self, maxDisappeared=30, maxDistance=50):
        self.nextObjectID = 0
        self.objects = OrderedDict() 
        self.disappeared = OrderedDict()
        self.trajectories = OrderedDict() # Memorizza i punti per calcolare la direzione

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
            cX = int(x + w / 2.0)
            cY = int(y + h / 2.0)
            inputCentroids[i] = (cX, cY)

        if len(self.objects) == 0:
            for i in range(0, len(inputCentroids)):
                self.register(inputCentroids[i])
        
        else:
            objectIDs = list(self.objects.keys())
            objectCentroids = list(self.objects.values())

            D = dist.cdist(np.array(objectCentroids), inputCentroids)
            rows = D.min(axis=1).argsort()
            cols = D.argmin(axis=1)[rows]

            usedRows = set()
            usedCols = set()

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

# --- FINE CLASSE CENTROID TRACKER ---

load_dotenv()

# Configurazione del Monitoraggio
DURATION = int(os.getenv("MONITORING_DURATION_MINUTES", 600))
LINE_PERCENTAGE = int(os.getenv("COUNTING_LINE_PERCENTAGE", 70))

# Email Configuration
EMAIL_SENDER = os.getenv("EMAIL_SENDER")
EMAIL_PASSWORD = os.getenv("EMAIL_PASSWORD")
EMAIL_RECIPIENT = os.getenv("EMAIL_RECIPIENT")

# Modello di Rilevamento e Log
CASCADE = cv2.CascadeClassifier("haarcascade_fullbody.xml")
LOG_FILE = "flusso_log.txt"

# Variabili Globali per il Conteggio
TRACKER = CentroidTracker(maxDisappeared=30, maxDistance=50) 
TOTAL_IN = 0
TOTAL_OUT = 0
# Dizionario per impedire il doppio conteggio dello stesso attraversamento
OBJECT_COUNTED = {} 

def detect_and_track(frame, line_y):
    """
    Rileva le persone, le traccia e aggiorna i contatori di Ingresso/Uscita.
    """
    global TOTAL_IN, TOTAL_OUT

    if CASCADE.empty():
        return

    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    
    # Rilevamento (trova le nuove posizioni)
    rects = CASCADE.detectMultiScale(
        gray, 
        scaleFactor=1.1, 
        minNeighbors=5, 
        minSize=(30, 60), 
        flags=cv2.CASCADE_SCALE_IMAGE
    )
    
    # Aggiorna il tracker con le nuove posizioni
    objects = TRACKER.update(rects)
    
    # Itera su tutti gli oggetti tracciati
    for (objectID, centroid) in objects.items():
        
        # Assicurati che ci siano almeno 2 punti per calcolare la direzione
        if len(TRACKER.trajectories[objectID]) < 2:
            continue
        
        p1_y = TRACKER.trajectories[objectID][-2][1] # Vecchio punto Y
        p2_y = centroid[1]                          # Nuovo punto Y
        
        if objectID not in OBJECT_COUNTED:
            OBJECT_COUNTED[objectID] = False

        # 2. LOGICA DI CONTEGGIO: Se l'oggetto ha attraversato la linea E non è ancora stato contato
        if not OBJECT_COUNTED[objectID]:
            
            # Ingresso (Movimento verso il basso: sopra la linea -> sotto la linea)
            if p1_y < line_y and p2_y >= line_y:
                TOTAL_IN += 1
                OBJECT_COUNTED[objectID] = True 

            # Uscita (Movimento verso l'alto: sotto la linea -> sopra la linea)
            elif p1_y > line_y and p2_y <= line_y:
                TOTAL_OUT += 1
                OBJECT_COUNTED[objectID] = True 

def write_log(msg):
    """Scrive un messaggio nel file di log con timestamp."""
    ts = datetime.now().strftime("%H:%M:%S")
    line = f"[{ts}] {msg}"
    
    try:
        with open(LOG_FILE, 'a') as f:
            f.write(line + "\n")
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
        
        today_date = datetime.now().strftime('%d/%m/%Y')
        # Aggiunge i risultati finali al corpo dell'email
        msg = MIMEText(f"Report Flusso Turisti:\n\n{log}\n\nTOTALE INGRESSI: {TOTAL_IN}\nTOTALE USCITE: {TOTAL_OUT}")
        msg['Subject'] = f"Report Flusso Affluenza - {today_date}"
        msg['From'] = EMAIL_SENDER
        msg['To'] = EMAIL_RECIPIENT
        
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
    """Loop principale di monitoraggio del flusso."""
    global TOTAL_IN, TOTAL_OUT

    # Inizializzazione Log
    try:
        with open(LOG_FILE, 'w') as f:
            f.write(f"Inizio monitoraggio flusso: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n")
            f.write(f"Durata configurata: {DURATION} minuti\n")
            f.write("-" * 50 + "\n\n")
    except IOError:
        print("✗ Impossibile creare il file di log. Esco.")
        return
    
    # Apri fotocamera (indice 0 è tipico)
    cap = cv2.VideoCapture(0)
    
    if not cap.isOpened():
        write_log("⚠️ Errore: Impossibile aprire la fotocamera.")
        print("✗ Fotocamera non disponibile. Esco.")
        send_email() 
        return
    
    # Calcola la posizione della Linea Virtuale
    ret, frame = cap.read()
    if not ret:
        print("✗ Impossibile leggere il primo frame. Esco.")
        cap.release()
        return

    FRAME_HEIGHT = frame.shape[0]
    COUNTING_LINE_Y = int(FRAME_HEIGHT * (LINE_PERCENTAGE / 100))
    
    write_log(f"Fotocamera aperta. Linea Virtuale Y: {COUNTING_LINE_Y} ({LINE_PERCENTAGE}%)")

    start_time = datetime.now()
    end_time = start_time + timedelta(minutes=DURATION)
    
    # Loop di monitoraggio
    while datetime.now() < end_time:
        ret, frame = cap.read()
        
        if not ret:
            time.sleep(1)
            continue
        
        # Esegue Rilevamento, Tracking e Conteggio
        detect_and_track(frame, COUNTING_LINE_Y)
        
        # Rimuove gli ID non più tracciati dal dizionario di conteggio
        for objectID in list(OBJECT_COUNTED.keys()):
            if objectID not in TRACKER.objects:
                del OBJECT_COUNTED[objectID]
                
        # Breve attesa per non sovraccaricare la CPU
        cv2.waitKey(1) 
    
    # Ferma la telecamera e termina
    cap.release()
    cv2.destroyAllWindows()
    
    write_log(f"Monitoraggio terminato. Risultati finali:")
    write_log(f"TOTALE INGRESSI: {TOTAL_IN}")
    write_log(f"TOTALE USCITE: {TOTAL_OUT}")
    
    print("\nInvio report via email...")
    
    send_email()
    
    # Elimina file di log locale
    try:
        if os.path.exists(LOG_FILE):
            os.remove(LOG_FILE)
    except Exception as e:
        print(f"Avviso: Impossibile eliminare il file di log locale: {e}")

    print("✓ Monitoraggio Flusso completato.")

if __name__ == "__main__":
    main()

⚙️ Script Bash: start_flusso_monitor.sh
Crea questo script nella directory principale. Deve essere reso eseguibile e utilizzato da Cron.
#!/bin/bash

# =================================================================
# SCRIPT DI AVVIO PER IL MONITORAGGIO DEL FLUSSO TURISTI (Per Cron)
# =================================================================

# !!! MODIFICA QUESTO PERCORSO !!!
# Deve puntare alla posizione esatta della tua cartella Vision sul Raspberry Pi
PROJECT_DIR="/home/settoretecnico/Downloads/Vision"

# Vai alla directory del progetto
cd "$PROJECT_DIR" || { echo "Errore CRON: La directory di progetto non è stata trovata in $PROJECT_DIR"; exit 1; }

# Attiva l'ambiente virtuale
source venv/bin/activate || { echo "Errore CRON: Impossibile attivare l'ambiente virtuale venv. Verifica il percorso."; exit 1; }

# Esegui lo script Python e reindirizza l'output a un log di sistema
python presenze_flusso.py >> flusso_monitoraggio_output.log 2>&1

# Disattiva l'ambiente virtuale
deactivate

# Fine script

⏰ Automazione (Cron Setup)
Per avviare il monitoraggio automaticamente all’orario di apertura (es. 08:00) e tutti i giorni della settimana:
Rendi lo script di avvio eseguibile:
chmod +x start_flusso_monitor.sh

Apri l’editor Cron:
crontab -e

Aggiungi la seguente riga. Ricorda di verificare il percorso assoluto!
# Avvia lo script alle 08:00, tutti i giorni (dal lunedì alla domenica)
0 8 * * * /home/settoretecnico/Downloads/Vision/start_flusso_monitor.sh

⚠️ Note Importanti
Verifica la Directory: Assicurati che PROJECT_DIR in start_flusso_monitor.sh e il percorso nel Cron job siano assolutamente corretti. Un errore impedisce l’avvio automatico.
Log di Sistema: Per verificare l’esecuzione e diagnosticare problemi all'avvio o al tracking, controlla il file flusso_monitoraggio_output.log.
Taratura: La variabile COUNTING_LINE_PERCENTAGE nel file .env è cruciale. Potrebbe essere necessario fare dei test per posizionare la linea virtuale (e la telecamera) nel punto più efficace per il conteggio.
