# Kubernetes — Vodič i najbolje prakse

> Sažet referentni materijal za rad na Kubernetes projektima. Ovaj fajl služi kao instrukcija agentu: kada radim projekat, drži se principa i obrazaca opisanih ovde.

---

## 1. Uvod

**Kubernetes (k8s)** je platforma za automatski deployment, skaliranje i upravljanje kontejnerima. Sve u Kubernetesu je izloženo kroz **resurse** — svaki resurs ima jasnu namenu.

- Docker poznaje mali broj resursa: `Service`, `Network`, `Volume`, `Secret`, `Config`.
- Kubernetes poznaje znatno više: `Pod`, `Volume`, `Secret`, `ConfigMap`, `Service`, `Ingress`, `Job`, `CronJob`, `ReplicaSet`, `Deployment`, `StatefulSet` itd.
- Mogu se definisati i proizvoljni resursi preko **CRD-a** (Custom Resource Definitions). Primer: `S3Bucket` CRD koji preko kontrolera komunicira sa AWS API-jem.

### Alati

- **Lokalni klaster:** `k3d`, `Minikube`, `kind`.
- **Upravljanje resursima:** `kubectl`.
- **Učitavanje lokalnih image-a u klaster:**
  - `k3d image import <image>`
  - `minikube image load <image>`
- Inače, sve image koje koristim treba da budu javne na nekom registry-u (npr. DockerHub).

### Resursi vs. objekti

- **Resurs** je tip (npr. `Pod`, `Deployment`, `Secret`).
- **Objekat** je konkretna instanca (npr. `pod/auth-server`, `deployment/booking`).

### Scope resursa

- **Namespace-scoped** (`Pod`, `Deployment`, `ConfigMap`, `Secret`, `Ingress`...) — moraju pripadati nekom namespace-u; nazivi su jedinstveni unutar namespace-a.
- **Cluster-wide** — ne pripadaju namespace-u; nazivi su jedinstveni na nivou celog klastera.

---

## 2. Pod

Pod je grupa od jednog ili više kontejnera koji dele storage (volumes) i mrežu.

**Najbolje prakse:**

- Najčešće Pod ima **jedan glavni kontejner** — samu aplikaciju.
- **Sidecar kontejneri** za pomoćne uloge (npr. log agent koji prosleđuje logove agregatoru).
- **Init kontejneri** za pripremu pre pokretanja aplikacije (npr. migracija baze, kreiranje Kafka topic-a, health check zavisnosti).
- Kontejneri unutar istog Pod-a komuniciraju preko `localhost:port` jer dele mrežu.

### Primer: `nginx.yml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.14.2
      ports:
        - containerPort: 80
```

**Komande:**

```bash
kubectl apply -f nginx.yml      # kreiranje
kubectl delete -f nginx.yml     # brisanje
```

**CLI alternativa (brzo, bez yaml-a):**

```bash
kubectl run --port 80 --image nginx:1.14.2 nginx
kubectl delete pod nginx
```

### Healthcheck — liveness i readiness probes

Uvek definisati obe sonde za production aplikacije:

- **livenessProbe** — da li je aplikacija živa; ako nije, Kubernetes restartuje kontejner.
- **readinessProbe** — da li je aplikacija spremna da prima saobraćaj; ako nije, Service je isključuje iz load balancing-a.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
    - name: counter
      image: danijelradakovic/counter
      ports:
        - containerPort: 8000
      livenessProbe:
        httpGet:
          port: 8000
          path: /probe/liveness
        initialDelaySeconds: 3   # čekaj pre prve provere
        periodSeconds: 3         # provera svake 3 sekunde
      readinessProbe:
        httpGet:
          port: 8000
          path: /probe/readiness
        initialDelaySeconds: 3
        periodSeconds: 3
```

**Pre apply-a, kreirati namespace ako ne postoji:**

```bash
kubectl create namespace demo
kubectl -n demo apply -f pod.yaml
```

---

## 3. Deployment

Koristi se kada je potrebno više replika istog Pod-a. Deployment garantuje da je broj željenih replika **uvek** ispunjen — ako replika padne, kreira novu.

**Najbolje prakse:**

- Skoro nikad ne pravim "go" Pod direktno — uvek preko Deployment-a, osim za jednokratne testove.
- `selector.matchLabels` mora odgovarati `template.metadata.labels`.
- Definisati `imagePullPolicy: IfNotPresent` za lokalni razvoj sa preload-ovanim image-om.

### Primer: `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: counter
  labels:
    app: counter
spec:
  replicas: 2
  selector:
    matchLabels:
      app: counter
  template:
    metadata:
      name: counter
      labels:
        app: counter
    spec:
      containers:
        - name: counter
          image: danijelradakovic/counter
          imagePullPolicy: IfNotPresent
          livenessProbe:
            httpGet:
              port: 8000
              path: /probe/liveness
            initialDelaySeconds: 3
            periodSeconds: 3
          readinessProbe:
            httpGet:
              port: 8000
              path: /probe/readiness
            initialDelaySeconds: 3
            periodSeconds: 3
      restartPolicy: Always
```

```bash
kubectl -n demo apply -f deployment.yaml
```

---

## 4. Service

### Problem koji rešava

- Pod-ovi mogu da komuniciraju preko IP-ja, ali IP **nije statičan** (menja se na restart).
- Sa više replika treba load balancing.
- Treba način da se Pod-ovima pristupa po **imenu**.

### Šta Service radi

- **Service discovery** — pristup preko imena servisa.
- **Load balancing** preko više replika iz istog Deployment-a.
- Koristi **liveness probe** za detekciju da li je Pod živ i **readiness probe** za odluku da li slati saobraćaj.

### Komunikacija

- Iz **istog namespace-a:** `service_name:port`
- Iz **drugog namespace-a:**
  - `service_name.namespace:port`
  - `service_name.namespace.svc.cluster.local:port` (FQDN)

### Primer: `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: counter
spec:
  selector:
    app: counter
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
```

```bash
kubectl -n demo apply -f service.yaml
```

**CLI alternativa:**

```bash
kubectl -n demo expose deployment counter --port 8000
```

### Testiranje load balancing-a

```bash
kubectl -n demo run -it --rm --image curlimages/curl:8.00.1 curl -- sh
# unutar pod-a:
curl http://counter:8000
```

Inspekcija servisa:

```bash
kubectl -n demo describe service counter
# Endpoints prikazuje IP:port svih Pod-ova obuhvaćenih servisom
```

---

## 5. Ingress

### Kada se koristi

Service omogućava komunikaciju **unutar** klastera. Za pristup **van** klastera koristim Ingress.

**Bitno:**

- Ingress izlaže **samo HTTP (80) i HTTPS (443)** portove.
- Za druge portove izvan klastera koristim `Service` tipa `NodePort` ili `LoadBalancer`.
- Ponašanje Ingress-a zavisi od klastera (GKE, EKS, k3d, Minikube se razlikuju).
- Ingress kreira LoadBalancer koji rutira saobraćaj ka odgovarajućem Service-u, koji dalje šalje Pod-u.
- Postoji **Ingress operator** koji upravlja konfiguracijom.

### Primer Echo aplikacije + Ingress

```yaml
# echo-server.yaml
kind: Pod
apiVersion: v1
metadata:
  name: echo
  labels:
    app: echo
spec:
  containers:
    - name: echo
      image: 'kicbase/echo-server:1.0'
---
kind: Service
apiVersion: v1
metadata:
  name: echo
spec:
  selector:
    app: echo
  ports:
    - port: 8080
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo
spec:
  rules:
    - http:
        paths:
          - pathType: Prefix
            path: /counter
            backend:
              service:
                name: counter
                port:
                  number: 8000
          - pathType: Prefix
            path: /echo
            backend:
              service:
                name: echo
                port:
                  number: 8080
```

```bash
kubectl -n demo apply -f echo-server.yaml
kubectl -n demo apply -f ingress.yaml
kubectl -n demo describe ingress demo
```

**Best practice — razdvajanje:** Umesto jednog velikog Ingress resursa, mogu se napraviti odvojeni Ingress resursi po aplikaciji (sa istim `ingressClassName: nginx`). Lakše za održavanje i nezavisno menjanje.

---

## 6. ConfigMap

Koristi se za **konfiguracije** koje Pod-ovi konzumiraju kao:

1. environment varijable,
2. argumente komandne linije,
3. konfiguracione fajlove (montirane u kontejner).

`data` sekcija je kolekcija ključ–vrednost parova.

### 6.1 Kao environment varijable (sve odjednom)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: booking
data:
  LOG_LEVEL: "debug"
  FETCH_TIMEOUT: "1m"
---
apiVersion: v1
kind: Pod
metadata:
  name: booking
spec:
  containers:
    - name: app
      image: alpine
      envFrom:
        - configMapRef:
            name: booking
```

### 6.2 Selektivno + preimenovanje env varijable

```yaml
spec:
  containers:
    - name: app
      image: alpine
      env:
        - name: HTTP_TIMEOUT
          valueFrom:
            configMapKeyRef:
              name: booking
              key: FETCH_TIMEOUT
```

### 6.3 Kao argumenti komandne linije

```yaml
spec:
  containers:
    - name: app
      image: alpine
      envFrom:
        - configMapRef:
            name: booking
      command: ["/bin/sh", "-c"]
      args: ["echo $FETCH_TIMEOUT"]
```

### 6.4 Kao konfiguracioni fajl (volume mount)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: booking
data:
  dev.toml: |
    mode=dev
    timeout=1m
---
spec:
  containers:
    - name: app
      image: alpine
      command: ["/bin/app"]
      args: ["--config /etc/booking/dev.toml"]
      volumeMounts:
        - name: config
          mountPath: /etc/booking
  volumes:
    - name: config
      configMap:
        name: booking
```

### 6.5 Više fajlova u ConfigMap-i, mountuje se samo jedan (`subPath`)

```yaml
data:
  dev.toml: |
    mode=dev
    timeout=1m
  prod.toml: |
    mode=prod
    timeout=90s
---
volumeMounts:
  - name: config
    mountPath: /etc/booking/dev.toml   # putanja unutar kontejnera
    subPath: dev.toml                  # ključ/fajl iz volume-a
volumes:
  - name: config
    configMap:
      name: booking
```

### Pravila i ponašanje ConfigMap-e

- ConfigMap i Pod **moraju biti u istom namespace-u**. Ako Pod-u treba ConfigMap iz drugog namespace-a — kopirati je.
- **Mount-ovane ConfigMap-e se automatski ažuriraju** kad se izmene.
  - Korisno za feature flag-ove, log level i sl.
  - Aplikacija mora **sama da prati promenu fajla** (npr. watcher tred).
  - Konzistentnost između Pod-ova **nije zagarantovana** — postoji kratak period razlike.
- Ako se koristi `subPath` u `volumeMounts` ili kao **environment varijable**, ažuriranje **NIJE automatsko** — Pod treba restartovati.
- Postoje i **Immutable ConfigMap-e** (samo čitanje, mogu se samo obrisati) — koristiti za stabilne konfiguracije, daje bolje performanse.

---

## 7. Secrets, Volumes i ostali resursi

> PDF ove sekcije ostavlja kao TODO. Smernice koje treba primenjivati u projektu:

- **Secrets** — koristiti za sve osetljivo (lozinke, tokeni, TLS sertifikati). Nikad ne stavljati osetljive podatke u ConfigMap.
- **PersistentVolume / PersistentVolumeClaim** — za podatke koji preživljavaju restart Pod-a (baze, fajl storage). Razmotriti `subPathExpr` i `emptyDir` za specifične scenarije.
- **Quality of Service** — postaviti `requests` i `limits` (hard i soft) za CPU/memoriju. Referenca: <https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/>
- **Namespace limiti i kvote** — `ResourceQuota` i `LimitRange` po namespace-u.
- **Topologija** — Node taints + tolerations, pod eviction, pod affinity/anti-affinity.
- **RBAC** — least privilege; ServiceAccount, Role/ClusterRole, RoleBinding/ClusterRoleBinding.

---

## 8. Operativni cheatsheet

```bash
# Namespace
kubectl create namespace demo
kubectl get ns

# Apply / delete
kubectl -n demo apply -f <file.yaml>
kubectl -n demo delete -f <file.yaml>

# Pregled
kubectl -n demo get pods
kubectl -n demo get all
kubectl -n demo describe pod <name>
kubectl -n demo logs <pod-name> -f

# Debug / shell
kubectl -n demo exec -it <pod> -- sh
kubectl -n demo run -it --rm --image curlimages/curl:8.00.1 curl -- sh

# Skaliranje
kubectl -n demo scale deployment counter --replicas=5

# Expose (brzi service)
kubectl -n demo expose deployment counter --port 8000
```

---

## 9. Konvencije za projekat (instrukcije za agenta)

Kada radim na projektu, sledi ove principe:

1. **Strukturira manifest-e po resursu**: `pod.yaml` / `deployment.yaml` / `service.yaml` / `ingress.yaml` / `configmap.yaml`. Više dokumenata u jednom fajlu razdvajati `---`.
2. **Uvek koristi namespace** — ne deploy-uj u `default`. Komandama dodaj `-n <namespace>`.
3. **Deployment, ne sirov Pod** — sirov Pod samo za jednokratni debug.
4. **Liveness + Readiness probe** za svaku aplikaciju.
5. **ConfigMap za sve što nije tajna**, **Secret za sve što je tajna**.
6. **Service discovery preko imena** (`service-name:port`), ne preko IP-ja.
7. **Ingress samo za HTTP/HTTPS**; za ostalo `NodePort`/`LoadBalancer`.
8. **Image-i moraju biti dostupni** — ili na javnom registry-u, ili učitani u lokalni klaster (`k3d image import` / `minikube image load`).
9. **Labels & selectors konzistentno** — `app: <ime>` minimum; dodati `version`, `tier` po potrebi.
10. **Resource requests/limits** definisati od početka, ne naknadno.
11. **Namespace-scoped resursi** — pamti da ConfigMap/Secret moraju biti u istom namespace-u kao Pod.
12. **Auto-update pravila ConfigMap-e** — ako koristim `subPath` ili `env`, planiram restart Pod-a kao deo flow-a.
13. **Komentariši YAML** kratko gde je logika ne-očigledna (npr. `# čekaj pre prve probe`).

---

## 10. Reference

- Quality of Service za Pod-ove: <https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/>
- Primer aplikacije: Dojo (PostgreSQL + Ingress + Helm + monitoring).
- UI dashboard: **Headlamp**.
