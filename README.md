🌳 Parco Monitor: Rilevamento Leggero su Raspberry PiQuesto progetto implementa un sistema di Computer Vision estremamente leggero su un Raspberry Pi per monitorare un parco pubblico. Il suo obiettivo è catturare foto a intervalli regolari e salvarle solo se non viene rilevata alcuna figura umana, inviando poi un riepilogo giornaliero via email.Utilizza il Classificatore a Cascata di Haar di OpenCV per garantire un'alta stabilità e un basso consumo di RAM sulla CPU del Raspberry Pi.1. PrerequisitiAssicurati che il tuo sistema Raspberry Pi sia operativo e che tu abbia accesso al terminale.Hardware: Raspberry Pi 4 (4 GB di RAM consigliati) e una fotocamera compatibile (USB o CSI).Software: Python 3.x.Rimozione Ollama: Confermiamo che i modelli LLM pesanti che causavano il memory killing sono stati rimossi.2. Setup dell'Ambiente e DipendenzeÈ essenziale utilizzare un ambiente virtuale per isolare le librerie del progetto.2.1. Creazione e Attivazione dell'AmbienteAssicurati di essere nella cartella corretta (~/Downloads/Vision):Bash# 1. Crea l'ambiente virtuale
python3 -m venv venv

# 2. Attiva l'ambiente virtuale
source venv/bin/activate
# Dovresti vedere (venv) apparire all'inizio del tuo prompt.
2.2. Installazione delle LibrerieCon l'ambiente virtuale attivo, installa le dipendenze necessarie:Bashpip install opencv-python schedule python-dotenv
2.3. Download del Modello CVPer il rilevamento di persone (corpi interi) senza utilizzare il modulo DNN instabile, usiamo un classificatore a cascata.Bash# Scarica il file del classificatore di corpi interi (fullbody)
wget https://raw.githubusercontent.com/opencv/opencv/master/data/haarcascades/haarcascade_fullbody.xml -O haarcascade_fullbody.xml
3. Configurazione E-mail (Credenziali)Lo script invia un riepilogo giornaliero e richiede le credenziali SMTP.3.1. Creazione o Verifica del file .envIl file che contiene le credenziali deve essere chiamato .env per essere letto automaticamente dal comando load_dotenv(). (Se insisti nel chiamarlo exampre.env, devi modificarlo nello script).Crea/Modifica il file .env nella cartella del progetto:Snippet di codice# Dati Email
# USARE LA PASSWORD PER APP (Gmail) per ragioni di sicurezza
EMAIL_SENDER="tua_email@gmail.com"
EMAIL_PASSWORD="la_tua_password_app_o_standard" 
EMAIL_RECIPIENT="destinatario_email@dominio.it"
4. Esecuzione del Monitoraggio4.1. Avvio e FunzionamentoAssicurati che il tuo file parco_monitor.py sia l'ultima versione che utilizza il Classificatore a Cascata e che l'ambiente virtuale sia attivo.Bashpython parco_monitor.py
4.2. Logica OperativaComponenteDettagliIntervallo CatturaOgni 5 minuti (300 secondi).Metodo di AnalisiClassificatore a Cascata di Haar (haarcascade_fullbody.xml).Condizione di SalvaLa foto viene salvata SOLO SE il classificatore non rileva corpi umani.Report E-mailInviato ogni giorno alle 21:00.ArchiviazioneLe foto inviate via email vengono spostate da parco_photos/ a archive_photos/.4.3. Esecuzione Persistente (Background)Per far sì che il monitoraggio continui anche dopo aver chiuso il terminale (consigliato per l'uso a lungo termine), utilizza nohup:Bash# Avvia lo script in background. L'output viene salvato in nohup.out.
nohup python parco_monitor.py & 

# Per controllare lo stato:
tail -f nohup.out 

# Per fermare il processo, trova il PID e uccidilo (kill):
ps aux | grep parco_monitor.py 
# E poi: kill <PID>
