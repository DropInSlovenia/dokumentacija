# Poročilo P2 — Docker + Azure VM

## DropInSlovenia

**Skupina:** Matija Dukarić (vodja), Maj Donko, Luka Manfreda   
**Skupina:** Matija Dukarić (vodja), Maj Donko, Luka Manfreda  
**Datum oddaje:** \***\*\_\_\_\*\***  
**GitHub:** https://github.com/DropInSlovenia

---

## Razdelitev nalog

| Task    | Opis                              | Assignee                      |
| ------- | --------------------------------- | ----------------------------- |
| TASK-01 | `output: standalone` v Next.js    | Maj Donko                     |
| TASK-02 | Dockerfile — Frontend             | Maj Donko                     |
| TASK-03 | Dockerfile — Backend              | Luka Manfreda                 |
| TASK-04 | Dockerfile — Kotlin server        | Luka Manfreda                 |
| TASK-05 | docker-compose.yml                | Matija Dukarić                |
| TASK-06 | Lokalni Docker test (vsi servisi) | Vsi                           |
| TASK-07 | Azure Student račun               | Matija Dukarić                |
| TASK-08 | Ustvaritev Linux VM               | Matija Dukarić                |
| TASK-09 | SSH ključi in dostop              | Vsi                           |
| TASK-10 | Port forwarding — dokumentacija   | Maj Donko                     |
| TASK-11 | Tip in kapaciteta diska           | Maj Donko                     |
| TASK-12 | Poraba virov v naročnini          | Luka Manfreda                 |
| TASK-13 | Namestitev Dockerja + swap na VM  | Matija Dukarić                |
| TASK-14 | Deploy aplikacije na VM           | Luka Manfreda                 |
| TASK-15 | Pisanje poročila P2               | Matija Dukarić (koordinacija) |

---

## 0. Struktura projekta in repozitoriji


Naš projekt DropInSlovenia je razdeljen v **tri ločene GitHub repozitorije**, vsak za svoj servis:

| Repozitorij                               | Tehnologija       | Namen                          |
| ----------------------------------------- | ----------------- | ------------------------------ |
| `github.com/DropInSlovenia/frontend`      | Next.js (React)   | Uporabniški vmesnik            |
| `github.com/DropInSlovenia/backend`       | Node.js / Express | REST API, MongoDB komunikacija |
| `github.com/DropInSlovenia/kotlin-server` | Kotlin / Ktor     | Scraping in zunanji podatki    |

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

**Zakaj ločeni repozitoriji?** Vsak servis ima svojo ekipo (v realnem svetu), svojo tehnologijo in svoj deployment cikel. Z ločenimi repozitoriji se Docker slika za frontend zgradi samo ko se frontend koda spremeni — ne ob vsaki spremembi backenda.

**Kje je `docker-compose.yml`?** Za lokalni razvoj in za Azure VM imamo `docker-compose.yml` ali v četrtem infrastructure repozitoriju, ali pa ga ročno ustvarimo na VM. Ta datoteka poveže vse tri servise v enotno aplikacijo.

---

## 0.1 MongoDB Atlas — nastavitev oblačne baze


*Avtor: Matija Dukarić*

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


*Avtor: Vsi*

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
version: '3.8'  # verzija Docker Compose formata

services:

  # =========================
  # Kotlin backend / service
  # =========================
  kotlin-server:
    build:
      context: ./PrincipiProjekt/desktopApp  # mapa kjer je Dockerfile za Kotlin aplikacijo
      dockerfile: Dockerfile               # ime Dockerfile (lahko se izpusti, če je default)
    container_name: kotlin-server         # ime containerja
    ports:
      - "8080:8080"                       # host:container port mapping
    restart: unless-stopped               # restart če crasha ali ob rebootu

  # =========================
  # Node.js / backend API
  # =========================
  backend:
    build:
      context: ./backendProjekt            # lokacija backend kode
      dockerfile: Dockerfile
    container_name: backend
    ports:
      - "3000:3000"
    env_file: .env                        # naloži environment spremenljivke iz .env datoteke
    environment:
      - MONGODB_URI=${MONGODB_URI}        # MongoDB connection string
      - JWT_SECRET=${JWT_SECRET}          # secret za JWT avtentikacijo
      - KOTLIN_SERVICE_URL=http://kotlin-server:8080  # interni Docker network URL za Kotlin service
    depends_on:
      - kotlin-server                     # backend se zažene šele po kotlin-server
    restart: unless-stopped

  # =========================
  # Frontend (Next.js / React)
  # =========================
  frontend:
    build:
      context: ./webApp                   # lokacija frontend kode
      dockerfile: Dockerfile
      args:
        - NEXT_PUBLIC_API_URL=http://localhost:3000/api  # URL API-ja (viden v browserju)
    container_name: frontend
    ports:
      - "3001:3001"
    depends_on:
      - backend                          # frontend čaka backend
    restart: unless-stopped

```

`.env` datoteka (v istem folderju kot `docker-compose.yml`, **nikoli v git**):

```env
MONGODB_URI=mongodb+srv://dropinslovenia-user:GESLO@cluster0.xxxxx.mongodb.net/dropinslovenia
JWT_SECRET=nek_dolg_nakljucen_string_tukaj_vsaj_32_znakov
GOOGLE_PLACES_API_KEY=AIza...
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

Prikaz repozitorija, kjer so not vidni kotlin streznik, fronent, backend,  .env, in yaml:

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

![alt text](slike/yamlTask.png)
---

## 2. Dostop do storitve Azure

_Avtor: Matija Dukarić_

Na računu vodje skupine smo uspešno aktivirali Azure for Students naročnino. Postopek je potekal preko Microsoft Azure portala, kjer smo se prijavili s študentskim e-mail naslovom in opravili verifikacijo statusa študenta. Po uspešni aktivaciji smo pridobili dostop do brezplačnih Azure storitev, vključno z dobroimetjem in brezplačnimi urami virtualnega strežnika, brez vnosa kreditne kartice.

Jira task:

![alt text](slike/vmJira.png)

---

## 3. Vpostavitev virtualne naprave

_Avtor: Matija Dukarić, SSH ključi: Maj Donko, Luka Manfreda_

### 3.1 Parametri VM


| Parameter | Vrednost |
|-----------|----------|
| Subscription | Azure for Students |
| Resource group | DropInSlovenia_group_05191709 |
| VM name | dropinslovenia-vm |
| Region | Austria East (Zone 2) |
| Image | Ubuntu Server 24.04 LTS |
| Size | Standard B2ts v2 (2 vCPU, 1 GiB RAM) |
| Authentication | Password |
| Public IP | DA |

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

**Kaj je SSH?** SSH (Secure Shell) je protokol za varno oddaljeno upravljanje strežnikov prek ukazne vrstice. Z njim se povežemo na Azure VM kot da bi sedeli pred njim.

**Javni/zasebni ključ:** SSH deluje na principu para ključev. **Zasebni ključ** ostane na tvojem računalniku (nikoli ga ne deli z nikomer). **Javni ključ** dodaš na strežnik. Ob prijavi strežnik preveri ali imaš ustrezni zasebni ključ — brez gesla. To je varnejše od gesla ker geslo je mogoče uganjati (brute force), matematičnega ključa pa ne.

**`authorized_keys`** je datoteka na strežniku ki vsebuje seznam vseh javnih ključev katerim je dostop dovoljen. Vsaka vrstica je en ključ — enega na člana.

**Korak 1 — Vsak član na svojem računalniku generira SSH ključ:**

```bash
# Generiramo SSH ključ (ed25519 je modern in varen algoritem)
ssh-keygen -t ed25519 -C "ime.priimek@student.um.si"
# Pritisni Enter za privzeto lokacijo (~/.ssh/id_ed25519)
# Passphrase: po želji (Enter za prazno)


# Prikažemo javni ključ in ga pošljemo Matiji (npr. prek Discord/WhatsApp)
cat ~/.ssh/id_ed25519.pub
# Izpis (primer):
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... ime.priimek@student.um.si
```

**Korak 2 — Matija na VM doda ključe vseh treh članov:**

```bash
# Prva prijava z geslom (ki smo ga nastavili pri ustvaritvi VM)
ssh azureuser@<PUBLIC_IP>


# Ustvari .ssh mapo z ustreznimi pravicami
mkdir -p ~/.ssh
chmod 700 ~/.ssh
# (700 = samo lastnik ima dostop)


# Odpri authorized_keys in prilepi ključe vseh treh
nano ~/.ssh/authorized_keys
# V urejevalnik prilepi vse tri javne ključe, vsak v svojo vrstico:
# ssh-ed25519 AAAAC3... matija@student.um.si
# ssh-ed25519 AAAAC3... maj@student.um.si
# ssh-ed25519 AAAAC3... luka@student.um.si
# Shrani: Ctrl+X, Y, Enter


# Nastavi pravice (OBVEZNO — SSH zavrne prijavo če so pravice napačne)
chmod 600 ~/.ssh/authorized_keys
# (600 = samo lastnik lahko bere in piše)
```

**Korak 3 — Vsak testira SSH dostop brez gesla:**

```bash
ssh azureuser@<PUBLIC_IP>
# Mora se prijaviti BREZ gesla (samo s ključem)
# Pričakovana ukazna vrstica: azureuser@dropinslovenia-vm:~$
```

📸 _Slika: Matija — uspešna SSH prijava na VM_
`[VSTAVI SLIKO TUKAJ]`

📸 _Slika: Maj — uspešna SSH prijava na VM_
`[VSTAVI SLIKO TUKAJ]`

📸 _Slika: Luka — uspešna SSH prijava na VM_
`[VSTAVI SLIKO TUKAJ]`

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
<img width="1326" height="873" alt="image" src="https://github.com/user-attachments/assets/a368408c-121b-4252-a631-b0b2ff0554b9" />
---

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

**Zakaj swap?** Azure B1s VM ima samo **1 GB RAMa**. Naša aplikacija teče v treh containerjih skupaj:

- Next.js frontend: ~150–200 MB
- Node.js backend: ~100 MB
- Kotlin JVM: ~256 MB (omejeno z `-Xmx256m`)
- Ubuntu OS: ~200 MB

Skupaj: ~700–800 MB samo za aplikacijo. Ko Docker gradi slike ob zagonu, poraba začasno naraste nad 1 GB — VM bi "zmrznil". Dodamo **2 GB swap datoteko** — del diska ki ga OS uporablja kot razširitev RAMa (počasnejši od pravega RAMa a dovolj za naš primer).

```bash
# Prijava na VM
ssh azureuser@<PUBLIC_IP>


# Posodobitev sistema (vedno najprej posodobi)
sudo apt update && sudo apt upgrade -y


# Namestitev Dockerja z uradnim skriptom
# (skript samodejno doda Docker apt repozitorij in namesti Docker Engine)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh


# Dodamo trenutnega uporabnika v docker skupino
# (brez tega bi morali pisati "sudo docker" pred vsakim ukazom)
sudo usermod -aG docker $USER


# POMEMBNO: Odjavimo se in prijavimo nazaj da group membership velja
exit
ssh azureuser@<PUBLIC_IP>


# Preverimo namestitev
docker --version
docker compose version
```

**Dodamo swap (OBVEZNO za B1s VM):**

```bash
# Ustvarimo 2 GB swap datoteko na disku
sudo fallocate -l 2G /swapfile


# Nastavimo pravice (samo root sme brati swap)
sudo chmod 600 /swapfile


# Inicializiramo swap datoteko
sudo mkswap /swapfile


# Aktiviramo swap
sudo swapon /swapfile


# Dodamo v /etc/fstab da swap ostane aktiven po vsakem rebootu VM
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab


# Preverimo — mora pokazati Swap: 2.0G
free -h
```

📸 _Slika: `docker --version` in `docker compose version` na VM_
`[VSTAVI SLIKO TUKAJ]`

📸 _Slika: `free -h` — vidni RAM in swap_
`[VSTAVI SLIKO TUKAJ]`

---

### 5.2 Prenos kode in zagon

_Avtor: Luka Manfreda_

**Izziv z ločenimi repozitoriji:** Ker imamo 3 ločene git repozitorije, jih moramo vse klonirati na VM v ustrezno strukturo map. `docker-compose.yml` pričakuje vse tri v isti nadrejeni mapi.

**Korak 1 — Kloniramo repozitorije:**

```bash
# Na VM ustvarimo delovno mapo
mkdir -p ~/dropinslovenia
cd ~/dropinslovenia


# Kloniramo vse tri repozitorije
git clone https://github.com/DropInSlovenia/frontend.git
git clone https://github.com/DropInSlovenia/backend.git
git clone https://github.com/DropInSlovenia/kotlin-server.git


# Preverimo strukturo
ls -la
# Mora videti: frontend/  backend/  kotlin-server/
```

**Korak 2 — Ustvarimo `.env` datoteko z dejanskimi vrednostmi:**

```bash
# Na VM ustvarimo .env (Matija vnese prave vrednosti)
nano ~/dropinslovenia/.env
```

Vsebina `.env`:

```env
MONGODB_URI=mongodb+srv://dropinslovenia-user:GESLO@cluster0.xxxxx.mongodb.net/dropinslovenia
JWT_SECRET=nek_dolg_nakljucen_string_tukaj_vsaj_32_znakov
GOOGLE_PLACES_API_KEY=AIza...
```

> **Varnostna opomba:** Ta datoteka ostane samo na VM. Nikoli je ne commitamo v git. Če jo brišemo ali VM resetiramo, jo moramo znova ročno ustvariti.

**Korak 3 — Ustvarimo `docker-compose.yml` za Azure VM:**

`docker-compose.yml` za Azure VM se razlikuje od lokalnega v enem delu — `NEXT_PUBLIC_API_URL` mora biti javni IP VM, ne `localhost`:

```bash
nano ~/dropinslovenia/docker-compose.yml
```

Vsebina (enaka kot lokalna, le z zamenjano vrednostjo za frontend):

```yaml
# (enako kot v sekciji 1.5, le ta vrstica drugačna:)
args:
  - NEXT_PUBLIC_API_URL=http://<PUBLIC_IP>:3000/api
  # Zamenjaj <PUBLIC_IP> z dejanskim IP naslovom Azure VM!
```

**Zakaj je to potrebno?** `NEXT_PUBLIC_API_URL` se vtisne v JavaScript bundle med Docker buildom. Frontend teče v **uporabnikovem brskalniku** — ne v Docker omrežju. Ko brskalnik naredi API klic, mora doseči backend prek javnega IP-ja, ne prek `localhost` (ki bi kazal na uporabnikov računalnik).

**Korak 4 — Zagon:**

```bash
cd ~/dropinslovenia


# Zgradi slike in zaženi v ozadju
# Opozorilo: Kotlin build traja ~5-10 minut, ostala dva ~2-3 minute
docker compose up --build -d


# Sproti opazuj loge med buildom
docker compose logs -f


# Ko je vse zagnano, preveri status
docker ps
```

**Test dostopnosti iz interneta** (vsak na svojem računalniku v brskalniku):

```
http://<PUBLIC_IP>:3001                    ← Frontend
http://<PUBLIC_IP>:3000/api                ← Backend
http://<PUBLIC_IP>:8080/events/maribor     ← Kotlin server
```

**Posodobitev kode na VM** (ko pushate spremembe na git):

```bash
cd ~/dropinslovenia/frontend   # ali backend ali kotlin-server
git pull
cd ~/dropinslovenia
docker compose up --build -d frontend  # rebuild samo spremenjenega servisa
```

📸 _Slika: `docker ps` na VM — vsi 3 containerji Up_
`[VSTAVI SLIKO TUKAJ]`

📸 _Slika: Aplikacija dostopna v brskalniku na `http://<PUBLIC_IP>:3001`_
`[VSTAVI SLIKO TUKAJ]`

📸 _Slika: API klic na `http://<PUBLIC_IP>:8080/events/maribor` vrne JSON_
`[VSTAVI SLIKO TUKAJ]`

---

## Morebitne težave in rešitve

| Težava                                        | Vzrok                                                         | Rešitev                                                                       |
| --------------------------------------------- | ------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `docker compose up` zamrzne pri Kotlin buildu | VM zmanjka RAMa med Gradle buildom                            | Preveri da je swap aktiven (`free -h`), dodaj 2GB swap                        |
| Frontend ne more doseči backenda              | `NEXT_PUBLIC_API_URL` kaže na `localhost` namesto na javni IP | Posodobi `docker-compose.yml` z dejanskim `<PUBLIC_IP>` in rebuildaj frontend |
| SSH dostop zavrnjen                           | Napačne pravice na `authorized_keys`                          | Na VM: `chmod 600 ~/.ssh/authorized_keys` in `chmod 700 ~/.ssh`               |
| MongoDB connection error                      | IP Azure VM ni dovoljen v Atlas                               | Dodaj `0.0.0.0/0` v Atlas Network Access                                      |
| Port ni dostopen iz interneta                 | NSG pravilo manjka                                            | Dodaj Inbound rule za ustrezni port v Azure NSG                               |
| [Opiši svojo težavo]                          | [Opiši vzrok]                                                 | [Opiši rešitev]                                                               |

---

_Poročilo P2 — DropInSlovenia | Maj 2026_
