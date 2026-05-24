# Kubernetes Operatori — Najbolje Prakse i Vodič za Implementaciju

> **Namena dokumenta:** Ovo je referentni vodič i instrukcije za AI agenta prilikom rada na projektu Kubernetes operatora pomoću Kubebuilder-a (Go). Sadrži pravila, konvencije i obavezne prakse koje agent mora poštovati prilikom generisanja koda i konfiguracija.

---

## 1. Osnovni koncepti

### Šta je operator?
- **Operator** = Pod/Deployment koji sluša izmene određenih objekata i procesira ih dovodeći ih u željeno stanje.
- Sastoji se od skupa **kontrolera** (control loops).
- **Operator pattern** = kombinacija *Custom Resource (CR)* + *Custom Controller*.

### Načini proširivanja Kubernetesa
1. **Custom Resource Definition (CRD) + operator** ← ovaj pristup koristimo
2. Konfiguracija Aggregation Layer-a

### Ključni pojmovi
- **CRD (Custom Resource Definition)** — definicija novog tipa resursa (RESTful endpoint + OpenAPI v3.0 šema na API Serveru).
- **Reconciler** — struktura koja implementira *reconcile petlju*.
- **Reconcile petlja** — petlja koja na izmene objekata procesira ih u skladu sa logikom kontrolera.

---

## 2. Podešavanje okruženja

**Obavezni alati i verzije:**
- Go `>= 1.25.6`
- Kubebuilder `>= 4.11.0`
- k3d `>= 5.8.3`

**Bash completion:**
```bash
sudo sh -c 'k3d completion bash > /etc/bash_completion.d/k3d'
sudo sh -c 'kubebuilder completion bash > /etc/bash_completion.d/kubebuilder'
```

**Lokalni klaster (`cluster.yaml`):**
```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: local
servers: 1
```
```bash
k3d cluster create --config cluster.yaml
```

---

## 3. Kreiranje projekta

### Inicijalizacija
```bash
kubebuilder init \
  --domain <tvoj-domen> \
  --project-name <ime-projekta> \
  --repo github.com/<user>/<repo>
```

**Konvencije:**
- `--domain`: osnova za API Group. Pun naziv grupe: `<group-name>.<domain>` (npr. `core.jutsu.com`, `billing.jutsu.com`).
- `--repo`: naziv Go modula.

### Dodavanje API-ja (CRD + kontroler)
```bash
kubebuilder create api \
  --group <group> \
  --version v1 \
  --kind <Kind> \
  --resource \
  --controller \
  --plural <plural>
```

**Generišu se:**
- `api/v1/<kind>_types.go` — definicija CRD (Spec, Status)
- `api/v1/groupversion_info.go`
- `internal/controller/<kind>_controller.go` — implementacija kontrolera
- `internal/controller/<kind>_controller_test.go` — unit testovi
- `internal/controller/suite_test.go`

---

## 4. Implementacija CRD-a

### Obavezni delovi CRD-a
| Polje | Značenje |
|-------|----------|
| `ApiVersion` | Grupa i verzija (`<group>.<domain>/<version>`) |
| `Kind` | Naziv resursa u **PascalCase** |
| `Scope` | `Namespaced` ili `Cluster` |
| `Spec` | Željeno stanje (definiše korisnik) |
| `Status` | Trenutno stanje (upisuje operator) |
| `Singular` | Jednina, za `kubectl` (npr. `dojo`) |
| `Plural` | Množina, za URL putanje (npr. `dojos`) |

> Kubebuilder automatski generiše `ApiVersion`, `Kind`, `Scope`, `Singular`, `Plural`. **Ti definišeš samo `Spec` i `Status`.**

### Pravila za `Spec`

**Pravilo o pokazivačima:**
- **Obavezni atributi → vrednosti** (`int`, `string`)
- **Opcioni atributi → pokazivači** (`*int`, `*string`)

**Razlog:** ako je opcioni `string` prazan (`""`), ne znamo da li je nedodeljen ili namerno postavljen — pokazivač eliminiše tu nejasnoću i sprečava neželjene efekte kod **Server Side Apply**.

**Validacija pomoću Kubebuilder markups:**
```go
type DojoSpec struct {
    AccountId string `json:"accountId"`
    Title     string `json:"title"`

    // +kubebuilder:default:="Postgres"
    Database *Database `json:"database"`

    CredentialsRef corev1.SecretReference `json:"credentialsRef"`

    // +kubebuilder:validation:Minimum=0
    // +kubebuilder:default:=1
    // +optional
    Replicas *int32 `json:"replicas"`
}
```

**Enumeracije:**
```go
// +kubebuilder:validation:Enum=Postgres;Mongo
type Database string
const (
    DatabasePostgres Database = "Postgres"
    DatabaseMongo    Database = "Mongo"
)
```

### Pravila za `Status`

**SVI atributi `Status`-a moraju biti opcioni** — pri prvom kreiranju objekta `Status` je prazan; kontroler ga kasnije popunjava.

**Standardne `Conditions` (preporuka):**
- `Available` — resurs spreman za korišćenje.
- `Progressing` — resurs se procesira ka novom željenom stanju.
- `Degraded` — nastala je neoporavljiva greška.

**Pravila uzajamne isključivosti:**
- `Degraded` i `Progressing` su **uzajamno isključivi** (`Degraded=True` ⇒ `Progressing=False`).
- `Available` **nije isključiv** sa ostalima:
  - `Available=True, Progressing=True` → ima replika, ali se skalira.
  - `Available=True, Degraded=True` → ima replika, ali postoji nepopravljiva greška.

**Pomoćne metode (obavezno implementirati):**
- `setUnrecoverableErrorStatus` → `Degraded=True, Progressing=False`
- `setProgressStatus` → `Degraded=False, Progressing=True`

**`Reason` taksonomija (konvencija, ne enforce-ovano):**

Kad postavljaš `Degraded=True`, koristi semantički `Reason` da signaliziraš **uzrok greške**:

| Reason | Značenje | Primer |
|--------|----------|--------|
| `Stalled` | Greška u **konfiguraciji** koju je korisnik napravio. **Nepopravljivo bez user intervencije** (mora se izmeniti `Spec`). | Pogrešan `selector`, nevalidan tip polja, conflict u OwnerReferences |
| `Failed` | Greška u **klaster infrastrukturi** — operator je sve odradio kako treba, ali klaster nije uspeo da kreira/update-uje resurs. | Cluster `Create()` poziv vraća internal error, quota prekoračena, scheduling neuspeh |
| `Init` / `Creating` / `Scaling` | Normalan napredak (`Progressing=True`, ne `Degraded`). | Prvi prolaz petlje, kreiranje child resursa, čekanje da Deployment dostigne `Ready` |

> **Zašto razlika bitna:** `Stalled` poručuje korisniku "moraš da promeniš YAML", `Failed` poručuje SRE-u "klaster ima problem". Drugi alati (Argo, Headlamp) prikazuju `Reason` direktno u UI.

**Backward/forward kompatibilnost:** `Status` je deo API-a. Drugi kontroleri (npr. ArgoCD koristi `Progressing`) zavise od njega — **ne uvodi breaking changes**.

```go
type DojoStatus struct {
    // +listType=map
    // +listMapKey=type
    // +optional
    Conditions []metav1.Condition `json:"conditions,omitempty"`

    // +optional
    Credentials corev1.SecretReference `json:"credentials,omitempty"`

    // +optional
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`
    // +optional
    UpdatedReplicas int32 `json:"updatedReplicas,omitempty"`
    // +optional
    AvailableReplicas int32 `json:"availableReplicas,omitempty"`
    // +optional
    ReadyStatus string `json:"readyStatus,omitempty"`
}
```

### Generisanje i instalacija
```bash
make manifests generate    # generiše CRD u config/crd/bases/<group>_<plural>.yaml
make install               # instalira CRD na klaster
```

---

## 5. Implementacija kontrolera (Reconciler)

### Reconciler fundamentals
Reconciler je **stateless** — ne pamti prethodno stanje. Mora biti **idempotentan**:

> **Idempotency:** operacija primenjena više puta daje isti rezultat kao prva primena.

Bez idempotentnosti može doći do beskonačne petlje. Idempotentnost se postiže korišćenjem `Status` objekta — Reconciler iz `Status`-a zaključuje u kojoj fazi se nalazi.

### Level-based dizajn (NE edge-based)
> Sistem mora ispravno raditi na osnovu **trenutnog vs. željenog** stanja, bez obzira na to koliko je međupromena propušteno.

Ako se objekat 5 puta promenio dok se petlja izvršava — interesuje nas **samo poslednja izmena**.

### Skelet Reconcile metode
```go
func (r *DojoReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    dojo := &corev1.Dojo{}
    if err := r.Get(ctx, req.NamespacedName, dojo); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }
    // logika...
    return ctrl.Result{}, nil
}
```

**`client.IgnoreNotFound(err)`** — kada objekat nije pronađen, on je obrisan i nema potrebe za daljim procesiranjem.

### Povratne vrednosti `Reconcile`
| Povratna vrednost | Značenje |
|-------------------|----------|
| `ctrl.Result{}, nil` | Uspešno — zaustavlja reconciling dok se objekat ne promeni |
| `ctrl.Result{}, err` | Greška — Exponential Backoff Requeue |
| `ctrl.Result{RequeueAfter: time.Minute}, nil` | Polling — vrati u Workqueue posle vremena |
| ~~`ctrl.Result{Requeue: true}, nil`~~ | **Deprecated** — koristi polling |

### Zlatna pravila Reconciler-a

1. **Jedna izmena objekta po iteraciji** — API Server prati `metadata.resourceVersion`. Slanje stare verzije → `409 Conflict`.
2. **Posle svakog `r.Update()` proveriti `!apierrors.IsConflict(err)`** — ignoriši konflikt i petlja će se ponovo okinuti.
3. **`No-op Update` ne pokreće broadcast** — koristi to da izbegneš beskonačne petlje.
4. **Reconciler ne menja `Spec`** — sme samo `Status` (i izuzetno `Scale`/`Finalizer`).
5. **Ne čekaj resurse u petlji** (`wait_until_running` ❌). Petlja ima timeout. Umesto toga: završi iteraciju i okini polling sa `RequeueAfter`.
6. **Sasvim je normalno da se petlja okida više puta** dok ne dođe do željenog stanja.

### Kategorizacija grešaka
- **Oporavljive** (npr. `409 Conflict`) → ignoriši i pusti petlju da se ponovo okine.
- **Neoporavljive** (npr. pogrešno konfigurisan Deployment, neispravan selector/ownership) → postavi `Degraded=True` u `Status`-u sa odgovarajućim `Reason`-om.

### r.Get() vs r.APIReader
- **`r.Get()`** — čita iz **keša** (uobičajeno). Keš može imati zastarelu verziju → moguć `409`.
- **`r.APIReader`** — direktno API Server (zaobilazi keš). **Ne zloupotrebljavaj** — udaraš u rate limit (default `--max-requests-inflight=400`).

### `client.Client` u pozadini koristi Informer
Komponente Informer-a:
- **Reflector** — gleda izmene na API Serveru.
- **Cache** — čuva kompletne objekte (Metadata, Spec, Status).
- **Workqueue** — metapodaci (`namespace/name`); uklanja duplikate ključeva → poštuje level-based dizajn.

Pri dobavljanju iz keša, biblioteka pravi **duboku kopiju** (deep copy) da spreči direktnu izmenu keša. Zato `make generate` generiše `api/v1/zz_generated_deepcopy.go`.

### Happy-path kroz 5 iteracija (mentalna mapa)

Reconciler radi **jednu izmenu po iteraciji** i izlazi. Petlja se okida ponovo na svaki `Update()`. Tipičan tok od `kubectl apply` do `Available=True`:

| Iter | Akcija | Posle iteracije |
|------|--------|-----------------|
| **1** | Pročitaj CR. Postavi `Status.Conditions = [Progressing=True, Reason=Init]`. `r.Status().Update()` → izlaz. | Cache update → reflektor → workqueue → triggeruje sledeću petlju. |
| **2** | CR ima status. `r.Get(deployment)` → `NotFound`. Pozovi `r.Create(deployment)` sa `SetControllerReference`. Postavi `Reason=Creating`. Izlaz. | Deployment kreiran, owned od strane CR-a. |
| **3** | Deployment postoji ali `ReadyReplicas=0`. Ne menjaj `Spec`. Možda update `Status.AvailableReplicas` na osnovu Deployment-ovog statusa. Izlaz, ostavi `Progressing=True`. | Čekamo da Kubernetes pokrene pod-ove. |
| **4** | Deployment ima `ReadyReplicas > 0`. Postavi `Available=True, Progressing=False, Reason=Available`. Izlaz. | Aplikacija prima saobraćaj. |
| **5** | User izmeni `spec.replicas` (npr. 2→5). Reconciler vidi mismatch sa Deployment-om, pozove `r.Update(deployment)` sa novim brojem. Vraća se na iteraciju 3 dok se ne ujednači. | Skaliranje. |

**Pravila koja čine ovaj tok ispravnim:**
- Svaka iteracija **bilo radi jednu izmenu, bilo samo čita**. Nikad dve izmene u jednom prolazu (vodi ka `409 Conflict`).
- Iz `Status`-a (`Reason`, `Conditions`) Reconciler zna gde je stao. To je *jedini* način — Reconciler je **stateless**.
- Ako iteracija 2 dobije `409` na `Create()` (neko drugi je već kreirao), to je benigno → ignoriši, sledeća iteracija će videti `Found`.
- Ako Deployment nikad ne dođe u `Ready` (ImagePullBackOff, OOM, scheduling): nakon nekog vremena postavi `Degraded=True, Reason=Failed, Message="<detalji>"`. Petlja staje dok user ne izmeni `Spec` ili dok problem ne nestane sam.

> **Heuristika:** ako tvoj Reconciler hoće da uradi >1 izmena u istoj iteraciji, razdvoji ih kroz iteracije preko `Status` polja. Primer: prvo postavi `Status.DatabaseSecret=...`, izađi. Sledeća iteracija vidi popunjeno polje i nastavlja sa kreiranjem Deployment-a.

---

## 6. Ownership i Garbage Collection

### Postavljanje ownership-a
```go
if err := ctrl.SetControllerReference(dojo, dep, r.Scheme); err != nil {
    return nil, err
}
```

**Šta ovo radi:**
- Postavlja `metadata.ownerReferences` na child objekat.
- Označava `controller: true` (samo jedan primarni owner).
- Pri brisanju Dojo objekta, **Kubernetes Garbage Collector (GC)** automatski briše Deployment.

**Pravila:**
- `controller: true` — samo **jedan** primarni owner po objektu.
- `blockOwnerDeletion: true` (default za kontrolere) — owner ne može biti obrisan dok se ne obriše child.
- GC briše objekat iz `etcd` čim ostane bez ijednog owner-a.

> **Najbolja praksa: jedan owner po objektu.** Više ownership-ova može blokirati brisanje (objekat ostaje u `Terminating` stanju). Umesto toga koristi **labele i selektore**.

### Posmatranje child resursa
```go
func (r *DojoReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&corev1.Dojo{}).
        Owns(&appsv1.Deployment{}).      // okida petlju za izmene Deployment-a koji je owned
        Named("dojo").
        Complete(r)
}
```

`Owns()` filtrira tako da se petlja okida **samo za child objekte koje vlasništvo Dojo-a** — optimizuje keš i memoriju.

---

## 7. Subresources

### Šta su podresursi?
Virtuelni endpoint-i koji dele jedan Kubernetes objekat radi specifične logike, validacije i RBAC-a:

| Podresurs | Namena |
|-----------|--------|
| `/status` | Izmene samo `Status`-a (Spec se ignoriše) |
| `/scale` | Standardni interfejs za HPA |
| `/finalizer` | Upravlja procesom brisanja |
| `/log`, `/exec`, `/portforward`, `/approval` | Privremene/streaming akcije |

### Implementacija `/scale` (za HPA i `kubectl scale`)
1. **Dodaj `Selector` u `Status`:**
```go
type DojoStatus struct {
    // ...
    // +optional
    Selector string `json:"selector,omitempty"`
}
```

2. **Konfiguriši markup-e na Kind strukturi:**
```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:path=dojos
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas,selectorpath=.status.selector
type Dojo struct {
    metav1.TypeMeta   `json:",inline"`
    // ...
}
```

### Print kolone i shortName
```go
// +kubebuilder:resource:shortName=dj,categories={all}
// +kubebuilder:printcolumn:name="READY",type="string",JSONPath=".status.readyStatus"
// +kubebuilder:printcolumn:name="UP-TO-DATE",type="integer",JSONPath=".status.updatedReplicas"
// +kubebuilder:printcolumn:name="AVAILABLE",type="integer",JSONPath=".status.availableReplicas"
// +kubebuilder:printcolumn:name="ACCOUNT",type="string",JSONPath=".spec.accountId"
// +kubebuilder:printcolumn:name="AGE",type="date",JSONPath=".metadata.creationTimestamp"
type Dojo struct {...}
```

> Posle izmene markup-a obavezno: `make uninstall && make manifests generate install run`.

---

## 8. Watches() — praćenje eksternih objekata

Kada operator zavisi od objekata koje **NE poseduje** (npr. Secret koji generiše drugi operator), koristi `Watches()`.

### Tri parametra `Watches()`
| Parametar | Uloga |
|-----------|-------|
| `Object` | Tip resursa koji se posmatra |
| `Predicate` | Filter — koji objekti idu dalje |
| `EventHandler` | Mapira filtrirani objekat na `[]reconcile.Request` |

### Primer: praćenje CNPG Secret-a
```go
cnpgLabelPredicate := predicate.NewPredicateFuncs(
    func(obj client.Object) bool {
        _, hasLabel := obj.GetLabels()["cnpg.io/cluster"]
        return hasLabel
    },
)

return ctrl.NewControllerManagedBy(mgr).
    For(&corev1.Dojo{}).
    Owns(&appsv1.Deployment{}).
    Owns(&k8scorev1.Secret{}).
    Watches(
        &k8scorev1.Secret{},
        handler.EnqueueRequestsFromMapFunc(r.findDojosForSecret),
        builder.WithPredicates(cnpgLabelPredicate),
    ).
    Named("dojo").
    Complete(r)
```

> **Predikat je krucijalan.** Bez njega EventHandler se okida na **bilo koju** izmenu Secret-a u klasteru → ozbiljan udar na CPU i memoriju operatora.

### Indeksi za efikasnu pretragu
**Bez indeksa** moraš dobaviti sve objekte i iterirati — `O(N)`. **Sa indeksom** je `O(1)`.

**Konfiguracija indeksa u `main.go`:**
```go
if err := mgr.GetFieldIndexer().IndexField(ctx, &corev1.Dojo{},
    ".spec.credentialsRef.name",
    func(rawObj client.Object) []string {
        dojo := rawObj.(*corev1.Dojo)
        if dojo.Spec.CredentialsRef.Name == "" {
            return nil
        }
        return []string{dojo.Spec.CredentialsRef.Name}
    },
); err != nil {
    setupLog.Error(err, "unable to set up the indexer for Dojo")
    os.Exit(1)
}
```

**Korišćenje indeksa u `r.List()`:**
```go
err := r.List(ctx, dojos,
    client.InNamespace(secret.Namespace),
    client.MatchingFields{SecretIndexField: secret.Name},
)
```

### Filtriranje po tipu događaja
`Watches()` po default-u prati `CREATE, UPDATE, DELETE, GENERIC`. Razmisli koji su tebi relevantni:
- Za izmene Secret-a — najčešće samo `UPDATE`.
- `DELETE` događaji možda ne mogu biti smisleno obrađeni (aplikacija jednostavno neće raditi bez baze).

---

## 9. RBAC

### Markup-i u kontroleru
```go
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core.jutsu.com,resources=dojos,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core.jutsu.com,resources=dojos/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=core.jutsu.com,resources=dojos/finalizers,verbs=update
```

### Pravilo
- **`make run`** — koristi `~/.kube/config` (admin permisije) → uvek radi.
- **Stvarni deployment** — koristi `config/rbac` (generisan iz markup-a). Ako markupi nisu dodati → operator **ne radi**.
- Posle izmene RBAC markup-a: `make manifests`.

---

## 10. Dependency: CloudNativePG (CNPG)

> **Pravilo:** za standardne komponente (baza podataka, message broker...) **koristi gotove operatore**, ne implementiraj svoju minimalnu verziju.

### Instalacija CNPG
```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm upgrade --install cnpg \
  --namespace cnpg-system \
  --create-namespace \
  cnpg/cloudnative-pg
```

### Kreiranje Postgres klastera
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: dojo
spec:
  instances: 1
  bootstrap:
    initdb:
      database: dojo
      owner: dojo
      postInitApplicationSQL:
        - CREATE TABLE weapons(id text not null, name text not null, PRIMARY KEY(id))
        - ALTER TABLE weapons OWNER TO dojo;
  storage:
    size: 1Gi
```

> `postInitApplicationSQL` izvršava root korisnik. **Eksplicitno postavi `OWNER TO <appuser>`** za sve tabele, inače aplikacija neće moći da im pristupi.

CNPG kreira Secret-e (`<name>-app`, `<name>-ca`, `<name>-replication`, `<name>-server`) sa kredencijalima (URI, host, port, user, password, dbname, ...).

---

## 11. Sample-ovi i testiranje

### Sample konfiguracije
- Lokacija: `config/samples/<group>_<version>_<kind>.yaml`
- Primena: `kubectl apply -f config/samples/...` ili `kubectl apply -k config/samples`

### Unit testovi
- Lokacija: `internal/controller/<kind>_controller_test.go`
- Ručno se okida svaka iteracija reconcile petlje.
- **Ograničenje:** brisanje namespace-a nije podržano → **svaki test mora ići u poseban namespace**.

---

## 12. Pakovanje i deployment

### Iz-kutije pogodnosti od Kubebuilder-a

Kubebuilder generiše `main.go` koji već uključuje:

- **Leader election** — ako pokrećeš operator sa **više replika** (HA setup), samo *jedna* radi Reconcile u datom trenutku. Ostale stoje u rezervi i preuzimaju ako primary padne. Aktivira se preko `--leader-elect` flag-a u `manager.yaml`-u. Bez ovoga, više replika bi paralelno menjalo iste objekte → race condition i konstantni `409 Conflict`.
- **Prometheus `/metrics` endpoint** — manager izlaže `/metrics` na portu `:8443` (TLS-protected) sa metrikama kao `workqueue_depth`, `controller_runtime_reconcile_total`, `reconcile_errors_total`. Treba ti samo `ServiceMonitor` da Prometheus scrape-uje. Kubebuilder generiše `config/prometheus/monitor.yaml` — uključi ga u Helm chart.
- **Health probes** — `/healthz` i `/readyz` su pripremljeni u manager-u; samo eksponiraj port.

> **Pravilo:** ne piši ove stvari sam. Kubebuilder skelet je već production-ready u tom delu. Tvoja briga je samo Reconcile logika.

### Container slika
```bash
docker login
make docker-buildx IMG=<repo>/<image>:<tag>
```

### Kustomize deployment
```bash
make deploy IMG=<repo>/<image>:<tag>
make undeploy
```
- Generiše `config/manager/manager.yaml`.
- **Sve dodatne izmene** prave se u `config/default/` folderu.
- Koristi `~/.kube/config` kredencijale.

### Helm deployment
```bash
make build-installer IMG=<repo>/<image>:<tag>
kubebuilder edit --plugins=helm/v2-alpha
# za naredne izmene:
kubebuilder edit --plugins=helm/v2-alpha --force
```

```bash
helm upgrade --install <name> ./dist/chart \
  --namespace <name>-system \
  --create-namespace
```

Chart se generiše u `dist/chart`.

---

## 13. Eksterni resursi i Finalizer-i

Kada kontroler upravlja resursima **van Kubernetes klastera** (Discord API, eksterne baze, cloud resursi...) — **MORA** koristiti **Finalizer-e**.

**Razlog:** GC briše samo Kubernetes objekte. Bez finalizer-a, brisanjem CR-a ostaju "siročad" eksterni resursi (npr. Discord kanal).

**Finalizer pattern:**
1. Kada se objekat kreira → dodaj finalizer u `metadata.finalizers`.
2. Kada se objekat briše → Kubernetes ga stavlja u `Terminating`, a **NE briše** dok je finalizer prisutan.
3. Reconciler detektuje `DeletionTimestamp != nil` → izvršava cleanup eksternog resursa → uklanja finalizer.
4. Kubernetes završava brisanje.

---

## 14. Kontrolna lista pre commit-a

Pre svakog `git commit`-a:

- [ ] `make manifests generate` — regenerisani CRD-ovi i deepcopy.
- [ ] Svi novi atributi u `Spec`-u imaju validacione markup-e.
- [ ] Svi atributi u `Status`-u označeni kao `+optional`.
- [ ] Reconciler je idempotentan (ne radi nikakvu akciju ako je već u željenom stanju).
- [ ] Reconciler vraća `client.IgnoreNotFound(err)` za case kada objekat ne postoji.
- [ ] Sve `r.Update()` provere prate `!apierrors.IsConflict(err)` logiku.
- [ ] Reconciler **ne menja** `Spec`.
- [ ] Postavljen `OwnerReference` za sve child objekte koji se kreiraju.
- [ ] `Owns()` ili `Watches()` konfigurisan u `SetupWithManager()`.
- [ ] `Watches()` ima **predikat** koji filtrira nepotrebne događaje.
- [ ] RBAC markup-i postoje za sve resurse koje kontroler dotiče.
- [ ] Indeksi (`IndexField`) konfigurisani u `main.go` ako se radi `r.List()` po polju.
- [ ] Za eksterne resurse: implementiran finalizer.
- [ ] Conditions postavljene na sve relevantne tačke (`Available`, `Progressing`, `Degraded`).

---

## 15. Brze komande (cheatsheet)

| Akcija | Komanda |
|--------|---------|
| Inicijalizacija projekta | `kubebuilder init --domain ... --project-name ... --repo ...` |
| Dodavanje API-ja | `kubebuilder create api --group ... --version v1 --kind ... --resource --controller --plural ...` |
| Generisanje CRD i deepcopy | `make manifests generate` |
| Instalacija CRD-a | `make install` |
| Pokretanje operatora lokalno | `make run` |
| Deinstalacija CRD-a | `make uninstall` |
| Build container slike | `make docker-buildx IMG=...` |
| Deployment (Kustomize) | `make deploy IMG=...` |
| Undeploy (Kustomize) | `make undeploy` |
| Apply sample-a | `kubectl apply -k config/samples` |
| Skaliranje preko CR | `kubectl scale <kind> <name> --replicas <n>` |

---

## 16. Antipatterni — šta NIKAD ne raditi

❌ **Reconciler menja `Spec` objekta.** Spec je vlasništvo korisnika.
❌ **Čekanje resursa u petlji** (`wait_until_running`). Koristi `RequeueAfter`.
❌ **Više izmena objekta po iteraciji.** Vodi ka `409 Conflict`.
❌ **Korišćenje `r.APIReader` umesto `r.Get()` rutinski.** Udaraš rate limit.
❌ **`Watches()` bez predikata.** Operator se okida na sve eventove → CPU/memorija eksplozija.
❌ **`r.List()` bez indeksa kad se traži po polju.** O(N) umesto O(1).
❌ **Više primarnih owner-a (`controller: true`) na istom objektu.** Konflikti i blokirano brisanje.
❌ **Obavezni atributi kao pokazivači u `Spec`-u.** Pravi nejasnoću pri Server Side Apply.
❌ **Atributi u `Status`-u koji nisu `+optional`.** Prvi `apply` propada.
❌ **Breaking changes u `Status` polju.** Razbiješ ArgoCD i druge alate.
❌ **Eksterni resursi bez finalizer-a.** Ostavljaš siročad.
❌ **Implementiranje sopstvenog operatora za standardne sisteme** (npr. Postgres) umesto korišćenja gotovih (CNPG).

---

## 17. Komunikacija komponenti (mental model)

```
┌──────────┐  REST   ┌──────────────┐  write   ┌──────┐
│  kubectl │────────▶│  API Server  │─────────▶│ etcd │
└──────────┘         │              │          └──────┘
                     │  - Webhooks  │
                     │  - Broadcast │
                     └──────┬───────┘
                            │ events (broadcast)
                            ▼
                     ┌──────────────┐
                     │   Operator   │
                     │ ┌──────────┐ │
                     │ │ Informer │ │ (Reflector + Cache + Workqueue)
                     │ └────┬─────┘ │
                     │      ▼       │
                     │ ┌──────────┐ │
                     │ │Reconciler│ │
                     │ └──────────┘ │
                     └──────────────┘
```

- API Server je **jedini** koji upisuje u `etcd`.
- Pre upisa pozivaju se **Webhook-ovi**.
- Posle upisa **broadcast** ka svim zaintresovanim komponentama.
- Operator preko Informer-a dobija eventove → Workqueue → Reconciler.
- `r.Get()` čita iz **keša** (Informer cache); `r.Update()` direktno API Serveru.

---

> **Završna napomena za AI agenta:** Ako bilo kojem zadatku nedostaje kontekst (npr. tačan domen, naziv grupe, image registry), **pitaj korisnika pre nego što pretpostaviš**. Konvencije iz ovog dokumenta su prioritet — ne odstupaj od njih bez eksplicitne potvrde.
