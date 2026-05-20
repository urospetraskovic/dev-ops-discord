# Docker, Kubebuilder i CI/CD — praktični vodič za Windows početnika

> **Šta je ovo:** Detaljna, "klik-na-klik" uputstva za stvari oko kontejnera, Kubebuilder-a i povezivanja CI/CD pipeline-ova kroz svih 5 repozitorijuma.
>
> **Šta NIJE:** Konceptualni vodič — to je [tutorial-pocetnik.md](tutorial-pocetnik.md). Pre nego što kreneš ovde, **pročitaj sekcije 0–5 prvog tutorijala** da znaš šta je image, šta je pod, šta je Helm.
>
> **Pretpostavka:** Windows 10/11, PowerShell, nešto znanja iz frontend-a (Node/npm). Ne pretpostavlja se znanje Docker-a, Kubernetes-a ili Go-a.
>
> **Stil:** Svaki korak ima jasan input/output. Posle svakog koraka **provera** — kako da znaš da je OK.

---

## Sadržaj

- [1. Pravljenje svih naloga (90 minuta — uradi odjednom)](#1-pravljenje-svih-naloga-90-minuta--uradi-odjednom)
- [2. Windows-specifični setup (WSL2, Docker Desktop, alati)](#2-windows-specifični-setup-wsl2-docker-desktop-alati)
- [3. Docker — od nule do prvog kontejnera](#3-docker--od-nule-do-prvog-kontejnera)
- [4. Docker — pravljenje slike Shop backend-a (Go)](#4-docker--pravljenje-slike-shop-backend-a-go)
- [5. Push image-a na DockerHub](#5-push-image-a-na-dockerhub)
- [6. Kubebuilder na Windows-u — instalacija (WSL2 ruta)](#6-kubebuilder-na-windows-u--instalacija-wsl2-ruta)
- [7. Prvi Kubebuilder projekat — Hello Operator](#7-prvi-kubebuilder-projekat--hello-operator)
- [8. Povezivanje 5 repozitorijuma — mentalni model](#8-povezivanje-5-repozitorijuma--mentalni-model)
- [9. CI/CD setup za svih 5 repo-a (sekvencijalno)](#9-cicd-setup-za-svih-5-repo-a-sekvencijalno)
- [10. Cross-repo workflow — kako sve klikne](#10-cross-repo-workflow--kako-sve-klikne)
- [11. Tipični problemi na Windows-u i rešenja](#11-tipični-problemi-na-windows-u-i-rešenja)
- [12. Praktične vežbe — uradi pa ti je u glavi](#12-praktične-vežbe--uradi-pa-ti-je-u-glavi)

---

## 1. Pravljenje svih naloga (90 minuta — uradi odjednom)

**Strategija:** otvoriš sve naloge u jednom dahu, sve credentials ide u password manager (1Password, Bitwarden, ili `.txt` na sigurnom). Posle se ne vraćaš na to.

### 1.1. GitHub — lični nalog

Ako imaš već, preskoči. Ako nemaš:

1. Idi na https://github.com/signup
2. Email → password (snažan!) → username (ne mora biti tvoje ime, ovo je javno).
3. Verify email.
4. **Settings → SSH and GPG keys → New SSH key** — vrlo korisno za rad sa `git` bez kucanja password-a:
   ```powershell
   # U PowerShell-u
   ssh-keygen -t ed25519 -C "tvoj.email@example.com"
   # Pritisni Enter na sva pitanja (default lokacija, prazan passphrase OK za početak)
   Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
   ```
   Zatim na GitHub-u nalepi sadržaj clipboard-a u "Key" polje.
5. Test:
   ```powershell
   ssh -T git@github.com
   # Treba da pišu: "Hi <username>! You've successfully authenticated..."
   ```

### 1.2. GitHub — organizacija za tim

**Bitno:** projekat traži 5 repo-a koji su pod istom "krovnom" organizacijom. Lakše je za pristup, RBAC, billing (mada je sve besplatno za studente).

1. https://github.com/account/organizations/new → odaberi **Free** plan.
2. Organization name: `shophub-<vase-prezime>` ili `shophub-team-X` — ime mora biti jedinstveno na celom GitHub-u.
3. Email: jednog od članova tima.
4. This organization belongs to: **My personal account** (ili "business").
5. Posle pravljenja, idi u **People → Invite member** → dodaj sva tri člana.
6. Svi članovi prihvataju pozivnicu sa email-a → klik "Join".

**Provera:** Pod https://github.com/orgs/<vaš-org>/people vidiš sva 3 člana sa rolom "Member".

### 1.3. DockerHub nalog

1. Idi na https://hub.docker.com/signup
2. Username (npr. `shophubteam`), email, password.
3. Verify email.
4. **Account Settings → Security → New Access Token**:
   - Description: `github-actions-ci`
   - Access permissions: **Read, Write, Delete**
   - Klikni **Generate**.
   - **KOPIRAJ TOKEN SADA** — neće se prikazati ponovo.
   - Sačuvaj kao `DOCKERHUB_TOKEN` u password manager-u.
5. Sačuvaj i tvoj username — biće `DOCKERHUB_USERNAME` u GitHub secrets.

**Provera:**
```powershell
docker login -u <tvoj-username>
# Password: <nalepi token>
# Treba: "Login Succeeded"
```

### 1.4. Discord razvojni server + bot (za Fazu 2)

Trebaće ti za **DiscordChannel CRD** operator-a. Najlakše uradi sad.

**Server:**
1. U Discord aplikaciji → levo dole "+" → "Create My Own" → "For me and my friends" → ime "ShopHub Dev".
2. Idi u **Server Settings → Widget → Server ID** — kopiraj broj (npr. `1234567890123456789`). To je **GuildID** koji ide u DiscordChannel spec.

**Bot:**
1. https://discord.com/developers/applications → **New Application** → naziv "ShopHub Bot".
2. Levo meni → **Bot** → **Reset Token** → **KOPIRAJ TOKEN** — sačuvaj kao `DISCORD_BOT_TOKEN`.
3. Pod **Privileged Gateway Intents** uključi **MESSAGE CONTENT INTENT** (ako planiraš da bot čita poruke; za webhooks nije obavezno).
4. **OAuth2 → URL Generator**:
   - Scopes: ✅ `bot`
   - Bot Permissions: ✅ `Manage Channels`, ✅ `Manage Webhooks`, ✅ `Send Messages`
   - Kopiraj generisani URL → otvori u browseru → "Add to Server" → izaberi "ShopHub Dev" server.
5. **Provera:** u Discord-u vidiš bot kao člana servera (offline boja je OK, online se vidi tek kad operator počne da koristi token).

### 1.5. MetaMask wallet (za Fazu 5)

Da odmah uradiš, jer Sepolia ETH faucet ima 24h cooldown.

1. Instaliraj MetaMask ekstenziju u browser-u: https://metamask.io/download/
2. **Create a new wallet** → ne koristi pravi nalog, ovo je za testnet.
3. **Sačuvaj seed phrase** (12 reči) na sigurnom — ne na disku, na papir ili u password manager.
4. **Add Network → Sepolia**:
   - Network name: `Sepolia`
   - RPC URL: `https://rpc.sepolia.org` (ili Alchemy/Infura)
   - Chain ID: `11155111`
   - Currency symbol: `SepoliaETH`
   - Block explorer: `https://sepolia.etherscan.io`
5. Prebaci na Sepolia mrežu (gore desno dropdown).
6. Iskopiraj svoju adresu (klik na ime naloga → kopira se).
7. Idi na https://sepoliafaucet.com ili https://www.alchemy.com/faucets/ethereum-sepolia → nalepi adresu → "Send Me ETH".
8. Sačekaj 1-5 minuta, otvori MetaMask → treba da vidiš ~0.5 SepoliaETH.

### 1.6. Sačuvaj credentials negde

Otvori `.txt` fajl u password manager-u sa:
```
GitHub org: https://github.com/<vaš-org>
DockerHub username: <username>
DockerHub token (CI): <dugačak string>
Discord Guild ID: <broj>
Discord Bot Token: <dugačak string>
MetaMask Sepolia address: 0x...
MetaMask seed phrase: (12 reči, ne deli ovo nikad)
```

**Pravilo:** ništa od ovoga ne ide u Git. NIKAD. Ako slučajno commit-uješ token — odmah ga revoke-uj na DockerHub-u/Discord-u i napravi novi.

---

## 2. Windows-specifični setup (WSL2, Docker Desktop, alati)

**Bitno:** Kubebuilder, k3d, mnogi Linux DevOps alati rade **mnogo bolje u WSL2** (Windows Subsystem for Linux) nego u native PowerShell-u. Preporuka: glavni rad u WSL2 + VS Code Remote.

### 2.1. Instalacija WSL2

PowerShell (kao administrator):
```powershell
wsl --install
# Restartuj računar
wsl --set-default-version 2
wsl --install -d Ubuntu-22.04
```

Posle restart-a, otvori Start menu → **Ubuntu 22.04** → prvi put traži username + password (može isti kao Windows nalog, ali drugačiji password je sigurnije).

**Provera:**
```powershell
wsl -l -v
#   NAME            STATE           VERSION
# * Ubuntu-22.04    Running         2
```

### 2.2. Docker Desktop sa WSL2 backend-om

1. Skini sa https://www.docker.com/products/docker-desktop/ → instaliraj.
2. Tokom instalacije: ✅ "Use WSL 2 instead of Hyper-V".
3. Posle: **Settings → Resources → WSL Integration** → uključi **Ubuntu-22.04**.
4. Apply & Restart.

**Provera (iz WSL):**
```bash
# Otvori Ubuntu terminal (ne PowerShell!)
docker --version
docker run hello-world
# Treba: "Hello from Docker!"
```

### 2.3. Alati u WSL2 Ubuntu

```bash
# Update prvo
sudo apt update && sudo apt upgrade -y

# Osnovni alati
sudo apt install -y curl wget git build-essential unzip

# Go (verzija 1.25.6 ili novija)
wget https://go.dev/dl/go1.25.6.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.25.6.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bashrc
source ~/.bashrc
go version

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client

# k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
k3d version

# Helm
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
helm version

# Kubebuilder
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder && sudo mv kubebuilder /usr/local/bin/
kubebuilder version
```

**Provera svih alata:**
```bash
docker --version       # 25.x+
go version             # go1.25.6
kubectl version --client
k3d version            # k3d 5.8+
helm version           # v3.14+
kubebuilder version    # 4.11+
git --version
```

### 2.4. VS Code Remote — radiš u WSL-u, editor je na Windows-u

1. U VS Code instaliraj ekstenziju **WSL** (od Microsoft-a).
2. U WSL Ubuntu terminalu:
   ```bash
   cd ~
   code .
   ```
   Otvoriće se VS Code koji "živi" u WSL — sve komande u terminalu su Linux, fajlovi su iste, ali editor je tvoj Windows VS Code.
3. Lako prebacivanje: gore levo "WSL: Ubuntu-22.04" ti pokazuje da si u remote modu.

**Bitno:** ne radi `git clone` u `C:\Users\...` ako planiraš da koristiš WSL alate na tim fajlovima — performanse su katastrofalne (cross-filesystem). **Sve drži u WSL home-u**: `~/code/shop-operator`, itd.

### 2.5. PowerShell vs WSL — kad koji

| Šta radiš | Gde |
|------------|-----|
| `git`, `docker`, `kubectl`, `helm` | **WSL** (brže, kompatibilnije) |
| `kubebuilder`, Go projekat | **WSL** (mnogo manje problema) |
| `npm`, frontend dev server | može oba, ali **WSL** za konzistentnost |
| Windows-only alati (Visual Studio, Office) | PowerShell / GUI |
| Brza provera (`docker ps`, `kubectl get pods`) | može oba — Docker Desktop sinhronizuje |

**Pravilo:** sva komandna magija u ovom tutorijalu pretpostavlja **WSL Ubuntu terminal**, osim ako nije izričito napisano "PowerShell".

---

## 3. Docker — od nule do prvog kontejnera

Cilj sekcije: razumeš šta su image, layer, container, registry — kroz ruku.

### 3.1. "Hello World" u kontejneru

```bash
docker run hello-world
```

Šta se desilo:
1. Docker je proverio da li ima image `hello-world` lokalno → nije.
2. Skinuo ga je sa Docker Hub-a (`docker.io/library/hello-world:latest`).
3. Pokrenuo kontejner iz tog image-a.
4. Kontejner je ispisao poruku → izašao.

```bash
docker ps                 # vidi koje kontejnere imaš pokrenute (verovatno nijedan)
docker ps -a              # vidi sve, uključujući i one koji su završili
docker images             # vidi sve image-e lokalno
```

### 3.2. Pokretanje nginx-a sa portom

```bash
docker run -d -p 8080:80 --name moj-nginx nginx:1.27-alpine
```

Razlaganje komande:
- `-d` — detached (u pozadini)
- `-p 8080:80` — port 8080 na hostu mapiraj na port 80 u kontejneru
- `--name moj-nginx` — ime kontejnera (lakše nego `<random-id>`)
- `nginx:1.27-alpine` — image (alpine = slim verzija)

**Provera:**
```bash
docker ps
# Vidiš moj-nginx, port 0.0.0.0:8080->80/tcp

# Otvori u browseru: http://localhost:8080
# Treba: "Welcome to nginx!"

# Logovi
docker logs moj-nginx

# Uđi u kontejner
docker exec -it moj-nginx sh
# unutar: ls /usr/share/nginx/html/
# exit kad si gotov

# Zaustavi i obriši
docker stop moj-nginx
docker rm moj-nginx
```

### 3.3. Layeri — zašto Docker image-i nisu jedan fajl

Image se sastoji od **layera**. Svaki `RUN`, `COPY`, `ADD` u Dockerfile-u kreira novi layer. Layeri se keširaju.

```bash
docker history nginx:1.27-alpine
# Vidiš listu layera, veličinu i koja komanda ih je napravila
```

Primer:
```dockerfile
FROM alpine:3.20             # layer 1: alpine baza (~7 MB)
RUN apk add curl             # layer 2: dodaje curl (~2 MB)
COPY hello.txt /             # layer 3: kopira fajl (1 B)
```

Ako menjaš `hello.txt`, samo layer 3 se ponovo gradi. Layer 1 i 2 se uzimaju iz keša → build je brz.

**Pravilo:** **stavljaj stvari koje se retko menjaju na vrh, a često menjaju na dno**. Zato je u Go Dockerfile-u tipično:

```dockerfile
COPY go.mod go.sum ./        # retko se menja
RUN go mod download           # keširaj download
COPY . .                      # često se menja — ovo na dno
RUN go build ...
```

### 3.4. Volumes — kako da kontejner ima persistentne podatke

Kontejner je "efemeran" — kad ga obrišeš, sve unutar nestaje. Za podatke koji preživljavaju → **volume**.

```bash
# Postgres bez volume-a (gubi podatke na restart)
docker run -d --name pg1 -e POSTGRES_PASSWORD=test postgres:16

# Postgres sa volume-om (preživljava)
docker volume create pgdata
docker run -d --name pg2 \
  -e POSTGRES_PASSWORD=test \
  -v pgdata:/var/lib/postgresql/data \
  postgres:16

# Vidi sve volume-ove
docker volume ls
```

Bind mount (folder sa hosta) — koristi se za dev:
```bash
docker run -d -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx:1.27-alpine
# Tvoj lokalni html folder je sada serviran preko nginx-a
```

### 3.5. Docker Compose — više kontejnera odjednom

Compose je za "lokalni stack" — više servisa koji komuniciraju. **U projektu ga koristimo za integration testove** (sekcija 5.2 specifikacije).

`docker-compose.yaml`:
```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: shopdb
      POSTGRES_USER: shop
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"

  app:
    build: .                  # gradi iz lokalnog Dockerfile-a
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: "postgres://shop:secret@db:5432/shopdb?sslmode=disable"
    depends_on:
      - db
```

Komande:
```bash
docker compose up -d         # pokrene sve, u pozadini
docker compose logs -f app   # logovi jednog servisa
docker compose ps            # status
docker compose down          # zaustavi i obriši
docker compose down -v       # + obriši volume-ove
```

> **Bitno za projekat:** `docker compose` je dovoljan za integration testove ako ti je Testcontainers previše složen.

---

## 4. Docker — pravljenje slike Shop backend-a (Go)

Konkretan primer. Sve komande su iz WSL Ubuntu terminala.

### 4.1. Mini Go projekat

```bash
mkdir -p ~/code/shop-backend-demo
cd ~/code/shop-backend-demo
go mod init github.com/test/shop-backend-demo
go get github.com/gin-gonic/gin
```

Napravi `main.go`:
```bash
cat > main.go << 'EOF'
package main

import (
    "net/http"
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    r.GET("/probe/liveness", func(c *gin.Context) {
        c.String(http.StatusOK, "ok")
    })
    r.GET("/probe/readiness", func(c *gin.Context) {
        c.String(http.StatusOK, "ok")
    })
    r.GET("/", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"app": "shop", "version": "0.1.0"})
    })
    r.Run(":8080")
}
EOF
```

Test lokalno:
```bash
go run main.go
# u drugom terminalu:
curl http://localhost:8080
# {"app":"shop","version":"0.1.0"}
# Ctrl+C u prvom terminalu da prekineš
```

### 4.2. Dockerfile (multi-stage, non-root, slim)

```bash
cat > Dockerfile << 'EOF'
# syntax=docker/dockerfile:1.6
ARG GO_VERSION=1.25.6

# Stage 1: builder
FROM golang:${GO_VERSION}-alpine AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download
COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 GOOS=linux go build -ldflags='-s -w' -o /out/shop ./...

# Stage 2: release (mala slika)
FROM alpine:3.20 AS release
RUN apk add --no-cache ca-certificates && \
    addgroup -S app && adduser -S app -G app
COPY --from=builder /out/shop /app/shop
RUN chown -R root:root /app && chmod 0755 /app/shop
USER app
WORKDIR /app
EXPOSE 8080
ENTRYPOINT ["/app/shop"]
EOF
```

Razlaganje:
- `# syntax=docker/dockerfile:1.6` — koristi noviju verziju Dockerfile sintakse (omogućava `--mount=type=cache`).
- `ARG GO_VERSION` — može da se override-uje: `docker build --build-arg GO_VERSION=1.26.0 ...`
- `--mount=type=cache,target=/go/pkg/mod` — Go modul keš preživljava između build-ova → brzo.
- `-ldflags='-s -w'` — strip debug info → manji binarni fajl.
- `CGO_ENABLED=0` — static binary, ne treba C library u finalnom image-u.
- `USER app` — ne root!

### 4.3. `.dockerignore`

Inače Docker kopira **sve** iz tekućeg foldera u build context — uključujući `.git/`, `node_modules/`, `*.log`. Spori build, veliki context.

```bash
cat > .dockerignore << 'EOF'
.git
.gitignore
*.md
.idea
.vscode
*.log
bin
dist
EOF
```

### 4.4. Build i pokretanje

```bash
docker build -t shop-backend:0.1.0 .
# Prvi put traje 2-3 minuta (skida Go i Alpine).
# Drugi put (sa keš) je 5-10 sekundi.

docker images | grep shop-backend
# shop-backend   0.1.0   abc123   2 minutes ago   25MB

docker run -d -p 8080:8080 --name shop-test shop-backend:0.1.0

curl http://localhost:8080
# {"app":"shop","version":"0.1.0"}

curl http://localhost:8080/probe/liveness
# ok

docker logs shop-test
docker stop shop-test
docker rm shop-test
```

### 4.5. Hadolint provera

```bash
docker run --rm -i hadolint/hadolint < Dockerfile
# Ako ne kaže ništa → sve OK.
# Ako kaže "DL3008 Pin versions in apk add" → dodaj verziju u apk add ca-certificates=...
```

### 4.6. Inspekcija slike

```bash
docker history shop-backend:0.1.0
# Vidiš slojeve, veličine

docker image inspect shop-backend:0.1.0 | head -100
# JSON detalji — image config, env, entrypoint, layeri

# Ako koristiš `dive` alat (sjajan za inspekciju):
# go install github.com/wagoodman/dive@latest
# dive shop-backend:0.1.0
```

---

## 5. Push image-a na DockerHub

Sad da ovo isto image guraš na DockerHub — postaje dostupan svima.

### 5.1. Login

```bash
docker login -u <tvoj-dockerhub-username>
# Password: <DOCKERHUB_TOKEN koji si napravio u 1.3>
# Login Succeeded
```

### 5.2. Tag-uj image sa svojim username-om

DockerHub konvencija: `<username>/<repo>:<tag>`

```bash
docker tag shop-backend:0.1.0 <username>/shop-backend:0.1.0
docker tag shop-backend:0.1.0 <username>/shop-backend:latest

docker images | grep shop-backend
# vidiš sad 3 reda — lokalni i 2 tag-ovana
```

### 5.3. Push

```bash
docker push <username>/shop-backend:0.1.0
docker push <username>/shop-backend:latest
# Prvi put traje minut-dva
```

### 5.4. Provera

1. Otvori https://hub.docker.com/repositories/<username> u browser-u.
2. Vidiš `shop-backend` repozitorijum.
3. Klikni → vidiš tag-ove `0.1.0` i `latest`.

```bash
# Test — povuci sa drugog mesta
docker rmi <username>/shop-backend:0.1.0
docker pull <username>/shop-backend:0.1.0
docker run -d -p 8081:8080 --name s2 <username>/shop-backend:0.1.0
curl http://localhost:8081
docker stop s2 && docker rm s2
```

### 5.5. Tipovi tag-ova — best practice

- `0.1.0` — **immutable** (nikad ne menjaj sadržaj jednom kad je pushovan).
- `0.1` — moving tag, uvek pokazuje na najnoviji `0.1.x`.
- `0` — uvek najnoviji `0.x.x`.
- `latest` — najnoviji uopšte (najopasnije, ne koristi u K8s manifestima).

U CI pipeline-u ovo automatski radimo (sekcija 9).

---

## 6. Kubebuilder na Windows-u — instalacija (WSL2 ruta)

**Najveća greška: pokušaj da pokreneš Kubebuilder na native Windows-u.** Ne radi pouzdano. WSL2 je jedini razuman put.

### 6.1. Provera da je sve spremno u WSL

```bash
# Iz Ubuntu terminala
go version              # >= 1.25.6
kubebuilder version     # >= 4.11.0
docker version          # client + server radi
kubectl version --client
```

Ako bilo šta fali → vrati se na sekciju 2.3.

### 6.2. Šta Kubebuilder generiše

Kubebuilder je generator skeleta. **Ne piše ti operator** — pravi strukturu projekta + boilerplate. Ti pišeš `Reconcile()` logiku.

Šta dobijaš:
- `api/v1/<kind>_types.go` — definicija CRD-a (Spec + Status).
- `internal/controller/<kind>_controller.go` — Reconcile metoda.
- `config/crd/bases/<group>_<plural>.yaml` — generisan CRD YAML.
- `config/rbac/` — RBAC manifest-i.
- `config/manager/` — Deployment za operator Pod.
- `Makefile` — komande (`make manifests`, `make install`, `make run`).

### 6.3. Hello Operator — pravimo `Counter` CRD

Da uvežbaš Kubebuilder pre nego što kreneš sa Shop operator-om.

```bash
mkdir -p ~/code/counter-operator
cd ~/code/counter-operator
kubebuilder init --domain example.com --project-name counter-operator --repo github.com/test/counter-operator
```

Posle ovoga vidiš:
```bash
ls
# Dockerfile  Makefile  PROJECT  README.md  bin  cmd  config  go.mod  go.sum  hack  internal  test
```

Dodaj API:
```bash
kubebuilder create api --group apps --version v1 --kind Counter --resource --controller --plural counters
# Create Resource [y/n]? y
# Create Controller [y/n]? y
```

Sad vidiš:
```bash
ls api/v1/
# counter_types.go  groupversion_info.go  zz_generated.deepcopy.go

ls internal/controller/
# counter_controller.go  counter_controller_test.go  suite_test.go
```

### 6.4. Definiši Spec i Status

Otvori `api/v1/counter_types.go` u VS Code. Pronađi `CounterSpec` i `CounterStatus` struct-ove, zameni ih sa:

```go
type CounterSpec struct {
    // +kubebuilder:default:=0
    InitialValue int32 `json:"initialValue"`

    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:default:=1
    IncrementBy int32 `json:"incrementBy"`
}

type CounterStatus struct {
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // +optional
    CurrentValue int32 `json:"currentValue,omitempty"`
    // +optional
    LastIncrement metav1.Time `json:"lastIncrement,omitempty"`
}
```

Imports na vrhu fajla treba da uključuje:
```go
import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
```

### 6.5. Generiši CRD i deepcopy

```bash
make manifests generate
# manifests: regeneriše YAML CRD-ove
# generate: regeneriše zz_generated.deepcopy.go
```

Provera:
```bash
cat config/crd/bases/apps.example.com_counters.yaml | head -30
# Treba da vidiš OpenAPI spec sa initialValue i incrementBy
```

### 6.6. Reconcile logika — naš mini operator

Otvori `internal/controller/counter_controller.go`. Pronađi `Reconcile` metodu, zameni sadržaj:

```go
func (r *CounterReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := ctrl.LoggerFrom(ctx)

    counter := &appsv1.Counter{}
    if err := r.Get(ctx, req.NamespacedName, counter); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Inicijalizuj
    if counter.Status.CurrentValue == 0 && counter.Spec.InitialValue != 0 {
        counter.Status.CurrentValue = counter.Spec.InitialValue
        counter.Status.LastIncrement = metav1.Now()
        if err := r.Status().Update(ctx, counter); err != nil {
            return ctrl.Result{}, err
        }
        log.Info("Initialized counter", "value", counter.Status.CurrentValue)
        return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
    }

    // Inkrement
    counter.Status.CurrentValue += counter.Spec.IncrementBy
    counter.Status.LastIncrement = metav1.Now()
    if err := r.Status().Update(ctx, counter); err != nil {
        return ctrl.Result{}, err
    }
    log.Info("Incremented", "value", counter.Status.CurrentValue)

    return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
}
```

Dodaj `time` u imports:
```go
import (
    "context"
    "time"
    // ...
)
```

### 6.7. Lokalni klaster i pokretanje

```bash
# Pravimo k3d cluster ako nije
k3d cluster create local --servers 1 --agents 1

# Instaliraj CRD na klaster
make install

# Pokreni operator lokalno (van klastera)
make run
# Vidiš log "Starting Controller" — Reconcile još nije ništa video
```

Drugi terminal:
```bash
cat > /tmp/counter.yaml << 'EOF'
apiVersion: apps.example.com/v1
kind: Counter
metadata:
  name: my-counter
spec:
  initialValue: 10
  incrementBy: 5
EOF

kubectl apply -f /tmp/counter.yaml
```

U prvom terminalu vidiš:
```
INFO  Initialized counter  value=10
INFO  Incremented  value=15
INFO  Incremented  value=20
```

Provera u drugom terminalu:
```bash
kubectl get counter my-counter -o yaml
# Vidiš status.currentValue raste svakih 10 sekundi

kubectl describe counter my-counter
# Vidiš Events i Status
```

Cleanup:
```bash
kubectl delete -f /tmp/counter.yaml
# Ctrl+C u prvom terminalu da zaustaviš operator
make uninstall
```

**Sad razumeš celu mašinu:** spec → reconcile → status. Sve ostalo (3 CRD-a, ownership, finalizer, RBAC) je elaboracija ovog.

### 6.8. Pravljenje image-a operator-a

```bash
make docker-build IMG=<username>/counter-operator:0.1.0
# Pravilo: Kubebuilder ima ugrađen `make docker-build`
docker images | grep counter-operator

# Push
docker login
make docker-push IMG=<username>/counter-operator:0.1.0
```

Deploy na klaster (van `make run`):
```bash
make deploy IMG=<username>/counter-operator:0.1.0
kubectl get pods -n counter-operator-system
# Vidiš operator-controller-manager-... Pod-a Running
```

Cleanup:
```bash
make undeploy
```

---

## 7. Prvi Kubebuilder projekat — Hello Operator

Ako si uradio sekciju 6, već imaš ovo. Sekcija 7 je samo da naglasim **šta si naučio i kako to mapira na Shop operator**.

### 7.1. Mapping mini-projekta na pravi

| Counter operator | Shop operator |
|------------------|---------------|
| `CounterSpec` ima `initialValue`, `incrementBy` | `ShopSpec` ima `title`, `availability`, `database`, `walletAddress` |
| Inicijalizacija u Reconcile-u | Prvi reconcile pravi CNPG `Cluster`, čeka Secret |
| Inkrement svakih 10s | Reconcile se okida na svaku promenu (Spec, child resursa) |
| Status: `CurrentValue` | Status: `Conditions`, `ReadyReplicas`, `URL` |
| 1 CRD | 3 CRD-a (Shop, DiscordChannel, Wallet) — svaki ima svoj kontroler |

### 7.2. Šta dodaješ na Shop operatoru što Counter nema

1. **Owner references** — kad pravi child resurse, postavi Shop CR kao "owner". Brisanje Shop-a automatski briše child-ove (GC).
2. **Watches sa predikatima** — okini Reconcile kad se CNPG Secret pojavi.
3. **Finalizer** — DiscordChannel CRD; pre brisanja, briše Discord kanal preko Discord API-ja.
4. **RBAC markup-i** — `// +kubebuilder:rbac:...` iznad Reconcile metode.
5. **Idempotentnost** — pre `Create`, proveri `Get` da ne dupliraš.

Sve ovo je u [kubernetes_operatori_elegancija.md](vezbe%20pomocni%20materijali/kubernetes_operatori_elegancija.md), pažljivo pratiti.

---

## 8. Povezivanje 5 repozitorijuma — mentalni model

Ovo je gde se najlakše izgubiš. **5 repo-a, 3 image-a, 3 chart-a, jedan klaster.** Hajde da vidimo kako pričaju.

### 8.1. Šta gde sedi

```
+--------------------+      build       +-------------------------+
| shop-operator repo |----------------->| DockerHub image:        |
| (Go kod)           |   push:          | <user>/shop-operator    |
+--------------------+   on push to main+-------------------------+
         |                                            ^
         | helm chart kopira CRD-ove                  |
         | (make manifests + sync)                    | helm install
         v                                            |
+--------------------+      package     +-------------------------+
| helm-charts repo   |----------------->| DockerHub OCI chart:    |
| (charts/shop-      |   push to main:  | <user>/shop-operator    |
|  operator/...)     |   helm push      | (Helm chart)            |
+--------------------+                  +-------------------------+

+--------------------+      build       +-------------------------+
| shop repo          |----------------->| <user>/shop-backend     |
| (Go + React)       |                  | <user>/shop-frontend    |
+--------------------+                  +-------------------------+

+--------------------+      build       +-------------------------+
| shophub repo       |----------------->| <user>/shophub-backend  |
| (Go + React)       |                  | <user>/shophub-frontend |
+--------------------+                  +-------------------------+

+--------------------+   sva 3 chart-a (shop, shophub, shop-operator)
| helm-charts repo   |--------v
+--------------------+        v
                         OCI registry (DockerHub)
                              ^
                              |
+--------------------+   referencira chart-ove
| kube-state repo    |   sa OCI URL-ovima
| (clusters/local/   |
|  helm.yaml fajlovi)|
+--------------------+
         |
         | ArgoCD ili manuelno helm install
         v
+----------------------------+
| Kubernetes klaster (k3d)   |
| - cnpg-system              |
| - redis-operator           |
| - shop-operator-system     |
| - shophub-system           |
| - tenant-alice, tenant-bob |
+----------------------------+
```

### 8.2. Dataflow primer — kreiranje shop-a

Šta se desi kad korisnik klikne "Create Shop" u ShopHub UI:

```
1. Browser → POST /api/shops na ShopHub backend
2. ShopHub backend → preko client-go pravi Shop CR u namespace tenant-alice
3. Kubernetes API Server → snima u etcd
4. Shop operator (gleda Shop CR-ove) → primi event, pokrene Reconcile
5. Reconcile pravi:
   - CNPG Cluster CR        → CNPG operator pravi Postgres Pod
   - Wallet CR              → Shop operator (drugi kontroler) pravi wallet Secret
   - DiscordChannel CR      → Shop operator (treći kontroler) zove Discord API
   - Deployment (shop backend, shop frontend)
   - Service, Ingress
   - ServiceMonitor
6. Kad sve bude Running, Reconcile update-uje Shop.Status.URL i Conditions
7. ShopHub UI polluje GET /api/shops/<name>/status → vidi URL, prikazuje korisniku
```

### 8.3. Verzije — kako ih sinhronizujemo

**Problem:** ako Shop operator zna o `Shop` v1 sa poljem `walletAddress`, ali ti pushuješ chart sa CRD-om v2 koji ima `walletAddresses` (plural), klaster pada.

**Rešenje za projekat:** sve verzije idu u koraku. SemVer:
- shop-operator image `0.1.0` ↔ shop-operator chart `0.1.0` ↔ CRD verzija `v1`.
- Bumpaj sve troje istovremeno kad menjaš API.

**Workflow za prelaz 0.1.0 → 0.2.0:**

1. U `shop-operator` repo-u: dodaj polje u `shop_types.go`.
2. `make manifests generate`.
3. Bumpaj verziju u `Makefile` (`VERSION=0.2.0`) — utiče na image tag.
4. Tag: `git tag v0.2.0 && git push --tags`. CI gradi image.
5. U `helm-charts` repo-u: kopiraj nove CRD-ove u `charts/shop-operator/crds/`, bumpaj `Chart.yaml` verziju na `0.2.0`, bumpaj `values.yaml` `image.tag: 0.2.0`.
6. PR → merge → CI publishuje chart `0.2.0` na OCI.
7. U `kube-state` repo-u: bumpaj `clusters/local/shop-operator/helm.yaml` `version: 0.2.0`.
8. PR → merge → (ako ArgoCD) automatski deploy, inače `helm upgrade` ručno.

> **U praksi za projekat** verovatno nećeš imati više verzija u toku razvoja. Jedna `0.1.0` traje do odbrane.

---

## 9. CI/CD setup za svih 5 repo-a (sekvencijalno)

**Strategija:** ne radi sve odjednom. Uradi **jedan repo do kraja**, pa drugi, pa treći. Učenje kroz repeticiju.

### 9.1. Redosled

1. **shop-operator** — najsloženiji CI (Go + Docker + integration testovi). Uradi prvo da naučiš pattern.
2. **shop** — sličan, ali monorepo (backend + frontend). Reuse 80% workflow-a.
3. **shophub** — kopija shop-a.
4. **helm-charts** — drugačiji (Helm lint + OCI push).
5. **kube-state** — najlakši (samo YAML validacija).

### 9.2. Šta dodaješ u svaki repo (checklist)

Za svaki od 5 repo-a uradi:

- [ ] `.github/workflows/ci.yml` (CI pipeline).
- [ ] `.github/workflows/commit-lint.yml` (validacija commit poruka).
- [ ] **Secrets** u repo settings (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`).
- [ ] **Branch protection** (PR, approval, linear history, status checks).

### 9.3. Repo 1: shop-operator — detaljan workflow

Posle prvog commit-a u shop-operator, dodaj `.github/workflows/ci.yml`:

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
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: go vet
        run: go vet ./...

      - name: staticcheck
        run: |
          go install honnef.co/go/tools/cmd/staticcheck@latest
          staticcheck ./...

      - name: Hadolint
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile

  test:
    name: Unit + Integration tests
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true

      - name: Install envtest binaries
        run: |
          go install sigs.k8s.io/controller-runtime/tools/setup-envtest@latest
          setup-envtest use 1.30.x --bin-dir /tmp/envtest

      - name: Unit tests
        env:
          KUBEBUILDER_ASSETS: /tmp/envtest/k8s/1.30.x-linux-amd64
        run: go test ./... -race -coverprofile=cover.out -timeout 10m

      - name: Coverage summary
        run: go tool cover -func=cover.out

  build:
    name: Build & Push image
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Compute version
        id: ver
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            VERSION="${GITHUB_REF#refs/tags/v}"
          else
            # Untagged commit na main-u
            VERSION="0.0.0-$(git rev-parse --short HEAD)"
          fi
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "Version: ${VERSION}"

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

**Postavi secrets:**

1. Repo → **Settings → Secrets and variables → Actions → New repository secret**:
   - Name: `DOCKERHUB_USERNAME`, Value: tvoj DockerHub username.
   - Name: `DOCKERHUB_TOKEN`, Value: token koji si napravio.

**Commit i push:**
```bash
cd ~/code/shop-operator
mkdir -p .github/workflows
# nalepi sadržaj ci.yml fajla
git add .github/workflows/ci.yml
git commit -m "ci: add build, test, and image push pipeline"
git push
```

**Prvi run:**
- Idi u repo → **Actions** tab.
- Vidiš "CI" workflow run u toku.
- Sačekaj 3-5 minuta.
- Lint job treba da prođe; test job možda padne (ako nema testova) → to je OK za sad.

**Bitno za branch protection:**
- Posle prvog uspešnog run-a, idi u **Settings → Branches → main**.
- "Require status checks to pass before merging" → odaberi `lint`, `test`, `build` (pojaviće se posle prvog run-a).

### 9.4. Repo 2 i 3: shop i shophub — monorepo workflow

Razlika od shop-operator: imaš `backend/` i `frontend/` foldere. Pipeline gradi 2 image-a (`shop-backend` i `shop-frontend`).

`shop/.github/workflows/ci.yml`:
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
  NODE_VERSION: '20'

jobs:
  backend-lint:
    name: Backend lint
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: backend } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
          cache-dependency-path: backend/go.sum
      - run: go vet ./...
      - uses: hadolint/hadolint-action@v3.1.0
        with: { dockerfile: backend/Dockerfile }

  backend-test:
    name: Backend tests
    runs-on: ubuntu-latest
    needs: backend-lint
    defaults: { run: { working-directory: backend } }
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: true
          cache-dependency-path: backend/go.sum
      - run: go test ./... -race -timeout 5m
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/test?sslmode=disable

  frontend-lint-build:
    name: Frontend lint & build
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: frontend } }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --watchAll=false
      - run: npm run build
      - uses: hadolint/hadolint-action@v3.1.0
        with: { dockerfile: frontend/Dockerfile }

  build-backend-image:
    runs-on: ubuntu-latest
    needs: backend-test
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - id: ver
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
          else
            echo "version=0.0.0-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
          fi
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
          cache-from: type=gha
          cache-to: type=gha,mode=max

  build-frontend-image:
    runs-on: ubuntu-latest
    needs: frontend-lint-build
    if: github.event_name == 'push' && (github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/v'))
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - id: ver
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
          else
            echo "version=0.0.0-$(git rev-parse --short HEAD)" >> $GITHUB_OUTPUT
          fi
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: ./frontend
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/shop-frontend:${{ steps.ver.outputs.version }}
            ${{ secrets.DOCKERHUB_USERNAME }}/shop-frontend:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

**Šta je drugačije od shop-operator-a:**

- `services: postgres` — GitHub Actions automatski pokreće Postgres container kao bočni servis (lakše od Testcontainers).
- 4 job-a paralelno: backend-lint → backend-test → build-backend-image, frontend-lint-build → build-frontend-image.
- 2 Docker image-a: `shop-backend` i `shop-frontend`.

Identičan workflow ide u `shophub` repo, samo zameni "shop" → "shophub" i tagove.

### 9.5. Repo 4: helm-charts — Helm specifičan workflow

`helm-charts/.github/workflows/ci.yml`:
```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
    tags: ['v*']

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-helm@v4
        with: { version: 'v3.14.0' }

      - name: Helm lint
        run: |
          set -e
          for chart in charts/*/; do
            echo "=== Linting $chart ==="
            helm dependency update "$chart" || true
            helm lint "$chart"
          done

      - name: Helm template
        run: |
          set -e
          for chart in charts/*/; do
            echo "=== Rendering $chart ==="
            helm template release-test "$chart" > /dev/null
          done

  publish:
    runs-on: ubuntu-latest
    needs: lint
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - uses: azure/setup-helm@v4
        with: { version: 'v3.14.0' }

      - name: OCI login
        run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | helm registry login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin docker.io

      - name: Package and push charts
        run: |
          set -e
          for chart in charts/*/; do
            name=$(basename "$chart")
            version=$(grep '^version:' "$chart/Chart.yaml" | awk '{print $2}')
            echo "=== Packaging $name version $version ==="
            helm dependency update "$chart" || true
            helm package "$chart" -d /tmp
            helm push "/tmp/${name}-${version}.tgz" "oci://docker.io/${{ secrets.DOCKERHUB_USERNAME }}"
          done
```

> **Bitno:** ovaj pipeline pretpostavlja da bumpaš `version:` u `Chart.yaml` kad menjaš chart. Ako pokušaš da push-uješ istu verziju 2x, OCI registry će vratiti grešku. Zato u praksi: na svaku PR koji menja chart, bump version.

### 9.6. Repo 5: kube-state — najlakši

`kube-state/.github/workflows/ci.yml`:
```yaml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Validate YAML
        run: |
          # Instaliraj yq
          sudo wget -qO /usr/local/bin/yq https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64
          sudo chmod +x /usr/local/bin/yq

          # Proveri da su svi helm.yaml fajlovi valjani
          set -e
          for f in $(find clusters -name 'helm.yaml'); do
            echo "=== Validating $f ==="
            yq eval '.' "$f" > /dev/null
            # Mora imati polja: chart, version, namespace
            yq eval '.chart' "$f" > /dev/null
            yq eval '.version' "$f" > /dev/null
            yq eval '.namespace' "$f" > /dev/null
          done

      - name: Validate cluster.yaml
        run: |
          yq eval '.' clusters/local/cluster.yaml > /dev/null
```

### 9.7. Postavljanje secrets za sva 5 repo-a (jedan put)

Možeš ovo ručno kroz UI (Settings → Secrets), ali ako tim ima Org plan, lakše je **Organization secret**:

1. Idi u GitHub org → **Settings → Secrets and variables → Actions**.
2. **New organization secret**:
   - Name: `DOCKERHUB_USERNAME`, Value: ..., Repository access: **Selected repositories** → odaberi svih 5.
   - Name: `DOCKERHUB_TOKEN`, Value: ..., isti repo access.

Tako sa **2 secret-a** opslužiš sva 5 repo-a. Bonus: rotacija — ako menjaš token, menjaš na jednom mestu.

### 9.8. Branch protection — primeni na svih 5 odjednom

Branch protection se mora konfigurisati **po repo-u** (org se ne podržava). Najlakše: kopiraj postavke iz prvog repo-a.

Za svaki repo:
1. Settings → Branches → Add branch protection rule.
2. Branch name pattern: `main`
3. Check-uj:
   - ✅ Require a pull request before merging
   - ✅ Require approvals: **1**
   - ✅ Dismiss stale pull request approvals when new commits are pushed
   - ✅ Require status checks to pass before merging
     - U searchbox-u dodaj: `lint`, `test`, `build` (gde postoje)
   - ✅ Require branches to be up to date before merging
   - ✅ Require linear history *(zahtev 5.1)*
   - ✅ Include administrators
4. Save.

> **Tip:** GitHub CLI (`gh`) može da automatizuje ovo:
> ```bash
> gh api repos/<org>/<repo>/branches/main/protection --method PUT --input protection.json
> ```
> Ali za 5 repo-a, ručno je brže.

---

## 10. Cross-repo workflow — kako sve klikne

**Pitanje koje će ti se postaviti:** "Promenim Shop CRD u shop-operator repo-u, kako se to propagira do klastera?"

### 10.1. Tok promene CRD-a

```
[1] Programer A menja api/v1/shop_types.go u shop-operator repo-u
        |
        | git commit + push na feature granu
        v
[2] Otvara PR ka main
        |
        | CI pokreće lint + test
        v
[3] Drugi član tima odobri PR (Faza 5.1 zahtev)
        |
        | merge (linear history — squash ili rebase)
        v
[4] CI na main-u: build image, push na DockerHub kao 0.0.0-abc1234
        |
        | (opciono) git tag v0.2.0 → CI gradi 0.2.0 tag
        v
[5] Programer A pravi PR u helm-charts repo-u:
    - kopira nove CRD-ove iz shop-operator/config/crd/bases/ u
      helm-charts/charts/shop-operator/crds/
    - bumpaje version u Chart.yaml na 0.2.0
    - bumpaje image.tag u values.yaml na 0.2.0
        |
        | drugi član odobri, merge
        v
[6] CI helm-charts: helm push chart na OCI registry kao 0.2.0
        |
        v
[7] Programer A pravi PR u kube-state repo-u:
    - bumpa clusters/local/shop-operator/helm.yaml version: 0.2.0
        |
        | drugi član odobri, merge
        v
[8] Ako ima ArgoCD: automatski sync. Bez ArgoCD: ručno:
        helm upgrade shop-operator oci://docker.io/<user>/shop-operator \
          --version 0.2.0 -n shop-operator-system \
          -f kube-state/clusters/local/shop-operator/values.yaml
        |
        v
[9] Klaster sada ima novi CRD. Postojeći Shop CR-ovi koriste defaults
    za nova polja (ako su pravilno postavljeni u CRD-u sa +kubebuilder:default).
```

### 10.2. Automatizacija — `helm-sync` Makefile

U `shop-operator` repo-u, dodaj u `Makefile`:

```makefile
HELM_CHARTS_REPO ?= ../helm-charts

.PHONY: helm-sync
helm-sync: manifests
	@echo "Syncing CRDs to helm-charts repo at $(HELM_CHARTS_REPO)"
	@cp config/crd/bases/*.yaml $(HELM_CHARTS_REPO)/charts/shop-operator/crds/
	@echo "Done. Remember to bump version in Chart.yaml and create PR."
```

Korišćenje:
```bash
cd ~/code/shop-operator
make helm-sync HELM_CHARTS_REPO=~/code/helm-charts
cd ~/code/helm-charts
# pregledaj diff, bumpaj verziju, commit
```

> **Bolja varijanta (bonus):** GitHub Action u shop-operator repo-u koji **automatski** otvara PR u helm-charts repo-u kad se push-uje tag. Traži cross-repo PAT (personal access token). Za projekat — ručno je dovoljno.

### 10.3. Demo pripreme — "bootstrap from scratch"

Pred odbranu, **test scenario**: obrišeš celi klaster, prateći samo `kube-state/README.md`, treba da podigneš sve.

```bash
# Reset
k3d cluster delete local

# Idi u kube-state repo, prati README:
cd ~/code/kube-state

# 1. Klaster
k3d cluster create --config clusters/local/cluster.yaml

# 2. CNPG
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace \
  -f clusters/local/cnpg/values.yaml

# 3. Redis operator
helm repo add redis-operator https://spotahome.github.io/redis-operator
helm install redis-operator redis-operator/redis-operator -n redis-operator --create-namespace

# 4. kube-prometheus-stack (sub-chart kroz shophub, ili samostalno za jasnoću)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace \
  -f clusters/local/kube-prometheus-stack/values.yaml

# 5. Shop operator (iz OCI)
helm install shop-operator oci://docker.io/<user>/shop-operator --version 0.1.0 \
  -n shop-operator-system --create-namespace \
  -f clusters/local/shop-operator/values.yaml

# 6. ShopHub
helm install shophub oci://docker.io/<user>/shophub --version 0.1.0 \
  -n shophub-system --create-namespace \
  -f clusters/local/shophub/values.yaml
```

Sve treba da bude Running:
```bash
kubectl get pods -A
# cnpg-system            cnpg-controller-manager-...      Running
# redis-operator         redis-operator-...               Running
# monitoring             kps-grafana-...                  Running
# shop-operator-system   shop-operator-controller-...     Running
# shophub-system         shophub-...                      Running
```

---

## 11. Tipični problemi na Windows-u i rešenja

### 11.1. Docker Desktop ne startuje

Simptom: kliknem ikonu, ne otvara se.

Rešenja po redu:
1. Restart računara.
2. Otvori Services → "Docker Desktop Service" → Start.
3. Reset Docker: Settings → Troubleshoot → Clean / Purge data.
4. Reinstaliraj sa "Add user to docker-users group" opcijom.

### 11.2. WSL2 i Docker se ne sinhronizuju

Simptom: `docker ps` u WSL kaže "Cannot connect to the Docker daemon".

Rešenje:
1. Docker Desktop → Settings → Resources → WSL Integration → uključi tvoj Ubuntu.
2. Restart WSL: u PowerShell-u kao admin → `wsl --shutdown` → ponovo otvori Ubuntu terminal.

### 11.3. `kubebuilder init` pada sa "permission denied"

Simptom: pokrećeš `kubebuilder init` u `/mnt/c/Users/...` folderu i hoće permission greške.

Rešenje: **Radi u WSL home folderu** (`~/code/...`), ne u Windows folderu mountovanom u `/mnt/c/`. Cross-filesystem permissions su problematične.

### 11.4. `make run` u operator-u baca "no kubeconfig found"

Simptom: operator pokušava da se konektuje na klaster, ne nalazi config.

Rešenje:
```bash
# Proveri da kubectl radi
kubectl config current-context
# k3d-local

# Ako "current-context not set":
k3d kubeconfig merge local --kubeconfig-switch-context
# Sad treba:
kubectl get nodes
```

### 11.5. Build je spor (5+ minuta)

Simptom: `docker build` traje predugo svaki put.

Rešenja:
1. Provera da je `--mount=type=cache` u Dockerfile-u.
2. `.dockerignore` postoji i isključuje `.git`, `node_modules`, itd.
3. Layeri po pravilima — retko promenljive na vrhu.
4. BuildKit uključen (default u novim Docker verzijama).

### 11.6. CI prolazi lokalno ali pada na GitHub-u

Simptom: `go test ./...` radi lokalno, ali u Actions-u pada.

Najčešći razlozi:
1. **Različita verzija Go-a** — pin u `actions/setup-go` na `1.25.6` kao lokalno.
2. **Test zavisi od /tmp/cache koji ne postoji u runner-u** — koristi `t.TempDir()`.
3. **Race condition** — testovi prolaze sekvencijalno, padaju paralelno. `go test -race` lokalno.
4. **Env varijable** — Actions runner nema tvoje `.env`. Postavi kroz `env:` u workflow-u.

### 11.7. `helm install` kaže "chart not found" za OCI

Simptom: `helm install foo oci://docker.io/<user>/shop-operator --version 0.1.0` ne radi.

Provera:
1. `docker login` (Helm OCI koristi isti registry kao Docker).
2. Verzija postoji: `helm pull oci://docker.io/<user>/shop-operator --version 0.1.0` — ako ovo radi, install treba da radi.
3. Da li je chart bio public push-ovan? DockerHub OCI je default public, ali proveri u DockerHub UI da li se vidi.

### 11.8. `git push` traži password svaki put

Rešenje: koristiš HTTPS, prebaci na SSH:
```bash
cd ~/code/shop-operator
git remote set-url origin git@github.com:<org>/shop-operator.git
git push
# Traži passphrase samo prvi put ako si postavio SSH agent
```

Da agent zapamti:
```bash
echo 'eval $(ssh-agent -s) > /dev/null' >> ~/.bashrc
echo 'ssh-add ~/.ssh/id_ed25519 2> /dev/null' >> ~/.bashrc
```

---

## 12. Praktične vežbe — uradi pa ti je u glavi

Ovo nisu deo projekta — to su **vežbice** koje preporučujem da uradiš pre nego što kreneš sa pravim radom. Svaka 30-60 minuta.

### Vežba 1: Hello Docker

**Cilj:** Pokreni nginx u kontejneru, izmeni HTML, vidi promenu.

```bash
mkdir ~/vezba1 && cd ~/vezba1
echo "<h1>Moja prva web stranica</h1>" > index.html

docker run -d --name web \
  -v $(pwd):/usr/share/nginx/html:ro \
  -p 8080:80 nginx:1.27-alpine

curl http://localhost:8080
# <h1>Moja prva web stranica</h1>

echo "<h1>Izmenjeno!</h1>" > index.html
curl http://localhost:8080
# <h1>Izmenjeno!</h1>

docker stop web && docker rm web
```

### Vežba 2: Build vlastiti image

**Cilj:** Napravi multi-stage Go image koji vraća "hello".

Prati sekciju 4 ovog tutorijala. Cilj je da:
- Build prolazi.
- `docker images` pokazuje da je <30MB.
- `hadolint` ne prijavljuje greške.
- Image radi: `curl http://localhost:8080` vraća JSON.

### Vežba 3: Lokalni klaster sa 3 nginx-a

**Cilj:** k3d klaster, deploy nginx Deployment-a sa 3 replike, Service, Ingress.

```bash
k3d cluster create vezba3 --port "8888:80@loadbalancer"

cat > nginx-stack.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          ports: [ { containerPort: 80 } ]
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports: [ { port: 80, targetPort: 80 } ]
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  rules:
    - http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: web
                port: { number: 80 }
EOF

kubectl apply -f nginx-stack.yaml
kubectl get pods    # treba 3 nginx pod-a
kubectl get svc
kubectl get ingress

curl http://localhost:8888    # Welcome to nginx!

# Scale-uj na 5
kubectl scale deployment web --replicas=5
kubectl get pods

# Cleanup
k3d cluster delete vezba3
```

### Vežba 4: Hello Operator

Sekcija 6.3–6.7 ovog tutorijala. Counter operator. **Ako nisi uradio — uradi sada.** To je 70% učenja Kubebuilder-a.

### Vežba 5: Push prvi image na DockerHub

**Cilj:** Image iz Vežbe 2 push-uj na DockerHub.

```bash
docker tag shop-backend:0.1.0 <username>/test-shop:0.1.0
docker push <username>/test-shop:0.1.0

# Otvori https://hub.docker.com/r/<username>/test-shop
# Treba da vidiš svoj image
```

### Vežba 6: Helm chart sa parametrima

**Cilj:** Napravi Helm chart koji deploy-uje nginx sa parametrizovanim brojem replika.

```bash
helm create vezba-chart
# Pravi default strukturu

# Edituj vezba-chart/values.yaml:
# replicaCount: 1 → 3
# image.repository: nginx → nginx
# image.tag: "" → "1.27-alpine"
# service.type: ClusterIP

helm lint vezba-chart
helm template test vezba-chart            # vidi rendered YAML

# Instaliraj na klaster
helm install moj-web vezba-chart
kubectl get pods                          # vidi 3 nginx pod-a
kubectl get svc                           # vidi service moj-web-vezba-chart

# Promeni broj replika preko --set
helm upgrade moj-web vezba-chart --set replicaCount=5
kubectl get pods                          # sad 5

# Cleanup
helm uninstall moj-web
```

### Vežba 7: CI prolazi na test repo-u

**Cilj:** Napravi privatan test repo na GitHub-u, commit-uj jednostavan Go projekat sa CI workflow-om iz sekcije 9.3. Vidi da pipeline radi end-to-end.

Posle ove vežbe, kad uradiš isto za pravi shop-operator repo, znaš tačno šta očekivati i gde je problem ako nešto pada.

---

## Završna napomena

**Realan tempo:**
- Sekcije 1–2 (nalozi + Windows setup): **dan 1** — uradi sa svojim timom.
- Sekcije 3–5 (Docker): **dan 2-3** — pojedinačno, hands-on.
- Sekcije 6–7 (Kubebuilder vežba): **dan 4-5** — uradi Counter operator pre nego što diraš Shop.
- Sekcija 8 (mentalni model): **sat vremena** kad pravite arhitekturu.
- Sekcija 9 (CI/CD): **nedelja 2-3 projekta** — jedan repo dnevno.
- Sekcije 10–12 (workflow, problemi, vežbe): kao referenca dok radite.

**Najveći savet:** **uradi vežbe.** Posebno Counter operator (sekcija 6). Polusat čitanja Kubebuilder dokumentacije zameni tri sata praktičnog rada.

I — kao u prvom tutorijalu — **pitaj odmah** kad nešto nije jasno. Bolje 5 pitanja danas, nego 50 u nedelji 9.

Srećno! 🚀
