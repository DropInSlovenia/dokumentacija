# Poročilo P2 — Docker + Azure VM

## DropInSlovenia

**Skupina:** Matija Dukarić (vodja), Maj Donko, Luka Manfreda  
**GitHub:** https://github.com/DropInSlovenia

---

## Vodenje sprinta

Vsi taski znotraj sprinta:

![alt text](slike/sprint.png)

---

## 0. Struktura projekta in repozitoriji

Naš projekt DropInSlovenia je razdeljen v **tri ločene GitHub repozitorije**, vsak za svoj servis:

| Repozitorij                            | Tehnologija       | Namen                          |
| -------------------------------------- | ----------------- | ------------------------------ |
| `github.com/DropInSlovenia/webApp`     | Next.js (React)   | Uporabniški vmesnik            |
| `github.com/DropInSlovenia/backend`    | Node.js / Express | REST API, MongoDB komunikacija |
| `github.com/DropInSlovenia/desktopApp` | Kotlin / Ktor     | Scraping in zunanji podatki    |

**Kako servisi komunicirajo med seboj:**

```
Uporabnik (brskalnik)
       │
       ▼
   Frontend :3001   ──── API klici ────▶   Backend :3000
                                                │
                                                │ HTTP klic
                                                ▼
                                        Kotlin server :8080
                                                │
                                                │ scraping
                                                ▼
                                          Zunanji viri
                                        (Google Places API ipd.)

Backend :3000  ──── MongoDB URI ────▶   MongoDB Atlas (oblak)
```

**Zakaj ločeni repozitoriji?** Vsak servis ima svojo vlogo, svojo tehnologijo in svoj deployment cikel. Recimo backend in kotlin streznik delujeta kot celota in sta neodvisna of fontenda ali namize aplikacije, saj gre za REST APi.

---

## 0.1 MongoDB Atlas — nastavitev oblačne baze

_Avtor: Matija Dukarić_

Za našo aplikacijo uporabljamo MongoDB Atlas, kar pomeni, da baza podatkov ni nameščena lokalno, ampak deluje v oblaku. To je prednost, ker lahko do baze dostopata tako lokalni razvoj kot tudi Azure VM, brez dodatnih namestitev MongoDB na posameznih računalnikih.

Najprej sem na strani MongoDB Atlas ustvaril brezplačen račun in projekt z imenom DropInSlovenia. Nato sem ustvaril M0 free cluster v regiji Europe West, kar je dovolj za razvoj in manjše projekte.

Za dostop do baze sem ustvaril uporabnika dropinslovenia-user z varnim geslom in mu dodelil pravice za branje in pisanje (ali admin dostop). Da omogočim povezavo iz različnih naprav, sem dovolil dostop iz vseh IP naslovov (0.0.0.0/0), kar je primerno za razvojno okolje.

Po tem sem iz Atlas konzole pridobil connection string, ga prilagodil (vnesel geslo) in shranil v .env datoteko pod MONGODB_URI, ki ni del GIT repozitorija zaradi varnosti.

Prikaz nastavljenega clusterja:

![MongoDB Atlas dashboard z ustvarjenim cluster-jem](slike/mongoAtlas.png)

Prikaz baze v uporabi:

![MongoDB Atlas baza v uporabi](slike/baza.png)

Potrdilo delovanja MongoDb Atlas baze:

![MongoDB v delovanju](slike/delovanjeBaze.png)

Task v jiri:

![alt text](slike/taskMongo.png)

---

## 1. Lokalna namestitev Dockerja

### 1.0 Predpogoj — namestitev Dockerja lokalno

_Avtor: Vsi_

Potrdilo delujocega dockerja:

![alt text](slike/dockerPotrdilo.png)

---

### 1.1 Next.js config — predpogoj za Docker

_Avtor: Maj Donko_

Preden pišemo Dockerfile za frontend, moramo v `frontend/next.config.ts` dodati eno obvezno vrstico:

```typescript
// frontend/next.config.ts
const nextConfig: NextConfig = {
  output: "standalone",
  async rewrites() {
    const apiUrl =
      process.env.API_URL ??
      process.env.NEXT_PUBLIC_API_URL ??
      "http://localhost:3000";

    return [
      {
        source: "/api/:path*",
        destination: `${apiUrl}/api/:path*`,
      },
    ];
  },
};

export default nextConfig;
```

**Zakaj `output: standalone`?** Brez tega Next.js v Docker container skopira celotno `node_modules` mapo (pogosto večja od 500 MB). Z `standalone` outputom Next.js zgradi minimalen bundle (~10–30 MB) z vsem kar potrebuje za zagon — brez razvojnih odvisnosti. Container je manjši, hitrejši za prenos in hitrejši za zagon.

---

### 1.2 Dockerfile — Frontend (Next.js)

_Avtor: Maj Donko_

**Kaj je Dockerfile?** Dockerfile je tekstovna datoteka z navodili za gradnjo Docker slike. Vsaka vrstica je en korak. Docker izvede korake od zgoraj navzdol in shrani rezultat kot sliko (image), ki jo nato zaženemo kot container.

**Multi-stage build** je tehnika kjer Docker zgradi sliko v več fazah — vsaka faza ima svojo `FROM` direktivo. V **builder** fazi imamo vsa razvojna orodja in izvajamo `npm run build`. V **runner** fazi pa vzamemo samo tisto kar je potrebno za zagon. Iz builder faze v runner fazo prenesemo samo rezultat builda — brez `node_modules`, brez izvorne kode. Rezultat je majhen produkcijski container.

`NEXT_PUBLIC_API_URL` se nastavi med buildom kot build argument (`ARG`). To je URL do Node.js backend API-ja. Ko gradimo za Azure VM, ga nastavimo na javni IP VM-ja. Next.js ta URL **vtisne v JavaScript bundle med kompilacijo** — zato ga moramo podati med `docker build`, ne ob zagonu containerja.

```dockerfile
# FAZA 1: Build
FROM node:20-alpine AS builder

WORKDIR /app

# Build argument — API URL se nastavi med docker build
# Privzeta vrednost je za lokalni razvoj
ARG NEXT_PUBLIC_API_URL=http://localhost:3000/api
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL

# Docker cache optimizacija: najprej samo package.json
# Če se package.json ne spremeni, Docker preskoči npm ci pri naslednjem buildu
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# FAZA 2: Production runner (brez razvojnih orodij)
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=3001
ENV HOSTNAME="0.0.0.0"

# Kopiramo samo rezultat builda iz prve faze
# --from=builder pomeni: vzemi iz faze z imenom "builder"
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3001

CMD ["node", "server.js"]
```

**Test lokalno:**

```bash
cd frontend
docker build -t dropinslovenia/frontend:test .
docker run -p 3001:3001 dropinslovenia/frontend:test
# Odpri brskalnik: http://localhost:3001
```

**Razlaga `docker run` zastavic:**

- `-p 3001:3001` — poveži port 3001 na tvojem računalniku s portom 3001 v containerju (format: `HOST:CONTAINER`)
- `-t dropinslovenia/frontend:test` — poimenuj sliko

**Potek gradnje:**

📸 _Slika: `docker build` uspešno zaključen za frontend_
<img width="1181" height="516" alt="image" src="https://github.com/user-attachments/assets/b3fbd0ca-ad26-4d33-81bd-02c0ce4833f1" />

📸 _Slika: Frontend dostopen na http://localhost:3001_
<img width="2524" height="1291" alt="image" src="https://github.com/user-attachments/assets/c8860500-5cef-4a9d-9f73-aa2a39dcf418" />

---

### 1.3 Dockerfile — Backend (Node.js/Express)

_Luka Manfreda_

Backend komunicira z dvema zunanjima sistemoma:

- **MongoDB Atlas** prek `MONGODB_URI` connection stringa (oblačna baza)
- **Kotlin strežnikom** prek `KOTLIN_SERVER_URL` — to je interni Docker URL (`http://kotlin-server:8080`) ki deluje samo znotraj Docker omrežja

Ti podatki se **ne shranijo v Docker sliko**, podamo jih kot environment spremenljivke ob zagonu prek `.env` datoteke.

```dockerfile
FROM node:22.13.1-alpine

WORKDIR /app

# kopira package file
COPY package*.json ./

# Installa samo production dependencies
RUN npm ci --omit=dev

# kopira source kodo
COPY . .

EXPOSE 3000

CMD ["npm", "start"]
```

- `FROM node:22.13.1-alpine` -> uporabi Node.js sliko
- `WORKDIR /app` -> nastavi delovni direktorij v kontejnerju na /app. Vsi naslednji ukazi se izvajajo v tej mapi.
- `COPY package\*.json ./` -> kopira package.json in package-lock.json
- `RUN npm ci --omit=dev` -> namesti samo produkcijske odvisnosti
- `COPY . .` -> kopira celotno izvorno kodo v kontejner
- `EXPOSE 3000` -> Pove, da aplikacija uporablja port 3000.
- `CMD ["npm", "start"]` -> Privzeti ukaz ob zagonu kontejnerja: zazene npm start
  **Opomba:** Če vaš backend vstopno točko imenuje drugače (npr. `index.js` ali `app.js`), prilagodi zadnjo vrstico CMD ustrezno.

**Lokalno se to testira tako**

```bash
cd backend
docker build -t dropinslovenia/backend:test .
docker run -p 3000:3000 \
  -e MONGODB_URI="mongodb+srv://dropinslovenia-user:GESLO@cluster0.xxx.mongodb.net/dropinslovenia" \
  -e JWT_SECRET="test_secret_za_lokalni_test" \
  -e KOTLIN_SERVER_URL="http://host.docker.internal:8080" \
  dropinslovenia/backend:test
```

Backend container in uspesen API klic:
![Backend container](slike/BackendDockerContainer.png)
![uspesen API klic](slike/BackendApiCall.png)

---

### 1.4 Dockerfile, Kotlin Ktor strežnik

_Luka Manfreda_

```dockerfile
# build stage
FROM gradle:8.5-jdk21 AS builder

WORKDIR /app

COPY build.gradle.kts settings.gradle.kts gradlew ./
COPY gradle ./gradle
COPY composeApp ./composeApp

RUN chmod +x ./gradlew

RUN ./gradlew :composeApp:packageUberJarForCurrentOS --no-daemon

# runtime stage
FROM eclipse-temurin:21-jre-alpine

# Namesti Chromium
RUN apk add --no-cache \
    chromium \
    chromium-chromedriver \
    nss \
    freetype \
    harfbuzz \
    ca-certificates \
    ttf-freefont

WORKDIR /app

COPY --from=builder /app/composeApp/build/compose/jars/*.jar app.jar

ENV JAVA_OPTS="-Xmx256m"
ENV CHROME_BIN=/usr/bin/chromium-browser

EXPOSE 8080

CMD ["sh", "-c", "java -cp app.jar org.dropinslovenia.server.ApplicationKt $JAVA_OPTS"]
```

- `FROM gradle:8.5-jdk21 AS builder` -> uporabi Gradle in jdk 21, ta faza je samo za build
- `WORKDIR /app` -> nastavi delovnik direktorij na /app
- `COPY build.gradle.kts settings.gradle.kts gradlew ./ COPY gradle ./gradle COPY composeApp ./composeApp` -> kopira Gradle konfiguracijo in projekt
- `RUN chmod +x ./gradlew` -> doda izvrsilne pracice za Gradle wrapper
- `RUN ./gradlew :composeApp:packageUberJarForCurrentOS --no-daemon` -> zgradi JAR, vsebuje vse dependencies, --no-daemon = brez Gradle background procesa
- `FROM eclipse-temurin:21-jre-alpine` -> uporabi Eclipse Temurin JRE 21, to je runtime java slika

````bash
RUN apk add --no-cache \
    chromium \
    chromium-chromedriver \
    nss \
    freetype \
    harfbuzz \
    ca-certificates \
    ttf-freefont``` ->
````

- Namesti Chromium browser in potrebne knjižnice
- `WORKDIR /app` -> nastavi runtime delovni direktorij
- `COPY --from=builder /app/composeApp/build/compose/jars/*.jar app.jar` -> kopira zgrajen .jar iz build faze v runtime sliki
- `ENV JAVA_OPTS="-Xmx256m"` -> Nastavi Java memory limit: najvec 256 MB RAM za JVM
- `ENV CHROME_BIN=/usr/bin/chromium-browser` -> nastavi pot do Chromium browserja za aplikacijo.
- `EXPOSE 8080` -> Aplikacija poslusa na portu 8080
- `CMD ["sh", "-c", "java -cp app.jar org.dropinslovenia.server.ApplicationKt $JAVA_OPTS"]` -> zazene java aplikacijo, app.jar vsebuje celoten backend, ApplicationKt je entry potin kotlin server.

**Build testiranje:**

```bash
cd kotlin-server
docker build -t dropinslovenia/kotlin-server:test .
docker run -p 8080:8080 \
  -e GOOGLE_PLACES_API_KEY="ključ" \
  dropinslovenia/kotlin-server:test

curl http://localhost:8080/events/maribor
```

Kotlin server container:
![Kotlin server container](slike/ktor-serverContainer.png)

---

### 1.5 docker-compose.yml

_Avtor: Matija Dukarić_

V projektu uporabljamo kombiniran pristop k .env datotekam. Pri zagonu celotnega sistema preko Docker Compose uporabljamo eno centralno .env datoteko, ki zagotavlja enotne nastavitve za vse servise (backend, frontend in kotlin-server).

Hkrati lahko vsak servis deluje tudi samostojno, zato ima lahko svoj lokalni .env, kar omogoča neodvisen razvoj in testiranje posameznih komponent.

Tak pristop omogoča večjo fleksibilnost, lažji razvoj ter dosledno upravljanje občutljivih podatkov, ki se nikoli ne shranjujejo v Git.

```yaml
services:
  # Kotlin service for desktop/microservice logic
  kotlin-server:
    build:
      # Path to Kotlin project
      context: ./PrincipiProjekt/desktopApp
      dockerfile: Dockerfile
    container_name: kotlin-server

    # Expose Kotlin API on port 8080
    ports:
      - "8080:8080"

    # Restart container automatically if it crashes
    restart: unless-stopped

  # Main backend service (Node.js/Express)
  backend:
    build:
      # Path to backend source code
      context: ./backendProjekt
      dockerfile: Dockerfile
    container_name: backend

    # Backend API available on localhost:3000
    ports:
      - "3000:3000"

    # Load environment variables from .env file
    env_file: .env

    environment:
      # MongoDB connection string
      - MONGODB_URI=${MONGODB_URI}

      # Secret key for JWT authentication
      - JWT_SECRET=${JWT_SECRET}

      # Internal Docker network URL for Kotlin service
      - KOTLIN_SERVICE_URL=http://kotlin-server:8080

    # Start Kotlin service before backend
    depends_on:
      - kotlin-server

    # Keep backend running unless manually stopped
    restart: unless-stopped

  # Frontend service (Next.js application)
  frontend:
    build:
      # Path to frontend project
      context: ./webApp

      # Dockerfile location inside frontend folder
      dockerfile: frontend/Dockerfile

      args:
        # Public API URL used during frontend build
        - NEXT_PUBLIC_API_URL=http://localhost:3000

    container_name: frontend

    # Frontend accessible on localhost:3001
    ports:
      - "3001:3001"

    environment:
      # Internal backend URL inside Docker network
      - API_URL=http://backend:3000

    # Backend must start before frontend
    depends_on:
      - backend

    # Restart frontend automatically if needed
    restart: unless-stopped
```

`.env` datoteka (v istem folderju kot `docker-compose.yml`, **nikoli v git**):

```env
# FRONTEND (Next.js)

NEXT_PUBLIC_API_URL=http://localhost:3000
NEXT_PUBLIC_WS_URL=ws://localhost:3000


#  BACKEND (Node API)

PORT=3000

# MongoDB Atlas
MONGODB_URI=mongodb+srv://<username>:<password>@<cluster>.mongodb.net/<database>?retryWrites=true&w=majority

MONGO_USER=
MONGO_KEY=

# JWT avtentikacija
JWT_SECRET=replace-with-strong-secret
JWT_EXPIRES_IN=1d

# povezava na Kotlin servis ( v Dockerju NE localhost)
KOTLIN_SERVICE_URL=http://localhost:8080

# CORS pravila
CORS_ORIGIN=http://localhost:3001

# rate limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX=100

# dodatni tokeni
ACCESS_TOKEN_SECRET=
REFRESH_TOKEN_SECRET=


# KOTLIN SERVER

GOOGLE_PLACES_KEY=

#  v Dockerju mora biti internal URL
API_BASE_URL=http://kotlin-server:8080/

AUTH_EMAIL=
AUTH_PASSWORD=
```

Celoten sistem zaženemo z uporabo Docker Compose, ki avtomatsko zgradi in poveže vse tri servise (frontend, backend in kotlin-server). Z ukazom docker compose up --build se aplikacije zgradijo iz Dockerfile-ov in zaženejo kot ločeni containerji v skupnem Docker omrežju.

Za lažje upravljanje lahko sistem zaženemo tudi v detached načinu (-d), kar omogoča delovanje v ozadju brez blokiranja terminala. Stanje containerjev preverimo z docker ps, loge posameznih servisov pa spremljamo z docker compose logs.

Zaustavitev celotnega sistema izvedemo z ukazom docker compose down, ki ustavi in odstrani vse povezane containerje.

**Zagon vsega skupaj:**

```bash
# Zgradi vse slike in zaženi v ozadju
docker compose up --build

# Za zagon v ozadju (detached mode):
docker compose up --build -d

# Preveri status
docker ps

# Ustavi vse
docker compose down
```

Prikaz repozitorija, kjer so not vidni kotlin streznik, fronent, backend, .env, in yaml:

![alt text](slike/repoDokaz.png)

Uporaba ukaza `docker compose up --build`:

![alt text](slike/dokazUk.png)

Uporaba ukaza `docker ps`:

![alt text](slike/dockerPs.png)

Delujoc backend (REST API):

![alt text](slike/backendRest.png)

Delujoc fronend (ni še popolnoma končan):

![alt text](slike/frontendDokaz.png)

Slika jira taska:

## ![alt text](slike/yamlTask.png)

## 2. Dostop do storitve Azure

_Avtor: Matija Dukarić_

Na računu vodje skupine smo uspešno aktivirali Azure for Students naročnino. Postopek je potekal preko Microsoft Azure portala, kjer smo se prijavili s študentskim e-mail naslovom in opravili verifikacijo statusa študenta. Po uspešni aktivaciji smo pridobili dostop do brezplačnih Azure storitev, vključno z dobroimetjem in brezplačnimi urami virtualnega strežnika, brez vnosa kreditne kartice.

Jira task:

![alt text](slike/vmJira.png)

---

## 3. Vpostavitev virtualne naprave

_Avtor: Matija Dukarić, SSH ključi: Maj Donko, Luka Manfreda_

### 3.1 Parametri VM

| Parameter      | Vrednost                             |
| -------------- | ------------------------------------ |
| Subscription   | Azure for Students                   |
| Resource group | DropInSlovenia_group_05191709        |
| VM name        | dropinslovenia-vm                    |
| Region         | Austria East (Zone 2)                |
| Image          | Ubuntu Server 24.04 LTS              |
| Size           | Standard B2ts v2 (2 vCPU, 1 GiB RAM) |
| Authentication | Password                             |
| Public IP      | DA                                   |

![alt text](slike/parametri.png)

![alt text](slike/vmKorak.png)

### Težave:

Pri ustvarjanju VM-ja smo naleteli na omejitve, zato nismo mogli popolnoma slediti zahtevam. Region West Europe ni bil na voljo za naše naročnino Azure for Students, zato smo izbrali Austria East (Zone 2), ki je bila najbližja razpoložljiva možnost.
Posledično tudi velikosti Standard B1s ni bilo mogoče izbrati, saj ta velikost v regiji Austria East ni bila na voljo. Izbrali smo Standard B2ts v2 (2 vCPU, 1 GiB RAM), ki je bila najbližja ustrezna alternativa s podobno količino pomnilnika.
Vse ostale nastavitve — Ubuntu Server 24.04 LTS, Azure for Students naročnina, avtentikacija z geslom in odprti SSH vrata — smo ohranili enake, kot je bilo zahtevano.

Jira task:

![alt text](slike/vmJira2.png)

---

### 3.2 SSH dostop vseh članov

---

### Kaj je SSH in zakaj ključi?

**SSH** (Secure Shell) je protokol za varno oddaljeno upravljanje strežnikov prek ukazne vrstice. Z njim se povežemo na Azure VM kot da bi sedeli pred njim — iz kateregakoli računalnika, kjerkoli na svetu.

**Zakaj ključi namesto gesla?**

SSH podpira dve metodi prijave:

- **Geslo** — preprosto, ampak ranljivo. Napadalci lahko avtomatsko preizkušajo tisoče gesel na sekundo (brute force napad).
- **Par ključev** — varnejše. Temelji na matematičnem problemu ki ga z današnjo računalniško močjo ni mogoče rešiti v razumnem času.
  Par ključev sestavljata:
- **Zasebni ključ** (`id_ed25519`) — ostane **samo na tvojem računalniku**, nikoli ga ne deli z nikomer, nikamor ne nalagaj
- **Javni ključ** (`id_ed25519.pub`) — tega daš na strežnik; iz njega ni mogoče izpeljati zasebnega ključa
  Ob prijavi strežnik preveri: _"Imaš zasebni ključ ki ustreza javnemu ključu ki sem ga shranil?"_ Če da — dostop dovoljen, brez gesla.

**`authorized_keys`** je datoteka na strežniku ki vsebuje seznam vseh dovoljenih javnih ključev. Vsaka vrstica je en ključ — po en na člana ekipe.

---

### Dostop preko gesla:

Do VM smo dostopali preko uporabnišgeka imena in gesla, ki smo jih nastavili ob nastavitvi VM:

![alt text](slike/geslo.png)


## SSH ključi

### Korak 1 — Vsak član je generiral SSH ključ na svojem računalniku

#### Windows

Odpremo **PowerShell**:

```bash
ssh-keygen -t ed25519 -C "ime.priimek@student.um.si"
```

- `-t ed25519` — algoritem za generiranje ključa (ed25519 je moderen, hiter in varen)
- `-C "..."` — komentar/oznaka ključa, ponavadi email; pomaga pri razlikovanju ključev

#### macOS / Linux

Odpremo terminal in vpišemo isti ukaz:

```bash
ssh-keygen -t ed25519 -C "ime.priimek@student.um.si"
```

#### Prikažemo javni ključ

Ko je ključ generiran, prikažemo javni ključ:

**Windows (PowerShell):**

```bash
type C:\Users\TvojeIme\.ssh\id_ed25519.pub
```

**macOS / Linux:**

```bash
cat ~/.ssh/id_ed25519.pub
```

Izpis izgleda takole (primer):

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAbCdEfGhIjK... ime.priimek@student.um.si
```

---

### Korak 2 — Dodajanje javnih ključev od vseh 3 članov na VM:

#### 2.1 — Prva prijava na VM z geslom

Ker ključev še nismo dodali, se prvič prijavimo z geslom ki smo ga nastavili ob ustvarjanju VM-ja:

![alt text](slike/prijavaGeslo.png)

Ob prvič vpraša:

```
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Vpišemo `yes` in Enter. Nato vpišemo geslo VM-ja.

#### 2.2 — Ustvarimo `.ssh` mapo z ustreznimi pravicami

![alt text](slike/chmod.png)

`mkdir -p` ustvari mapo `~/.ssh` (če že obstaja, ne vrne napake).
`chmod 700` pomeni: **samo lastnik** lahko bere, piše in odpira mapo. SSH bo zavrnil prijavo s ključem če ima mapa preveč odprte pravice — to je varnostna zahteva SSHja.

#### 2.3 — Dodamo javne ključe vseh članov

```bash
nano ~/.ssh/authorized_keys
```

`nano` je preprost urejevalnik besedil v terminalu. Odpre se prazna datoteka. Vanjo prilepimo javne ključe vseh treh članov — vsak v svojo vrstico:

![alt text](slike/gesla.png)

#### 2.4 — Nastavimo pravice na `authorized_keys`

```bash
chmod 600 ~/.ssh/authorized_keys
```

`chmod 600` pomeni: **samo lastnik** lahko bere in piše datoteko. SSH bo **zavrnil prijavo s ključem** če ima datoteka preveč odprte pravice (npr. 644 ali 777) — to je stroga varnostna zahteva.

### Korak 3 — Vsak član testira prijavo brez gesla

Ko so vsi kljuci dodani, vsak član na svojem računalniku preizkusi prijavo.
Tokrat **ne vpraša za geslo** — prijava mora uspeti samodejno s ključem. Uspešna prijava izgleda tako:

![alt text](slike/brezPrijave.png)

Jira task:

![alt text](slike/jirat2.png)

---

## 4. Odgovori na Azure vprašanja

### 4.1 Kje in kako omogočite "port forwarding"?

_Avtor: Maj Donko_

**Kaj je NSG?** Network Security Group (NSG) je virtualni požarni zid ki nadzira omrežni promet do in od Azure virov. Vsak VM v Azure avtomatično dobi priložen NSG.

**Kako deluje?** NSG vsebuje seznam **pravil** (rules). Vsako pravilo določa: iz katerega vira (Source), na kateri port, kateri protokol in ali je promet dovoljen (Allow) ali zavrnjen (Deny). Pravila so urejena po prioriteti — nižja številka = višja prioriteta.

**Privzeto stanje:** Ko ustvarimo VM, so vsa vhodna vrata zaprta razen porta 22 (SSH). To je varnostno načelo najmanjših privilegijev — odpiramo samo tisto kar nujno potrebujemo.

**Port forwarding** na Azure pomeni dodajanje Inbound security rules v NSG za porte naše aplikacije.

**Pot v portalu:** VM → **Networking** → **Network settings** → **Create port rule → Inbound port rule**

Za vsak port naše aplikacije dodamo pravilo:

| Port | Ime pravila    | Namen                    |
| ---- | -------------- | ------------------------ |
| 3000 | allow-backend  | Node.js/Express REST API |
| 3001 | allow-frontend | Next.js frontend         |
| 8080 | allow-kotlin   | Kotlin Ktor server       |
| 9000 | allow-webhook  | Webhook strežnik (P3)    |

Za vsako pravilo nastavimo:

- **Source:** Any
- **Source port ranges:** `*`
- **Destination:** Any
- **Destination port ranges:** npr. `3001`
- **Protocol:** TCP
- **Action:** Allow
- **Priority:** npr. 1010, 1020, 1030, 1040 (vsak naslednji +10)

> **Zakaj je privzeto vse zaprto?** Vsak odprt port je potencialna napadalna površina. Napadalec ki skenira internet bi na odprtem portu 3000 videl naš Node.js API in ga poskušal izkoristiti. Odpiramo samo kar nujno rabimo.

## <img width="1326" height="873" alt="image" src="https://github.com/user-attachments/assets/a368408c-121b-4252-a631-b0b2ff0554b9" />

### 4.2 Kakšen tip diska je bil dodan navidezni napravi in kakšna je njegova kapaciteta?

_Avtor: Maj Donko_

**Pot v portalu:** VM -> Settings -> Disks

Naši navidezni napravi je bil samodejno dodan OS disk tipa **Premium SSD LRS** s kapaciteto **30 GB**.

**Razlaga:**

- **Premium SSD** je osnovni SSD za večino trenutnih VM-ov na Azur-u
- **LRS** (Locally Redundant Storage) pomeni da Azure podatke replicira **trikrat znotraj istega podatkovnega centra** v Amsterdamu. Če en fizični disk odpove, se podatki ohranijo. Ne varuje pred izpadom celotnega podatkovnega centra — za to bi potrebovali ZRS ali GRS.
- **30 GB** je privzeta velikost OS diska za Ubuntu VM v Azure.
  <img width="1629" height="71" alt="image" src="https://github.com/user-attachments/assets/b6d7841d-78a8-448d-9eeb-748d24fc5bd1" />

---

### 4.3 Stanje porabe virov

_Luka Manfreda_

**Poraba virov prikazuje:**

- Skupno porabo v evrih razčlenjeno po storitvah (Virtual Machines, Storage, Bandwidth...)
- Graf porabe skozi čas
- Koliko od 100 € dobroimetja smo že porabili
- Napoved porabe do konca meseca

![Poraba virov](slike/porabaVirov.png)

---

## 5. Vpostavitev Dockerja na Azure VM

### 5.1 Namestitev Dockerja in swap

_Avtor: Matija Dukarić_

---

### Korak 1: Prijava na VM prek SSH

To je spet isti postopek kot prej:

![alt text](slike/brezPrijave.png)

### Korak 2: Posodobitev sistema

Ko smo prijavljeni, najprej posodobimo seznam paketov in namestimo najnovejše varnostne popravke. To je dobra praksa pred vsako namestitvijo:
![alt text](slike/posodobitev.png)

- `apt update` — prenese aktualni seznam razpoložljivih paketov
- `apt upgrade -y` — namesti vse posodobitve (`-y` samodejno potrdi vse)

---

### Korak 3: Namestitev Dockerja

Docker nameščamo z uradnim skriptom ki ga pripravi Docker sam. Skript samodejno zazna operacijski sistem, doda Docker repozitorij in namesti Docker Engine.

Prenesemo in namestimo namestitveno skripo:

![alt text](slike/names.png)

#### Preverimo namestitev

![alt text](slike/namestitev.png)

---

### Korak 4: Dodajanje swap datoteke

**Zakaj swap?** Naš Azure Standard B2ts v2 VM ima samo **1 GiB RAMa**. Naša aplikacija teče v treh containerjih skupaj:

| Komponenta       | Poraba RAMa                    |
| ---------------- | ------------------------------ |
| Next.js frontend | ~150–200 MB                    |
| Node.js backend  | ~100 MB                        |
| Kotlin JVM       | ~256 MB (omejeno z `-Xmx256m`) |
| Ubuntu OS        | ~200 MB                        |
| **Skupaj**       | **~700–800 MB**                |

Ko Docker gradi slike ob zagonu, poraba začasno naraste nad 1 GB — VM bi "zmrznil" ali se sesul. Rešitev je **swap datoteka** — rezerviran del diska, ki ga operacijski sistem začasno uporablja kot razširitev RAMa. Je počasnejši od pravega RAMa, a za naš primer povsem zadostuje.

> **Opomba:** Originalna navodila predvidevajo Standard B1s (1 vCPU, 1 GB RAM), mi pa smo zaradi problemov (ki smo jih opisali pri namestitvi) naročnine Azure for Students izbrali Standard B2ts v2 (2 vCPU, 1 GiB RAM). Količina RAMa je enaka, zato so vsi koraki — vključno z dodajanjem swapa — identični.

# Ustvarimo 2 GB swap datoteko, nastavimo pravice, inicializiramo in aktiviramo jo ter jo dodamo v /etc/fstab za trajno delovanje

![alt text](slike/swap.png)

Jira task:
![alt text](slike/jirat.png)

---

### 5.2 Prenos kode in zagon

_Luka Manfreda_

Docker containerji na VM:
![alt text](slike/dockerVMContainers.png)

Aplikacija je ob zagonu na VM dostopna na http://68.210.138.63:3001:
![alt text](slike/aplikacijaDostopna.png)

Api klic: http://68.210.138.63:8080/events/maribor vrne evente:
![alt text](slike/dokazApiMariborEvents.png)

---

