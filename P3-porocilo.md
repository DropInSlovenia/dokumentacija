# Poročilo P3 — CI/CD (Docker Hub + GitHub Actions + Webhook)
## DropInSlovenia


**Skupina:** Matija Dukarić (vodja), Maj Donko, Luka Manfreda  
**Datum oddaje:** ___________  
**GitHub:** https://github.com/DropInSlovenia  
**Docker Hub:** https://hub.docker.com/u/dropinslovenia


---


## Razdelitev nalog


| Task | Opis | Assignee |
|------|------|----------|
| TASK-16 | Docker Hub račun in repozitoriji | Matija Dukarić |
| TASK-17 | Ročni push Docker slik | Maj (frontend), Luka (backend + kotlin) |
| TASK-18 | GitHub Secrets v vseh repozitorijih | Matija Dukarić |
| TASK-19 | GitHub Actions workflow — Frontend | Maj Donko |
| TASK-20 | GitHub Actions workflow — Backend | Luka Manfreda |
| TASK-21 | GitHub Actions workflow — Kotlin | Matija Dukarić |
| TASK-22 | Deploy skripta na VM | Luka Manfreda |
| TASK-23 | Webhook konfiguracija + systemd | Matija Dukarić |
| TASK-24 | Test celotnega CI/CD pipeline | Vsi |
| TASK-25 | Varnostna analiza + UFW firewall | Maj (dokumentacija), Matija (implementacija) |
| TASK-26 | Opis dodatnih GitHub Actions workflows | Maj Donko |
| TASK-27 | Pisanje poročila P3 | Luka (koordinacija) |


---


## 0. Pregled CI/CD arhitekture


*Avtor: Matija Dukarić*


**Kaj je CI/CD?** CI/CD (Continuous Integration / Continuous Delivery) je praksa kjer se vsaka sprememba kode samodejno zgradi, testira in dostavi v produkcijo — brez ročnega posredovanja. Cilj je da vsak `git push` na `main` branch samodejno posodobi aplikacijo na strežniku.


**Naš CI/CD tok:**


```
Razvijalec naredi spremembo v kodi
         │
         ▼
    git push na main
         │
         ▼
  GitHub Actions zazna push
         │
         ├─── Zgradi Docker sliko iz Dockerfile
         ├─── Naloži sliko na Docker Hub (nova verzija)
         │
         ▼ (samo če je zgornje uspelo)
  Pošlje webhook HTTP zahtevo na Azure VM
         │
         ▼
  VM: webhook strežnik sprejme zahtevo
         │
         ├─── Preveri HMAC podpis (varnostno preverjanje)
         ├─── Zažene deploy.sh skripto
         │       ├─── Ustavi stari container
         │       ├─── Prenese novo sliko z Docker Hub
         │       └─── Zažene nov container
         ▼
  Aplikacija je posodobljena na VM ✓
```


**Zakaj ločen Docker Hub?** GitHub Actions gradi sliko na GitHub strežnikih, ne na našem VM-ju. Sliko mora "prenesti" na VM — Docker Hub je vmesno skladišče (registry). VM ne more direktno dostopati do GitHub Actions build okolja.


**Zakaj 3 ločeni workflow-i?** Ker imamo 3 ločene repozitorije (frontend, backend, kotlin-server), vsak dobi svojo workflow datoteko. Ko pushamo spremembo samo v backend repozitorij, se zgradi in deployira samo backend — frontend in kotlin-server ostaneta nedotaknjena.


---


## 1. Docker Hub Container Registry


### 1.1 Ustvaritev računa in repozitorijev
*Avtor: Matija Dukarić*


**Kaj je Docker Hub?** Docker Hub je javni register Docker slik — podobno kot GitHub za kodo, le da shranjuje Docker slike. Ko GitHub Actions zgradi sliko, jo naloži sem. Ko VM potrebuje novo verzijo, jo prenese od sem.


**Zakaj javni repozitoriji?** Docker Hub v brezplačnem računu dovoli samo en privatni repozitorij. Za šolski projekt javni repozitoriji niso problem — Docker slike same po sebi ne vsebujejo gesel ali skrivnosti (te so v environment spremenljivkah ki se podajo ob zagonu, ne med buildom).


**Zakaj Access Token namesto gesla?** Access Token je poseben ključ z omejenimi pravicami (samo branje/pisanje slik, ne upravljanje računa). Če bi bil token odtujen (npr. uhajanje iz GitHub Secrets), ga v sekundi pobrišemo in naredimo novega — brez da bi morali menjati geslo računa. Geslo ima polne pravice, token pa ne.


**Koraki:**
1. Odpri https://hub.docker.com in ustvari račun z imenom `dropinslovenia`
2. Ustvari repozitorije:
   - **Create Repository** → ime `frontend` → Public → Create
   - **Create Repository** → ime `backend` → Public → Create
   - **Create Repository** → ime `kotlin-server` → Public → Create
3. Ustvari Access Token:
   - Klikni na avatar (zgoraj desno) → **Account Settings**
   - Levi meni → **Security** → **Access Tokens** → **New Access Token**
   - Ime: `github-actions`
   - Access permissions: `Read & Write`
   - Klikni **Generate** → **shrani token takoj** — prikazan je SAMO enkrat!


📸 *Slika: Docker Hub dashboard z vsemi 3 repozitoriji*
`[VSTAVI SLIKO TUKAJ]`


---


### 1.2 CLI ukazi za container registry
*Avtor: Maj Donko (frontend), Luka Manfreda (backend + kotlin)*


Dokumentirani ukazi in razlaga:


```bash
# Prijava na Docker Hub
# Ko te vpraša za geslo, vnesi ACCESS TOKEN (ne pravega gesla)
docker login -u dropinslovenia


# Gradnja slike z imenom in tagom
# Format: docker build -t <username>/<repozitorij>:<tag> <pot_do_konteksta>
docker build -t dropinslovenia/frontend:latest ./frontend


# Nalaganje slike na Docker Hub
# Docker Hub prepozna username/repozitorij in naloži tja
docker push dropinslovenia/frontend:latest


# Prenos slike z Docker Hub na lokalni računalnik ali VM
docker pull dropinslovenia/frontend:latest


# Pregled vseh lokalnih slik
docker images


# Brisanje lokalne slike (sprosti prostor na disku)
docker rmi dropinslovenia/frontend:latest


# Zagon containerja neposredno iz Docker Hub slike (brez lokalnega builda)
docker run -d -p 3001:3001 dropinslovenia/frontend:latest
```


**`:latest` vs `:<git-sha>` tag — zakaj oba:**


`:latest` je konvencionalni tag za zadnjo verzijo. GitHub Actions ga prepiše ob vsakem deploymentu — vedno kaže na trenutno najnovejšo sliko. Enostavno za referenco.


`:<git-sha>` je tag z Git commit hash-em (npr. `:a3f8c2e1d4b7`). Vsak commit dobi svojo edinstveno sliko ki se **nikoli ne prepiše**. S tem imamo zgodovino slik in lahko kadarkoli vrnemo na točno določen commit z `docker pull dropinslovenia/frontend:a3f8c2e`.


V GitHub Actions workflowu naložimo obe oznaki hkrati:
```yaml
tags: |
  dropinslovenia/frontend:latest
  dropinslovenia/frontend:${{ github.sha }}
```


---


### 1.3 Nalaganje slik na Docker Hub
*Avtor: Maj Donko (frontend), Luka Manfreda (backend + kotlin)*


Preden vzpostavimo avtomatizirani CI/CD pipeline, slike **ročno naložimo** na Docker Hub. Namen je dvojen: preveriti da `docker build` in `docker push` delujeta pravilno, in imeti veljavne slike na Docker Hub preden prvič zaženemo GitHub Actions.


```bash
# Prijava (vnesi access token kot geslo)
docker login -u dropinslovenia


# --- Maj: Frontend ---
docker build -t dropinslovenia/frontend:latest ./frontend
docker push dropinslovenia/frontend:latest


# --- Luka: Backend ---
docker build -t dropinslovenia/backend:latest ./backend
docker push dropinslovenia/backend:latest


# --- Luka: Kotlin server ---
docker build -t dropinslovenia/kotlin-server:latest ./kotlin-server
docker push dropinslovenia/kotlin-server:latest
```


> [Opiši morebitne težave pri buildu ali pushu in kako si jih rešil]


📸 *Slika: Terminal z uspešnim `docker push` za frontend*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Docker Hub — frontend repozitorij z naloženo `latest` sliko in časovnim žigom*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Docker Hub — backend repozitorij*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Docker Hub — kotlin-server repozitorij*
`[VSTAVI SLIKO TUKAJ]`


---


## 2. GitHub Actions Workflows


### 2.1 GitHub Secrets
*Avtor: Matija Dukarić*


**Kaj so GitHub Secrets?** GitHub Secrets so šifrirane spremenljivke shranjene v GitHub repozitoriju. V workflow YAML datotekah jih referenciramo z `${{ secrets.IME }}`. Vrednosti so šifrirane in nikoli vidne v logih — niti lastniku repozitorija po shranitvi.


**Zakaj ne direktno v YAML?** Ker so workflow datoteke v git repozitoriju in git je (v našem primeru) javen. Kdorkoli bi videl geslo ali token v plaintext YAML datoteki.


**Zakaj potrebujemo `WEBHOOK_SECRET`?** Webhook strežnik na VM mora preveriti da zahteva res prihaja iz GitHub Actions in ne od nekoga ki je uganil naš URL. HMAC podpis z deljenim secretom to zagotovi — brez pravega secreta ni mogoče ustvariti veljavnega podpisa.


Za vsak repozitorij (frontend, backend, kotlin-server) nastavi secrets:
**GitHub → Settings → Secrets and variables → Actions → New repository secret**


| Secret | Vrednost | Namen |
|--------|----------|-------|
| `DOCKERHUB_USERNAME` | `dropinslovenia` | Uporabniško ime za Docker Hub login |
| `DOCKERHUB_TOKEN` | access token iz 1.1 | Ključ za push slik (ne pravo geslo) |
| `VM_HOST` | javni IP Azure VM (npr. `20.123.45.67`) | Za webhook URL in frontend build arg |
| `VM_USERNAME` | `azureuser` | Za referenco v workflowih (info) |
| `WEBHOOK_SECRET` | output od `openssl rand -hex 32` | HMAC ključ za podpisovanje webhook zahtev |


**Generiranje webhook secreta** (enkrat, Matija):
```bash
openssl rand -hex 32
# Primer outputa: a3f8c2e1d4b756c8f0e9d2a1b3c4d5e6...
# Ta string shrani:
# 1. Kot GitHub Secret WEBHOOK_SECRET v vseh 3 repozitorijih
# 2. Kot vrednost v hooks.json na VM (sekcija 3.3)
# Morata biti IDENTIČNA!
```


📸 *Slika: GitHub Settings → Secrets stran z vidnimi imeni secretov (vrednosti so skrite)*
`[VSTAVI SLIKO TUKAJ]`


---


### 2.2 Workflow — Frontend
*Avtor: Maj Donko*


**Kaj je GitHub Actions workflow?** Workflow je YAML datoteka v `.github/workflows/` mapi repozitorija ki opisuje avtomatizirane korake. GitHub ga samodejno zazna in izvaja ob določenih dogodkih (npr. push na main).


**Struktura našega workflow-a:**


Workflow ima **dva joba** ki tečeta zaporedno:
1. `build-and-push` — zgradi Docker sliko in jo naloži na Docker Hub
2. `notify-server` — pošlje webhook sporočilo na Azure VM


**Ključna beseda `needs: build-and-push`** pove GitHub Actions da naj `notify-server` job počaka na uspešen zaključek `build-and-push` joba. Če build ne uspe (npr. napaka v kodi), se webhook ne pošlje in na VM ostane stara delujoča verzija. To je namerna odločitev — ne deployiramo pokvarjene verzije.


**`${{ github.sha }}`** je unikatni Git commit hash (npr. `a3f8c2e`). Z njim tagnemo Docker sliko da vemo točno kateri commit je v produkciji. Če bi nova verzija pokvarila aplikacijo, bi z `docker pull dropinslovenia/frontend:<stari-sha>` vrnili na prejšnjo.


**`build-args`** prenese `NEXT_PUBLIC_API_URL` med Docker buildom. Next.js ta URL **vtisne v JavaScript bundle med kompilacijo** — zato ga moramo podati takrat. Vrednost je `http://<VM_HOST>:3000/api` — javni IP Azure VM.


Datoteka: `frontend/.github/workflows/ci-cd.yml`


```yaml
name: CI/CD Frontend


on:
  push:
    branches: [ main ]


jobs:
  # JOB 1: Zgradimo Docker sliko in jo naložimo na Docker Hub
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout kode
        uses: actions/checkout@v4
        # Prenese kodo iz repozitorija na GitHub Actions runner


      - name: Prijava na Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
          # Uporabi secrets, ne plaintext vrednosti!


      - name: Gradnja in push Docker slike
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/frontend:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/frontend:${{ github.sha }}
          build-args: |
            NEXT_PUBLIC_API_URL=http://${{ secrets.VM_HOST }}:3000/api
            # VM_HOST je javni IP Azure VM — vtisne se v Next.js bundle


  # JOB 2: Obvesti VM — zažene se SAMO če je Job 1 uspel
  notify-server:
    runs-on: ubuntu-latest
    needs: build-and-push
    # needs: pomeni "čakaj na uspešen zaključek build-and-push"
    steps:
      - name: Pošlji webhook na VM
        uses: distributhor/workflow-webhook@v3
        with:
          webhook_url: http://${{ secrets.VM_HOST }}:9000/hooks/deploy-frontend
          webhook_secret: ${{ secrets.WEBHOOK_SECRET }}
          data: '{"service": "frontend"}'
          # data: je JSON payload ki ga pošljemo — deploy.sh bo prebral "service"
```


**Kako preveriti da workflow deluje:**
1. Naredi kakršno koli spremembo v kodi (npr. dodaj vrstico v README)
2. `git add . && git commit -m "test ci/cd" && git push`
3. Pojdi na GitHub → **Actions** zavihek → vidiš workflow ki se izvaja
4. Klikni na run → vidiš oba joba v realnem času z logi


📸 *Slika: GitHub Actions — uspešen frontend workflow run z obema zelenima joboma*
`[VSTAVI SLIKO TUKAJ]`


---


### 2.3 Workflow — Backend
*Avtor: Luka Manfreda*


Backend workflow je strukturno enak frontend workflowu z eno razliko — ni `build-args`. Backend (Node.js) bere environment spremenljivke dinamično ob zagonu z `process.env.KOTLIN_SERVER_URL` — ne med buildom. Zato ni treba ničesar vtiskati v sliko.


Datoteka: `backend/.github/workflows/ci-cd.yml`


```yaml
name: CI/CD Backend


on:
  push:
    branches: [ main ]


jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4


      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}


      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/backend:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/backend:${{ github.sha }}


  notify-server:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - name: Webhook na VM
        uses: distributhor/workflow-webhook@v3
        with:
          webhook_url: http://${{ secrets.VM_HOST }}:9000/hooks/deploy-backend
          webhook_secret: ${{ secrets.WEBHOOK_SECRET }}
          data: '{"service": "backend"}'
```


📸 *Slika: GitHub Actions — uspešen backend workflow run*
`[VSTAVI SLIKO TUKAJ]`


---


### 2.4 Workflow — Kotlin server
*Avtor: Matija Dukarić*


Kotlin workflow je enak backend workflowu. Posebnost: Gradle mora ob prvem buildu prenesti vse odvisnosti (~200 MB) kar traja **5–10 minut**. To je normalno — GitHub Actions runner nima predpomnjenega Gradle cache-a. Ob naslednjih buildih je enako počasen ker Docker vsakič začne od začetka v čistem okolju. Optimizacija bi bila dodati `actions/cache` za `.gradle` mapo — za to nalogo ni zahtevano.


Datoteka: `kotlin-server/.github/workflows/ci-cd.yml`


```yaml
name: CI/CD Kotlin Server


on:
  push:
    branches: [ main ]


jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4


      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}


      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/kotlin-server:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/kotlin-server:${{ github.sha }}


  notify-server:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - name: Webhook na VM
        uses: distributhor/workflow-webhook@v3
        with:
          webhook_url: http://${{ secrets.VM_HOST }}:9000/hooks/deploy-kotlin
          webhook_secret: ${{ secrets.WEBHOOK_SECRET }}
          data: '{"service": "kotlin-server"}'
```


📸 *Slika: GitHub Actions — uspešen kotlin workflow run*
`[VSTAVI SLIKO TUKAJ]`


---


### 2.5 Možni dodatni GitHub Actions workflows
*Avtor: Maj Donko*


**1. Lint workflow** — preverja kakovost kode ob vsakem pull requestu:
```yaml
name: Lint
on: [pull_request]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run lint
```
*Korist za naš projekt:* ESLint napake ujamemo preden gredo v `main` branch. Zmanjšamo število bugov v produkciji in ohranimo konsistenten stil kode v vseh treh repozitorijih.


**2. Test workflow** — poganja unit teste avtomatično ob vsakem PR-ju:
```yaml
name: Tests
on: [pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
```
*Korist za naš projekt:* Zagotavlja da novi commiti ne zlomijo obstoječih API endpointov ali komponent. Posebej koristno za backend kjer bi testirali MongoDB operacije in REST API odgovore.


**3. Security scan workflow** — preverja ranljivosti v npm odvisnostih:
```yaml
name: Security
on:
  push:
    branches: [ main ]
jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
```
*Korist za naš projekt:* `npm audit` preveri ali katera od naših npm knjižnic vsebuje znane varnostne ranljivosti (CVE). Workflow nas opozori preden ranljiva verzija gre v produkcijo.


**4. Branch protection + status checks:**
V GitHub Settings → Branches → Add rule → Require status checks before merging → izberi `lint` in `test` workflowe.
*Korist:* Direktni push na `main` je blokiran. Vsaka sprememba mora iti prek pull requesta in mora prestati lint + test. To je standardna praksa v ekipnem razvoju — nihče ne more "slučajno" pokvariti produkcije.


---


## 3. Webhook


### 3.1 Namestitev webhook strežnika
*Avtor: Matija Dukarić*


**Kaj je `webhook` paket?** `webhook` je majhen HTTP strežnik napisan v Go ki posluša na določenem portu (pri nas 9000). Ko dobi HTTP zahtevo na `/hooks/<id>`, preveri pogoje (pri nas HMAC podpis) in zažene nastavljeno skripto. Je lahek, zanesljiv in enostaven za konfigurirati.


**Zakaj ne pišemo lastnega webhook strežnika?** Bi lahko — v Node.js ali Python — ampak `webhook` paket je že preizkušena rešitev ki pravilno implementira varnostno preverjanje. Pisanje lastnega bi pomenilo tudi pisanje HMAC preverjanja, kar je bolj kompleksno in bolj ranljivo za napake.


```bash
# Na VM
sudo apt install webhook -y


# Preverimo verzijo
webhook --version
```


---


### 3.2 Deploy skripta
*Avtor: Luka Manfreda*


**Logika skripte:** Skripta sprejme ime servisa (`frontend`, `backend` ali `kotlin-server`) kot argument `$1`. `case` stavek določi ustrezno Docker sliko, ime containerja in port mapping. Nato:
1. Ustavi stari container (`docker stop`) — `2>/dev/null` tišimo napake v primeru da container sploh ne teče
2. Odstrani stari container (`docker rm`) — container mora biti odstranjen preden zaženemo novega z istim imenom
3. Prenese novo sliko z Docker Hub (`docker pull`) — vedno potegne `latest` tag, ki je bil ravno posodobljen z GitHub Actions
4. Zažene nov container z `--restart unless-stopped` — ostane po rebootu VM
5. Vse korake zapiše v `/var/log/deploy.log` z datumom in uro za sledenje


**Zakaj `--env-file` in ne direktno spremenljivke?** `.env` datoteka na VM vsebuje MongoDB URI, JWT secret itd. Z `--env-file` te spremenljivke naložimo v container brez da bi jih pisali v skripto (kjer bi bile vidne v logih ali v kodi).


Datoteka: `/opt/deploy/deploy.sh`


```bash
#!/bin/bash
# Deploy skripta — zažene se ob vsakem webhook klicu
# Argument $1: ime servisa (frontend, backend, kotlin-server)


SERVICE=$1
DOCKERHUB_USER="dropinslovenia"
ENV_FILE="/home/azureuser/.env"
LOG_FILE="/var/log/deploy.log"


echo "========================================" >> $LOG_FILE
echo "[$(date)] Deploy zahtevano za: $SERVICE" >> $LOG_FILE


# Določimo parametre glede na servis
case $SERVICE in
  "frontend")
    IMAGE="$DOCKERHUB_USER/frontend:latest"
    CONTAINER="frontend"
    PORT_MAPPING="3001:3001"
    ;;
  "backend")
    IMAGE="$DOCKERHUB_USER/backend:latest"
    CONTAINER="backend"
    PORT_MAPPING="3000:3000"
    ;;
  "kotlin-server")
    IMAGE="$DOCKERHUB_USER/kotlin-server:latest"
    CONTAINER="kotlin-server"
    PORT_MAPPING="8080:8080"
    ;;
  *)
    echo "[$(date)] NAPAKA: Neznan servis '$SERVICE'" >> $LOG_FILE
    exit 1
    ;;
esac


# Ustavimo in odstranimo stari container
# 2>/dev/null: ne prikazuj napak če container ne obstaja (ob prvem deployu)
echo "[$(date)] Ustavljam stari container: $CONTAINER" >> $LOG_FILE
docker stop $CONTAINER 2>/dev/null || echo "[$(date)] Container ni tekel" >> $LOG_FILE
docker rm $CONTAINER 2>/dev/null || echo "[$(date)] Container ni obstajal" >> $LOG_FILE


# Potegnemo najnovejšo sliko z Docker Hub
echo "[$(date)] Prenašam novo sliko: $IMAGE" >> $LOG_FILE
docker pull $IMAGE


# Zaženemo nov container
echo "[$(date)] Zaganjam nov container: $CONTAINER" >> $LOG_FILE
docker run -d \
  --name $CONTAINER \
  -p $PORT_MAPPING \
  --env-file $ENV_FILE \
  --restart unless-stopped \
  $IMAGE


echo "[$(date)] Deploy uspešno zaključen: $SERVICE" >> $LOG_FILE
```


**Namestitev na VM:**
```bash
# Ustvarimo mapo in datoteko
sudo mkdir -p /opt/deploy
sudo nano /opt/deploy/deploy.sh
# (prilepimo vsebino zgoraj)


# Naredimo skripto izvršljivo
sudo chmod +x /opt/deploy/deploy.sh


# Ustvarimo log datoteko z ustreznimi pravicami
sudo touch /var/log/deploy.log
sudo chmod 666 /var/log/deploy.log


# Ročni test skripte (preden testiramo cel pipeline)
/opt/deploy/deploy.sh backend
# Počakaj ~30 sekund
docker ps
# Mora prikazati backend container z svežim CREATED časom
cat /var/log/deploy.log
# Mora prikazati deploy loge
```


📸 *Slika: Ročni test skripte — terminal z outputom in `docker ps` po zagonu*
`[VSTAVI SLIKO TUKAJ]`


---


### 3.3 Webhook konfiguracija in systemd servis
*Avtor: Matija Dukarić (konfiguracija), Maj Donko (systemd servis)*


**Struktura `hooks.json`:** Vsak hook objekt ima:
- `id` — del URL poti (`/hooks/<id>`). GitHub Actions pošlje zahtevo na ta URL.
- `execute-command` — skripta ki se zažene ob uspešnem klicu
- `pass-arguments-to-command` — prenese vrednost `service` iz JSON payload-a kot argument `$1` skripti. Payload je `{"service": "frontend"}` ki ga pošlje GitHub Actions.
- `trigger-rule` z HMAC-SHA256 — webhook se sproži SAMO če je zahteva podpisana s pravilnim secretom. GitHub Actions podpiše vsak request z `WEBHOOK_SECRET` — webhook strežnik preveri ta podpis.


**Zakaj systemd?** Webhook strežnik mora teči ves čas, tudi po rebootu VM. `systemctl enable webhook` registrira servis za samodejni zagon ob startu sistema. `Restart=always` zagotovi da se servis restarata ob morebitni napaki.


**Datoteka `/etc/webhook/hooks.json`:**


```json
[
  {
    "id": "deploy-frontend",
    "execute-command": "/opt/deploy/deploy.sh",
    "command-working-directory": "/home/azureuser",
    "pass-arguments-to-command": [
      { "source": "payload", "name": "service" }
    ],
    "trigger-rule": {
      "match": {
        "type": "payload-hmac-sha256",
        "secret": "VAŠ_WEBHOOK_SECRET_TUKAJ",
        "parameter": {
          "source": "header",
          "name": "X-Hub-Signature-256"
        }
      }
    }
  },
  {
    "id": "deploy-backend",
    "execute-command": "/opt/deploy/deploy.sh",
    "command-working-directory": "/home/azureuser",
    "pass-arguments-to-command": [
      { "source": "payload", "name": "service" }
    ],
    "trigger-rule": {
      "match": {
        "type": "payload-hmac-sha256",
        "secret": "VAŠ_WEBHOOK_SECRET_TUKAJ",
        "parameter": {
          "source": "header",
          "name": "X-Hub-Signature-256"
        }
      }
    }
  },
  {
    "id": "deploy-kotlin",
    "execute-command": "/opt/deploy/deploy.sh",
    "command-working-directory": "/home/azureuser",
    "pass-arguments-to-command": [
      { "source": "payload", "name": "service" }
    ],
    "trigger-rule": {
      "match": {
        "type": "payload-hmac-sha256",
        "secret": "VAŠ_WEBHOOK_SECRET_TUKAJ",
        "parameter": {
          "source": "header",
          "name": "X-Hub-Signature-256"
        }
      }
    }
  }
]
```


> Zamenjaj `VAŠ_WEBHOOK_SECRET_TUKAJ` z istim stringom ki si ga vpisal v GitHub Secret `WEBHOOK_SECRET`.


```bash
sudo mkdir -p /etc/webhook
sudo nano /etc/webhook/hooks.json
# (prilepi vsebino zgoraj, zamenjaj secret)
```


**Datoteka `/etc/systemd/system/webhook.service`:**


```ini
[Unit]
Description=Webhook Server za DropInSlovenia deploy
After=network.target docker.service
# After: zaženi webhook šele ko je omrežje in Docker pripravljeno


[Service]
User=azureuser
ExecStart=/usr/bin/webhook -hooks /etc/webhook/hooks.json -port 9000 -verbose
# -verbose: izpisuje loge vsakega klica — koristno za debugging
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
# journal: logi gredo v systemd journal (berljivo z: journalctl -u webhook)


[Install]
WantedBy=multi-user.target
```


```bash
sudo nano /etc/systemd/system/webhook.service
# (prilepi vsebino zgoraj)


# Reload systemd da zazna novo datoteko
sudo systemctl daemon-reload


# Omogoči samodejni zagon ob startu
sudo systemctl enable webhook


# Zaženi takoj
sudo systemctl start webhook


# Preveri status
sudo systemctl status webhook
# Mora pisati: Active: active (running)
```


**Test webhook strežnika** (katerikoli član s svojega računalnika):
```bash
curl http://<PUBLIC_IP>:9000/hooks/deploy-backend
# Pričakovan odgovor: "Hook rules were not satisfied."
# To je OK! Pomeni da hook deluje, ampak zahteva pravilni HMAC podpis.
# Napačen podpis (ali brez podpisa) = zavrnjena zahteva
```


📸 *Slika: `systemctl status webhook` — active (running)*
`[VSTAVI SLIKO TUKAJ]`


---


### 3.4 Test celotnega pipeline
*Avtor: Maj Donko, Luka Manfreda, Matija Dukarić*


Ko je vse nastavljeno, testiramo celoten tok od kode do produkcije. Vsak član naredi commit in push v svojem repozitoriju.


**Potek testa korak za korakom:**


```
1. Naredi spremembo in pushaj (katerikoli repozitorij)
   git add .
   git commit -m "test: CI/CD pipeline test"
   git push origin main


2. Opazuj GitHub Actions (https://github.com/DropInSlovenia/<repo>/actions)
   - Vidiš workflow ki se je sprožil
   - Job 1 (build-and-push): ~3 minute (Kotlin ~10 min)
   - Job 2 (notify-server): ~10 sekund


3. Preveri Docker Hub (https://hub.docker.com/u/dropinslovenia)
   - Odpri ustrezni repozitorij
   - Timestamp "Last pushed" mora biti svež (pred minutami)


4. Preveri na VM:
   ssh azureuser@<PUBLIC_IP>
   cat /var/log/deploy.log        # Svež deploy log z aktualnim časom
   docker ps                       # CREATED čas mora biti svež (pred minutami)
```


**Odpravljanje težav:**
- Če GitHub Actions workflow ne uspe: klikni na rdeči X, preberi log napake
- Če webhook ne pride do VM: preveri `sudo systemctl status webhook` na VM, preveri NSG pravilo za port 9000
- Če deploy skripta ne zažene containerja: preberi `/var/log/deploy.log` za podrobnosti


📸 *Slika: GitHub Actions — oba joba zelena (build + notify)*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: Docker Hub — nova slika z novim časovnim žigom po pushu*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: `/var/log/deploy.log` na VM — vidni deploy logi*
`[VSTAVI SLIKO TUKAJ]`


📸 *Slika: `docker ps` na VM — container ima svež CREATED časovni žig*
`[VSTAVI SLIKO TUKAJ]`


---


## 4. Varnost pri Webhookih
*Avtor dokumentacije: Maj Donko | Avtor implementacije: Matija Dukarić*


### 4.1 Identificirane varnostne luknje


**Luknja 1: Webhook URL je javno dostopen brez avtentikacije**
> Kdorkoli ki pozna IP in port 9000 lahko pošlje HTTP zahtevo. Brez zaščite bi napadalec s primerno zahtevo sprožil deploy kadarkoli — kar bi povzročilo restart servisov ali naložitev zlonamerne slike.
*Rešitev:* HMAC-SHA256 podpis — GitHub podpiše vsak request s secretom, webhook strežnik podpis preveri preden zažene skripto. Brez veljavnega podpisa se skripta ne izvede. **Že implementirano v `hooks.json`.**


**Luknja 2: HTTP namesto HTTPS**
> Webhook komunikacija poteka po nešifriranem HTTP. Napadalec na omrežju med GitHub in VM (man-in-the-middle) bi videl vsebino zahtev. Čeprav je payload podpisan z HMAC (kar ščiti pred manipulacijo), bi napadalec videl kdaj in za kateri servis se sprožijo deployi — koristna informacija za načrtovanje napada.
*Rešitev:* Dodati TLS certifikat prek nginx reverse proxy in Let's Encrypt (`certbot`). Za to nalogo ni zahtevano, je pa priporočljivo v produkciji.


**Luknja 3: Deploy skripta teče s pravicami `azureuser` ki ima dostop do `.env`**
> Skripta bere `/home/azureuser/.env` ki vsebuje MongoDB URI in JWT secret. Napadalec ki bi uspešno sprožil webhook bi izvedel skripto ki dostopa do vseh teh skrivnosti. Prav tako ima `azureuser` dostop do Dockerja — s privileji Dockerja je mogoče eskalirati na root.
*Rešitev:* Ločen `deployer` user z minimalnimi pravicami:
```bash
sudo useradd -r -s /bin/bash deployer
sudo usermod -aG docker deployer
# Webhook servis poženemo kot 'deployer' — spremenimo User= v webhook.service
```


**Luknja 4: Webhook secret je v `hooks.json` v čistem tekstu**
> Vsakdo z SSH dostopom do VM (vsi 3 člani) vidi secret. Hkrati je datoteka berljiva z `sudo cat` brez posebnih pravic.
*Rešitev:* Branje secreta iz environment spremenljivke:
```bash
# V /etc/systemd/system/webhook.service dodamo:
Environment=WEBHOOK_SECRET=vrednost_iz_openssl
# V hooks.json pa spremenimo:
# "secret": "${WEBHOOK_SECRET}"
# Tako se secret ne shranjuje v nobeni konfiguraciji datoteki
```


---


### 4.2 Implementirane varnostne rešitve


**HMAC-SHA256 podpis (implementirano v hooks.json):**


HMAC (Hash-based Message Authentication Code) deluje takole:
1. GitHub Actions zna naš `WEBHOOK_SECRET`
2. Ob vsakem webhook klicu GitHub izračuna SHA-256 hash payload-a XOR z secretom
3. Ta podpis doda v `X-Hub-Signature-256` HTTP header
4. Webhook strežnik na VM izračuna isti podpis in ga primerja
5. Če se podpisa ujemata → zahteva je legitimna, skripta se zažene
6. Če se ne ujemata → zahteva se zavrne, skripta se NE zažene


Brez poznavanja secreta je nemogoče ustvariti veljaven podpis — napad je matematično nemogoč.


**UFW Firewall — omejitev porta 9000 samo na GitHub IP naslove (implementirano):**


```bash
# Najprej preverimo da je UFW nameščen
sudo apt install ufw -y


# Najprej dovolimo SSH (da se ne zaključimo sami ven!)
sudo ufw allow 22
sudo ufw allow 3000
sudo ufw allow 3001
sudo ufw allow 8080


# Zapri port 9000 za vse
sudo ufw deny 9000


# Dovoli samo GitHub Actions IP range
# (IP naslovi ki jih GitHub Actions runners dejansko uporabljajo)
sudo ufw allow from 140.82.112.0/20 to any port 9000
sudo ufw allow from 185.199.108.0/22 to any port 9000
sudo ufw allow from 192.30.252.0/22 to any port 9000


# Aktiviraj UFW
sudo ufw --force enable


# Preveri pravila
sudo ufw status verbose
```


> **Opomba:** GitHub IP naslovi se občasno spremenijo. Aktualni seznam je vedno na https://api.github.com/meta pod `actions` ključem. Ob morebitnih problemih s webhookom v prihodnosti preveri ali so IP-ji še veljavni.


📸 *Slika: `sudo ufw status` z vidnimi firewall pravili*
`[VSTAVI SLIKO TUKAJ]`


---


## Morebitne težave in rešitve


| Težava | Vzrok | Rešitev |
|--------|-------|---------|
| GitHub Actions workflow ne uspe pri `build-and-push` | Napačen `DOCKERHUB_TOKEN` secret | Regeneriraj access token na Docker Hub in posodobi secret |
| Webhook ne pride do VM (job 2 rdeč) | Port 9000 zaprt v NSG ali webhook servis ne teče | Preveri NSG pravilo za port 9000; `systemctl status webhook` na VM |
| Deploy skripta ne zamenja containerja | Container z istim imenom že teče in `docker run` vrne napako | Preveri da `docker stop` in `docker rm` uspešno delujeta ročno |
| `docker pull` v deploy skripti vrne `unauthorized` | VM ni prijavljen v Docker Hub | Za javne slike `docker pull` ne zahteva prijave — preveri ime slike |
| `.env` manjka na VM po rebootu | `.env` je bil ustvarjen ročno in se ne obnovi samodejno | Ustvari `/home/azureuser/.env` ročno po vsakem rebootu VM ali shrani v varno lokacijo |
| Kotlin build traja >15 minut v GitHub Actions | Gradle prenaša vse odvisnosti vsakič znova | Dodaj `actions/cache` za `.gradle` mapo v workflow — opcijsko |
| [Opiši svojo težavo] | [Opiši vzrok] | [Opiši rešitev] |


---


*Poročilo P3 — DropInSlovenia | Maj 2026*
