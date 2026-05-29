# Vežbe 5 — Operatori, Helm, eksterni resursi (Discord)

> **Izvor:** transkript poslednjih vežbi (`vezbe 5.odt`). Ovde je sažeto i grupisano po temama. Originalni naglasci su zadržani; gde je nešto bilo nejasno u govoru, dodate su kratke beleške.

---

## 1. Graceful shutdown i obrada signala

- Kad koristimo `sleep infinity` / idle state, brisanje traje ~10s dok ne istekne TERM timeout, pa onda SIGKILL.
- **Pravi pristup**: obraditi `SIGTERM` u Go kodu — odraditi biznis logiku, vratiti greške, zatvoriti konekcije, **pa tek onda** izaći iz procesa.
- Bez pravilne obrade signala Kubernetes mora da forcira brisanje preko KILL signala što ostavlja zaglavljene konekcije/transakcije.

## 2. OwnerReferences i Garbage Collector

### Tri vrednosti koje kontrolišu GC
- `apiVersion` + `kind` + `name` + `uid` — koji je objekat vlasnik
- `controller: true` — ko je primarni kontroler
- `blockOwnerDeletion: true` — sprečava brisanje vlasnika dok dependent objekat postoji

### Lanac brisanja
Kad obrišeš objekat sa OwnerReferences chain-om, GC pokreće dependency walk u memoriji:
1. Pronalazi sve dependent objekte
2. Briše ih redom (počevši od dna lanca)
3. Dok god deployment nije obrisan, Pod ostaje u `Terminating`
4. Dok god kontejner ne izađe iz docker engine-a, deployment ostaje u `Terminating`

### Block Owner Deletion — kada NE koristiti
- Jedan objekat može imati **više** vlasnika u `OwnerReferences`, ali samo **jedan** sme biti `controller: true`.
- Ako više objekata pokazuje na istog dependent-a sa `blockOwnerDeletion: true`, dolazi do **deadlock-a** — niko ne može da bude obrisan.
- **Preporuka:** ne koristi `blockOwnerDeletion` osim u 1-na-1 vezama. Za 1-na-više implementiraj **labele** i sam sprovodi logiku zavisnosti u kontroleru.

## 3. Arhitektura kontrolera (reflector + informer)

### Dijagram koji moramo znati u sred noći

```
                ┌──────────────┐
                │  API server  │  ← jedini sme da edituje etcd
                └──────┬───────┘
                       │ Watch
                       ▼
                 ┌───────────┐
                 │ Reflector │  ← kešira sve obekvene resurse
                 └─────┬─────┘
                       │ Mapping + Predicate
                       ▼
                ┌──────────────┐
                │ Work Queue   │  ← dedup po `namespace/name`
                └──────┬───────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Reconciler loop │
              └─────────────────┘
```

- **API server** je jedini koji edituje bazu (etcd). Ni kubelet, ni scheduler, ni controller manager.
- **Reflector** posmatra promene, sve trpa u **cache** u memoriji procesa.
- Bez `Watches` predikata, reflector **trpa SVE resurse** odgovarajućeg tipa u cache → **memory leak** kako klastersko stanje raste.
- **Work queue** drži samo `namespace/name` (jedinstven ID) i radi **deduplikaciju** — ako 5 promena udari na isti CR, reconciler se okida samo jednom za poslednje stanje.
- **Get u reconcileru čita iz cache-a**, ne ide direktno na API server. Zato je važno da watches dobro filtrira.

### Predicate vs EventHandler
- **Predicate** — filter NA ULAZU u reflektor (pre kešovanja). Ovde isključiš tipove događaja koji te ne zanimaju.
- **EventHandler** — mapping NA IZLAZU iz reflektora (pre upisa u queue). Ovde rešavaš 1-na-više slučaj: jedan secret može da pokrene reconciler za 5 različitih Shop-ova.

```go
.Watches(
    &corev1.Secret{},
    handler.EnqueueRequestsFromMapFunc(r.findShopForCNPGSecret),
    builder.WithPredicates(cnpgSecretPredicate),
)
```

## 4. Indeksi za O(1) pretragu

Bez indeksiranja, traženje "koji Shop koristi ovaj secret" je **O(N)** — iteracija kroz svu kolekciju.

Sa indeksom, mapa kešira `secret-name → shop-name` rezultat → **O(1)**.

Indekse konfigurišemo u `main.go` preko `FieldIndexer`:

```go
if err := mgr.GetFieldIndexer().IndexField(ctx, &appsv1.Shop{},
    ".spec.title",
    func(obj client.Object) []string {
        return []string{obj.(*appsv1.Shop).Spec.Title}
    }); err != nil {
    return err
}
```

> Asistent je rekao: "ne morate ovako da radite, ali je preporuka" — za projekat može i bez ako je skup malo Shop-ova.

## 5. Subresursi u CRD-ovima

REST API princip: resurs ima **pod-resurse** koji su delovi šire celine. Primer iz REST sveta:
- `POST /api/v1/booking` — kreira ceo booking
- `GET /api/v1/booking/{id}/bed_number` — vraća samo broj kreveta

Kod Kubernetes CRD-ova imamo tri ugrađena subresursa:

| Subresurs | Šta radi | Ko ga menja |
|---|---|---|
| `/status` | drži observed state | samo reconciler (`r.Status().Update()`) |
| `/scale` | broj replika (integration sa HPA) | autoscaler ili `kubectl scale` |
| `/finalizers` | lista finalizera za eksterne resurse | reconciler pre brisanja |

### `+kubebuilder:subresource:status`
Default svaki novi CRD ima ovo. Reconciler može da menja samo Status; **Spec menja korisnik** (kroz `kubectl apply`).

Ako reconciler pokuša PUT na Spec, Kubernetes vraća **403 Forbidden** — namerno.

### `+kubebuilder:subresource:scale`

```go
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas,selectorpath=.status.selector
```

Tri obavezne stvari:
1. **specpath** — koje polje Spec-a drži desired replicas
2. **statuspath** — koje polje Status-a drži ready replicas (autoscaler ovo čita)
3. **selectorpath** — kako se label selector formira (npr. `app=dojo`)

Bez `selectorpath`, `kubectl scale` radi ali HPA ne može da pronađe pod-ove.

**Plus**: treba ti `selector` field u Spec-u (string, opciono) i odgovarajući RBAC marker:

```go
// +kubebuilder:rbac:groups=apps.shophub.local,resources=shops/scale,verbs=get;update;patch
```

### Print kolone

```go
// +kubebuilder:printcolumn:name="TITLE",type="string",JSONPath=".spec.title"
// +kubebuilder:printcolumn:name="DB",type="string",JSONPath=".spec.database"
// +kubebuilder:printcolumn:name="READY",type="integer",JSONPath=".status.readyReplicas"
```

Toliko jednostavno. `kubectl get shops` postaje čitljiv.

## 6. CloudNativePG operator za PostgreSQL

### Filozofija
Ne pišemo svoj postgres kontroler — to je **previše posla** i naša implementacija ne bi bila industrijska. CNPG je standard, koristimo njegove CRD-ove:
- `Cluster` — instance, storage, bootstrap
- `Database` — schema unutar cluster-a (može više baza po klasteru)
- `Backup`, `ScheduledBackup`, `Pooler` — dodatne stvari

### Bootstrap CR primer
Ono što naš operator pravi za svaki Shop sa `database: postgres`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-shop
spec:
  instances: 1
  bootstrap:
    initdb:
      database: app
      owner: app
      # opciono: secret postojeći, inače CNPG generiše
  storage:
    size: 1Gi
```

### Auto-generated Secret
CNPG kreira `<cluster-name>-app` Secret sa poljima:
- `username`
- `password`
- `host` (interno cluster DNS)
- `port` (5432)
- `dbname`
- `uri` (full PG connection string)

**Naš operator postavlja `envFrom: secretRef: <cluster-name>-app`** u Deployment-u Shop-a → Shop backend dobija env vars `PGHOST`, `PGUSER`, `PGPASSWORD`, `PGDATABASE`, `URI` bez ručnog parsiranja.

### Watches za CNPG Secret
Naš operator treba da reaguje **samo na `Update`** event za CNPG secret:
- `Create` — beskorisno (secret se kreira pre Deployment-a)
- `Update` — zna se da je CNPG rotirao password → update env vars + restart Pod-ova
- `Delete` — **nismo handlovali**. Asistent: u dokumentaciji napišite "ne dirajte ovaj secret rukom". Adekvatno rešenje ne postoji; operator čeka da neko vrati secret.

```go
.Watches(
    &corev1.Secret{},
    handler.EnqueueRequestsFromMapFunc(r.findShopForCNPGSecret),
    builder.WithPredicates(predicate.Funcs{
        UpdateFunc: func(e event.UpdateEvent) bool { return hasCNPGLabel(e.ObjectNew) },
        CreateFunc: func(_ event.CreateEvent) bool { return false },
        DeleteFunc: func(_ event.DeleteEvent) bool { return false },
    }),
)
```

## 7. Helm chart filozofija za projekat

### `values.yaml` sa "enabled" prekidačima

```yaml
postgres:
  enabled: true
  # CNPG cluster konfiguracija...

mongo:
  enabled: false
  # Mongo konfiguracija...

app:
  enabled: false  # uključi tek u drugoj iteraciji
  name: my-shop
  database: postgres
```

### Iterativan deploy — zašto NE sve odjednom

Asistent: kad uradiš `helm install` sa svim `enabled: true`:
1. Helm prosleđuje sve manifeste API server-u u **paralelnoj** instalaciji
2. Aplikacija pokuša da se startuje pre nego što secret postoji
3. **Race condition** — Pod-ovi crashuju, helm ne vidi to kao grešku (success vraća, jer manifestе je samo applied)
4. Najveći problem helma — nije za logiku stanja, samo za templating

**Pravi pristup**:
1. Iteracija 1: `postgres.enabled: true`, `app.enabled: false` → CNPG digne bazu
2. Sačekaj da CNPG kreira secret (`kubectl wait`)
3. Iteracija 2: `app.enabled: true` → tek sada Deployment, koji odmah ima secret

### Bitnami helm charts kao reference

Asistent je rekao da pogledamo https://github.com/bitnami/charts za primere:
- `bitnami/postgresql`
- `bitnami/kafka`
- `bitnami/mongodb`

Svaki ima ogroman `values.yaml` (~1000 linija) — ali to je za production. Za studentski projekat 20-30 linija u values je sasvim dovoljno.

## 8. RBAC markeri

Default kubebuilder markeri pokrivaju samo "naše" CRD-ove (`shops`, `discordchannels`, `wallets`). Kad počnemo da kontrolišemo eksterne resurse (CNPG Cluster, Secret, Deployment), **moramo eksplicitno** da proširimo RBAC.

```go
// +kubebuilder:rbac:groups=apps,resources=deployments,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=postgresql.cnpg.io,resources=clusters,verbs=get;list;watch;create;update;patch;delete
```

Bez ovog, reconciler dobija `403 Forbidden` od API server-a.

## 9. Unit testovi za operator

### Princip
Mi ručno pokrećemo `Reconcile()` u testu:

```go
func TestShopReconciler_Reconcile_creates_deployment(t *testing.T) {
    // setup fake client + scheme
    cl := fake.NewClientBuilder().WithScheme(scheme).Build()
    r := &ShopReconciler{Client: cl, Scheme: scheme}

    // create input Shop CR
    shop := &appsv1.Shop{...}
    require.NoError(t, cl.Create(ctx, shop))

    // run Reconcile manually
    _, err := r.Reconcile(ctx, ctrl.Request{NamespacedName: types.NamespacedName{
        Namespace: shop.Namespace, Name: shop.Name,
    }})
    require.NoError(t, err)

    // assert Deployment was created
    dep := &k8sappsv1.Deployment{}
    require.NoError(t, cl.Get(ctx, types.NamespacedName{...}, dep))
}
```

**Pripaziti:**
- **`envtest` ne podržava brisanje namespace-a** (framework limitacija). Standardni `BeforeEach/AfterEach` koji briše namespace ne radi.
- Workaround: kreiraj **nov namespace** za svaki test (sa randomizovanim sufiksom) umesto da brišeš stari.

## 10. Make deploy vs make run

| Komanda | Gde radi operator | Kad koristiti |
|---|---|---|
| `make run` | lokalno, kao Go proces u WSL-u, koristi tvoj kubeconfig | razvoj, debug, brze iteracije |
| `make deploy` | unutar cluster-a, kao Pod (Deployment), koristi ServiceAccount | demo, produkcijska simulacija |

`make deploy` pod haubom:
1. `docker build` → slika sa tagom (npr. `v0.4.0`)
2. `docker push` na DockerHub
3. `kustomize build config/default | kubectl apply -f -` → kreira namespace, Deployment, RBAC

Asistent: na odbrani moraće `make deploy` da radi end-to-end, ne samo `make run`. **Helm chart koji smo napravili (Faza 7.1) je infrastruktura za ovo.**

## 11. Finalizeri za eksterne resurse

### Problem
Discord kanal **ne živi u Kubernetes-u**. Ako obrišemo `DiscordChannel` CR, K8s GC briše CR — ali kanal na Discord server-u ostaje "siroče".

### Rešenje
Finalizer — stringovni marker u `metadata.finalizers` koji **sprečava** GC da završi brisanje dok ga reconciler ne ukloni.

```go
const discordFinalizer = "shophub.local/discord-channel"

func (r *DiscordChannelReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    ch := &notifyv1.DiscordChannel{}
    if err := r.Get(ctx, req.NamespacedName, ch); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Dodaj finalizer pri prvom reconcile-u
    if ch.DeletionTimestamp.IsZero() {
        if !controllerutil.ContainsFinalizer(ch, discordFinalizer) {
            controllerutil.AddFinalizer(ch, discordFinalizer)
            return ctrl.Result{}, r.Update(ctx, ch)
        }
        // ... normalna reconcile logika ...
    }

    // Ako je u brisanju
    if !ch.DeletionTimestamp.IsZero() && controllerutil.ContainsFinalizer(ch, discordFinalizer) {
        if err := r.deleteDiscordChannel(ctx, ch); err != nil {
            return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
        }
        controllerutil.RemoveFinalizer(ch, discordFinalizer)
        return ctrl.Result{}, r.Update(ctx, ch)
    }

    return ctrl.Result{}, nil
}
```

### Kad god treba finalizer?
**Uvek kad kontroler manageuje resurs van Kubernetes-a:**
- Discord kanali / poruke
- Cloud resursi (AWS S3 bucket, GCP project)
- Spoljne baze (Stripe customer, Auth0 user)
- Bilo šta sa "lifecycle" van API server-a

### Asistent o prometheus alarmima preko Discord-a (Verzija 5)

Tema poslednjih vežbi je bila i da:
- Definišemo `PrometheusRule` resurse (npr. CPU > 80% → alarm)
- Koristimo `discord-go` biblioteku ili Alertmanager webhook
- Pravimo CRD-ove: `DiscordServer`, `DiscordChannel`, `DiscordChannelGroup`
- Svaki ima sekcije `warnings`, `critical`, ko slušaju koje kanale

**Mi smo to već implementirali** (PR #6 — DiscordChannel reconciler sa finalizer-om). Sledeća iteracija je integracija sa Alertmanager-om (Faza 6 deeper).

---

## Šta IMAMO iz ove sesije i šta SLEDI

### Već implementirano u našem repo-u
- ✅ Graceful shutdown u ShopHub backendu (`http.Server.Shutdown` + SIGTERM)
- ✅ OwnerReferences chain (Shop → Cluster + Deployment + Service + ServiceMonitor)
- ✅ Watches sa default predicate (preko `Owns()`)
- ✅ `+kubebuilder:subresource:status` na svim CRD-ovima
- ✅ Print kolone na sva 3 CRD-a
- ✅ CNPG integracija sa bootstrap + `<name>-app` secret
- ✅ Helm chart za shop-operator (Faza 7.1)
- ✅ RBAC markeri za deployments, services, secrets, CNPG, MongoDB
- ✅ DiscordChannel finalizer za external API cleanup

### Ostalo iz vežbi (potencijalne TODO stavke)
- ❌ `+kubebuilder:subresource:scale` za `kubectl scale shop`
- ❌ HPA integracija (zahtev nije eksplicitan u spec-u 4.1, ali bi bilo bonus)
- ❌ FieldIndexer za O(1) pretragu (nice-to-have)
- ❌ Watches sa custom predicate koji filtrira samo CNPG-labeled secrets (jer trenutno `Owns(Cluster)` posredno pokriva ovo)
- ❌ PrometheusRule resurs za alarme (Faza 6 deeper)
- ❌ Alertmanager → Discord integration
- ❌ Bitnami-style helm chart za ShopHub aplikaciju (Faza 7.2)
- ❌ Unit testovi po šablonu iz sekcije 9

### Ono što je asistent posebno **naglasio za odbranu**
- Dijagram reflektor/informer/work queue/reconciler — "u sred noći"
- `make deploy` mora da radi end-to-end (ne samo `make run`)
- Iterativan helm deploy (postgres prvo, app posle)
- Operator se **oslanja na druge operatore** (CNPG za postgres, Mongo community za mongo) — naš ne sme da pokušava da pravi StatefulSet-ove direktno
- `BlockOwnerDeletion` deadlock — znaj da postoji
- Finalizer pattern za eksterne resurse
