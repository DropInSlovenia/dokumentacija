# Poročilo P2 — Docker + Azure VM
## DropInSlovenia


**Skupina:** Matija Dukarić (vodja), Maj Donko, Luka Manfreda  
**Datum oddaje:** ___________  
**GitHub:** https://github.com/DropInSlovenia


---


## Razdelitev nalog


| Task | Opis | Assignee |
|------|------|----------|
| TASK-01 | `output: standalone` v Next.js | Maj Donko |
| TASK-02 | Dockerfile — Frontend | Maj Donko |
| TASK-03 | Dockerfile — Backend | Luka Manfreda |
| TASK-04 | Dockerfile — Kotlin server | Luka Manfreda |
| TASK-05 | docker-compose.yml | Matija Dukarić |
| TASK-06 | Lokalni Docker test (vsi servisi) | Vsi |
| TASK-07 | Azure Student račun | Matija Dukarić |
| TASK-08 | Ustvaritev Linux VM | Matija Dukarić |
| TASK-09 | SSH ključi in dostop | Vsi |
| TASK-10 | Port forwarding — dokumentacija | Maj Donko |
| TASK-11 | Tip in kapaciteta diska | Maj Donko |
| TASK-12 | Poraba virov v naročnini | Luka Manfreda |
| TASK-13 | Namestitev Dockerja + swap na VM | Matija Dukarić |
| TASK-14 | Deploy aplikacije na VM | Luka Manfreda |
| TASK-15 | Pisanje poročila P2 | Matija Dukarić (koordinacija) |


---


## 0. Struktura projekta in repozitoriji


*Avtor: Matija Dukarić*


Naš projekt DropInSlovenia je razdeljen v **tri ločene GitHub repozitorije**, vsak za svoj servis:


| Repozitorij | Tehnologija | Namen |
|-------------|-------------|-------|
| `github.com/DropInSlovenia/frontend` | Next.js (React) | Uporabniški vmesnik |
| `github.com/DropInSlovenia/backend` | Node.js / Express | REST API, MongoDB komunikacija |
| `github.com/DropInSlovenia/kotlin-server` | Kotlin / Ktor | Scraping in zunanji podatki |


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


Naša aplikacija ne sme imeti baze nameščene lokalno — uporabljamo **MongoDB Atlas**, ki je upravljana MongoDB baza v oblaku (brezplačni tier M0 je zadosten).


**Zakaj oblačna baza in ne lokalna?** Lokalna baza bi bila dostopna samo na eni napravi. Z Atlas bazo do nje dostopata tako lokalni razvoj kot Azure VM, brez da bi morali namestiti MongoDB kjerkoli.


**Koraki za nastavitev Atlas baze:**


1. Odpri https://cloud.mongodb.com in ustvari brezplačni račun
2. Ustvari nov projekt: **New Project** → ime `DropInSlovenia`
3. Ustvari cluster: **Build a Database** → izberi **M0 Free** → Region: Europe West
4. Ustvari database user:
   - **Database Access** → **Add New Database User**
   - Username: `dropinslovenia-user`
   - Password: generiraj varno geslo in si ga shrani
   - Role: **Atlas admin** (ali Read and write to any database)
5. Dovoli dostop z vseh IP naslovov (za Azure VM z dinamičnim IP):
   - **Network Access** → **Add IP Address** → **Allow Access from Anywhere** (`0.0.0.0/0`)
   - Opomba: V produkciji bi omejili samo na IP Azure VM-ja
6. Pridobi connection string:
   - **Database** → **Connect** → **Drivers** → izberi Node.js
   - Kopiraj connection string, izgleda tako:
   ```
   mongodb+srv://dropinslovenia-user:<password>@cluster0.xxxxx.mongodb.net/dropinslovenia
   ```
   - Zamenjaj `<password>` s pravim geslom in shrani v `.env`


```env
# .env (NIKOLI v git!)
MONGODB_URI=mongodb+srv://dropinslovenia-user:GESLO@cluster0.xxxxx.mongodb.net/dropinslovenia
```


**Preveri delovanje:**
```bash
# Lokalno — testiraj da se backend poveže z Atlas
cd backend
node -e "
const mongoose = require('mongoose');
mongoose.connect(process.env.MONGODB_URI)
  .then(() => { console.log('Atlas OK!'); process.exit(0); })
  .catch(err => { console.error(err); process.exit(1); });
"
```


![MongoDB Atlas dashboard z ustvarjenim cluster-jem](slike/mongoAtlas.png) 



📸 *Slika: Network Access z `0.0.0.0/0` pravilom*
`[VSTAVI SLIKO TUKAJ]`


---


## 1. Lokalna namestitev Dockerja


### 1.0 Predpogoj — namestitev Dockerja lokalno


*Avtor: Vsi*


Preden pišemo Dockerfila, mora vsak član imeti Docker nameščen lokalno.


**Windows / Mac:** Prenesi in namesti **Docker Desktop** z https://www.docker.com/products/docker-desktop/


**Linux (Ubuntu):**
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
# Odjavi se in prijavi nazaj
docker --version
```


**Preveri namestitev:**
```bash
docker --version
# Docker version 24.x.x
docker compose version
# Docker Compose version v2.x.x
docker run hello-world
# Mora izpisati "Hello from Docker!"
```


**Kaj je Docker in zakaj ga uporabljamo?**
Docker je orodje za pakiranje aplikacij v **containerje** — izolirane enote ki vsebujejo aplikacijo in vse njene odvisnosti (Node.js, Python, Java...). Container teče enako na vsakem računalniku, ne glede na operacijski sistem. Rešuje problem "pri meni dela, pri tebi ne" — v containerju je okolje vedno isto.


---


### 1.1 Next.js config — predpogoj za Docker
*Avtor: Maj Donko*


Preden pišemo Dockerfile za frontend, moramo v `frontend/next.config.ts` dodati eno obvezno vrstico:


```typescript
// frontend/next.config.ts
const nextConfig = {
  output: 'standalone',  // OBVEZNO za Docker
}
export default nextConfig
```


**Zakaj `output: standalone`?** Brez tega Next.js v Docker container skopira celotno `node_modules` mapo (pogosto večja od 500 MB). Z `standalone` outputom Next.js zgradi minimalen bundle (~10–30 MB) z vsem kar potrebuje za zagon — brez razvojnih odvisnosti. Container je manjši, hitrejši za prenos in hitrejši za zagon.


Spremembo commitas in pushas na git.


---


### 1.2 Dockerfile — Frontend (Next.js)
*Avtor: Maj Donko*


**Kaj je Dockerfile?** Dockerfile je tekstovna datoteka z navodili za gradnjo Docker slike. Vsaka vrstica je en korak. Docker izvede korake od zgoraj navzdol in shrani rezultat kot sliko (image), ki jo nato zaženemo kot container.


**Multi-stage build** je tehnika kjer Docker zgradi sliko v več fazah — vsaka faza ima svojo `FROM` direktivo. V **builder** fazi imamo vsa razvojna orodja in izvajamo `npm run build`. V **runner** fazi pa vzamemo samo tisto kar je potrebno za zagon. Iz builder faze v runner fazo prenesemo samo rezultat builda — brez `node_modules`, brez izvorne kode. Rezultat je majhen produkcijski container.


`NEXT_PUBLIC_API_URL` se nastavi med buildom kot build argument (`ARG`). To je URL do Node.js backend API-ja. Ko gradimo za Azure VM, ga nastavimo na javni IP VM-ja. Next.js ta URL **vtisne v JavaScript bundle med kompilacijo** — zato ga moramo podati med `docker build`, ne ob zagonu containerja.


```dockerfile
# FAZA 1: Build
FROM node:20-alpine AS builder


WORKDIR /app


# Docker cache optimizacija: najprej samo package.json
# Če se package.json ne spremeni, Docker preskoči npm ci pri naslednjem buildu
COPY package*.json ./
RUN npm ci


COPY . .


# Build argument — API URL se nastavi med docker build
# Privzeta vrednost je za lokalni razvoj
ARG NEXT_PUBLIC_API_URL=http://localhost:3000/api
ENV NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL


RUN npm run build


# FAZA 2: Production runner (brez razvojnih orodij)
FROM node:20-alpine AS runner


WORKDIR /app


ENV NODE_ENV=production


# Kopiramo samo rezultat builda iz prve faze
# --from=builder pomeni: vzemi iz faze z imenom "builder"
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public


EXPOSE 3001


ENV PORT=3001
ENV HOSTNAME="0.0.0.0"


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
> [Opiši morebitne težave in kako si jih rešil]


📸 *Slika: `docker build` uspešno zaključen za frontend*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Frontend dostopen na http://localhost:3001*
`[VSTAVI SLIKO TUKAJ]`


---


### 1.3 Dockerfile — Backend (Node.js/Express)
*Avtor: Luka Manfreda*


Za backend ne potrebujemo multi-stage builda ker Node.js ne kompilira kode — izvorna koda je direktno izvršljiva. Ključna optimizacija je `--only=production` pri `npm ci` — s tem Docker ne namesti `devDependencies` (npr. Jest, ESLint, nodemon), ki v produkciji niso potrebni. Container je manjši in varnejši.


Backend komunicira z dvema zunanjima sistemoma:
- **MongoDB Atlas** prek `MONGODB_URI` connection stringa (oblačna baza)
- **Kotlin strežnikom** prek `KOTLIN_SERVER_URL` — to je interni Docker URL (`http://kotlin-server:8080`) ki deluje samo znotraj Docker omrežja


Ti podatki se **ne shranijo v Docker sliko** — podamo jih kot environment spremenljivke ob zagonu prek `.env` datoteke. To je varnostna praksa: slika sama ne vsebuje nobenih gesel.


```dockerfile
FROM node:20-alpine


WORKDIR /app


# Samo produkcijske odvisnosti (brez devDependencies)
COPY package*.json ./
RUN npm ci --only=production


# Kopiramo izvorno kodo
COPY . .


EXPOSE 3000


CMD ["node", "server.js"]
```


**Opomba:** Če vaš backend vstopno točko imenuje drugače (npr. `index.js` ali `app.js`), prilagodi zadnjo vrstico CMD ustrezno.


**Test lokalno:**
```bash
cd backend
docker build -t dropinslovenia/backend:test .
docker run -p 3000:3000 \
  -e MONGODB_URI="mongodb+srv://dropinslovenia-user:GESLO@cluster0.xxx.mongodb.net/dropinslovenia" \
  -e JWT_SECRET="test_secret_za_lokalni_test" \
  -e KOTLIN_SERVER_URL="http://host.docker.internal:8080" \
  dropinslovenia/backend:test
```


**Zakaj `host.docker.internal`?** Ko testiramo backend container lokalno (brez docker-compose), Kotlin server teče neposredno na tvojem računalniku ali v ločenem containerju. `host.docker.internal` je posebno DNS ime ki znotraj Docker containerja kaže na tvoj računalnik (host). V docker-compose okolju tega ne rabimo — tam backend naslovi kotlin-server po imenu servisa.


**Test API-ja:**
```bash
# Preveri da backend odgovarja
curl http://localhost:3000/api
# Ali v brskalniku: http://localhost:3000/api
```


📸 *Slika: Backend container teče, API klic uspešen*
`[VSTAVI SLIKO TUKAJ]`


---


### 1.4 Dockerfile — Kotlin Ktor strežnik
*Avtor: Luka Manfreda*


Kotlin zahteva **Gradle** za prevajanje — to je razlog za multi-stage build. V **builder** fazi imamo celoten JDK (Java Development Kit) in Gradle, ki prevedeta Kotlin kodo in ustvarita JAR datoteko. V **runner** fazi potrebujemo samo **JRE** (Java Runtime Environment), ne celotnega JDK — JRE je ~3x manjši ker vsebuje samo runtime, ne kompajlerja.


`-Xmx256m` je JVM flag ki omeji porabo RAMa na 256 MB. Azure B1s VM ima skupaj samo 1 GB RAMa. JVM brez omejitve privzeto rezervira 25% sistemskega RAMa (~256 MB) samo za "heap". Skupaj z Node.js backend-om (~100 MB), Next.js frontend-om (~150 MB) in operacijskim sistemom (~200 MB) bi presegli 1 GB — VM bi začel pagirati na disk ali pa bi containerji padli z OOM (Out Of Memory) napako.


```dockerfile
# FAZA 1: Build z Gradle
FROM gradle:8.5-jdk21 AS builder


WORKDIR /app


# Najprej kopiramo build datoteke (cache optimizacija)
COPY gradle/ gradle/
COPY gradlew gradlew.bat ./
COPY settings.gradle.kts build.gradle.kts ./
COPY gradle/libs.versions.toml gradle/


RUN chmod +x gradlew


# Kopiramo izvorno kodo
COPY src/ src/


# Gradle build — traja ~3-5 minut prvič (prenos odvisnosti z interneta)
# -x test preskoči teste med Docker buildom
RUN ./gradlew jvmJar --no-daemon -x test


# FAZA 2: Samo Java Runtime (brez Gradle in JDK)
FROM eclipse-temurin:21-jre-alpine


WORKDIR /app


# Kopiramo samo JAR datoteko iz builder faze
COPY --from=builder /app/build/libs/*.jar app.jar


EXPOSE 8080


# -Xmx256m: omejimo RAM porabo na Azure VM
CMD ["java", "-Xmx256m", "-jar", "app.jar"]
```


**Opomba glede Gradle task imena:** Zgornji Dockerfile predpostavlja da ima vaš projekt `jvmJar` Gradle task (tipično za Kotlin Multiplatform projekte). Če imate navaden Kotlin/JVM projekt, je task morda samo `jar` ali `bootJar` (Spring Boot). Preverite z:
```bash
cd kotlin-server
./gradlew tasks | grep -i jar
# Prikaže vse razpoložljive JAR taske
```


**Test lokalno:**
```bash
cd kotlin-server
docker build -t dropinslovenia/kotlin-server:test .
# Opozorilo: prvi build traja ~5-10 minut ker Gradle prenese vse odvisnosti


docker run -p 8080:8080 \
  -e GOOGLE_PLACES_API_KEY="vaš_ključ" \
  dropinslovenia/kotlin-server:test


# Test:
curl http://localhost:8080/events/maribor
```


> [Opiši morebitne težave z Gradle buildom in kako si jih rešil]


📸 *Slika: Kotlin server container teče, `/events/maribor` vrne podatke*
`[VSTAVI SLIKO TUKAJ]`


---


### 1.5 docker-compose.yml
*Avtor: Matija Dukarić*


**Kaj je Docker Compose?** `docker-compose.yml` je datoteka ki opisuje vse servise naše aplikacije in jih zažene skupaj z enim ukazom (`docker compose up`). Brez Compose bi morali za vsak container ročno pisati dolge `docker run` ukaze z vsemi parametri.


**Interno Docker omrežje:** Docker Compose avtomatično ustvari skupno interno omrežje za vse servise. Znotraj tega omrežja se servisi naslavljajo po **imenu servisa** (npr. `http://kotlin-server:8080`), ne po IP naslovu. To pomeni da backend v svoji kodi ne potrebuje vedeti IP naslova Kotlin strežnika — vedno bo dosegljiv na `http://kotlin-server:8080`.


**`depends_on`** pove Dockerju vrstni red zaganjanja. Backend čaka da se kotlin-server zažene, ker ob zagonu morda takoj naredi klic nanj. Frontend čaka backend iz istega razloga. Opomba: `depends_on` čaka da se container **zažene**, ne da je aplikacija v njem **pripravljena** — za to bi potrebovali `healthcheck`.


**`restart: unless-stopped`** poskrbi da se container avtomatično ponovno zažene po rebootu VM ali po morebitni napaki v aplikaciji. Ne bo se restartal samo če ga ročno ustaviš z `docker stop`.


**`env_file: .env`** naloži environment spremenljivke iz `.env` datoteke. Ta datoteka je v `.gitignore` — nikoli ne gre v git repozitorij ker vsebuje gesla!


```yaml
version: '3.8'


services:


  kotlin-server:
    build:
      context: ./kotlin-server
      dockerfile: Dockerfile
    container_name: kotlin-server
    ports:
      - "8080:8080"
    env_file: .env
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:8080/events/maribor"]
      interval: 30s
      timeout: 10s
      retries: 3


  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: backend
    ports:
      - "3000:3000"
    environment:
      - MONGODB_URI=${MONGODB_URI}
      - JWT_SECRET=${JWT_SECRET}
      - KOTLIN_SERVER_URL=http://kotlin-server:8080
    depends_on:
      - kotlin-server
    restart: unless-stopped


  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        - NEXT_PUBLIC_API_URL=http://localhost:3000/api
    container_name: frontend
    ports:
      - "3001:3001"
    depends_on:
      - backend
    restart: unless-stopped
```


`.env` datoteka (v istem folderju kot `docker-compose.yml`, **nikoli v git**):


```env
MONGODB_URI=mongodb+srv://dropinslovenia-user:GESLO@cluster0.xxxxx.mongodb.net/dropinslovenia
JWT_SECRET=nek_dolg_nakljucen_string_tukaj_vsaj_32_znakov
GOOGLE_PLACES_API_KEY=AIza...
```


**Opomba o strukturi repozitorijev:** Ker imamo 3 ločene repozitorije, `docker-compose.yml` pričakuje da so vsi klonirani v isti nadrejeni mapi:


```
~/dropinslovenia/
├── frontend/          ← git clone .../frontend
├── backend/           ← git clone .../backend
├── kotlin-server/     ← git clone .../kotlin-server
├── docker-compose.yml ← ta datoteka
└── .env               ← lokalne spremenljivke (v .gitignore)
```


**Zagon vsega skupaj:**
```bash
# Zgradi vse slike in zaženi v ozadju
docker compose up --build


# Za zagon v ozadju (detached mode):
docker compose up --build -d


# Preveri status
docker ps


# Oglej loge
docker compose logs --tail=50


# Ustavi vse
docker compose down
```


**Razlaga `--build`:** Ta zastavica pove Docker Compose da naj vedno znova zgradi slike iz Dockerfilov. Brez nje bi Docker Compose uporabil že obstoječo sliko (cached) tudi če si spremenil kodo. Ob prvem zagonu je `--build` vedno potreben.


📸 *Slika: `docker compose up --build` — logi vseh 3 servisov*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: `docker ps` — vsi 3 containerji z statusom `Up`*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Frontend v brskalniku na http://localhost:3001*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Kotlin server API klic na http://localhost:8080/events/maribor*
`[VSTAVI SLIKO TUKAJ]`


---


## 2. Dostop do storitve Azure
*Avtor: Matija Dukarić*


**Kaj je Azure?** Microsoft Azure je platforma za računalništvo v oblaku — ponuja virtualne strežnike, baze, omrežja in stotine drugih storitev. Mi bomo uporabili **virtualni strežnik (VM)** na katerem bomo poganjali naše Docker containerje.


**Azure for Students** je Microsoftov program ki študentom brezplačno ponuja:
- 100 € dobroimetja za katero koli Azure storitev
- 750 ur/mesec virtualnega strežnika B1s za 12 mesecev (brezplačno brez porabe dobroimetja)
- Brez kreditne kartice — samo studenstki email


**Koraki registracije:**
1. Odpri https://azure.microsoft.com/en-us/free/students/
2. Klikni **Start free**
3. Prijavi se s **študentskim e-mailom** (`xx1234@student.um.si`)
4. Verifikacija: Microsoft preveri da email pripada šolski instituciji — sledi navodilom
5. Nikjer ne vnašaj kreditne kartice — ni zahtevano
6. Po uspešni registraciji se prikaže Azure portal s "Azure for Students" naročnino


> [Opiši morebitne težave pri verifikaciji in kako si jih rešil]


📸 *Slika: Azure portal po uspešni prijavi — viden "Azure for Students" subscription*
`[VSTAVI SLIKO TUKAJ]`


---


## 3. Vpostavitev virtualne naprave
*Avtor: Matija Dukarić, SSH ključi: Maj Donko, Luka Manfreda*


### 3.1 Parametri VM


**Kaj je virtualna naprava (VM)?** VM je računalnik ki teče kot program na Microsoftovem fizičnem strežniku. Dobimo lastni Linux operacijski sistem, IP naslov in popoln SSH dostop — enako kot fizičen strežnik, le da je virtualen.


| Parameter | Vrednost | Zakaj |
|-----------|----------|-------|
| Subscription | Azure for Students | Brezplačna naročnina |
| Resource group | dropinslovenia-rg | Logična skupina vseh virov projekta |
| VM name | dropinslovenia-vm | Ime naše navidezne naprave |
| Region | West Europe | Najbližji datacenter (Amsterdam) |
| Image | Ubuntu Server 24.04 LTS | Stabilna Linux distribucija, LTS = dolgotrajna podpora |
| Size | Standard B1s | Vključeno v brezplačni tier: 1 vCPU, 1 GB RAM |
| Authentication | Password | Enostavno za začetek, SSH ključe dodamo ročno |
| Public IP | [VSTAVI IP] | Javni naslov prek katerega dostopamo do VM |


**Koraki v portalu:**


```
Portal: https://portal.azure.com
Pot: Virtual Machines → Create → Azure virtual machine


1. Basics:
   - Subscription: Azure for Students
   - Resource group: Create new → dropinslovenia-rg
   - Virtual machine name: dropinslovenia-vm
   - Region: (Europe) West Europe
   - Image: Ubuntu Server 24.04 LTS - x64 Gen2
   - Size: Standard_B1s (klikni "See all sizes" če ni vidno)
   - Authentication type: Password
   - Username: azureuser
   - Password: [dolgo geslo, si ga zapomni!]
   - Public inbound ports: Allow selected → SSH (22)


2. Disks: privzeto (Standard SSD)


3. Networking: privzeto


4. Review + Create → Create
```


Deployment traja ~2 minuti. Po koncu: **Go to resource** → zabeležiš **Public IP address** (npr. `20.123.45.67`).


> [Opiši morebitne težave in kako si jih rešil]


📸 *Slika: "Your deployment is complete" stran*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: VM overview z vidnim Public IP*
`[VSTAVI SLIKO TUKAJ]`


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


📸 *Slika: Matija — uspešna SSH prijava na VM*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Maj — uspešna SSH prijava na VM*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Luka — uspešna SSH prijava na VM*
`[VSTAVI SLIKO TUKAJ]`


---


## 4. Odgovori na Azure vprašanja


### 4.1 Kje in kako omogočite "port forwarding"?
*Avtor: Maj Donko*


**Kaj je NSG?** Network Security Group (NSG) je virtualni požarni zid ki nadzira omrežni promet do in od Azure virov. Vsak VM v Azure avtomatično dobi priložen NSG.


**Kako deluje?** NSG vsebuje seznam **pravil** (rules). Vsako pravilo določa: iz katerega vira (Source), na kateri port, kateri protokol in ali je promet dovoljen (Allow) ali zavrnjen (Deny). Pravila so urejena po prioriteti — nižja številka = višja prioriteta.


**Privzeto stanje:** Ko ustvarimo VM, so vsa vhodna vrata zaprta razen porta 22 (SSH). To je varnostno načelo najmanjših privilegijev — odpiramo samo tisto kar nujno potrebujemo.


**Port forwarding** na Azure pomeni dodajanje Inbound security rules v NSG za porte naše aplikacije.


**Pot v portalu:** VM → **Networking** → **Network settings** → **Create port rule → Inbound port rule**


Za vsak port naše aplikacije dodamo pravilo:


| Port | Ime pravila | Namen |
|------|-------------|-------|
| 3000 | allow-backend | Node.js/Express REST API |
| 3001 | allow-frontend | Next.js frontend |
| 8080 | allow-kotlin | Kotlin Ktor server |
| 9000 | allow-webhook | Webhook strežnik (P3) |


Za vsako pravilo nastavimo:
- **Source:** Any
- **Source port ranges:** `*`
- **Destination:** Any
- **Destination port ranges:** npr. `3001`
- **Protocol:** TCP
- **Action:** Allow
- **Priority:** npr. 1010, 1020, 1030, 1040 (vsak naslednji +10)


> **Zakaj je privzeto vse zaprto?** Vsak odprt port je potencialna napadalna površina. Napadalec ki skenira internet bi na odprtem portu 3000 videl naš Node.js API in ga poskušal izkoristiti. Odpiramo samo kar nujno rabimo.


📸 *Slika: NSG inbound rules z dodanimi pravili za porte 3000, 3001, 8080, 9000*
`[VSTAVI SLIKO TUKAJ]`


---


### 4.2 Kakšen tip diska je bil dodan navidezni napravi in kakšna je njegova kapaciteta?
*Avtor: Maj Donko*


**Pot v portalu:** VM → **Disks**


Naši navidezni napravi je bil samodejno dodan OS disk tipa **Standard SSD (LRS)** s kapaciteto **30 GB**.


**Razlaga:**
- **Standard SSD** (v nasprotju s Standard HDD ali Premium SSD) — zmogljivost je med HDD in Premium SSD. Za naš primer (strežniška aplikacija z Docker) je popolnoma zadosten.
- **LRS** (Locally Redundant Storage) pomeni da Azure podatke replicira **trikrat znotraj istega podatkovnega centra** v Amsterdamu. Če en fizični disk odpove, se podatki ohranijo. Ne varuje pred izpadom celotnega podatkovnega centra — za to bi potrebovali ZRS ali GRS.
- **30 GB** je privzeta velikost OS diska za Ubuntu VM v Azure.


📸 *Slika: Disks sekcija v Azure portalu z vidnim tipom in kapaciteto diska*
`[VSTAVI SLIKO TUKAJ]`


---


### 4.3 Kje preverimo stanje trenutne porabe virov v naročnini "Azure for students"?
*Avtor: Luka Manfreda*


**Pot v portalu:** Iskalno polje zgoraj → vpišemo **"Cost Management"** → **Cost analysis**


ALI: **Subscriptions** → **Azure for Students** → levi meni → **Cost Management → Cost analysis**


**Kaj prikazuje Cost Management?**
- Skupno porabo v evrih razčlenjeno po storitvah (Virtual Machines, Storage, Bandwidth...)
- Graf porabe skozi čas
- Koliko od 100 € dobroimetja smo že porabili
- Napoved porabe do konca meseca


> **Pozor:** Poraba bo vidna šele ~24 ur po vzpostavitvi VM-ja. Strežnik zbirata in agregira podatke z zamudo. Posnetek zaslona naredi dan po vzpostavitvi.


📸 *Slika: Cost Management / Cost analysis v Azure portalu*
`[VSTAVI SLIKO TUKAJ]`


---


## 5. Vpostavitev Dockerja na Azure VM


### 5.1 Namestitev Dockerja in swap
*Avtor: Matija Dukarić*


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


📸 *Slika: `docker --version` in `docker compose version` na VM*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: `free -h` — vidni RAM in swap*
`[VSTAVI SLIKO TUKAJ]`


---


### 5.2 Prenos kode in zagon
*Avtor: Luka Manfreda*


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


📸 *Slika: `docker ps` na VM — vsi 3 containerji Up*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Aplikacija dostopna v brskalniku na `http://<PUBLIC_IP>:3001`*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: API klic na `http://<PUBLIC_IP>:8080/events/maribor` vrne JSON*
`[VSTAVI SLIKO TUKAJ]`


---


## Morebitne težave in rešitve


| Težava | Vzrok | Rešitev |
|--------|-------|---------|
| `docker compose up` zamrzne pri Kotlin buildu | VM zmanjka RAMa med Gradle buildom | Preveri da je swap aktiven (`free -h`), dodaj 2GB swap |
| Frontend ne more doseči backenda | `NEXT_PUBLIC_API_URL` kaže na `localhost` namesto na javni IP | Posodobi `docker-compose.yml` z dejanskim `<PUBLIC_IP>` in rebuildaj frontend |
| SSH dostop zavrnjen | Napačne pravice na `authorized_keys` | Na VM: `chmod 600 ~/.ssh/authorized_keys` in `chmod 700 ~/.ssh` |
| MongoDB connection error | IP Azure VM ni dovoljen v Atlas | Dodaj `0.0.0.0/0` v Atlas Network Access |
| Port ni dostopen iz interneta | NSG pravilo manjka | Dodaj Inbound rule za ustrezni port v Azure NSG |
| [Opiši svojo težavo] | [Opiši vzrok] | [Opiši rešitev] |


---


*Poročilo P2 — DropInSlovenia | Maj 2026*


