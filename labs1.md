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

### 1. Login

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
```

### 2. Nuovo progetto

```bash
oc new-project cli-project
```

---

### 3. Crea Pod

```bash
oc run web-server --image registry.access.redhat.com/ubi10/httpd-24
```

```bash
oc get pods
```

---

### 4. Espone Pod

```bash
oc expose pod web-server --port=8080 --target-port=8080 --name=web-server-svc
```

```bash
oc get service
```

---

### 5. Crea Route

```bash
oc expose svc web-server-svc
```

```bash
oc get route
```

---

### 6. Test

```bash
curl web-server-svc-cli-project.apps.ocp4.example.com
```

---

### 7. Conversione in Deployment

```bash
oc delete pod web-server
```

```bash
oc create deployment web-server --image=registry.access.redhat.com/ubi10/httpd-24
```

```bash
oc get pods
oc get route
```

```bash
curl web-server-svc-cli-project.apps.ocp4.example.com
```

---

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
 
