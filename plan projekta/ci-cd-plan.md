# CI/CD plan — kompletan pipeline za ShopHub projekat

> **Razlog hitnosti:** asistent je na poslednjim vežbama eksplicitno rekao da CI/CD treba implementirati **što ranije**, ne na kraju. Idealno na početku projekta, ali nije katastrofa ako se uradi posle Faze 2-3.
>
> **Spec referenca:** zahtev 5.2 traži CI pipeline za ShopHub, Shop i Shop operator repo-e koji pokriva build + unit/integration testove + image build i publish.
>
> **Trenutno stanje (2026-05-28):** imamo samo `commitlint` workflow + kubebuilder default test/lint/e2e workflow-e u `shop-operator` repo-u (svi prolaze). Image build + publish nemamo. ShopHub i Shop repo-i nemaju test/lint workflow-e.

---

## 1. Šta čini "kompletan CI/CD" za ovaj projekat

CI/CD se deli na **CI (Continuous Integration)** — sve što se okida na PR/push, validira kod, i **CD (Continuous Deployment/Delivery)** — automatski deploy/release artefakta.

Po prioritetu:

### CI — okida se na svaki PR ka main

| # | Workflow | Šta radi | Repo-i |
|---|---|---|---|
| 1 | `commitlint` | Validira Conventional Commits format | svi 5 |
| 2 | `test` | `go build` + `go test` + `go vet` | shop-operator, shop, shophub |
| 3 | `lint` | `golangci-lint run` | shop-operator, shop, shophub |
| 4 | `e2e` (samo shop-operator) | `kind` cluster + integracioni testovi | shop-operator |
| 5 | `helm-lint` | `helm lint charts/*` | helm-charts |
| 6 | `docker-build` | Build Docker image (NE push, samo validacija) | shop-operator, shop, shophub |
| 7 | `hadolint` | Linter za Dockerfile | shop-operator, shop, shophub |
| 8 | (opciono) `trivy-fs` | Skenira repo na vulnerabilities | svi |

**Branch protection** (već postavljeno za main):
- Require pull request before merging
- Require approvals: 1
- Require **status checks to pass** — sve gornje workflow-e treba dodati kao required
- Require linear history (squash merge default)

### CD — okida se na merge u main (push event)

| # | Workflow | Šta radi | Repo-i |
|---|---|---|---|
| 9 | `docker-publish` | Build + tag + push na DockerHub | shop-operator, shop, shophub |
| 10 | `helm-package-push` | `helm package` + `helm push` na OCI registry | helm-charts |
| 11 | `release-please` ili manual | SemVer tag iz Conventional Commits | svi |

### Optional (Faza 9 / bonus)
- `argocd-sync` ili `flux-trigger` — GitOps automatic deploy na live cluster
- `kubectl-apply` u CI runner-u (ne preporučljivo za production, OK za demo)

---

## 2. Konkretni GitHub Actions workflow fajlovi

### `.github/workflows/test.yml` (Go repo-i)

```yaml
name: Test
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    name: Run on Ubuntu
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: go mod tidy
      - run: go build ./...
      - run: go vet ./...
      - run: go test -race -coverprofile=coverage.out ./...
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ github.sha }}
          path: coverage.out
```

> **Napomena:** kubebuilder već generiše ovaj fajl u `shop-operator` repo-u. Treba ga **ručno dodati** u `shop`, `shophub` repo-ima.

### `.github/workflows/lint.yml`

```yaml
name: Lint
on:
  push:
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - uses: golangci/golangci-lint-action@v8
```

> Već imamo u shop-operator-u (radi). Replikuj u shop + shophub.

### `.github/workflows/docker-build.yml` (PR-time validation)

```yaml
name: Docker Build
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Lint Dockerfile
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
      - name: Build (no push)
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          tags: ghcr.io/${{ github.repository }}:pr-${{ github.event.pull_request.number || github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### `.github/workflows/docker-publish.yml` (CD — samo na push u main)

```yaml
name: Docker Publish
on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3

      - name: Compute version
        id: ver
        run: |
          if [[ "${GITHUB_REF}" == refs/tags/v* ]]; then
            echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
          else
            echo "version=0.0.0-$(date +%Y%m%d)-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
          fi

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build + push
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            docker.io/shophub-devoops/${{ github.event.repository.name }}:${{ steps.ver.outputs.version }}
            docker.io/shophub-devoops/${{ github.event.repository.name }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### `.github/workflows/helm-lint.yml` (helm-charts repo)

```yaml
name: Helm Lint
on:
  push:
  pull_request:

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
      - name: Lint charts
        run: |
          for chart in charts/*/; do
            helm lint "$chart"
          done
      - name: Template charts (sanity)
        run: |
          for chart in charts/*/; do
            helm template demo "$chart" --namespace test
          done
```

### `.github/workflows/helm-publish.yml` (helm-charts, CD)

```yaml
name: Helm Publish
on:
  push:
    branches: [main]
    paths: ['charts/**']

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
      - name: Login to DockerHub as OCI registry
        run: |
          echo "${{ secrets.DOCKERHUB_TOKEN }}" | \
            helm registry login docker.io -u ${{ secrets.DOCKERHUB_USERNAME }} --password-stdin
      - name: Package + push all charts
        run: |
          for chart in charts/*/; do
            helm package "$chart" -d /tmp/packaged
          done
          for pkg in /tmp/packaged/*.tgz; do
            helm push "$pkg" oci://docker.io/shophub-devoops
          done
```

---

## 3. Secrets koji treba podesiti u GitHub Repo Settings

Za **svaki repo koji pushuje image-e** (`shop-operator`, `shop`, `shophub`, `helm-charts`):

- `DOCKERHUB_USERNAME` — DockerHub korisničko ime (`shophub-devoops`)
- `DOCKERHUB_TOKEN` — Access token sa **Read/Write/Delete** pravima

### Kako se postavlja u GitHub-u
1. Repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
2. Name: `DOCKERHUB_USERNAME`, Value: `shophub-devoops` → Add
3. Name: `DOCKERHUB_TOKEN`, Value: (paste iz DockerHub) → Add

### Kako se generiše DockerHub token
1. https://hub.docker.com → ulogovati se kao `shophub-devoops`
2. Account Settings → Security → New Access Token
3. Description: `github-actions`, Scope: **Read, Write, Delete**
4. Generate → **kopiraj odmah** (vidi se samo jednom) → paste u GitHub secret
5. **Nikad ne stavljaj token u repo fajlove**, makar i privremeno

---

## 4. Semantic Versioning policy

Spec 5.2 traži SemVer za kontejner verzije. Strategija:

### Verzije iz git tag-ova
- `v0.1.0` — početak (već smo)
- `v0.1.1` — bugfix patches
- `v0.2.0` — minor features (recimo Wallet reconciler dodat)
- `v1.0.0` — production ready (odbrana)

### Kako se tag-uje
```bash
git tag -a v0.1.0 -m "Initial scaffold"
git push origin v0.1.0
```

Workflow `docker-publish` će na `refs/tags/v*` push event automatski tag-ovati image sa istom verzijom.

### Automatizacija (opciono)
**release-please** action automatski generiše tag-ove i CHANGELOG iz Conventional Commits:

```yaml
- uses: googleapis/release-please-action@v4
  with:
    release-type: go
```

Ako commit ima `feat:`, dize minor verziju. `fix:` digne patch. `feat!:` ili `BREAKING CHANGE:` u body-ju → major.

---

## 5. Redosled implementacije

Predlog redosleda da brzo zaštitimo kod (dok ne idemo previše unazad da pravimo image-e):

### Faza A — CI (validacija PR-a)
1. **shophub repo** — `test.yml` + `lint.yml` + `docker-build.yml`. ~30 min.
2. **shop repo** — kad Shop backend bude napisan (Faza 3). Iste workflow-e.
3. **helm-charts repo** — `helm-lint.yml`. ~10 min.

### Faza B — CD (publish na main)
4. **DockerHub token** generisanje + GitHub secrets u 3 repo-a. ~10 min.
5. **shop-operator repo** — `docker-publish.yml`. Posle prvog merge-a, image je na DockerHub-u kao `0.0.0-YYYYMMDD-sha7`. ~20 min.
6. **shophub repo** — isto. ~20 min.
7. **helm-charts repo** — `helm-publish.yml`. ~30 min.

### Faza C — Polished
8. **SemVer tagovanje** za prvu pravu verziju (`v0.1.0`). Manual ili release-please.
9. **Update shop-operator helm chart** da pokazuje na publish-ovan image: u `values.yaml` change `image.repository` iz `docker.io/shophub-devoops/shop-operator` na taj koji workflow gura.
10. **Test in-cluster deploy**: `helm install shop-operator oci://docker.io/shophub-devoops/shop-operator --version 0.1.0 -n shop-operator-system` → operator se pojavi kao Pod.

### Faza D — Bonus (može i kasnije)
11. Hadolint za sve Dockerfile-e
12. Trivy security scan
13. Coverage report uplod na Codecov
14. ArgoCD/Flux integration (zahtev 5.3 opciono)

---

## 6. Kako mi (operativno) ovo radimo

**Po jedan PR po Faza A koraku** — manji PR-ovi su lakši za review.

Šablon commit poruka:
```
ci(shop): add test workflow

- go build, go vet, go test on push and pull_request
- coverage uploaded as artifact
- triggers branch protection check
```

```
ci(shop-operator): add docker publish workflow

- triggers on push to main + tags (v*)
- builds and pushes to docker.io/shophub-devoops/shop-operator
- versions: latest, SHA-based for main, semver from tag
- requires DOCKERHUB_USERNAME + DOCKERHUB_TOKEN secrets
```

---

## 7. Šta NE moramo (za projektnu odbranu)

- **Multi-arch image-i** (amd64 + arm64). Pre-built za amd64 je dovoljno.
- **Image signing** (cosign, sigstore). Spec ne traži.
- **SBOM generisanje**. Ne traži se.
- **Pre-commit hooks**. Conventional Commits validira commitlint workflow.
- **CodeQL security scan**. GitHub default za public repo, ne moramo extra.
- **Reusable workflows**. Manja optimizacija, ne treba zbog 4 repo-a.
- **Matrix builds** (više Go verzija). Mi targetujemo Go 1.26.

---

## 8. Vremenski budžet — realistično

| Korak | Vreme |
|---|---|
| shophub test+lint+docker-build (PR) | 30 min |
| helm-charts helm-lint | 10 min |
| DockerHub token + GitHub secrets | 10 min |
| shop-operator docker-publish | 20 min |
| shophub docker-publish | 15 min (template iz shop-operator) |
| helm-charts helm-publish | 30 min |
| Tag v0.1.0 + verify image pulled | 20 min |
| **Faza A+B+C ukupno** | **~2h 15min** |

Posle ovog smo **gotovi sa eliminacionim CI/CD zahtevom (spec 5.2)** i možemo da se vratimo na Shop backend / observability / web3.

---

## 9. Verifikacija da CI/CD radi

Posle svake faze treba da vidimo:
- ✅ Status checks na PR-u (zeleno/crveno)
- ✅ Image na DockerHub-u (https://hub.docker.com/u/shophub-devoops)
- ✅ Helm chart na OCI registry (verify sa `helm pull oci://docker.io/shophub-devoops/shop-operator --version 0.1.0`)
- ✅ `helm install` chart-a koristi image koji workflow gura (bez `imagePullBackOff`)
- ✅ Operator radi end-to-end iz in-cluster deploya (ne samo `make run` lokalno)

**Demo na odbrani:** asistent traži da `make deploy` (ili helm install) radi end-to-end. Ovo je infrastruktura koja to omogućava.
