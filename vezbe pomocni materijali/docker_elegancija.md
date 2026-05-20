# Docker - Najbolje Prakse

Referentni dokument sa najboljim praksama za rad sa Dockerom. Koristiti kao smernice prilikom izrade projekata sa Docker Build, Docker Compose i Docker Swarm.

---

## 1. Docker Build

### 1.1. Bezbednosne preporuke

#### Multi-stage build
- **Obavezno koristiti multi-stage build** - drastično smanjuje veličinu slike i povećava bezbednost.
- Finalna slika treba da sadrži **samo izvršni deo aplikacije**, bez source code-a i build alata. Time se sprečava da neko unutar kontejnera izmeni source code, build-uje malicioznu verziju i pokrene je.
- **Kompajlirani jezici (Go, Rust, C++) su u prednosti** jer se pokreću kao binary fajlovi, dok interpretirani jezici (JavaScript, Python) zahtevaju source code za pokretanje.

#### Korisnici i permisije
- **Nikada ne pokretati aplikaciju kao root.** Kreirati novog korisnika u kontejneru sa permisijama `read + execute` (bez `write` permisije).
- Owner aplikacije podesiti kao root, a aplikaciju izvršavati pod novim korisnikom.
- **Kompajlirani jezici:** izmeniti permisije izvršne datoteke.
- **Interpretirani jezici:** izmeniti permisije svih source code fajlova i njihovih zavisnosti.

#### Slim verzije slika
- Koristiti **slim verzije** Docker slika - manje zavisnosti znači manje površine za ranjivosti.
- Primer: prelazak sa `node:8` (811 ranjivosti) na `node:10.23.1-buster-slim` (55 ranjivosti).

#### Ostale bezbednosne prakse
- **Koristiti `hadolint`** - alat za uočavanje propusta u Dockerfile-u.
- **Uvek koristiti exec varijantu** za `CMD` i `ENTRYPOINT` (npr. `CMD ["python", "app.py"]`), ne shell varijantu jer ne prosleđuje signale podprocesima.
- **Koristiti clean build context** - poseban folder sa neophodnim fajlovima:
  ```bash
  docker buildx /files
  ```
  Bolje keširanje, ignoriše potrebu za `.dockerignore` fajlom.

---

### 1.2. Dockerfile - Korisne tehnike

#### Alias preko ENV
```dockerfile
FROM python
ENV update='apt-get update -qq'
ENV install='apt-get install -qq'
RUN $update && $install apt-utils \
    curl \
    gnupg
```
> Napomena: `alias ll=ls -alh` ne radi u Dockerfile-u - samo u interaktivnom režimu (`docker run -it` ili `docker exec`).

#### Pipefail kod RUN sa pipe-om
Kada se koristi pipe u `RUN` komandi, **obavezno** podesiti shell da prekine izvršavanje ako prva komanda ne uspe:

```dockerfile
FROM python
SHELL ["/bin/bash", "-o", "pipefail", "-c"]
# Za alpine: SHELL ["/bin/ash", "-o", "pipefail", "-c"]
RUN curl -L ${SRC_URL} | tar -xz
```

#### ONBUILD - Generičke slike
Koristi se za definisanje generičke slike koja će biti nasleđena. Ide odlično u kombinaciji sa multi-stage build-om.

```dockerfile
FROM python:3.9-alpine3.13
LABEL maintainer "danijelradakovic@uns.ac.rs"
ONBUILD ADD requirements.txt /app
ONBUILD RUN pip install -r /app/requirements.txt
ONBUILD COPY . /app
WORKDIR /app
EXPOSE 5000
ENTRYPOINT ["python"]
CMD ["application.py"]
```
Aplikacija samo nasledi: `FROM python-prod`. Slika se može pregaziti po potrebi.

---

### 1.3. Concurrent Builds (BuildKit)

BuildKit konkurentno izvršava stage-ove. Više `--from` u istom stage-u izvršava se paralelno.

**Preporuke:**
- `--from` naredbe stavljati **jedne ispod drugih**.
- Sve što treba kopirati iz prethodnog stage-a stavljati u `/out` folder pa kopirati ceo folder, umesto pojedinačnih fajlova.

```dockerfile
FROM maven:3.6-jdk-8-alpine AS builder
...
FROM alpine:latest AS assets
RUN echo "Hello World" > /out/assets.html

FROM openjdk:8-jre-alpine AS release
COPY --from=builder /app/target/app.jar .
COPY --from=assets /out /assets
CMD java -jar /app/app.jar
```

---

### 1.4. CI/CD Stage-ovi

Poželjno je imati posebne stage-ove za različite svrhe i različite OS-ove.

#### Parametrizovan flavor
```dockerfile
ARG flavor=alpine
FROM maven:3.6-jdk-8-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn -e -B dependency:resolve
COPY src ./src
RUN mvn -e -B package -DskipTests

FROM openjdk:8-jre-${flavor} AS release
COPY --from=builder /app/target/app.jar .
CMD java -jar /app/app.jar
```

#### Test Stage (Unit + Integration)
```dockerfile
FROM builder AS unit-test
RUN mvn -e -B test

FROM release AS integration-test
RUN apk add --no-cache curl
RUN ./test/run.sh
```

#### Lint Stage
```dockerfile
FROM openjdk:8-jre-alpine AS lint
RUN wget https://github.com/checkstyle/checkstyle/releases/download/checkstyle-8.15/checkstyle-8.15-all.jar
COPY checks.xml .
COPY src /src
RUN java -jar checkstyle-8.15-all.jar -c checks.xml /src
```

---

### 1.5. RUN --mount Tipovi

#### Bind mount (umesto COPY)
```dockerfile
FROM maven:3.6-jdk-8-alpine AS builder
WORKDIR /app
RUN --mount=source=pom.xml,target=. mvn -e -B dependency:resolve
RUN --mount=source=./src mvn -e -B package -DskipTests
```

#### Cache mount (application cache)
Još bolje - koristi cache za zavisnosti:

```dockerfile
RUN --mount=target=. \
    --mount=type=cache,target=/root/.m2 \
    mvn package -DoutputDirectory=/
```

**Cache folderi po alatima:**
| Alat        | Cache folder           |
|-------------|------------------------|
| apt         | `/var/lib/apt/lists`   |
| go          | `~/.cache/go-build`    |
| go-modules  | `$GOPATH/pkg/mod`      |
| npm         | `~/.npm`               |
| pip         | `~/.pip`               |

#### Secret mount - Rukovanje secret-ima
**NIKADA** ne prosleđivati secret-e kao build argumente (`ARG`) - vidljivi su preko `docker image inspect`. **NIKADA** ih ne kopirati pa brisati - `skopeo` alat može pristupiti starim layer-ima.

**Rešenje:** `--mount=type=secret`

```dockerfile
RUN --mount=type=secret,id=db-psw,dst=/.secrets/db-psw,required \
    --mount=type=secret,id=db-user,dst=/.secrets/db-user,required \
    --mount=type=secret,id=db-schema,dst=/.secrets/db-schema,required \
    export DATABASE_PASSWORD=$(cat /.secrets/db-psw); \
    export DATABASE_USERNAME=$(cat /.secrets/db-user); \
    export DATABASE_SCHEMA=$(cat /.secrets/db-schema); \
    # ... koristiti secret-e ovde ...
```

#### SSH mount - Privatni Git repo
```dockerfile
FROM alpine
RUN apk add --no-cache openssh-client
RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan github.com >> ~/.ssh/known_hosts
ARG REPO_REF=main
RUN --mount=type=ssh,required \
    git clone git@github.com:org/repo /work && cd /work \
    && git checkout -b $REPO_REF
```

Korišćenje:
```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
docker build --ssh=default .
```

---

### 1.6. Package Manageri

#### APT (Debian/Ubuntu)
```dockerfile
RUN apt-get update -y && apt-get install -y --no-install-recommends \
    curl=7.64.0-4+deb10u2 \
    tar=1.30+dfsg-6 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```
**Pravila:**
- Koristiti `--no-install-recommends`.
- Obrisati cache: `rm -rf /var/lib/apt/lists/*`.
- **Fiksirati verzije paketa.**

#### Maven
```dockerfile
FROM maven:3.6-jdk-8-alpine
WORKDIR /app
COPY pom.xml .
RUN mvn -e -B dependency:resolve
COPY src ./src
RUN mvn -e -B package
```
Keširanje zavisnosti pomoću `dependency:resolve` plugina pre kopiranja izvornog koda.

---

### 1.7. Optimizacija
- Optimizovati kroz **cache**, ali paziti na garbage collector (cache jede memoriju).
- `RUN --mount=type=cache` koristi bound volume - **odlično za lokalni build, problematično za CI/CD** jer različiti runner-i imaju različite cache-eve.
- Za CI/CD koristiti **remote storage backend** (npr. inline cache).

---

## 2. Docker Compose

### 2.1. Templating sa `docker compose config`

`docker compose config` generiše finalni compose fajl iz template-a i env fajla.

#### Env fajl
```ini
# env.conf.template
STAGE=prod
POSTGRES_VERSION=13
```

#### Template
```yaml
# persistance.yaml
services:
  database:
    image: postgres:${POSTGRES_VERSION:-12}  # default: 12
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/database-password
      POSTGRES_USER_FILE: /run/secrets/database-username
      POSTGRES_DB_FILE: /run/secrets/database-schema
    volumes:
      - database-data:/var/lib/postgresql/data
volumes:
  database-data:
    name: clean_cadet_database_${STAGE:-dev}  # default: dev
```

#### Generisanje finalnog compose fajla
```bash
docker compose --env-file env.template.conf \
               --file persistance.yml \
               config > compose.yml
```

---

## 3. Docker Swarm

### 3.1. Osnove

Docker Swarm formira **kluster (umreženih čvorova)** za deployment servisa.

#### Tipovi čvorova
- **Master** - upravlja klusterom, samo na njemu se izvršavaju Docker komande.
  - Među master čvorovima postoji **lider** koji donosi odluke.
  - Izbor lidera radi se preko **Raft protokola**.
- **Slave** - izvršava aplikacije.

#### Saobraćaj
- **Control plane** - Docker komande, komunikacija između master čvorova.
- **Data plane** - saobraćaj samih aplikacija.

#### Preporuke za master čvorove
- **Uvek neparan broj** master čvorova (zbog segmentacije mreže).
- **Maksimum 7** master čvorova - više od toga otežava sinhronizaciju bez benefita.

### 3.2. Stack

Infrastruktura se definiše u **stack** YAML fajlu (sličan compose fajlu).

```bash
docker stack deploy stack.yml
```

Svaki servis mora imati `deploy` sekciju. Postoje dva tipa deployment-a:
- **replicated** (default) - definiše broj replika koje swarm raspoređuje po klusteru.
- **global** - po jedan kontejner na svakom čvoru klastera.

---

### 3.3. Docker Secrets

Koristi se za čuvanje osetljivih informacija i transfer do servisa.

#### Kreiranje secret-a
```bash
printf "%s" "${DATABASE_PASSWORD}" \
  | docker secret create "clean_cadet_database_password_dev" - > /dev/null
printf "%s" "${DATABASE_USERNAME}" \
  | docker secret create "clean_cadet_database_username_dev" - > /dev/null
printf "%s" "${DATABASE_SCHEMA}" \
  | docker secret create "clean_cadet_database_schema_dev" - > /dev/null
```

#### Korišćenje u stack-u (kao external)
```yaml
services:
  database:
    image: postgres:${POSTGRES_VERSION:-13}
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/database-password
      POSTGRES_USER_FILE: /run/secrets/database-username
      POSTGRES_DB_FILE: /run/secrets/database-schema
    secrets:
      - source: database-password
        target: database-password
      - source: database-username
        target: database-username
      - source: database-schema
        target: database-schema

secrets:
  database-password:
    name: clean_cadet_database_password_dev
    external: true
  database-username:
    name: clean_cadet_database_username_dev
    external: true
  database-schema:
    name: clean_cadet_database_schema_dev
    external: true
```

#### Pravila
- Secret se **ne može obrisati** dok ga koristi neki servis.
- Procedura izmene secret-a:
  1. Obrisati sve servise koji koriste secret.
  2. Obrisati secret.
  3. Kreirati novi secret.
  4. Ponovo kreirati servise.

#### Bolji pristup - verzionisanje secret-a
- Kreirati novi secret sa verzijom u nazivu.
- U stack fajlu izmeniti referencu na novu verziju, swarm će sam uraditi update:
  ```bash
  docker service update --secret-rm source=<old> --secret-add source=<new>
  ```
- Obrisati stari secret.

---

### 3.4. Docker Configs

Bind volume **NE radi** u Swarm-u jer vezuje host file system - nemoguće distribuirati config servisima na drugim čvorovima.

**Loše rešenje:** pakovati config u sliku - naporan rad ako se često menja.

**Dobro rešenje:** **Docker Configs** - distribuiraju konfiguracione fajlove servisima.

#### Definicija u stack-u
```yaml
services:
  gateway:
    image: nginx:${VERSION-16}
    configs:
      - source: nginx.conf
        target: /etc/nginx/nginx.conf
      - source: api_gateway.conf
        target: /etc/nginx/api_gateway.conf

configs:
  nginx.conf:
    name: nginx.conf-prod
    file: ./gateway/files/config/nginx.conf
  api_gateway.conf:
    name: api_gateway.conf-prod
    file: ./gateway/files/config/api_gateway.conf
```

> Fajl mora biti na master čvoru gde se radi deploy. Alternativa je `external: true`.

#### Pravila configs-a
- Config je **immutable** - update se dešava **samo ako se promeni naziv config-a**, ne i sadržaj.
- Config se **ne može obrisati** dok ga neki servis koristi.

#### Najbolji pristup - hash sufiks u nazivu
Dodati hash sadržaja fajla kao sufiks u nazivu - ime se menja kad se promeni sadržaj:

```yaml
configs:
  nginx.conf:
    name: nginx.conf-${nginx_conf_DIGEST}
    file: ./gateway/files/config/nginx.conf
  api_gateway.conf:
    name: api_gateway.conf-${api_gateway_conf_DIGEST}
    file: ./gateway/files/config/api_gateway.conf
```

**Implementacija:**
1. Postaviti DIGEST placeholder-e u stack template-u.
2. Generisati hash na osnovu sadržaja fajla.
3. Skratiti hash tako da ceo naziv config-a bude **maksimum 64 karaktera** (limit Docker-a).
4. Generisati env fajl koji mapira placeholder-e na hash:
   ```ini
   nginx_conf_DIGEST=truncedhashfromnginx
   api_gateway_conf_DIGEST=truncedhashfromapigateway
   ```
5. Kombinovati env fajl i template u finalni stack.

#### Pomoćni kontejner za hash
Postoji gotov kontejner `cleancadet/docker-config-hash`:
```bash
docker create --name config-hash cleancadet/docker-config-hash:latest
docker cp ../ config-hash
docker start config-hash
docker cp config-hash:/tmp/env .
docker rm config-hash

docker compose --env-file env --file app.yml config \
  | docker stack deploy --prune -c - app-prod

rm env
```

#### Configs sa osetljivim podacima
Ako konfiguracioni fajl sadrži osetljive informacije, koristiti opciju `--template-driver` i napisati Go template.

---

## Brzi Checklist za Projekat

### Build
- [ ] Multi-stage build (build stage + release stage).
- [ ] Slim/alpine bazna slika.
- [ ] Non-root korisnik u finalnoj slici.
- [ ] Exec forma za `CMD` i `ENTRYPOINT`.
- [ ] `hadolint` proveru proći.
- [ ] Pipefail shell ako se koristi pipe u `RUN`.
- [ ] Fiksirane verzije paketa u package manager-u.
- [ ] Obrisan apt cache.
- [ ] `--no-install-recommends` za apt.
- [ ] `--mount=type=cache` za zavisnosti.
- [ ] `--mount=type=secret` za secrets - **nikad** preko ARG.
- [ ] `.dockerignore` ili clean build context.
- [ ] Posebni stage-ovi za unit-test, integration-test, lint.

### Compose
- [ ] Templating preko `docker compose config` + env fajl.
- [ ] Default vrednosti za ENV varijable (`${VAR:-default}`).
- [ ] Volumes sa imenima koja sadrže stage (dev/prod).

### Swarm
- [ ] Neparan broj master čvorova (≤ 7).
- [ ] `deploy` sekcija u svakom servisu.
- [ ] Secrets kao `external: true`, nikad inline.
- [ ] Verzionisanje u nazivima secret-a.
- [ ] Configs umesto bind volume za konfiguracione fajlove.
- [ ] Hash sufiks u nazivu config-a (max 64 karaktera ukupno).
- [ ] `--template-driver` za config-e sa osetljivim podacima.
