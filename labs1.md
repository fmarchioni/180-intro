# 📦 Laboratori Container & OpenShift
## Link essenziali:

**Classroom**: https://rol.redhat.com/rol/app/classes/6a1d537a-d716-4e33-9066-a1035ca20d04

**Console OpenShift**: https://console-openshift-console.apps.ocp4.example.com

- Utenze:    
- admin/redhatocp
- developer/developer

**Esempio login da linea di comando:**  


```
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

## 🧪 Esercizio 1 – Esecuzione immagine da registry

Esegui un container HTTPD esposto sulla porta 8080:

```bash
podman run --rm -it -p 8080:8080 registry.access.redhat.com/ubi8/httpd-24
```

Verifica da browser:

```
http://localhost:8080
```

Interrompi il container con:

```
CTRL + C
```

---

## 🧪 Esercizio 2 – Creazione immagine custom

### 1. Crea directory di lavoro

```bash
mkdir demo-openshift && cd demo-openshift
```

### 2. Crea pagina HTML

```bash
cat <<EOF > index.html
Hello world!
EOF
```

### 3. Crea Containerfile

```bash
cat <<EOF > Containerfile
FROM registry.access.redhat.com/ubi8/httpd-24

COPY index.html /var/www/html/index.html

EXPOSE 8080
EOF
```

### 4. Build immagine

```bash
podman build -t my-app:v1 .
```

### 5. Verifica immagine

```bash
podman images
```

### 6. Avvia container

```bash
podman run -d --name web-test -p 8080:8080 localhost/my-app:v1
```

### 7. Test

Apri:

```
http://localhost:8080
```

Oppure:

```bash
curl localhost:8080
```

### 8. Comandi di gestione

```bash
podman ps
podman logs web-test
podman stop web-test
podman rm web-test
```

---

## 🧪 Esercizio 3 – OpenShift Web Console

### Accesso

Apri:

```
https://console-openshift-console.apps.ocp4.example.com
```

Login:

```
Username: admin
Password: redhatocp
```

---

### 1. Crea progetto

* Home → Projects → Create Project
* Nome: `test`

---

### 2. Crea Pod

* Workloads → Pods → Create Pod
* Immagine:

```
registry.access.redhat.com/ubi10/httpd-24
```

---

### 3. Crea Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example
  namespace: test
spec:
  selector:
    app: httpd
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

---

### 4. Test DNS interno

Dal terminal del Pod:

```bash
curl example:8080
```

---

### 5. Crea Route

* Networking → Routes → Create Route
* Seleziona:

  * Service
  * Porta 8080

Verifica accesso via browser.

---

## 🧪 Esercizio 4 – Deployment

### 1. Cancella Pod

* Workloads → Pods → Delete Pod

---

### 2. Crea Deployment

* Workloads → Deployments → Create Deployment

Configurazione:

```
Name: httpd
Image: registry.access.redhat.com/ubi10/httpd-24
```

---

### 3. Verifiche

* La route continua a funzionare
* Cancella un Pod → viene ricreato automaticamente
* Scala a 1 replica

---

## 🧪 Esercizio 5 – Route SSL

Modifica la Route:

* Networking → Routes → Edit Route

Impostazioni:

* ✅ Secure Route
* TLS Termination: `Edge`
* Insecure traffic: `None`

Verifica:

* Accesso via `https`
* Certificato dal browser (lucchetto)

---

### 🧹 Pulizia

Cancella il progetto:

* Home → Projects → `test` → Delete

---

## 🧪 Esercizio 6 – OpenShift CLI

In questo laboratorio impareremo a connetterci al cluster, gestire pod singoli e utilizzare i Deployment per esporre servizi web verso l'esterno.

## 1. Accesso al Cluster
Il primo passo è autenticarsi presso l'API server di OpenShift utilizzando le credenziali fornite.

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

## 2. Creazione del Progetto
Creiamo un namespace isolato per separare le nostre risorse da quelle degli altri utenti.

```bash
oc new-project cli-project
```

## 3. Test con Pod Singolo (Effimero)
Creiamo un pod direttamente dall'immagine Apache. Un pod creato con `oc run` non viene gestito da un controller: se viene rimosso, non verrà ricreato.

```bash
oc run pod-web --image=registry.access.redhat.com/ubi9/httpd-24
```

Verifica dei log per confermare che il server sia attivo:

```bash
oc logs pod-web
```

Eliminiamo il pod per testarne la natura effimera:

```bash
oc delete pod pod-web
```

## 4. Deployment e Alta Affidabilità
Utilizziamo ora un **Deployment**. A differenza del comando precedente, il Deployment garantisce che il pod sia sempre attivo e ne permette la scalabilità.

```bash
oc create deployment web-server --image=registry.access.redhat.com/ubi9/httpd-24
```



## 5. Esposizione del Servizio e Accesso Esterno
Per rendere l'applicazione raggiungibile, dobbiamo creare un **Service** (IP interno stabile) e una **Route** (URL pubblico).

Creazione del Service (porta interna **8080**):

```bash
oc expose deployment web-server --port=8080 --target-port=8080
```

Creazione della Route:

```bash
oc expose service web-server
```



## 6. Verifica Finale
Recuperiamo l'URL pubblico assegnato dal router di OpenShift:

```bash
oc get route web-server
```

Eseguiamo un test di connessione per verificare che la pagina web sia raggiungibile (sostituisci l'URL con quello reale ottenuto dal comando precedente):

```bash
curl web-server-cli-project.apps.ocp4.example.com
```

 
## 🧪 Esercizio 7 – Monitoraggio

### Console Web

* Overview:
  `Home → Overview`

* Progetto:
  `Home → Projects → cli-project`

* Pod metrics:
  `Workloads → Pods → (pod) → Metrics`

* Dashboard globali:
  `Observe → Dashboards`

---

## 🧪 Esercizio 8 – ConfigMap e Secret

### 1. Crea ConfigMap

```bash
oc create configmap web-config --from-literal=message="Hello from ConfigMap"
```

---

### 2. Crea Secret

```bash
oc create secret generic web-secret --from-literal=password="mypassword"
```

---

### 3. Inietta variabili nel Deployment

```bash
oc set env deployment/web-server --from=secret/web-secret
```

```bash
oc set env deployment/web-server --from=configmap/web-config
```

---

### 4. Accedi al Pod

```bash
oc get pods
```

```bash
oc rsh <nome-pod>
```

---

### 5. Verifica variabili

```bash
echo $PASSWORD
```

```bash
echo $MESSAGE
```
 
 
---

## 🧪 Esercizio 9 – OpenShift Templates

In questo esercizio vedremo come esplorare i template di sistema e istanziare un database PostgreSQL personalizzando i parametri di configurazione.

### 1. Preparazione dell'ambiente
Per prima cosa, creiamo un nuovo progetto dedicato e verifichiamo i template disponibili nel namespace globale di OpenShift.

```bash
oc new-project demo-templates
oc get templates -n openshift
```

### 2. Analisi e Deployment del Template
Prima di creare l'applicazione, analizziamo quali parametri accetta il template `postgresql-ephemeral`. Successivamente, lo istanziamo passando le nostre credenziali personalizzate.

```bash
# Analisi dei parametri del template
oc process --parameters postgresql-ephemeral -n openshift
```

```bash

# Creazione dell'applicazione tramite template
oc new-app --template=postgresql-ephemeral \
    -p DATABASE_SERVICE_NAME=mydb-demo \
    -p POSTGRESQL_USER=admin_user \
    -p POSTGRESQL_PASSWORD=SecretPassword123 \
    -p POSTGRESQL_DATABASE=demo_db
```

### 3. Accesso al Database e Test Funzionale
Una volta che il Pod è in stato *Running*, entriamo nel container per interagire con il database tramite il client `psql`.

> **Nota:** Sostituisci `mydb-demo-1-vgfkr` con il nome reale del tuo Pod (puoi trovarlo con `oc get pods`).

```bash
# Accesso remoto al Pod
oc rsh mydb-demo-1-vgfkr
```

### Operazioni SQL
Una volta all'interno della shell del Pod, connettiti al database ed esegui i comandi SQL:

```sql
-- Connessione al database
psql -U admin_user -d demo_db

-- Creazione tabella e inserimento dati
CREATE TABLE demo (id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO demo (name) VALUES ('OpenShift Rocks');

-- Verifica dei dati
SELECT * FROM demo;

-- Uscita
\q
exit
```

 
