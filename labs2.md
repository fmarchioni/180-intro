# Esercizio 1: Jobs e CronJobs

In questo esercizio imparerai a distribuire un database MySQL e a utilizzare Job e CronJob per l'inizializzazione e il popolamento automatico dei dati.

## 1.1 Accesso alla Console e Preparazione
1. Accedi alla console web: `https://console-openshift-console.apps.ocp4.example.com`
2. Seleziona **Red Hat Identity Management**.
3. Credenziali: Username `admin` / Password `redhatocp`.
4. Crea un nuovo Project denominato `storage-demo` (**Home** -> **Projects** -> **Create Project**).

## 1.2 Deployment di MySQL
Dalla Console Web:
1. Vai su **Workloads** -> **Deployments**.
2. Assicurati che il progetto selezionato in alto sia `storage-demo`.
3. Crea un Deployment "mysql" utilizzando l'immagine: `registry.ocp4.example.com:8443/rhel9/mysql-80`.
4. Inserisci le seguenti **Environment variables**:
   * `MYSQL_USER` = `developer`
   * `MYSQL_PASSWORD` = `developer`
   * `MYSQL_ROOT_PASSWORD` = `redhat`
5. Verifica che i Pod siano disponibili e scala a **1**.

## 1.3 Creazione del Servizio
Crea un file YAML o incolla nella sezione "Create Service" della console:

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

Verifica gli endpoint da terminale:
```bash
# Verifica che il Service punti correttamente al Pod
oc get endpoints mysql
```

## 1.4 Inizializzazione Database (Job)
Esegui il Job per creare lo schema del database:

```bash
# Crea il Job di inizializzazione dello schema DB da URL remoto
oc create -f https://raw.githubusercontent.com/fmarchioni/180-intro/refs/heads/main/files/job-create.yaml
```

## 1.5 Popolamento Automatico (CronJob)
Esegui il CronJob per inserire un messaggio ogni minuto:

```bash
# Crea il CronJob per il popolamento automatico
oc create -f https://raw.githubusercontent.com/fmarchioni/180-intro/refs/heads/main/files/cronjob-insert.yaml
```

## 1.6 Verifica e Pulizia
Dopo qualche minuto, verifica i dati:

```bash
# Esegue una query SQL all'interno del Pod MySQL per vedere i messaggi inseriti
echo "redhat" | oc exec -i $(oc get pod -l app=mysql -o jsonpath='{.items[0].metadata.name}') -- mysql -u root --password -h mysql -e "USE demo; SELECT * FROM messaggi;"
```

Pulizia delle risorse:
```bash
# Elimina il Job eseguito
oc delete job mysql-init-schema
```

```bash
# Elimina il CronJob
oc delete cronjob mysql-insert-messaggi
```

---

# Esercizio 2: Load Balancer e Storage

## 2.1 Esposizione tramite Load Balancer
1. Vai su **Network** -> **Services** -> **mysql**.
2. Seleziona il tab **YAML**.
3. Modifica `type: ClusterIP` in `type: LoadBalancer`.
4. Clicca su **Save**.

Test di connessione esterna:
```bash
# Connessione al database tramite l'IP del Load Balancer (eseguire da workstation)
mysql -u root -predhat -h 192.168.50.10
```

## 2.2 Test Persistenza (Senza Storage)
```bash
# Elimina il Pod attuale per simulare un crash (i dati andranno persi)
oc delete pod -l app=mysql
```

## 2.3 Aggiunta dello Storage Persistente
1. Vai su **Workloads** -> **Deployments** -> **mysql**.
2. Seleziona l'azione **Add Storage**.
3. Impostazioni:
   * **PVC Name**: `mysql`
   * **Mount Path**: `/var/lib/mysql`
   * **Size**: `1GiB`
4. Clicca **Save**.

## 2.4 Ripopolamento e Verifica Persistenza
Rilancia i job dell'esercizio precedente:

```bash
# Inizializza nuovamente il DB
oc create -f https://raw.githubusercontent.com/fmarchioni/180-intro/refs/heads/main/files/job-create.yaml
```

```bash
# Avvia il CronJob di inserimento
oc create -f https://raw.githubusercontent.com/fmarchioni/180-intro/refs/heads/main/files/cronjob-insert.yaml
```

Verifica ora la persistenza dopo l'eliminazione del Pod:
```bash
# Elimina nuovamente il Pod (ora con storage collegato)
oc delete pod -l app=mysql
```

```bash
# Verifica che i dati siano ancora presenti nonostante il riavvio del Pod
mysql -u root -predhat -h 192.168.50.10 -e "USE demo; SELECT * FROM messaggi;"
```

---

# Esercizio 3: Secrets

In questo esercizio sposteremo le credenziali in chiaro dalle variabili d'ambiente a un oggetto Secret.

## 3.1 Creazione del Secret
Dalla Console Web:
1. Vai su **Workloads** -> **Secrets** -> **Create Key/value Secret**.
2. **Secret name**: `mysql`.
3. Aggiungi i seguenti valori:
   * `MYSQL_USER` = `developer`
   * `MYSQL_PASSWORD` = `developer`
   * `MYSQL_ROOT_PASSWORD` = `redhat`

## 3.2 Associazione al Workload
1. Seleziona il secret appena creato `mysql`.
2. Clicca sul tasto in alto a destra **Add Secret to Workload**.
3. Seleziona il deployment `mysql`.

## 3.3 Pulizia e Verifica
1. Vai sul Deployment `mysql` -> Tab **Environment**.
2. Rimuovi le variabili d'ambiente inserite manualmente all'inizio (quelle non provenienti dal secret).
3. Clicca **Save**.

Verifica finale del funzionamento:
```bash
# Verifica finale della connessione con credenziali gestite via Secret
mysql -u root -predhat -h 192.168.50.10 -e "USE demo; SELECT * FROM messaggi;"
```
