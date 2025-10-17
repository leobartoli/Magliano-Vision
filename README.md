🌳 Parco Monitor: Sistema di Rilevamento Leggero Condizionale (Raspberry Pi)
Questo progetto implementa un sistema di Computer Vision ottimizzato per l'uso su Raspberry Pi 4. Lo scopo è monitorare continuamente un parco pubblico, con una logica specifica:
Ricerca Attiva: Il sistema cattura una foto ogni 10 secondi finché non rileva NESSUNA persona. Notifica Istantanea e Pausa: Quando il parco è vuoto, invia una notifica immediata via email e si mette in pausa automatica fino al giorno successivo. Resilienza: Il monitoraggio viene riattivato all'orario predefinito o dopo qualsiasi interruzione di corrente.
1. Architettura e Motore CV
Per garantire la massima stabilità e il minimo consumo di RAM sul Raspberry Pi, è stato scelto un approccio estremamente leggero:
Componente
Descrizione
Motore CV
Classificatore a Cascata di Haar (haarcascade_fullbody.xml). Questo metodo è molto più leggero del modulo DNN (Deep Neural Network), che ha causato problemi di stabilità sull'architettura ARM.
Logica di Schedulazione
La logica di Pausa e Ripresa è gestita interamente all'interno del codice Python, utilizzando datetime per calcolare l'orario di riattivazione (es. 06:00 del mattino successivo), eliminando la necessità della libreria schedule.
Resilienza
Il riavvio del programma dopo un reboot o un'interruzione di corrente è gestito dal servizio di sistema cron (@reboot).
2. Setup Iniziale e Dipendenze
Il progetto risiede nella directory ~/Downloads/Vision/.
2.1. Prerequisiti
	•	Raspberry Pi OS (64-bit consigliato)
	•	Python 3.x
	•	Fotocamera USB o CSI
2.2. Installazione (Ambiente Virtuale)
# Entra nella cartella del progetto cd ~/Downloads/Vision  # 1. Crea e attiva l'ambiente virtuale python3 -m venv venv source venv/bin/activate  # 2. Installa le librerie necessarie # opencv-python: Computer Vision # python-dotenv: Configurazione email pip install opencv-python python-dotenv 
2.3. Download del Modello CV
Scarica il modello leggero per il rilevamento di corpi umani (necessario per lo script):
wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_fullbody.xml -O haarcascade_fullbody.xml 
3. Configurazione E-mail e Credenziali
Le credenziali SMTP (Gmail o altro) devono essere fornite tramite un file nascosto.
3.1. Creazione del file .env
Crea un file chiamato .env nella directory del progetto.
# Il file deve essere chiamato .env per essere letto automaticamente nano .env 
Inserisci le tue credenziali e salva:
# Credenziali email (usare una Password per App se si usa Gmail) EMAIL_SENDER="tua_email@gmail.com" EMAIL_PASSWORD="la_tua_password_app_o_standard"  EMAIL_RECIPIENT="destinatario_email@dominio.it" 
4. Esecuzione e Gestione del Servizio
Per garantire il funzionamento continuo (24/7) e la resilienza agli shutdown.
4.1. Creazione dello Script di Avvio Shell
Il servizio di sistema richiede uno script wrapper (.sh) per attivare l'ambiente virtuale. Crea il file start_monitor.sh e rendilo eseguibile:
#!/bin/bash  # Directory di lavoro del progetto cd /home/settoretecnico/Downloads/Vision  # Attiva l'ambiente virtuale source venv/bin/activate  # Avvia lo script Python in background, reindirizzando l'output # Lo script DEVE rimanere in esecuzione continua per gestire la pausa/ripresa nohup python parco_monitor.py > monitor_output.log 2>&1 &  # Deattiva l'ambiente deactivate 
chmod +x start_monitor.sh 
4.2. Configurazione della Schedulazione con cron
Utilizza cron per gestire il riavvio automatico e quello giornaliero.
crontab -e 
Aggiungi le seguenti due righe, sostituendo il percorso completo:
# 1. Riavvia dopo qualsiasi riavvio del sistema (reboot) o interruzione di corrente. @reboot /home/settoretecnico/Downloads/Vision/start_monitor.sh  # 2. Riavvia il monitoraggio ogni giorno alle 06:00 (se per qualche motivo si è fermato inaspettatamente). 0 6 * * * /home/settoretecnico/Downloads/Vision/start_monitor.sh 
5. Logica Dettagliata dello Script
Il file parco_monitor.py opera in due stati gestiti internamente al loop principale:
Stato
Condizione/Azione
Ritardo
STATO ATTIVO
Persone Rilevate: Continua il ciclo di cattura. Parco Vuoto Rilevato: 1. Salva la foto. 2. Invia email immediata. 3. Imposta IS_PAUSED = True e calcola l'orario di ripresa.
10 secondi
STATO PAUSA
Monitoraggio Sospeso: Il programma attende senza catturare foto. Orario di Ripresa Raggiunto (es. 06:00): Imposta IS_PAUSED = False e riprende immediatamente il monitoraggio attivo.
5 minuti (tra i controlli dell'orario)
Questo approccio garantisce che il Raspberry Pi non sprechi energia elaborando immagini durante la notte o una volta che il parco vuoto è stato confermato.
