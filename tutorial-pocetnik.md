# ShopHub — tutorijal za frontend developera koji prvi put radi DevOps

> **Šta je ovo:** Onboarding vodič koji popunjava rupu između "znam React" i "mogu da pratim [to_do.md](plan%20projekta/to_do.md)".
>
> **Šta NIJE:** Nije ponavljanje koraka iz to_do.md. Detaljni koraci po fazama su tamo. Ovde su koncepti, mentalni modeli i ono što me je strah da preskočim.
>
> **Kontekst:** Tim od 3, >10 nedelja do odbrane, prioritet CI/CD + IaC. Cilj: 35–45/50 bodova.
>
> **Kako čitati:** Linearno od 0 do 5 — to su osnove. Dalje (6+) je referentno i pratiš po potrebi.

---

## Sadržaj

- [0. Šta ti treba pre nego što počneš](#0-šta-ti-treba-pre-nego-što-počneš)
- [1. Mali rečnik DevOps-a za frontend developera](#1-mali-rečnik-devops-a-za-frontend-developera)
- [2. Docker u 15 minuta](#2-docker-u-15-minuta)
- [3. Kubernetes u 30 minuta](#3-kubernetes-u-30-minuta)
- [4. Šta je Helm i zašto nam treba](#4-šta-je-helm-i-zašto-nam-treba)
- [5. Šta je Kubernetes operator i zašto ga pišemo](#5-šta-je-kubernetes-operator-i-zašto-ga-pišemo)
- [6. Faza 0 — kreiranje 5 repozitorijuma (eliminaciono)](#6-faza-0--kreiranje-5-repozitorijuma-eliminaciono)
- [7. Faza 1 — lokalno okruženje](#7-faza-1--lokalno-okruženje)
- [8. Faza 7 prošireno — helm-charts i kube-state](#8-faza-7-prošireno--helm-charts-i-kube-state)
- [9. Faza 8 prošireno — CI/CD pipeline-i](#9-faza-8-prošireno--cicd-pipeline-i)
- [10. Najčešće greške i kako ih izbeći](#10-najčešće-greške-i-kako-ih-izbeći)
- [11. Checklist pred odbranu](#11-checklist-pred-odbranu)
- [12. Učenje Go-a u 1 vikend](#12-učenje-go-a-u-1-vikend)
- [13. Resursi za dalje učenje](#13-resursi-za-dalje-učenje)

---

## 0. Šta ti treba pre nego što počneš

### 0.1. Alati koje obavezno instaliraj

Sve ove alate instaliraj **odmah, pre bilo čega drugog**. Verzije su iz [kubernetes_operatori_elegancija.md](vezbe%20pomocni%20materijali/kubernetes_operatori_elegancija.md) sekcija 2.

| Alat | Verzija (min) | Šta radi | Instalacija (Windows) |
|------|---------------|----------|------------------------|
| **Docker Desktop** | 25.x | Pravi i pokreće kontejnere | https://www.docker.com/products/docker-desktop/ |
| **Go** | 1.25.6 | Programski jezik za backend i operator | https://go.dev/dl/ |
| **kubectl** | 1.30+ | CLI za Kubernetes klaster | `winget install Kubernetes.kubectl` |
| **k3d** | 5.8.3+ | Lokalni Kubernetes klaster u Docker-u | `choco install k3d` ili sa GitHub releases |
| **Kubebuilder** | 4.11.0+ | Generator skeleta za Kubernetes operatore | https://github.com/kubernetes-sigs/kubebuilder/releases (preuzmeš binarku, staviš u PATH) |
| **Helm** | 3.14+ | Paket menadžer za Kubernetes (kao npm za K8s) | `choco install kubernetes-helm` |
| **Node.js** | 20+ | Za frontend (znaš već) | https://nodejs.org/ |
| **Git** | 2.40+ | Verzionisanje | https://git-scm.com/ |
| **VS Code** | bilo koja | Editor | sa Go ekstenzijom + Kubernetes ekstenzijom |

### 0.2. Provera da je sve instalirano

Otvori PowerShell i probaj:

```powershell
docker --version
go version
kubectl version --client
k3d version
kubebuilder version
helm version
node --version
git --version
```

Ako bilo šta vraća "command not found" — restartuj PowerShell. Ako i dalje ne radi — `PATH` env varijabla nema putanju do binarke. Dodaj je preko **Settings → System → Advanced → Environment Variables**.

### 0.3. GitHub naloga i organizacija

- Otvori **GitHub organizaciju** za tim (npr. `shophub-rs`) — besplatno za studente. Ide u **Settings → Organizations → New organization**.
- Pozovi sva tri člana kao članove.
- Svi commit-ovi treba da idu kroz PR-ove sa review-em — branch protection se postavlja **po repo-u**, ne na nivou org-a.

### 0.4. DockerHub nalog

Treba ti za publish-ovanje container image-a (zahtev 5.2) i Helm chart-ova kao OCI artefakata.

- Napravi nalog na https://hub.docker.com.
- Pod **Account Settings → Security → New Access Token** napravi token sa "Read, Write, Delete" pravima.
- Token sačuvaj na sigurnom — ide kao GitHub Actions secret pod imenom `DOCKERHUB_TOKEN`.
- Username takođe ide kao secret: `DOCKERHUB_USERNAME`.

### 0.5. Discord server za testiranje

Zahtev 3.1 traži **DiscordChannel CRD** — operator pravi kanale za notifikacije. Za to ti treba:

- Discord server (može i lični) — neka jedan član tima napravi `ShopHub Test Server`.
- **Discord bot** — `https://discord.com/developers/applications` → New Application → Bot → Reset Token (sačuvaj!).
- Pozovi bot na server sa **Manage Channels** + **Manage Webhooks** pravima.
- Bot token ide kao Kubernetes Secret kasnije; **nikad** ga ne commit-uj u Git.

### 0.6. MetaMask + Sepolia ETH

Za Fazu 5 (Web3 plaćanje) potreban je testnet wallet.

- Instaliraj MetaMask browser ekstenziju.
- Napravi novi nalog (NE koristi pravi nalog!).
- Sačuvaj seed frazu na sigurnom (testnet je, ali navika).
- U MetaMask → Networks → Add Network → **Sepolia testnet**.
- Idi na https://sepoliafaucet.com ili Alchemy faucet → uzmi 0.5 Sepolia ETH (besplatno).
- Za Sepolia USDT testnet adresu — postoji više testnih ERC-20 tokena, izabrati jedan koji ima faucet. Detalje u Fazi 5.

---

## 1. Mali rečnik DevOps-a za frontend developera

Pošto ti je frontend domaća zemlja, evo paralela:

| Frontend pojam | DevOps ekvivalent | Šta radi |
|-----------------|---------------------|----------|
| `npm install` | `helm install` | Instalira "paket" — samo umesto npm paketa, instalira Kubernetes resurse |
| `package.json` | `Chart.yaml` + `values.yaml` | Metapodaci paketa + default vrednosti |
| `node_modules/` | Container image | Sve zavisnosti aplikacije spakovano u jednom mestu |
| `vite build` → `dist/` | `docker build` → image | Build artefakt koji možeš negde da pošalješ |
| `npm run dev` | `kubectl apply -f deployment.yaml` | Pokrene aplikaciju (samo u K8s-u — pokreće Pod-ove) |
| Vercel / Netlify | Kubernetes klaster | Mesto gde aplikacija živi |
| `.env.production` | Kubernetes `Secret` + `ConfigMap` | Konfiguracija aplikacije po okruženju |
| React komponenta | Pod (kontejner) | Najmanja stavka aplikacije |
| `<Route>` u routeru | Kubernetes `Ingress` | Mapira URL putanje na komponente / servise |
| HMR (hot reload) | Reconcile loop operatora | Reaguje na izmene i automatski ažurira stanje |
| `useState` | Kubernetes `etcd` baza | Persistentno stanje sistema |
| ESLint | `hadolint` (za Dockerfile), `staticcheck` (za Go) | Linteri — provera koda pre commit-a |
| GitHub Actions na PR | Isto, GitHub Actions na PR | Identično, samo proverava više stvari (build + test + image + lint) |

**Velika razlika:** u frontend-u stanje (state) je u browseru i nestaje. U Kubernetes-u **sve je deklarativno** — opišeš željeno stanje, klaster ga doteruje. Nikad ne pišeš "pokreni 2 Pod-a"; pišeš "želim 2 Pod-a" i klaster osigurava da je tako.

To je **deklarativno vs imperativno** — koncept ti je poznat iz React-a (`setState({ count: 5 })` umesto `dom.innerText = "5"`). DevOps je još jače deklarativan.

---

## 2. Docker u 15 minuta

Sve detaljno je u [docker_elegancija.md](vezbe%20pomocni%20materijali/docker_elegancija.md). Ovo je koncept, ne ponavljanje.

### 2.1. Šta je Docker image

Frontend analog: image je kao `dist/` folder + sve što treba da bi taj `dist/` radio (Node, biblioteke, OS). **Jedan zip fajl koji sadrži kompletan softver sa zavisnostima.**

Razlika od virtuelne mašine: VM ima ceo OS (gigabajti). Image deli OS kernel sa hostom — kilobajti do megabajti.

### 2.2. Šta je kontejner

Kontejner = **instanca image-a koja se pokreće**. Image je kao class, kontejner je kao instance.

```powershell
docker run nginx          # image "nginx" → pokrene jedan kontejner
docker ps                  # vidi koje kontejnere imaš pokrenute
docker stop <container-id> # zaustavi kontejner
```

### 2.3. Dockerfile = recept

```dockerfile
FROM node:20-alpine          # baza (kao "extends" u CSS-u)
WORKDIR /app                 # promeni dir unutar slike
COPY package.json .          # kopiraj fajl iz hosta u sliku
RUN npm install              # izvrši komandu pri build-u
COPY . .                     # kopiraj sve ostalo
EXPOSE 3000                  # informacija "ovaj kontejner sluša port 3000"
CMD ["node", "server.js"]    # šta se pokrene kad startuješ kontejner
```

Build:
```powershell
docker build -t my-app:1.0 .   # tag = ime:verzija; "." = build context
docker run -p 3000:3000 my-app:1.0
```

### 2.4. Multi-stage build (OBAVEZNO za projekat)

Problem: ako u jednom stage-u kompajluješ Go, image je 1GB (jer ima Go kompajler + izvorni kod). Ne treba ti to u produkciji.

Rešenje: **dva stage-a**. Prvi kompajluje, drugi samo kopira binarku.

```dockerfile
# Stage 1: builder
FROM golang:1.25-alpine AS builder
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /out/app ./...

# Stage 2: release (mala slika)
FROM alpine:3.20
COPY --from=builder /out/app /app
USER 1000                 # non-root — bezbednosno bitno
EXPOSE 8080
ENTRYPOINT ["/app"]
```

Ovo je standard za projekat — pogledaj sekciju 1.1 [docker_elegancija.md](vezbe%20pomocni%20materijali/docker_elegancija.md).

### 2.5. Šta je `hadolint`

Linter za Dockerfile. Kao ESLint za frontend. Proverava da li si poštovao prakse (multi-stage, non-root, pin verzije, ne kombinuj `apt-get install` bez `--no-install-recommends`...).

```powershell
docker run --rm -i hadolint/hadolint < Dockerfile
```

**Pravilo za projekat:** CI mora obavljati hadolint check na svaki PR. Ako ne prolazi → PR se ne merge-uje.

### 2.6. Šta nikad NE raditi

- **Ne stavljaj secret-e u Dockerfile** (`ENV PASSWORD=...`). Vidljivi su preko `docker image inspect`. Koristi `--mount=type=secret` (sekcija 1.5 docker eleganta) ili Kubernetes Secret-e u produkciji.
- **Ne pokreći aplikaciju kao root** (`USER root`). Default je root, što je loše. Uvek `USER app` ili `USER 1000`.
- **Ne kombinuj `RUN apt update` i `RUN apt install` u dva sloja** — drugi koristi keš pa može da koristi zastareli indeks paketa. Spoji ih: `RUN apt update && apt install ...`.

---

## 3. Kubernetes u 30 minuta

Sve detaljno je u [kubernetes_elegancija.md](vezbe%20pomocni%20materijali/kubernetes_elegancija.md). Ovo je big picture.

### 3.1. Šta je klaster

Klaster = **gomila mašina (nodes) koje rade zajedno kao jedna**. Svaki node pokreće kontejnere. Klaster ima:
- **Control plane** (master nodes) — mozak. Ima API Server, scheduler, controller manager, etcd.
- **Worker nodes** — pokreću kontejnere.

Za projekat koristimo **k3d** — lagani lokalni klaster koji pokreće svaki node kao Docker kontejner. Tako možeš da imaš 3-node klaster na svom laptopu.

```powershell
k3d cluster create --config cluster.yaml
kubectl get nodes
# NAME                  STATUS   ROLES                  AGE   VERSION
# k3d-local-server-0    Ready    control-plane,master   1m    v1.30.x
# k3d-local-agent-0     Ready    <none>                 1m    v1.30.x
# k3d-local-agent-1     Ready    <none>                 1m    v1.30.x
```

### 3.2. Hierarchija resursa

Od najvišeg do najnižeg:

```
Klaster
├── Namespace (tipa "folder" u klasteru)
│   ├── Deployment (recept za Pod-ove)
│   │   ├── ReplicaSet (interno — broj replika)
│   │   │   └── Pod (1+ kontejnera koji rade zajedno)
│   │   │       └── Container (Docker kontejner)
│   ├── Service (DNS + load balancer ka Pod-ovima)
│   ├── Ingress (HTTP ruter)
│   ├── ConfigMap (konfiguracija — env, fajlovi)
│   ├── Secret (osetljiva konfiguracija)
│   └── ServiceAccount + RBAC (kome je dozvoljeno šta)
└── Cluster-scoped resursi (npr. Node, CRD, ClusterRole)
```

**Namespace** je analog "projekta" u GitLab-u — sve unutar je grupisano. Mi ćemo imati npr. `shophub-system`, `tenant-alice`, `tenant-bob` namespace-e.

### 3.3. Šta je Pod, Deployment, Service, Ingress

**Pod** — najmanja jedinica. **Jedan ili više kontejnera koji dele mrežu i storage.** Najčešće je jedan kontejner (tvoja aplikacija) + opciono "sidecar" (npr. log agent). Ne pravi Pod direktno — uvek preko Deployment-a.

**Deployment** — "želim N replika ovog Pod-a". Ako jedan Pod padne, kreira drugi. Ako menjaš image, radi rolling update (postupno menja Pod-ove jedan po jedan).

**Service** — "stabilan DNS i load balancer za grupu Pod-ova". Pod ima dinamički IP koji se menja na restart; Service ima stabilno ime. Recept: ako pišeš `http://shop-backend:8080`, K8s razmota to u IP nekog Pod-a iz Service-a.

**Ingress** — "ruter za HTTP saobraćaj sa interneta u klaster". Bez Ingress-a, klaster je zatvoren spolja. Sa Ingress-om: `https://shop.example.com/api/items → Service "shop-backend" → Pod`.

### 3.4. YAML manifest = deklarativno stanje

Sve u Kubernetes-u je YAML fajl. Ti opisuješ **šta želiš**, klaster doteruje.

```yaml
apiVersion: apps/v1
kind: Deployment            # tip resursa
metadata:
  name: shop-backend         # ime — jedinstveno u namespace-u
  namespace: tenant-alice
spec:
  replicas: 2                # ŽELIM 2 Pod-a
  selector:
    matchLabels:
      app: shop-backend
  template:
    metadata:
      labels:
        app: shop-backend    # mora se podudarati sa matchLabels iznad
    spec:
      containers:
        - name: shop
          image: docker.io/mile/shop-backend:1.0.0
          ports:
            - containerPort: 8080
          livenessProbe:        # obavezno — vidi sekciju 2 k8s eleganta
            httpGet:
              port: 8080
              path: /probe/liveness
          readinessProbe:
            httpGet:
              port: 8080
              path: /probe/readiness
```

Primena:
```powershell
kubectl apply -f deployment.yaml
kubectl get pods -n tenant-alice
kubectl logs -n tenant-alice shop-backend-abc123-xyz
kubectl describe pod shop-backend-abc123-xyz -n tenant-alice
```

### 3.5. Komande koje ćeš najčešće koristiti

```powershell
# Sve u namespace-u
kubectl get all -n shophub-system

# Vidi pod i njegove logove
kubectl get pods -n shophub-system
kubectl logs -n shophub-system <pod-name> -f      # -f = follow (kao tail -f)

# Ulazak u kontejner
kubectl exec -it -n shophub-system <pod-name> -- sh

# Detalji + greške
kubectl describe pod <pod-name> -n shophub-system

# Brisanje
kubectl delete -f deployment.yaml
# ili
kubectl delete pod <pod-name> -n shophub-system

# Port forward (pristup servisu sa lokalne mašine)
kubectl port-forward -n shophub-system service/shop-backend 8080:8080
# Sad otvori http://localhost:8080 u browseru

# Pretraga preko svih namespace-a
kubectl get pods -A
```

### 3.6. Custom Resource Definition (CRD)

**CRD = novi tip resursa koji ti dodaješ Kubernetes-u.** Kao klasa u TypeScript-u — definišeš novi tip.

Iz koraka — mi pravimo `Shop`, `DiscordChannel`, `Wallet` CRD-ove. Posle instalacije, možeš da pišeš:

```yaml
apiVersion: apps.shophub.local/v1
kind: Shop
metadata:
  name: alice-clothing
spec:
  title: "Alice's Clothing"
  availability: high
  database: postgres
  walletAddress: "0xAB..."
```

I onda `kubectl apply -f`. **Sam Kubernetes ne zna šta da radi sa Shop CR-om**. Za to nam treba **operator** — softver koji čita Shop CR i radi šta treba (pravi Deployment, Service, bazu...).

---

## 4. Šta je Helm i zašto nam treba

### 4.1. Problem koji rešava

Zamisli da treba da deploy-uješ aplikaciju koja ima:
- Deployment (backend)
- Deployment (frontend)
- Service za backend
- Service za frontend
- Ingress
- ConfigMap za backend env
- Secret za database password
- PersistentVolumeClaim za upload-e

To je 8 YAML fajlova. Ako menjaš ime aplikacije, moraš to menjati u 8 fajlova ručno. **Helm rešava problem template-iziranjem.**

### 4.2. Helm chart = paket

Chart je folder sa strukturom:

```
shop/
├── Chart.yaml           # metapodaci (ime, verzija, opis)
├── values.yaml          # default vrednosti — "konfiguracija"
└── templates/
    ├── deployment.yaml  # YAML sa {{ .Values.x }} placeholder-ima
    ├── service.yaml
    └── ingress.yaml
```

`Chart.yaml`:
```yaml
apiVersion: v2
name: shop
description: Shop application
type: application
version: 0.1.0           # verzija chart-a
appVersion: "1.0.0"      # verzija aplikacije unutra
```

`values.yaml`:
```yaml
replicaCount: 2
image:
  repository: docker.io/mile/shop-backend
  tag: 1.0.0
service:
  port: 8080
```

`templates/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-shop
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: shop
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Instalacija:
```powershell
helm install my-shop ./charts/shop
# ili sa override:
helm install my-shop ./charts/shop --set replicaCount=3 --set image.tag=2.0.0
```

Helm rendera template-e i šalje finalni YAML klasteru.

### 4.3. Dependencies (jako bitno za projekat)

Chart može da zavisi od drugog chart-a. To je kao `package.json` `dependencies`.

`Chart.yaml`:
```yaml
apiVersion: v2
name: shophub
version: 0.1.0
dependencies:
  - name: kube-prometheus-stack       # zahtev 3.3 — koristimo prometheus-stack
    version: "65.x.x"
    repository: https://prometheus-community.github.io/helm-charts
```

Posle:
```powershell
helm dependency update ./charts/shophub
```

Helm spusti `kube-prometheus-stack-65.x.x.tgz` u `charts/shophub/charts/` i instalira ga zajedno.

### 4.4. OCI publish

Helm chart-ove možeš push-ovati na DockerHub kao OCI artefakte (Open Container Initiative). To koristimo jer **specifikacija 5.3** traži da chart-ovi budu u `helm-charts` repo-u, a `kube-state` repo treba da ih referencira preko `chart: oci://...`.

```powershell
helm package ./charts/shop
# kreira shop-0.1.0.tgz

helm push shop-0.1.0.tgz oci://docker.io/<tvoj-username>
```

### 4.5. CRD folder u Helm chart-u — JAKO BITNO

U Helm 3, sve što stavši u `crds/` folder (ne `templates/`) — instalira se **pre** template-ova i **nikad se ne ažurira na upgrade**.

```
shop-operator/
├── Chart.yaml
├── values.yaml
├── crds/                              # CRD-ovi idu OVDE
│   ├── apps.shophub.local_shops.yaml
│   ├── notify.shophub.local_discordchannels.yaml
│   └── payments.shophub.local_wallets.yaml
└── templates/
    ├── deployment.yaml                # operator Pod
    ├── serviceaccount.yaml
    ├── clusterrole.yaml
    └── clusterrolebinding.yaml
```

**Zašto je to bitno:** CRD definicija (npr. dodavanje novog polja u Shop spec) zahteva pažljiv migration. Helm namerno ne dira CRD-ove na `helm upgrade` da bi te zaštitio od slučajnog brisanja podataka.

---

## 5. Šta je Kubernetes operator i zašto ga pišemo

Sve detaljno je u [kubernetes_operatori_elegancija.md](vezbe%20pomocni%20materijali/kubernetes_operatori_elegancija.md). Ovo je big picture.

### 5.1. Problem koji rešava

Specifikacija kaže: "korisnik kreira Shop, sistem deploy-uje **kompletnu aplikaciju** (backend + frontend + baza + wallet + Discord kanal) u Kubernetes-u."

Bez operatora, kada korisnik kreira "shop", backend bi morao da:
1. Generiše unique ime
2. Šalje 8 YAML fajlova klasteru (`kubectl apply`)
3. Čeka da budu Running
4. Update-uje state u sopstvenoj bazi

**Sa operatorom:**
1. Backend kreira samo **jedan** Shop CR (`kubectl apply -f shop.yaml`).
2. Operator (koji uvek radi u pozadini) hvata da je Shop kreiran.
3. Operator **sam** pravi 8 child resursa, prati ih, hvata greške.
4. Status Shop CR-a postaje izvor istine — backend čita status iz K8s-a, ne iz lokalne baze.

**Mental model:** operator je kao React komponenta — ima `Reconcile()` funkciju koja se okida na svaku promenu. U njoj porediš trenutno stanje sa željenim i rešavaš razliku.

```
React:                        Kubernetes operator:
state changes → render()      Shop CR changes → Reconcile()
JSX vs DOM diff               desired vs actual K8s state
React updates DOM             Operator creates/updates/deletes resources
```

### 5.2. Kubebuilder — generator skeleta

Kubebuilder je framework koji generiše 90% boilerplate koda za operator. Ti pišeš samo `Reconcile()` logiku.

```powershell
mkdir shop-operator
cd shop-operator
kubebuilder init --domain shophub.local --project-name shop-operator --repo github.com/<org>/shop-operator
kubebuilder create api --group apps --version v1 --kind Shop --resource --controller --plural shops
```

To pravi:
```
shop-operator/
├── api/v1/shop_types.go              # tu definišeš Spec i Status
├── internal/controller/shop_controller.go  # tu pišeš Reconcile
├── config/crd/                       # auto-generisan CRD YAML
├── config/rbac/                      # auto-generisan RBAC
├── config/manager/                   # Deployment za operator Pod
└── Makefile                          # `make manifests`, `make install`, `make run`
```

### 5.3. Reconcile loop u jednoj rečenici

**"Za svaku izmenu Shop CR-a, dovedi kompletan sistem u stanje koje Spec opisuje."**

Ključna pravila (sve detaljno u sekciji 5 [kubernetes_operatori_elegancija.md](vezbe%20pomocni%20materijali/kubernetes_operatori_elegancija.md)):

1. **Idempotentno** — ako pozoveš `Reconcile` dva puta zaredom, drugi put ništa ne radi (jer je stanje već dobro).
2. **Level-based** — ne zanima te šta se dešavalo, samo trenutno vs željeno stanje.
3. **Ne menjaš Spec** — samo Status.
4. **Ako se desi greška** — vrati error, K8s će automatski retry sa exponential backoff.
5. **Ne čekaj u petlji** — vrati `RequeueAfter: 30s` i pusti K8s da te ponovo okine.

### 5.4. Šta operator NE radi

- Ne čuva state na disku — `etcd` je njegova baza (preko Status objekta).
- Ne komunicira sa korisnikom direktno — korisnik priča sa ShopHub backend-om, ShopHub backend priča sa K8s API-jem (pravi Shop CR), operator hvata Shop CR.
- Nema HTTP endpointe (osim `/metrics` za Prometheus).

---

## 6. Faza 0 — kreiranje 5 repozitorijuma (eliminaciono)

**Ovo je dan 1 projekta. Pre koda.** Specifikacija zahtev 5.3 je eliminacioni — ako ovo nije pravilno organizovano, projekat se NE pregleda.

### 6.1. Pet repozitorijuma — šta ide gde

Pojednostavljeno:

| Repo | Šta sadrži | Vlasnik | Jezik |
|------|------------|---------|-------|
| `shop-operator` | Go kod operatora + CRD-ovi | Član A | Go |
| `shop` | Shop backend (Go/Gin) + Shop frontend (React/Vite) | Član B + C | Go + TS |
| `shophub` | ShopHub backend (Go) + ShopHub frontend (React/Vite) | Član B + C | Go + TS |
| `helm-charts` | **Sve** Helm chart-ove koje sami pišete | Tim zajedno | YAML |
| `kube-state` | Stanje klastera — koje chart-ove instaliramo i sa kojim values | Član A | YAML |

> **Često pitanje:** "Zašto je Shop backend i Shop frontend u istom repo-u?" Specifikacija kaže "svaki mikroservis svoj repo". Shop backend i Shop frontend su deo istog "vlasništva" (Shop), pa idu zajedno. Ako sutra dodaš `shop-payments` kao zaseban mikroservis, to ide u svoj repo.

### 6.2. Koraci za svaki repo (10 minuta po repo-u)

**Korak 1.** Idi na GitHub → tvoja organizacija → New repository.
- Name: `shop-operator` (pa `shop`, pa `shophub`, pa `helm-charts`, pa `kube-state`).
- **Public** (lakše ti je kod CI/CD, manje secret-a).
- Initialize sa README.md.
- License: MIT (preporuka).
- .gitignore: Go (za prva tri), None (za helm-charts i kube-state).

**Korak 2.** Naprav prvi commit sa Conventional Commit porukom:
```powershell
git clone https://github.com/<org>/shop-operator.git
cd shop-operator
# napravi malu izmenu (npr. dodaj liniju u README)
git add .
git commit -m "chore: initial structure"
git push
```

**Korak 3.** Branch protection — **NAJBITNIJI deo**. Ide u repo → Settings → Branches → Add branch protection rule.

Postavke za `main`:
- ✅ Require a pull request before merging
- ✅ Require approvals: **1**
- ✅ Dismiss stale pull request approvals when new commits are pushed
- ✅ Require status checks to pass before merging (kasnije ćeš dodati pipeline)
- ✅ Require branches to be up to date before merging
- ✅ Require linear history *(zahtev 5.1)*
- ✅ Include administrators (da i admini moraju da idu kroz PR)
- ❌ Allow force pushes (NIKAD)
- ❌ Allow deletions (NIKAD)

**Korak 4.** Postavi **GitHub Actions secrets** (Settings → Secrets and variables → Actions):
- `DOCKERHUB_USERNAME` — tvoj DockerHub username
- `DOCKERHUB_TOKEN` — token koji si napravio u sekciji 0.4

**Korak 5.** Conventional Commits — instaliraj `commitlint`:

`.github/workflows/commit-lint.yml`:
```yaml
name: Lint Commit Messages
on: [pull_request]
jobs:
  commitlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - uses: wagoid/commitlint-github-action@v6
```

Sa `commitlint.config.js` u root-u:
```js
module.exports = { extends: ['@commitlint/config-conventional'] };
```

**Ponovi koraci 1-5 za sva 5 repozitorijuma.** Vreme: oko sat vremena po članu tima.

### 6.3. Struktura `helm-charts` repozitorijuma

Specifikacija 5.3 doslovno propisuje strukturu. **Mora biti baš ovako**:

```
helm-charts/
├── README.md
└── charts/
    ├── shop/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   └── templates/
    ├── shophub/
    │   ├── Chart.yaml
    │   ├── values.yaml
    │   ├── charts/                # subchart-ovi (kube-prometheus-stack)
    │   └── templates/
    │       ├── NOTES.txt
    │       ├── _helpers.tpl
    │       ├── deployment.yaml
    │       ├── hpa.yaml
    │       ├── ingress.yaml
    │       ├── service.yaml
    │       └── serviceaccount.yaml
    └── shop-operator/
        ├── Chart.yaml
        ├── values.yaml
        ├── crds/
        └── templates/
```

**Bootstrap sve odmah** (dan 1) — prazni fajlovi sa `.gitkeep` su OK:

```powershell
cd helm-charts
mkdir charts
mkdir charts/shop
mkdir charts/shop/templates
mkdir charts/shophub
mkdir charts/shophub/templates
mkdir charts/shophub/charts
mkdir charts/shop-operator
mkdir charts/shop-operator/crds
mkdir charts/shop-operator/templates

# .gitkeep u svaki prazan folder
ni charts/shop/templates/.gitkeep
ni charts/shophub/templates/.gitkeep
ni charts/shop-operator/templates/.gitkeep
ni charts/shop-operator/crds/.gitkeep

# Minimalni Chart.yaml fajlovi
@'
apiVersion: v2
name: shop
description: Shop application chart
type: application
version: 0.1.0
appVersion: "0.1.0"
'@ | Out-File -FilePath charts/shop/Chart.yaml -Encoding utf8

# Isto za shophub i shop-operator (samo zameni "name" i "description").

git add .
git commit -m "chore: initial helm-charts structure"
git push
```

**Provera:**
```powershell
helm lint charts/shop          # mora proći
helm lint charts/shophub
helm lint charts/shop-operator
```

### 6.4. Struktura `kube-state` repozitorijuma

```
kube-state/
├── README.md
└── clusters/
    └── local/
        ├── cluster.yaml                  # k3d cluster config
        ├── shop-operator/
        │   ├── helm.yaml
        │   └── values.yaml
        ├── shophub/
        │   ├── helm.yaml
        │   └── values.yaml
        ├── cnpg/
        │   ├── helm.yaml
        │   └── values.yaml
        ├── redis-operator/
        │   ├── helm.yaml
        │   └── values.yaml
        └── kube-prometheus-stack/
            ├── helm.yaml
            └── values.yaml
```

`clusters/local/cluster.yaml`:
```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: local
servers: 1
agents: 2
ports:
  - port: 8080:80
    nodeFilters:
      - loadbalancer
  - port: 8443:443
    nodeFilters:
      - loadbalancer
```

`clusters/local/cnpg/helm.yaml`:
```yaml
chart: cnpg/cloudnative-pg
version: 0.21.x
namespace: cnpg-system
createNamespace: true
repository: https://cloudnative-pg.github.io/charts
```

`clusters/local/shop-operator/helm.yaml`:
```yaml
chart: oci://docker.io/<tvoj-dockerhub-username>/shop-operator
version: 0.1.0
namespace: shop-operator-system
createNamespace: true
```

`README.md` mora da sadrži uputstva kako se klaster diže od nule:

```markdown
# kube-state

Ovaj repo opisuje stanje lokalnog Kubernetes klastera za projekat ShopHub.

## Prerequisites
- Docker Desktop pokrenut
- k3d 5.8.3+, helm 3.14+, kubectl 1.30+

## Setup
1. Klaster: `k3d cluster create --config clusters/local/cluster.yaml`
2. CNPG: `helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace`
3. Redis op: `helm install redis-operator redis-operator/redis-operator -n redis-operator --create-namespace`
4. Shop operator: `helm install shop-operator oci://.../shop-operator -n shop-operator-system --create-namespace`
5. ShopHub: `helm install shophub oci://.../shophub -n shophub-system --create-namespace`
```

> **Bonus (+3 boda):** umesto manualnih `helm install`, koristi **ArgoCD** ili **Flux** GitOps alat. ArgoCD čita `kube-state` repo i automatski sinhronizuje klaster. Detalje u sekciji 8.5 ovog tutorijala.

### 6.5. Checkpoint Fazi 0

Pre nego što pomeriš dalje:

- [ ] Svih 5 repo-a postoji na GitHub-u.
- [ ] Branch protection postavljen na svakom (PR, approval, linear history).
- [ ] `DOCKERHUB_USERNAME` i `DOCKERHUB_TOKEN` secret-i u svakom (osim helm-charts/kube-state — tamo nije obavezno).
- [ ] commitlint workflow u svakom.
- [ ] `helm-charts` ima tačnu strukturu foldera; `helm lint` prolazi.
- [ ] `kube-state` ima `clusters/local/cluster.yaml` i README sa uputstvom za bootstrap.
- [ ] Svaki repo ima commit "chore: initial structure" sa Conventional Commits porukom.

**Ako je sve OK, projekat sad zadovoljava eliminacioni zahtev 5.3** — možeš da nastaviš.

---

## 7. Faza 1 — lokalno okruženje

Detalji su u [to_do.md sekcija 1](plan%20projekta/to_do.md). Ja ti dajem komentar za neke gotchas.

### 7.1. k3d podiže se za 60 sekundi

```powershell
cd kube-state/clusters/local
k3d cluster create --config cluster.yaml
kubectl cluster-info
kubectl get nodes
```

Ako Docker Desktop nije pokrenut → `k3d` neće raditi. **Uvek prvo proveri da li je Docker Desktop tray ikona "zelena"** (Settings → General → "Start Docker Desktop when you log in" je korisno).

### 7.2. Instalacija CNPG i Redis operator-a

```powershell
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo add redis-operator https://spotahome.github.io/redis-operator
helm repo update

helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace
helm install redis-operator redis-operator/redis-operator -n redis-operator --create-namespace

# Provera
kubectl get pods -n cnpg-system
kubectl get pods -n redis-operator
# Svi treba da budu Running (oko 30 sekundi).
```

### 7.3. Test CNPG-a (opciono ali korisno)

Naprav `test-pg.yaml`:
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: test-pg
  namespace: default
spec:
  instances: 1
  bootstrap:
    initdb:
      database: testdb
      owner: testuser
  storage:
    size: 1Gi
```

```powershell
kubectl apply -f test-pg.yaml
kubectl get cluster -n default          # videćeš test-pg
kubectl get pods -n default             # videćeš test-pg-1 Pod
kubectl get secret -n default           # videćeš test-pg-app, test-pg-ca, ...
```

Sekret `test-pg-app` sadrži kredencijale (`username`, `password`, `host`, `uri`, ...). To je **ono što će Shop operator čitati** kad pravi env za Shop backend.

```powershell
# Vidi sadržaj
kubectl get secret test-pg-app -o jsonpath='{.data.uri}' -n default | base64 -d
```

### 7.4. Loadovanje lokalnog Docker image-a u k3d

Najveća greška početnika: build-uješ image lokalno, pokrećeš Pod u k3d, dobijaš `ImagePullBackOff`. **Razlog:** k3d ne vidi tvoj lokalni Docker — svaki k3d node je zaseban Docker kontejner.

Rešenje:
```powershell
docker build -t shop-backend:dev .
k3d image import shop-backend:dev -c local
```

**Imenuje image bez "docker.io/..." prefixa za lokalni razvoj.** U Pod manifest-u onda:
```yaml
image: shop-backend:dev
imagePullPolicy: IfNotPresent     # ne pokušavaj da skineš sa registry-a
```

---

## 8. Faza 7 prošireno — helm-charts i kube-state

**Ti si rekao da te ovo najviše plaši — pa idemo polako i detaljno.**

### 8.1. Pravljenje shop-operator Helm chart-a

Pretpostavka: već si napravio Shop operator u `shop-operator/` repo-u (Faza 2), i `make manifests` ti je generisao CRD-ove u `shop-operator/config/crd/bases/`.

**Cilj:** prebaciti ove CRD-ove + deployment operator-a u `helm-charts/charts/shop-operator/`.

**Korak 1.** Kopiraj CRD-ove:

```powershell
# U shop-operator repo-u
make manifests generate

# Sad u helm-charts repo-u (sklon je problemu sa različitim direktorijumima — najlakše ručno)
cp ../shop-operator/config/crd/bases/*.yaml ./charts/shop-operator/crds/
```

> **Tip:** Napravi `Makefile` u helm-charts repo-u sa target-om `make sync-crds` koji to radi automatski. CI će kasnije ovo proveriti.

**Korak 2.** Napravi `Chart.yaml`:
```yaml
apiVersion: v2
name: shop-operator
description: Shop operator for ShopHub platform - manages Shop, DiscordChannel, and Wallet CRDs
type: application
version: 0.1.0
appVersion: "0.1.0"
keywords:
  - kubernetes
  - operator
  - shophub
maintainers:
  - name: <ime-tima>
    email: <email>
```

**Korak 3.** Napravi `values.yaml`:
```yaml
image:
  repository: docker.io/<tvoj-username>/shop-operator
  tag: 0.1.0
  pullPolicy: IfNotPresent

replicaCount: 1

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

# Discord — Secret koji operator koristi za Discord API
discord:
  botTokenSecretName: discord-bot-token
```

**Korak 4.** Templates:

`templates/serviceaccount.yaml`:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Release.Name }}-controller-manager
  namespace: {{ .Release.Namespace }}
```

`templates/clusterrole.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ .Release.Name }}-manager-role
rules:
{{- range .Files.Lines "config/rbac/role.yaml" }}
{{ . }}
{{- end }}
```

> **Najlakše:** umesto da rendaš RBAC iz fajla, samo kopiraj sadržaj `shop-operator/config/rbac/role.yaml` direktno u templates fajl. CRD ti `make manifests` regeneriše svaki put — automatizuj sync.

`templates/clusterrolebinding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: {{ .Release.Name }}-manager-binding
subjects:
  - kind: ServiceAccount
    name: {{ .Release.Name }}-controller-manager
    namespace: {{ .Release.Namespace }}
roleRef:
  kind: ClusterRole
  name: {{ .Release.Name }}-manager-role
  apiGroup: rbac.authorization.k8s.io
```

`templates/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-controller-manager
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}-controller-manager
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}-controller-manager
    spec:
      serviceAccountName: {{ .Release.Name }}-controller-manager
      containers:
        - name: manager
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          args:
            - --health-probe-bind-address=:8081
            - --metrics-bind-address=:8080
            - --leader-elect
          ports:
            - name: metrics
              containerPort: 8080
            - name: health
              containerPort: 8081
          livenessProbe:
            httpGet:
              path: /healthz
              port: health
            initialDelaySeconds: 15
          readinessProbe:
            httpGet:
              path: /readyz
              port: health
          resources:
{{ toYaml .Values.resources | indent 12 }}
```

`templates/service.yaml` (za `/metrics`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}-metrics
  namespace: {{ .Release.Namespace }}
  labels:
    app: {{ .Release.Name }}-controller-manager
spec:
  selector:
    app: {{ .Release.Name }}-controller-manager
  ports:
    - name: metrics
      port: 8080
      targetPort: metrics
```

`templates/servicemonitor.yaml` (uslovno — ako je Prometheus instaliran):
```yaml
{{- if .Values.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Release.Name }}-controller-manager
  namespace: {{ .Release.Namespace }}
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: {{ .Release.Name }}-controller-manager
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
{{- end }}
```

> **Zašto je `release: kube-prometheus-stack` label bitan:** default Prometheus selector pokupi samo ServiceMonitor-e sa tim label-om. Bez ovoga, metrike se neće skupljati. Detaljnije u [to_do.md sekcija 6.3](plan%20projekta/to_do.md).

**Korak 5.** Provera:
```powershell
helm lint charts/shop-operator
helm template charts/shop-operator                   # vidi rendered YAML
helm install --dry-run shop-operator charts/shop-operator -n shop-operator-system --create-namespace
# Ako ne paniči, pravi instalacija:
helm install shop-operator charts/shop-operator -n shop-operator-system --create-namespace
kubectl get pods -n shop-operator-system
```

### 8.2. Pravljenje shophub Helm chart-a sa dependencies

`charts/shophub/Chart.yaml`:
```yaml
apiVersion: v2
name: shophub
description: ShopHub platform with monitoring stack
type: application
version: 0.1.0
appVersion: "0.1.0"
dependencies:
  - name: kube-prometheus-stack
    version: "65.x.x"
    repository: https://prometheus-community.github.io/helm-charts
  - name: loki-stack
    version: "2.10.x"
    repository: https://grafana.github.io/helm-charts
    condition: loki-stack.enabled       # može da se isključi
```

```powershell
helm dependency update charts/shophub
# Skida kube-prometheus-stack i loki-stack u charts/shophub/charts/*.tgz
```

`charts/shophub/values.yaml`:
```yaml
shophub:
  image:
    repository: docker.io/<username>/shophub-backend
    tag: 0.1.0
  replicaCount: 2
  ingress:
    enabled: true
    host: shophub.local

# Override default values dependence-a
kube-prometheus-stack:
  grafana:
    adminPassword: changeme            # u produkciji: external Secret
    sidecar:
      dashboards:
        enabled: true                  # auto-import ConfigMap dashboard-a
        searchNamespace: ALL

loki-stack:
  enabled: true
  promtail:
    enabled: true
```

> **U pravoj produkciji:** `adminPassword` ide u Kubernetes Secret koji se referencira preko `existingSecret`. Za projekat, override preko CLI radi: `helm install --set kube-prometheus-stack.grafana.adminPassword=$env:GRAFANA_PWD`.

### 8.3. Pravljenje shop Helm chart-a

Ovaj chart **operator NE koristi direktno** — operator pravi Deployment/Service/Ingress sam, preko `client-go`. Ali specifikacija 5.3 ga pominje u strukturi.

**Predlog (najbezbednije):** chart kao "fallback" — ručna instalacija jednog Shop-a bez operatora.

`charts/shop/values.yaml`:
```yaml
title: "Demo Shop"
availability: standard
walletAddress: "0x0000000000000000000000000000000000000000"
database: postgres
backend:
  image:
    repository: docker.io/<username>/shop-backend
    tag: 0.1.0
frontend:
  image:
    repository: docker.io/<username>/shop-frontend
    tag: 0.1.0
ingress:
  host: demo-shop.local
```

`templates/shop.yaml` — pravi se Shop CR koji operator hvata:
```yaml
apiVersion: apps.shophub.local/v1
kind: Shop
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  title: {{ .Values.title | quote }}
  availability: {{ .Values.availability }}
  database: {{ .Values.database }}
  walletAddress: {{ .Values.walletAddress | quote }}
  image: {{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}
```

Tako "shop chart" postaje **demo helper** — `helm install demo-shop charts/shop -n tenant-demo` pravi Shop CR, operator dalje radi posao.

### 8.4. Publish chart-ova na DockerHub kao OCI

```powershell
# Login na DockerHub OCI registry
echo $env:DOCKERHUB_TOKEN | helm registry login -u $env:DOCKERHUB_USERNAME --password-stdin docker.io

# Pakuj
helm package charts/shop-operator
helm package charts/shophub
helm package charts/shop

# Push
helm push shop-operator-0.1.0.tgz oci://docker.io/<username>
helm push shophub-0.1.0.tgz oci://docker.io/<username>
helm push shop-0.1.0.tgz oci://docker.io/<username>
```

**Provera da je objavljeno:**
```powershell
helm pull oci://docker.io/<username>/shop-operator --version 0.1.0
```

### 8.5. ArgoCD GitOps — opcioni bonus (+3 boda)

ArgoCD prati `kube-state` repo i automatski sinhronizuje klaster. To je **GitOps**.

```powershell
# Instalacija ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Pristup UI
kubectl port-forward svc/argocd-server -n argocd 8443:443
# Otvori https://localhost:8443
# Username: admin
# Password: kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

Posle, registruj `kube-state` repo:
- ArgoCD UI → Settings → Repositories → Connect repo
- Tip: HTTPS, URL: https://github.com/<org>/kube-state.git

Pa naprav `Application` po komponenti:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shop-operator
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/kube-state.git
    targetRevision: main
    path: clusters/local/shop-operator
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: shop-operator-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Kad pushuješ promenu u `kube-state` repo, ArgoCD primeti i automatski primeni na klaster.

> **Ako biraš ovaj put** — strukturu `clusters/local/...` u kube-state repo-u prilagodiš da ArgoCD razume (svaka komponenta ima `Application` manifest umesto `helm.yaml` + `values.yaml`).

### 8.6. Checkpoint Faze 7

- [ ] `helm lint` prolazi za sva 3 chart-a.
- [ ] `helm template` rendera tačne YAML-ove (bez `<no value>`).
- [ ] `helm install --dry-run` prolazi.
- [ ] Sva 3 chart-a push-ovana kao OCI artefakti.
- [ ] `helm pull oci://...` radi sa drugog mašina.
- [ ] `kube-state` README sadrži korak-po-korak uputstvo.
- [ ] **Test "bootstrap from scratch":** obriši k3d klaster (`k3d cluster delete local`), pa prati samo README iz kube-state — sve treba da se digne.

---

## 9. Faza 8 prošireno — CI/CD pipeline-i

**Druga oblast koja te plaši — idemo polako.**

### 9.1. Šta je GitHub Actions

GitHub Actions je **CI/CD platforma ugrađena u GitHub**. Definiseš workflow u YAML fajlu pod `.github/workflows/`, on se okida na neki **event** (push, pull request, schedule), izvršava se na **runner** mašini (Linux/macOS/Windows VM), i kao output prijavljuje status nazad u GitHub.

Mental model: kao npm script, samo se pokreće u cloud-u kad neko gurne kod.

### 9.2. Tri pipeline-a (shop-operator, shop, shophub)

Svaki repo ima svoj `.github/workflows/ci.yml`. Struktura je ista:

1. **Trigger** — okida se na PR (validira) i push na main (publish-uje).
2. **Lint job** — staticcheck (Go), hadolint (Dockerfile), helm lint (chart-ovi).
3. **Test job** — unit + integration testovi (Testcontainers podiže Postgres).
4. **Build & push job** — gradi Docker image, push-uje na DockerHub sa SemVer tag-om. Pokreće se samo na main.

### 9.3. CI workflow za shop-operator (Go projekat)

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
    tags: ['v*']

env:
  GO_VERSION: '1.25.6'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Install staticcheck
        run: go install honnef.co/go/tools/cmd/staticcheck@latest

      - name: Staticcheck
        run: staticcheck ./...

      - name: Go vet
        run: go vet ./...

      - name: Hadolint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Unit tests
        run: go test ./... -short -race -coverprofile=cover.out

      - name: Integration tests
        run: go test ./... -run Integration -timeout 10m

      - name: Coverage report
        run: go tool cover -func=cover.out

  build-image:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Determine version
        id: ver
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            VERSION=${GITHUB_REF#refs/tags/v}
          else
            VERSION=$(git describe --tags --always --dirty)
          fi
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "Building version: ${VERSION}"

      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/shop-operator:${{ steps.ver.outputs.version }}
            ${{ secrets.DOCKERHUB_USERNAME }}/shop-operator:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Šta se dešava:**

1. **lint** job — proverava kvalitet koda (staticcheck) i Dockerfile-a (hadolint). Ako ne prolazi, gase ostali.
2. **test** job — pokreće unit i integration testove. Testcontainers automatski podiže Postgres kontejner u runner-u za integration testove.
3. **build-image** job — okida se SAMO na push na main ili tag (ne na PR). Build-uje Docker image, push-uje sa SemVer verzijom.

### 9.4. Branch protection — povezivanje sa CI

Posle prvog CI run-a, idi u repo → Settings → Branches → main rule:
- **Require status checks to pass before merging** → ✅
- Search for status checks → odaberi `lint`, `test`, `build-image`
- Sad bez prolaska svih CI job-ova, PR ne može merge.

> **Tip:** prva tri PR-a će ti pasti dok ne uštimaš pipeline. To je normalno. Lokalno testiraj sa `act` (https://github.com/nektos/act) da ne moraš push-ovati svaki put.

### 9.5. Pipeline za shop (Go backend + React frontend monorepo)

Pošto `shop` repo ima `backend/` i `frontend/` foldere, pipeline ih obrađuje paralelno.

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  backend-lint-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with: { go-version: '1.25.6', cache: true, cache-dependency-path: backend/go.sum }
      - run: go vet ./...
      - run: go test ./... -race

  frontend-lint-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: frontend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm', cache-dependency-path: frontend/package-lock.json }
      - run: npm ci
      - run: npm run lint
      - run: npm run test -- --watchAll=false
      - run: npm run build

  build-backend-image:
    runs-on: ubuntu-latest
    needs: backend-lint-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Determine version
        id: ver
        run: echo "version=$(git describe --tags --always)" >> $GITHUB_OUTPUT
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/shop-backend:${{ steps.ver.outputs.version }}
            ${{ secrets.DOCKERHUB_USERNAME }}/shop-backend:latest

  build-frontend-image:
    runs-on: ubuntu-latest
    needs: frontend-lint-test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    # ... isto kao backend, samo context: ./frontend i tag shop-frontend
```

### 9.6. Pipeline za helm-charts repo

Drugačiji — ovaj repo ne pravi Docker image-ove nego Helm chart-ove.

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
        with: { version: 'v3.14.0' }

      - name: Helm lint
        run: |
          for chart in charts/*/; do
            echo "Linting $chart"
            helm lint "$chart"
          done

      - name: Helm template
        run: |
          for chart in charts/*/; do
            helm template test "$chart"
          done

      - name: kubeval validation
        uses: stefanprodan/kube-tools@v1
        with:
          command: |
            for chart in charts/*/; do
              helm template test "$chart" | kubeval --strict --kubernetes-version "1.30.0"
            done

  publish:
    runs-on: ubuntu-latest
    needs: lint
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
        with: { version: 'v3.14.0' }

      - name: Login to DockerHub OCI
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | helm registry login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin docker.io

      - name: Package and push charts
        run: |
          for chart in charts/*/; do
            name=$(basename "$chart")
            helm package "$chart" -d /tmp
            version=$(grep '^version:' "$chart/Chart.yaml" | awk '{print $2}')
            helm push /tmp/"$name"-"$version".tgz oci://docker.io/${{ secrets.DOCKERHUB_USERNAME }}
          done
```

### 9.7. SemVer — kako da verzionišeš

[Semantic Versioning](https://semver.org/) — verzija je `MAJOR.MINOR.PATCH`:
- **MAJOR** (1.0.0 → 2.0.0): breaking change (nije kompatibilno unazad).
- **MINOR** (1.0.0 → 1.1.0): nova feature, kompatibilno.
- **PATCH** (1.0.0 → 1.0.1): bugfix, kompatibilno.

**Workflow za projekat:**

```powershell
# Lokalno, kad si spreman za release
git tag v0.1.0
git push --tags
```

CI pipeline koji detektuje tag pravi image sa tom verzijom (`shop-operator:0.1.0`).

**Alternativa — auto-versioning preko commit poruke:** alat `semantic-release` čita Conventional Commit poruke i automatski izračunava verziju (`feat:` → minor bump, `fix:` → patch bump, `BREAKING CHANGE` u footer → major bump). Bonus, jednostavno za team.

### 9.8. Integracioni testovi sa Testcontainers (Go)

Specifikacija 5.2 traži da CI pokreće integration testove. Najlakše rešenje: **Testcontainers** — biblioteka koja podiže Docker kontejnere iz Go testova.

`backend/internal/storage/postgres_integration_test.go`:
```go
//go:build integration

package storage_test

import (
    "context"
    "testing"
    "time"

    "github.com/testcontainers/testcontainers-go/modules/postgres"
    tcwait "github.com/testcontainers/testcontainers-go/wait"
    "github.com/stretchr/testify/require"
)

func TestPostgresIntegration(t *testing.T) {
    ctx := context.Background()

    pg, err := postgres.Run(ctx, "postgres:16-alpine",
        postgres.WithDatabase("test"),
        postgres.WithUsername("test"),
        postgres.WithPassword("test"),
        postgres.WithWaitStrategy(tcwait.ForLog("ready to accept connections").WithStartupTimeout(30*time.Second)),
    )
    require.NoError(t, err)
    defer pg.Terminate(ctx)

    connStr, err := pg.ConnectionString(ctx, "sslmode=disable")
    require.NoError(t, err)

    // Tvoj test sa stvarnom bazom
    // db, err := sql.Open("postgres", connStr)
    // ...
}
```

Pokreni:
```powershell
go test ./... -tags=integration -v -timeout 5m
```

Testcontainers se sam stara o tome da kontejner bude obrisan na kraju testa.

### 9.9. Checkpoint Faze 8

- [ ] Svaki repo ima `.github/workflows/ci.yml`.
- [ ] CI prolazi na trenutnom main-u (zeleni check pored commit-a).
- [ ] PR ne može da se merge-uje ako CI ne prolazi (branch protection postavljen).
- [ ] Push na main automatski build-uje image i push-uje na DockerHub.
- [ ] Image tagovi su SemVer (`0.1.0`, ne `latest-2024-04-29`).
- [ ] `helm-charts` repo na push-u automatski package + push chart-ove na OCI.
- [ ] commitlint workflow funkcioniše — PR sa lošom commit porukom se odbija.

---

## 10. Najčešće greške i kako ih izbeći

Stvari koje će ti pojesti vreme ako ne znaš unapred:

### 10.1. Docker / image gotchas

- **`ImagePullBackOff` u k3d** — image nije učitan u k3d klaster. Reši: `k3d image import shop-backend:dev -c local`.
- **`exec format error`** — pravi image-e na M1/Mac, pa pokreni u runner-u → arhitektura nije ista. Reši: `--platform linux/amd64` na build-u.
- **`USER app` ali fajlovi nisu njegovi** — `chown -R app:app /app` posle `COPY` u Dockerfile-u.
- **Slika je 1GB+** — verovatno nemaš multi-stage. Vidi sekciju 2.4.

### 10.2. Kubernetes / operator gotchas

- **`409 Conflict` u Reconcile** — pokušavaš da update-uješ objekat čiji `resourceVersion` je zastareo. Reši: ignoriši grešku, K8s će te ponovo okinuti. Vidi sekciju 5 [kubernetes_operatori_elegancija.md](vezbe%20pomocni%20materijali/kubernetes_operatori_elegancija.md).
- **`Watches()` bez predikata** — operator gori CPU jer se okida na svaku promenu Secret-a u klasteru. **Uvek koristi predikat** (sekcija 8 operator eleganta).
- **`OWNER TO` zaboravljen u CNPG `postInitApplicationSQL`** — root korisnik kreira tabele, app korisnik ne može da im pristupi. Vidi sekciju 10 operator eleganta.
- **Finalizer ostaje zauvek** — ako greška u cleanup-u, finalizer se ne uklanja, objekat ostaje u `Terminating` zauvek. Reši: u finalizer cleanup-u, ako resurs već ne postoji eksterno, smatraj OK.

### 10.3. Helm gotchas

- **CRD-ovi se ne update-uju na `helm upgrade`** — to je by design (vidi sekciju 4.5). Ručno `kubectl apply -f crds/*.yaml` ili obrisati i reinstalirati chart.
- **`<no value>` u template-u** — referenca u `values.yaml` nije postavljena. Pokreni `helm template --debug` da vidiš.
- **`Error: INSTALLATION FAILED: rendered manifests contain a resource that already exists`** — neko je već instalirao taj resurs ručno. `helm install --replace` ili obrisati ručno.
- **Subchart values override-ovani slabo** — sintaksa je `{ <subchart-name>: { ... } }` u values.yaml, sa **tačno istim imenom subchart-a**.

### 10.4. CI/CD gotchas

- **Secrets nisu vidljivi** — proveri da li si ih dodao u Settings → Secrets and variables → **Actions** (ne Codespaces ili Dependabot).
- **Pipeline pada sa `permission denied`** — DockerHub token nema "Write" pravo. Pravi novi token sa "Read, Write, Delete".
- **Test job traje 15+ min** — cachiraj `~/go/pkg/mod` i `~/.cache/go-build` preko `actions/cache@v4` ili `actions/setup-go@v5` sa `cache: true`.
- **`go test` ne pokušava integration test-ove** — koristi `//go:build integration` tag, i u CI: `go test -tags=integration`.

### 10.5. Web3 gotchas (Faza 5)

- **`insufficient funds for gas`** — testnet wallet nema dovoljno ETH. Sepolia faucet daje 0.5 ETH dnevno, treba 0.001 ETH za transakciju.
- **`Transfer` event ne stiže** — RPC node je out of sync. Koristi Alchemy ili Infura kao stabilan RPC.
- **`decimals()` je 6 (USDT) ali ti množiš sa 18** — USDT ima 6 decimala (`parseUnits(amount, 6)`), ETH ima 18. Uvek zovi `decimals()` jednom i keširaj.

---

## 11. Checklist pred odbranu

Tabela ide reverzno — od stvari koje ti gube projekat ako fale:

### Eliminacioni (bez ovoga nema pregleda)

- [ ] `helm-charts` repo ima tačnu strukturu iz specifikacije 5.3.
- [ ] `kube-state` repo ima tačnu strukturu iz specifikacije 5.3.
- [ ] Svi chart-ovi (shop, shophub, shop-operator) postoje u `helm-charts/charts/`.

### Bazni (za prolazak, ≥15/50)

- [ ] Pet repozitorijuma na GitHub-u sa Conventional Commits istorijom.
- [ ] Branch protection sa linear history, PR approval (zahtev 5.1).
- [ ] CI pipeline na svakom repo-u prolazi (zahtev 5.2).
- [ ] Shop CRD radi — `kubectl apply` pravi Pod-ove (zahtev 3.1).
- [ ] Shop aplikacija ima CRUD artikala i porudžbina (zahtev 2.1, 2.2).
- [ ] ShopHub može da kreira/briše/menja Shop CR-ove (zahtev 1.2).
- [ ] Image-i postoje na DockerHub-u sa SemVer tag-om.

### Standardni (za solidan rezultat, 30+)

- [ ] DiscordChannel CRD sa finalizer-om (zahtev 3.1).
- [ ] Wallet CRD (zahtev 3.1).
- [ ] Helm chart za Shop operator (zahtev 3.2).
- [ ] ShopHub Helm chart sa kube-prometheus-stack dependency (zahtev 3.3).
- [ ] Web3 plaćanje radi na Sepolia (zahtev 2.4).
- [ ] Grafana dashboard po Shop-u sa svim metrikama iz zahteva 4.1.
- [ ] Alarmi na Discord (zahtev 4.1).
- [ ] Klaster dashboard sa CPU/RAM/FS/network (zahtev 4.1).

### Bonus (za 40+)

- [ ] Web3 sign-in (SIWE) za ShopHub korisnike (opciono zahtev 1.1).
- [ ] Per-tenant Grafana pristup (opciono zahtev 4.1).
- [ ] ArgoCD / Flux GitOps u kube-state repo-u (opciono zahtev 5.3).
- [ ] MongoDB umesto/uz Redis (specifikacija dozvoljava).
- [ ] Test coverage ≥80% u backend-u.

### Demo priprema

- [ ] `demo.md` skripta od 10–15 koraka (vidi [to_do.md sekcija 9.1](plan%20projekta/to_do.md)).
- [ ] Probna prezentacija odrađena bar 1x bez greške.
- [ ] Tim zna ko priča o čemu na odbrani.
- [ ] Klaster može da se podigne sa nule prateći samo `kube-state/README.md`.
- [ ] Discord bot token, MetaMask wallet i sve credentials su spremni i radni.

---

## 12. Učenje Go-a u 1 vikend

Pošto znaš JS/TS, evo brze paralele:

| TypeScript | Go |
|-------------|-----|
| `let x: string = "hello"` | `var x string = "hello"` ili `x := "hello"` |
| `const x = 5` | `const x = 5` |
| `function add(a: number, b: number): number { return a + b }` | `func add(a int, b int) int { return a + b }` |
| `interface User { name: string }` | `type User struct { Name string }` |
| `async function load() { return await fetch(url) }` | `func load() (*http.Response, error) { return http.Get(url) }` |
| Generičke promise: `Promise<T>` | Vraćaš `(T, error)` |
| `try/catch` | `if err != nil { return err }` |
| `arr.map(x => x * 2)` | Eksplicitan `for` loop |
| `JSON.parse(s)` | `json.Unmarshal([]byte(s), &target)` |

### Najbitnije razlike

1. **Greške nisu exception-i** — funkcija vraća `(value, error)`. Uvek proveri:
   ```go
   user, err := getUser(id)
   if err != nil {
       return nil, fmt.Errorf("getUser: %w", err)
   }
   ```

2. **Pointers** — `*User` je pointer na `User`. `&user` daje pointer na `user`. `user.Name` automatski dereferiše ako je `user` pointer.

3. **Eksportovan vs neeksportovan** — ime sa velikim slovom je javno (`func MyFunc()`), malim je privatno (`func myFunc()`). Nema `public/private` keyword-a.

4. **Goroutine = lightweight thread** — `go myFunc()` pokrene funkciju u pozadini. Sinhronizacija preko `chan`-ova ili `sync.WaitGroup`.

5. **JSON tagovi na struct-u**:
   ```go
   type User struct {
       Name string `json:"name"`           // u JSON-u "name"
       Age  int    `json:"age,omitempty"`  // izostavi ako je 0
   }
   ```

### Mali Go primer — HTTP server (kako će izgledati Shop backend)

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"

    "github.com/gin-gonic/gin"
)

type Item struct {
    ID       string  `json:"id"`
    Name     string  `json:"name"`
    Price    float64 `json:"price"`
    Stock    int     `json:"stock"`
}

func main() {
    r := gin.Default()

    r.GET("/api/items", func(c *gin.Context) {
        // pretvori se da imaš bazu
        items := []Item{
            {ID: "1", Name: "T-shirt", Price: 19.99, Stock: 50},
        }
        c.JSON(http.StatusOK, items)
    })

    r.POST("/api/items", func(c *gin.Context) {
        var item Item
        if err := c.ShouldBindJSON(&item); err != nil {
            c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
            return
        }
        // sačuvaj u bazu...
        c.JSON(http.StatusCreated, item)
    })

    r.GET("/probe/liveness", func(c *gin.Context) {
        c.String(http.StatusOK, "ok")
    })

    log.Fatal(r.Run(":8080"))
}
```

### Resursi

- **Go by Example** — https://gobyexample.com/ — kratki primeri po temi.
- **Tour of Go** — https://go.dev/tour/ — interaktivni tutorijal.
- **Effective Go** — https://go.dev/doc/effective_go — kako se Go pravilno piše.

Realno, vikend praksa + Gin tutorial je dovoljno da pišeš Shop backend. Za operator (kubebuilder) treba dodatnih 3-4 dana.

---

## 13. Resursi za dalje učenje

### Kubernetes
- **Kubernetes The Hard Way** — https://github.com/kelseyhightower/kubernetes-the-hard-way — ako želiš da razumeš ispod motora (NIJE potrebno za projekat).
- **Kubernetes documentation** — https://kubernetes.io/docs/ — autoritativno.
- **k3d docs** — https://k3d.io — za lokalni klaster.

### Kubebuilder + Operator
- **Kubebuilder Book** — https://book.kubebuilder.io/ — ovo *moraš* pročitati.
- **operator-sdk samples** — https://github.com/operator-framework/operator-sdk/tree/master/testdata — primeri.
- **CNPG dokumentacija** — https://cloudnative-pg.io/documentation/ — kako se Postgres pravi preko CR-a.

### Helm
- **Helm Best Practices** — https://helm.sh/docs/chart_best_practices/ — kratko, korisno.
- **Helm chart examples** — https://artifacthub.io — pregled hiljada chart-ova, lepi su za učenje.

### Web3
- **ethers.js docs** — https://docs.ethers.org/v6/ — za frontend.
- **go-ethereum book** — https://goethereumbook.org/ — za backend verifikaciju.
- **MetaMask docs** — https://docs.metamask.io/wallet/ — wallet integracija.

### Observability
- **Prometheus query basics** — https://prometheus.io/docs/prometheus/latest/querying/basics/ — PromQL.
- **Grafana dashboard JSON model** — https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/modify-dashboard-settings/ — kako se piše dashboard.

### Conventional Commits
- **Specification** — https://www.conventionalcommits.org/en/v1.0.0/ — pravila.
- **commitlint** — https://commitlint.js.org/ — alat za enforce.

---

## Završna napomena

**Strategija za 10+ nedelja:**

| Nedelja | Fokus | Cilj |
|----------|-------|------|
| 1 | Faza 0 + Faza 1 | 5 repo-a, lokalni klaster, prvi Dockerfile |
| 2 | Učenje Go-a (svi članovi), kubebuilder skeleton | Shop CRD definisan |
| 3-4 | Faza 2 — Shop kontroler (član A) + Shop backend (član B) | `kubectl apply -f shop.yaml` pravi Pod-ove |
| 5 | Faza 3 — Shop frontend + Faza 4 — ShopHub backend | UI radi lokalno |
| 6 | Faza 4 — ShopHub frontend, DiscordChannel kontroler | Korisnik može da kreira Shop iz UI |
| 7 | Faza 5 — Web3 plaćanje | Demo wallet plaća na Sepoliji |
| 8 | Faza 6 — Observability stack | Grafana prikazuje metrike |
| 9 | Faza 7 — Helm chart-ovi finalizacija + Faza 8 — CI/CD | Sve push-uje na DockerHub |
| 10 | Bonus + Faza 9 — demo skripta | ArgoCD, per-tenant Grafana, polish |

**Pravilo:** ako u nedelji 6 nemaš ShopHub UI funkcionalan, **preskočiš Web3 sign-in** (samo plaćanje). Ako u nedelji 9 nemaš observability, **preskočiš Loki/Tempo** (samo Prometheus + Grafana metrike).

Eliminacioni je **uvek prioritet**. Bonusi nikad ne idu pre baznog.

Srećno! 🎯

> **Ako ti nešto u ovom tutorijalu nije jasno, pitaj.** Bolje 5 pitanja pre nego što počneš, nego 50 pitanja u 9. nedelji.
