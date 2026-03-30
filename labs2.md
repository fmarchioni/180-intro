
# Esercizio 1: Jobs e CronJobs

---

## 1. Accesso alla Console OpenShift

Apri il browser all'indirizzo:
`https://console-openshift-console.apps.ocp4.example.com`

Seleziona l'opzione **Red Hat Identity Management** e utilizza le seguenti credenziali:
* **Username:** `admin`
* **Password:** `redhatocp`

Esegui il login anche da terminale:

```
oc login -u admin -p redhatocp
```

---

## 2. Creazione Progetto

Crea da Console Web il Project "storage-demo":
* Percorso: **Home** -> **Projects** -> **Create Project**

---

## 3. Creazione Deployment MySQL

* Seleziona **Workloads** -> **Deployments**
* Clicca su **Create Deployment**
* Seleziona in alto il progetto **“storage-demo”**
* Usa l'immagine: `registry.ocp4.example.com:8443/rhel9/mysql-80`

Inserisci le seguenti **Environment variables** (in basso):
* `MYSQL_USER=developer`
* `MYSQL_PASSWORD=developer`
* `MYSQL_ROOT_PASSWORD=redhat`

*(Verifica che i Pods siano disponibili e scala il numero di repliche a 1)*

---

## 4. Creazione Servizio MySQL

Crea un Service denominato "mysql" utilizzando il seguente YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: storage-demo
spec:
  selector:
    app: mysql
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```

Verifica che il Service punti correttamente al pod:

```bash
# Verifica gli endpoints del servizio mysql per confermare il mapping con il Pod
oc get endpoints mysql
```

---

## 5. Inizializzazione Database (Job)

Crea lo schema del DB lanciando questo Job:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mysql-init-schema
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: mysql-client
        image: registry.ocp4.example.com:8443/rhel9/mysql-80
        command: ["sh", "-c"]
        args:
          - >
            mysql -h mysql -P 3306 -u root -predhat
            -e "
              CREATE DATABASE IF NOT EXISTS demo;
              USE demo;
              CREATE TABLE IF NOT EXISTS messaggi (
                id INT AUTO_INCREMENT PRIMARY KEY,
                message TEXT
              );
            "
```

Puoi usare la linea di comando per lanciarlo:

```bash
# Creazione del Job di inizializzazione tramite file remoto
oc create -f https://raw.githubusercontent.com/fmarchioni/180-intro/refs/heads/main/files/job-create.yaml
```

---

## 6. Popolamento Database (CronJob)

Lancia un CronJob per popolare il database ogni minuto:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-insert-messaggi
spec:
  schedule: "*/1 * * * *"   # ogni minuto
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: mysql-client
            image: registry.ocp4.example.com:8443/rhel9/mysql-80
            command: ["sh", "-c"]
            args:
              - >
                TS=$(date +%H:%M:%S);
                MSG="messaggio inviato: $TS";
                mysql -h mysql -P 3306 -u root -predhat
                -e "USE demo; INSERT INTO messaggi (message) VALUES ('$MSG');"
```

Puoi lanciarlo da linea di comando:

```bash
# Creazione del CronJob di inserimento dati tramite file remoto
oc create -f https://raw.githubusercontent.com/fmarchioni/180-intro/refs/heads/main/files/cronjob-insert.yaml
```

---

## 7. Verifica Dati e Pulizia

Dopo qualche minuto, verifica la presenza dei messaggi:

```bash
# Esegue una query SQL nel pod mysql per leggere la tabella messaggi
echo "redhat" | oc exec -i $(oc get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password -h mysql -e "USE demo; SELECT * FROM messaggi;"
```

Rimuovi le risorse Job e CronJob:

```bash
# Cancella l'istanza del Job completato
oc delete job mysql-init-schema
```

```bash
# Rimuove la schedulazione del CronJob
oc delete cronjob mysql-insert-messaggi
```

---

# Esercizio 2: Load Balancer

---

## 1. Modifica del Service

* Seleziona **Network** -> **Services** -> **mysql**
* Vai nel Tab **“YAML”**
* Cerca la riga `type: ClusterIP` e sostituiscila con: `type: LoadBalancer`
* Clicca su **“Save”**

---

## 2. Verifica Accesso Esterno

Apri il terminale della tua workstation:

```bash
# Connessione al database usando l'IP pubblico del Load Balancer
mysql -u root -predhat -h 192.168.50.10
```

Esegui i comandi SQL:

```sql
-- Seleziona il database demo
use demo;
```

```sql
-- Mostra i messaggi contenuti per verificare il funzionamento del CronJob
select * from messaggi;
```

```sql
-- Chiude la sessione mysql
exit;
```

> **Nota:** Lasciate questo tab del terminale aperto per i comandi successivi. Aprite un altro tab per i prossimi esercizi.

---

# Esercizio 3: Storage

---

## 1. Test perdita dati (Senza Storage)

Cancella il Pod utilizzato dal database:
* Percorso: **Workloads** -> **Pods** -> **Delete Pod** (seleziona quello in Running)

Verifica che i dati siano spariti (perché non c'è persistenza):

```bash
# Tenta l'accesso al DB: noterai che le tabelle create precedentemente non esistono più
mysql -u root -predhat -h 192.168.50.10 -e "use demo; select * from messaggi;"
```

---

## 2. Aggiunta Storage al Deployment

* Vai su **Workloads** -> **Deployments**
* Seleziona il Deployment **“mysql”**
* Seleziona il tab **Storage** o clicca su **Add Storage** dal menu azioni
* Scegli **Create new claim**
* **PersistentVolumeClaim Name:** `mysql`
* **Mount Path:** `/var/lib/mysql`
* **Size:** `1GiB`
* Clicca su **Save**

---

## 3. Inizializzazione e Popolamento (Con Storage)

```bash
# Ripristina lo schema del DB nel nuovo volume persistente
oc create -f https://raw.githubusercontent.com/fmarchioni/180-intro/refs/heads/main/files/job-create.yaml
```

```bash
# Riattiva il CronJob per riprendere il popolamento dei dati
oc create -f https://raw.githubusercontent.com/fmarchioni/180-intro/refs/heads/main/files/cronjob-insert.yaml
```

Attendi l'inserimento di qualche riga e verifica:

```bash
# Verifica che i nuovi messaggi vengano scritti correttamente
mysql -u root -predhat -h 192.168.50.10 -e "use demo; select * from messaggi;"
```

---

## 4. Verifica Persistenza

Cancella nuovamente il Pod:
* Percorso: **Workloads** -> **Pods** -> **Delete Pod**

Dopo che il nuovo Pod è attivo, verifica che i dati siano ancora presenti:

```bash
# Verifica che i dati siano persistiti sul volume PVC nonostante il riavvio del Pod
mysql -u root -predhat -h 192.168.50.10 -e "use demo; select * from messaggi;"
```

---

# Esercizio 4: Secrets

---

## 1. Creazione del Secret

Dalla Console Web:
* Percorso: **Workloads** -> **Secrets** -> **Create Key/value Secret**
* **Secret name:** `mysql`

Inserisci i seguenti attributi chiave/valore:
* `MYSQL_USER` = `developer`
* `MYSQL_PASSWORD` = `developer`
* `MYSQL_ROOT_PASSWORD` = `redhat`

---

## 2. Associazione del Secret al Deployment

* Dalla lista Secrets, clicca sul secret **“mysql”** appena creato.
* Clicca sul bottone in alto a destra: **Add Secret to Workload**.
* Seleziona il deployment **“mysql”**.

---

## 3. Pulizia Variabili d'Ambiente

* Spostati sul Deployment **“mysql”**.
* Seleziona il Tab **Environment**.
* Rimuovere le vecchie variabili d’ambiente inserite manualmente (quelle che non hanno l'icona del lucchetto/secret).
* Clicca su **“Save”**.

---

## 4. Verifica Finale

Verifica che la connessione sia ancora funzionante tramite le credenziali fornite dal Secret:

```bash
# Connessione al database per validare la configurazione tramite Secrets
mysql -u root -predhat -h 192.168.50.10
```

```sql
-- Ultimo controllo dei dati nel database persistente
use demo;
select * from messaggi;
```
