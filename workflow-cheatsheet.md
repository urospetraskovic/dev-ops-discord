# Workflow cheatsheet — Git, WSL, k3d za ShopHub projekat

> **Cilj ovog fajla:** sve "gotcha" stvari koje su nam već pojele vreme tokom setup-a, sažeto na jednom mestu. Ako negde u sledećoj sesiji nešto puca a ne sećamo se zašto — prvo ovde pogledaj.

---

## 1. Branch + commit + push + PR — pravi redosled

**Pravilo:** prvo se prebaci na novu feature granu, **pa onda** menjaj/commit-uj. Nikad ne commit-uj prvo na `main`.

```bash
# 1. Krećeš sa main-a koji je up-to-date sa origin/main
git checkout main
git pull

# 2. NOVA grana PRE bilo kakve izmene
git checkout -b feat/<kratak-opis>     # feat/, fix/, chore/, ci/, docs/

# 3. Menjaš fajlove, sredjuješ što ti treba
# ...

# 4. Verifikuj šta će u commit
git add -A
git status

# 5. Commit sa Conventional Commits porukom
git commit -m "$(cat <<'EOF'
feat(scope): kratak naslov ispod 70 karaktera

- bullet 1
- bullet 2
EOF
)"

# 6. Push (prvi put sa -u da postavi tracking)
git push -u origin feat/<kratak-opis>

# 7. Otvori PR (URL koji ti git ispiše posle push-a)
# 8. Čekaj commitlint workflow → Squash and merge na GitHub-u
```

**Zašto ovaj redosled:**
- Ako commit-uješ na `main` lokalno pre nego što napraviš feature granu, lokalni `main` ide ispred `origin/main`. Posle PR merge-a (squash), grane se "razilaze" i `git pull` pokušava merge commit umesto fast-forward → ulaziš u vim/merge editor → panic.

### Posle merge-a — sredi lokalni main

```bash
git checkout main
git fetch
git reset --hard origin/main          # bezbedno: tvoj commit je sad squashed na origin
git branch -D feat/<kratak-opis>      # obriši lokalnu granu
```

> `-D` (velika D) je potreban posle squash merge-a jer git ne vidi tvoje originalne commit-ove kao "predak" trenutnog main-a. Sadržaj je tu, samo SHA je drugi.

### Ako si već zabrljao (commit na main pre grane)

Da ne čekaš da PR merge stvori haos, sredi ga unapred:

```bash
git checkout -b feat/<kratak-opis>    # tvoj commit ostaje na ovoj grani
git checkout main
git reset --hard origin/main          # vrati main na remote stanje
git checkout feat/<kratak-opis>       # nastavi rad sa grane
```

---

## 2. WSL git — jednokratan setup

Posle čiste WSL instalacije:

```bash
# Identitet (mora biti isti kao u Windows git-u da svi tvoji commit-ovi imaju jednog autora)
git config --global user.name "Uroš Petrašković"
git config --global user.email "132670531+urospetraskovic@users.noreply.github.com"

# Line endings — bez ovog vidimo lažne "modified" na svim fajlovima koji su klonirani sa CRLF
git config --global core.autocrlf input

# Mark Windows-side repo-e kao bezbedne (sprečava "dubious ownership" grešku)
git config --global --add safe.directory "/mnt/c/Users/Korisnik/Documents/GitHub/devops/shop-operator"
# (Dodaj druge repo-e isto kad budu trebali iz WSL-a)

# Credential helper — koristi Windows Git Credential Manager (već autentifikovan kroz Windows)
git config --global credential.helper "/mnt/c/Program\ Files/Git/mingw64/bin/git-credential-manager.exe"
```

**Zašto credential helper:** GitHub je 2021. ukinuo password authentication za git operacije. Windows kubectl/git već koristi Git Credential Manager (OAuth browser flow, keširano u Windows Credential Manager). Ova linija kaže WSL git-u: "koristi isti keš". Bez ove linije WSL git pita za lozinku → fail.

**Provera da setup radi:** `git push` iz WSL-a treba da prođe bez prompt-a za lozinku.

---

## 3. WSL kubectl — jednokratan setup

WSL ima svoj odvojen Linux kubectl koji **ne deli kubeconfig sa Windows-om**. Bez setup-a:

```
kubectl get nodes
> error: current-context is not set
```

Reši tako što usmeriš WSL kubectl na isti kubeconfig koji k3d (pokrenut iz PowerShell-a) popunjava:

```bash
# Za trenutnu sesiju
export KUBECONFIG=/mnt/c/Users/Korisnik/.kube/config

# Persistuj u .bashrc da važi u svakoj sesiji
echo 'export KUBECONFIG=/mnt/c/Users/Korisnik/.kube/config' >> ~/.bashrc
```

**Provera:** `kubectl get nodes` u WSL-u treba da pokaže iste 3 noda kao iz PowerShell-a.

> Ovo radi jer Docker Desktop WSL integracija razrešava `host.docker.internal` iz WSL-a u host IP — što je gde k3d API server bind-uje port.

---

## 4. Pokretanje k3d klastera (sa nule, posle reboot-a)

```powershell
# Iz PowerShell-a (k3d je instaliran samo na Windows-u)
cd C:\Users\Korisnik\Documents\GitHub\devops\kube-state\clusters\local

# Ako klaster ne postoji
k3d cluster create --config cluster.yaml

# Ako postoji ali je zaustavljen (npr. posle reboot-a)
k3d cluster start local

# Ponovo instaliraj operatore ako klaster reset-ovan
helm install cnpg cnpg/cloudnative-pg -n cnpg-system --create-namespace
helm install community-operator mongodb/community-operator -n mongodb-operator --create-namespace

# Ponovo registruj naše CRD-ove (iz WSL-a, jer make je tamo)
# Otvori Ubuntu terminal:
# cd /mnt/c/Users/Korisnik/Documents/GitHub/devops/shop-operator
# make install
```

---

## 5. Kubebuilder scaffold (jednokratno po projektu) — WSL only

```bash
cd /mnt/c/Users/Korisnik/Documents/GitHub/devops/shop-operator

# Preduslovi (jednokratno na čistom WSL-u)
sudo apt-get install -y make build-essential

# Init projekta
kubebuilder init \
  --domain shophub.local \
  --project-name shop-operator \
  --repo github.com/shophub-devoops/shop-operator

# MULTI-GROUP — OBAVEZNO PRE create api, ako želiš >1 API grupu
kubebuilder edit --multigroup=true

# Generiši APIje
kubebuilder create api --group apps --version v1 --kind Shop \
  --resource --controller --plural shops
kubebuilder create api --group notify --version v1 --kind DiscordChannel \
  --resource --controller --plural discordchannels
kubebuilder create api --group payments --version v1 --kind Wallet \
  --resource --controller --plural wallets

# Generiši CRD YAML-ove iz markera + deepcopy metode
make manifests generate

# Instaliraj u klaster (treba KUBECONFIG iz sekcije 3)
make install
```

**Pogrešni redosledi:**
- `kubebuilder create api` pre `kubebuilder edit --multigroup=true` → druga `create api` (sa drugom grupom) pada sa "multiple groups are not allowed by default".
- `make manifests` bez `make` instaliranog → `executable file not found in $PATH`.

---

## 6. Najčešće greške koje smo lično prošli

| Simptom | Uzrok | Rešenje |
|---|---|---|
| `git pull` otvori vim sa merge commit-om | Commit-ovao si na lokalni main, pa kreirao granu, PR squashed — grane se razilaze | `:cq` u vim, `git merge --abort`, `git reset --hard origin/main` |
| `the branch 'feat/x' is not fully merged` na `git branch -d` | PR je merge-ovan kao squash → drugi SHA-evi | `git branch -D feat/x` (siguran je posle uspešnog merge-a) |
| `fatal: Authentication failed` iz WSL git push-a | Nema credential helper-a, GitHub ne podržava password auth | Vidi sekciju 2 |
| `current-context is not set` iz WSL kubectl-a | Linux kubectl ima sopstveni prazan kubeconfig | Vidi sekciju 3 |
| Lažne "modified" na svim fajlovima posle WSL git status | CRLF vs LF mismatch | `git config --global core.autocrlf input` |
| `make: command not found` u WSL-u | Ubuntu po defaultu nema make | `sudo apt install make build-essential` |
| `kubebuilder create api: multiple groups are not allowed by default` | Nije pokrenut `kubebuilder edit --multigroup=true` | Multi-group flag ide PRE `create api` komandi |
| `quay.io/spotahome/redis-operator: not found` | Spotahome projekat napušten, image-i obrisani | Koristi MongoDB Community Operator umesto Redis-a (spec dozvoljava) |
| `helm install ... CRD ... did not find expected node content` | Helm v4 stroži YAML parser, neki stari chart-ovi imaju malformed CRD | Probaj `--skip-crds` i instaliraj CRD ručno preko `kubectl apply -f` |
| `kubectl apply` za CRD: `metadata.annotations: Too long` | CRD prevelik za `last-applied-configuration` anotaciju | `kubectl apply --server-side -f` |

---

## 7. Quick reference — gde živi šta

| Tool | Gde se pokreće | Razlog |
|---|---|---|
| Docker Desktop | Windows tray | Backend za sve |
| k3d (cluster create/start/stop) | PowerShell | Priča sa Docker Desktop-om direktno |
| kubectl | PowerShell ILI WSL (oba rade posle setup-a iz sekcije 3) | Klijent ka API server-u |
| Helm | PowerShell ILI WSL (oba imaju Helm) | Klijent |
| kubebuilder, make, go, controller-gen | **WSL only** | Linux toolchain za operator dev |
| Git operacije za 5 repo-a | PowerShell ILI WSL (posle setup-a iz sekcije 2) | Oba rade isti repo na disku |
| VS Code | Windows | Sa otvorenim `devops` folderom kao root |
