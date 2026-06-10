# DropInSlovenia — Dokumentacija programske rešitve

> **Kako uporabljati ta dokument**
> - Vsi diagrami so napisani v **Mermaid** DSL. Na **GitHubu** (in v VS Code z
>   razširitvijo *Markdown Preview Mermaid*) se izrišejo samodejno. Za oddajo v PDF/Word
>   lahko diagrame izvozite kot slike prek <https://mermaid.live> (prilepiš kodo → Export PNG/SVG).
> - Ganttov diagram je izdelan iz **dejanskega Jira exporta** (`Jira.xml`, 141 zadev,
>   projekt SCRUM/DropInSlovenia); naloge so strnjene po epicih v fazni plan.
> - Razredni diagrami so izpeljani **neposredno iz izvorne kode** (backend, frontend,
>   PrincipiProjekt/desktopApp) — polja, metode in relacije ustrezajo dejanskemu stanju.
> - Tehnična referenca z vso vsebino projekta je v ločenem fajlu
>   [`PROJEKT-REFERENCA.md`](./PROJEKT-REFERENCA.md).

---

## Kazalo

1. [Prva stran](#1-prva-stran)
2. [Primeri uporabe](#2-primeri-uporabe)
3. [Arhitektura programske rešitve](#3-arhitektura-programske-rešitve)
4. [DevOps (CI/CD)](#4-devops-cicd)
5. [Varnost programske rešitve](#5-varnost-programske-rešitve)

---
---

# 1. Prva stran

<div align="center">

# 🇸🇮 DropInSlovenia

### Spletna aplikacija za odkrivanje, načrtovanje in vodenje izletov po Sloveniji

**Projektna dokumentacija**

</div>

<br>

### Člani skupine

| # | Ime in priimek | Vloga |
|---|---|---|
| 1 | **Matija Dukarić** | Vodja skupine (odda nalogo); backend (Trip API, WS), MongoDB Atlas, Azure VM, docker-compose, Docker Hub, webhook/systemd, Kotlin admin (Trips, Generator) |
| 2 | **Maj Donko** | Frontend (Next.js — auth, mapa, paneli, profil), Kotlin admin (Users UI), GitHub Actions frontend, Azure NSG/port forwarding, UFW dokumentacija |
| 3 | **Luka Manfreda** | Kotlin scraperji + Ktor strežnik, backend (JWT, auth middleware), DSL (lexer/parser), GitHub Actions backend, deploy skripta na VM |

### Povezave do repozitorijev

Koda je organizirana v GitHub organizaciji **[DropInSlovenia](https://github.com/DropInSlovenia)**, razdeljena na tri repozitorije:

| Komponenta | Repozitorij |
|---|---|
| Spletna aplikacija (frontend) | <https://github.com/DropInSlovenia/webApp> |
| Zaledni strežnik (backend) | <https://github.com/DropInSlovenia/backend> |
| Kotlin (scraping strežnik + namizna admin aplikacija) | <https://github.com/DropInSlovenia/desktopApp> |

### Ganttov diagram

Diagram prikazuje potek projekta **od vzpostavitve (marec 2026) do konca 3. letnika
(junij 2027)**. Faze 2. letnika so strnjene po Jira epicih (SCRUM-5 backend,
SCRUM-11 namizna aplikacija, SCRUM-28 setup, SCRUM-88 infrastruktura, SCRUM-89 DSL,
SCRUM-90 spletni vmesnik); datumi so vzeti iz dejanskih datumov ustvarjanja in
zaključka nalog v Jiri. Za 3. letnik je rezerviranih **10 praznih nalog**, ki jih
dopolnimo, ko bo znan obseg dela.

```mermaid
gantt
    title DropInSlovenia — časovnica projekta (2. in 3. letnik)
    dateFormat  YYYY-MM-DD
    axisFormat  %m/%y

    section Zagon projekta
    Idejna zasnova in izbira virov (SCRUM-7)              :done, vir,   2026-03-23, 2026-04-19
    Repozitoriji, Jira-GitHub, specifikacije (SCRUM-28)   :done, setup, 2026-04-16, 2026-05-08

    section Kotlin (SCRUM-11)
    Postavitev projekta in Google Places API              :done, kt1, 2026-04-06, 2026-04-23
    Scraperji dogodkov 6 mest in Wikipedia                :done, kt2, 2026-04-16, 2026-04-21
    Compose Desktop admin (Users, Trips, Scraper, Generator) :done, kt3, 2026-05-05, 2026-05-13

    section Backend (SCRUM-5)
    Okolje, User model, JWT avtentikacija                 :done, be1, 2026-04-23, 2026-05-01
    Trip model, CRUD in geo iskanje                       :done, be2, 2026-04-24, 2026-05-08
    WS živo vodenje in OSRM napotki                       :done, be3, 2026-04-24, 2026-05-17
    GeoJSON, places, popravki in izboljšave               :done, be4, 2026-05-14, 2026-05-30

    section DSL (SCRUM-89)
    BNF gramatika, lexer, parser, AST, pretty-printer     :done, dsl1, 2026-05-13, 2026-05-28
    Validator, GeoJSON exporter, OSRM, testni primeri     :done, dsl2, 2026-05-26, 2026-05-31

    section Frontend (SCRUM-90)
    Struktura, axios in JWT, auth strani, profil          :done, fe1, 2026-05-16, 2026-05-22
    Leaflet mapa, paneli, TripEditor                      :done, fe2, 2026-05-18, 2026-05-23
    WS klient, GPS streaming, LiveNavigation              :done, fe3, 2026-05-18, 2026-05-30
    UX popravki, aktivni izlet, ogledi                    :done, fe4, 2026-05-22, 2026-06-07

    section Infrastruktura in CI/CD (SCRUM-88)
    Dockerfile-i, docker-compose, lokalni test            :done, inf1, 2026-05-13, 2026-05-24
    Azure VM, SSH, NSG, swap, MongoDB Atlas               :done, inf2, 2026-05-13, 2026-05-24
    Docker Hub, GitHub Actions, webhook deploy, UFW       :done, inf3, 2026-05-28, 2026-06-07

    section Zaključek 2. letnika
    Dokumentacija in zagovor                              :active, doc, 2026-06-08, 2026-06-26

    section 3. letnik (rezervirano)
    Naloga 1 (dopolni)                                    :t1,  2026-10-01, 21d
    Naloga 2 (dopolni)                                    :t2,  2026-10-22, 21d
    Naloga 3 (dopolni)                                    :t3,  2026-11-12, 21d
    Naloga 4 (dopolni)                                    :t4,  2026-12-03, 21d
    Naloga 5 (dopolni)                                    :t5,  2027-01-07, 21d
    Naloga 6 (dopolni)                                    :t6,  2027-01-28, 21d
    Naloga 7 (dopolni)                                    :t7,  2027-02-18, 21d
    Naloga 8 (dopolni)                                    :t8,  2027-03-11, 21d
    Naloga 9 (dopolni)                                    :t9,  2027-04-01, 21d
    Naloga 10 (dopolni)                                   :t10, 2027-04-22, 21d
    Zaključek projekta in končni zagovor                  :milestone, konec, 2027-06-15, 0d
```

<!--
Opombe za vzdrževanje Gantta:
  • Faza "Idejna zasnova" se je začela pred prvim Jira vnosom (Jira export pokriva
    zadnjih 90 dni); začetni datum 2026-03-23 po potrebi prilagodi.
  • Nalogo 1–10 v sekciji "3. letnik" preimenuj, ko bodo znane naloge naslednjega leta.
  • Za sliko: https://mermaid.live → prilepi kodo → Actions → Export PNG/SVG.
-->

---
---

# 2. Primeri uporabe

## 2.1 Opis problema

Informacije, potrebne za načrtovanje izleta po Sloveniji — destinacije, nastanitve,
gostinska ponudba, znamenitosti in **aktualni dogodki** — so razpršene po številnih
ločenih virih (turistične platforme posameznih mest, Google, Wikipedija, zemljevidi).
Uporabnik mora podatke ročno združevati iz več strani, med samim izletom pa ni enotnega
orodja, ki bi ga **vodilo po načrtovani poti** in ga sproti obveščalo o dogajanju v bližini.

**DropInSlovenia** ta problem rešuje z eno aplikacijo, ki: (a) agregira točke interesa in
dogodke na interaktivnem zemljevidu, (b) omogoča sestavo večdnevnega potovanja s
postajami, in (c) med izletom v živo (prek GPS) izračunava pot, ETA in navodila do
naslednje postaje ter predlaga bližnje točke interesa.

### Matematična podlaga

Jedro rešitve temelji na nekaj geoprostorskih in optimizacijskih izračunih:

**1) Razdalja med dvema GPS točkama — Haversinova formula**
Uporabljena za izračun oddaljenosti točk interesa od uporabnika in za filter premika
(`min_moved`), ki prepreči odvečne poizvedbe.
(`backend/services/nearbyPlacesService.js`)

$$
a = \sin^2\!\left(\frac{\varphi_2 - \varphi_1}{2}\right) + \cos\varphi_1 \cdot \cos\varphi_2 \cdot \sin^2\!\left(\frac{\lambda_2 - \lambda_1}{2}\right)
$$

$$
d = 2R \cdot \operatorname{atan2}\!\left(\sqrt{a},\ \sqrt{1-a}\right)
$$

kjer je $R = 6\,371\,000\ \text{m}$ (polmer Zemlje), $\varphi$ geografska širina in
$\lambda$ geografska dolžina (v radianih).

**2) Iskanje potovanj v bližini — geoprostorska poizvedba**
Potovanje je vključeno v rezultat, če ima vsaj eno postajo $s$, za katero velja:

$$
\exists\, s \in \text{stops} : d(\text{uporabnik}, s) \le r_{\max}
$$

Izvedeno z MongoDB operatorjem `$near` nad `2dsphere` indeksom (privzeti $r_{\max} = 50\ \text{km}$).

**3) Najkrajša pot in ocenjeni čas prihoda (ETA)**
Pot med trenutno lokacijo in naslednjo postajo reši OSRM (Dijkstra/Contraction
Hierarchies nad cestnim grafom $G=(V,E)$ z utežmi $w$):

$$
\text{ETA} = \min_{P \in \text{poti}(u \to v)} \sum_{e \in P} w(e)
$$

Če je $\text{ETA} > 30\ \text{min}$, sistem sproži priporočilo bližnjih postankov (F14).

## 2.2 Primeri uporabe (sekvenčni diagrami)

> Diagrami so v Mermaid (`sequenceDiagram`). Za sliko: mermaid.live → Export.

### UC-1 — Registracija in prijava uporabnika (JWT)

Pokriva **F10**. Prikazuje preverjanje gesla (bcrypt) in izdajo para žetonov
(access + refresh).

```mermaid
sequenceDiagram
    actor U as Uporabnik
    participant FE as Frontend (Next.js)
    participant BE as Backend (Express)
    participant DB as MongoDB Atlas

    U->>FE: Vnese email in geslo
    FE->>BE: POST /api/auth/login {email, password}
    BE->>DB: User.findByEmail(email) (+passwordHash)
    DB-->>BE: Uporabniški dokument
    BE->>BE: bcrypt.compare(geslo, passwordHash)
    alt Geslo napačno ali uporabnik ne obstaja
        BE-->>FE: 401 { error: "Napačen email ali geslo" }
        FE-->>U: Prikaz napake
    else Račun deaktiviran (isActive = false)
        BE-->>FE: 403 { error: "Račun je deaktiviran" }
        FE-->>U: Prikaz napake
    else Uspešna prijava
        BE->>BE: Generiraj accessToken (15 min) + refreshToken (7 dni)
        BE->>DB: Shrani refreshToken k uporabniku
        BE-->>FE: 200 { accessToken, refreshToken, user }
        FE->>FE: writeToken(accessToken), authStore.setUser(user)
        FE-->>U: Preusmeritev v aplikacijo (zemljevid)
    end
```

### UC-2 — Iskanje bližnjih točk interesa na zemljevidu

Pokriva **F02, F03, F04**. Prikazuje agregacijo POI iz OpenStreetMap (Overpass) z
Haversine razvrščanjem in odpornostjo (fallback endpoint).

```mermaid
sequenceDiagram
    actor U as Uporabnik
    participant FE as Frontend (Leaflet)
    participant BE as Backend
    participant OV as Overpass / OSM

    U->>FE: Klik / premik na zemljevidu (lat, lon)
    FE->>BE: GET /api/nearby?lat&lon&categories&radius
    BE->>BE: parseNearbyConfig() + buildQuery() (Overpass QL)
    BE->>OV: POST data=<query> (primarni endpoint)
    alt Endpoint nedosegljiv (429/5xx/timeout)
        BE->>OV: POST <query> (rezervni endpoint)
    end
    OV-->>BE: Seznam elementov (POI z oznakami)
    BE->>BE: Haversine razdalja → sort → limit
    BE-->>FE: 200 { meta, places[] }
    FE-->>U: Izris markerjev POI na zemljevidu
```

### UC-3 — Vodenje v živo z navigacijo (WebSocket)

Pokriva **F11, F13, F14**. Prikazuje WebSocket sejo s preverjanjem JWT ob *upgrade*,
ponavljajoče se posodabljanje lokacije in priporočila ob dolgi vožnji.

```mermaid
sequenceDiagram
    actor U as Uporabnik
    participant FE as Frontend (LiveNavigation)
    participant BE as Backend (ws)
    participant DB as MongoDB
    participant OSRM as OSRM
    participant OV as Overpass

    U->>FE: Klikne "Začni izlet"
    FE->>BE: WS connect /live?token=JWT
    BE->>BE: verifyAccessToken() ob HTTP upgrade
    BE-->>FE: welcome
    FE->>BE: { type: "start_trip", tripId, profile }
    BE->>DB: Naloži Trip, nastavi activeSession.isActive = true
    BE-->>FE: start_trip { firstStop, currentStopIndex, totalStops }

    loop Ob vsakem GPS popravku
        FE->>BE: location_update { tripId, location:[lon,lat] }
        BE->>OSRM: route(trenutna lokacija → naslednja postaja)
        OSRM-->>BE: razdalja, trajanje (ETA), navodila
        BE-->>FE: location_update { nextStop, route, ETA }
        opt ETA > 30 min
            BE->>OV: Bližnji POI (postanki)
            OV-->>BE: places[]
            BE-->>FE: suggestion { places }
        end
    end

    U->>FE: Klikne "Obiskano"
    FE->>BE: mark_stop_visited { stopId }
    BE->>DB: stop.visitedAt = now, currentStopIndex++
    BE-->>FE: stop_visited { nextStop, newCurrentStopIndex, isLastStop }
    FE-->>U: Posodobi markerje + obvestilo
```

---
---

# 3. Arhitektura programske rešitve

## 3.1 Diagram arhitekture

Sistem je sestavljen iz **treh komponent** in se povezuje z več zunanjimi viri.

```mermaid
graph TB
    subgraph Klienti
        BR["🌐 Brskalnik<br/>(končni uporabnik)"]
        DT["🖥️ Compose Desktop<br/>(admin orodje)"]
    end

    subgraph "Frontend — webApp"
        FE["Next.js 16 + React 19<br/>Leaflet zemljevid<br/>Zustand · axios · WebSocket<br/>port 3001"]
    end

    subgraph "Backend — Node.js"
        BE["Express 4 (REST)<br/>ws 8 (WebSocket /live)<br/>JWT · helmet · rate-limit<br/>port 3000"]
    end

    subgraph "Kotlin — desktopApp"
        KT["Ktor 3 + Netty<br/>scraping API<br/>port 8080"]
    end

    subgraph "Zunanji viri"
        DB[("MongoDB Atlas<br/>2dsphere indeks")]
        OSRM["OSRM<br/>routing"]
        OV["Overpass / OSM<br/>POI"]
        GP["Google Places API"]
        WIKI["Wikipedia REST"]
        TUR["Turistične platforme<br/>(Visit*, Postojnsko)"]
    end

    BR -->|"HTTP REST (JSON)"| FE
    BR -.->|"WebSocket /live"| BE
    FE -->|"HTTP REST + JWT"| BE
    DT -->|"HTTP REST + JWT (OkHttp)"| BE

    BE -->|"Mongoose (TLS)"| DB
    BE -->|"axios HTTP"| OSRM
    BE -->|"axios HTTP"| OV
    BE -->|"axios HTTP"| KT

    KT -->|"HTTPS"| GP
    KT -->|"HTTPS"| WIKI
    KT -->|"Selenium + Ksoup"| TUR
```

## 3.2 Tehnologije

### Programski jeziki in izvajalna okolja

| Komponenta | Jezik | Prevajalnik / izvajalnik | Spletni strežnik |
|---|---|---|---|
| Frontend | **TypeScript** (TSX) | `tsc` / SWC (Next.js build) | Next.js standalone (Node.js), **port 3001** |
| Backend | **JavaScript** (Node.js, CommonJS) | interpreter **Node.js v22** | Express 4 + lasten `http` strežnik + `ws`, **port 3000** |
| Kotlin scraping | **Kotlin** (JVM) | `kotlinc` prek Gradle 8.14 | **Ktor 3 + Netty**, **port 8080** |
| Kotlin admin | **Kotlin** (JVM) | `kotlinc` prek Gradle | Compose for Desktop (ni strežnik) |

### Podatkovna baza

- **MongoDB** (gostovan na **MongoDB Atlas**), dostop prek **Mongoose 9** (ODM).
- Dve zbirki: `users` in `trips` (z vgnezdenimi `stops`).
- Geoprostorski **`2dsphere`** indeks na `stops.location` omogoča `$near` poizvedbe;
  dodatni indeksi na `tags`, `owner`, `(isPublic, startDate)`.

### Komunikacijski protokoli TCP/IP sklada

| Sloj | Protokol | Uporaba |
|---|---|---|
| Aplikacijski | **HTTP/1.1** (REST, JSON) | brskalnik/desktop ↔ backend ↔ Kotlin/zunanji viri |
| Aplikacijski | **WebSocket** (`ws://…/live`) | živo vodenje (dvosmerno, nizka latenca) |
| Aplikacijski | **HTTPS / TLS** | klici Google Places, Wikipedia, OSRM, Atlas |
| Aplikacijski | **MongoDB wire protocol** (prek TLS, `mongodb+srv`) | backend ↔ baza |
| Transportni | **TCP** | vsi zgornji |

### Odprta vrata (porti)

| Vrata | Storitev | Opomba |
|---|---|---|
| **3001** | Frontend (Next.js) | dostop končnega uporabnika |
| **3000** | Backend (HTTP REST **in** WebSocket) | isti port za REST in `/live` |
| **8080** | Kotlin Ktor scraping API | interni; kliče ga le backend |
| **9000** | `webhook` na produkcijski VM | sproži samodejni deploy (CI) |
| 27017 / 443 | MongoDB Atlas (`mongodb+srv`) | izhodna povezava backenda |
| 443 | OSRM, Overpass, Google Places, Wikipedia | izhodne povezave |

## 3.3 Knjižnice in API-ji

### Zunanji API-ji (kateri problem rešuje)

| API / vir | Komponenta | Problem, ki ga rešuje |
|---|---|---|
| **MongoDB Atlas** | Backend | Trajno shranjevanje; geoprostorske poizvedbe (`$near`). |
| **OSRM** | Backend | Izračun poti, razdalje, ETA in navigacijskih navodil. |
| **Overpass / OpenStreetMap** | Backend | Iskanje bližnjih POI po kategorijah — brezplačno, brez ključa. |
| **Google Places API** | Kotlin | Bogatejši podatki o krajih (ocene, naslov, fotografije, mnenja). |
| **Wikipedia REST API** | Kotlin | Kratek opis mesta (F15). |
| **Turistične platforme** | Kotlin | Aktualni dogodki po mestih (nimajo javnega API-ja → scraping). |

### Knjižnice po jezikih (in zakaj)

**Backend (JavaScript / Node.js):**

| Knjižnica | Zakaj / kateri problem rešuje |
|---|---|
| `express` | Minimalno, razširljivo REST ogrodje; de-facto standard za Node.js. |
| `mongoose` | ODM z validacijo shem, hooki (hash gesla, kaskadno brisanje) in indeksi. |
| `jsonwebtoken` | Stateless avtentikacija (access + refresh) brez sejne shrambe. |
| `bcryptjs` | Varno shranjevanje gesel (salt + hash, cost 12). |
| `ws` | Lahek WebSocket strežnik za živo vodenje, neodvisno od Express. |
| `helmet`, `cors`, `express-rate-limit` | Varnostni sloj (glavni, izvor, omejevanje zahtev). |
| `axios` | HTTP klient za klice zunanjih storitev. |
| `cheerio` | HTML parser (rezerva za scraping na Node strani). |
| `morgan`, `dotenv` | Logiranje zahtev; konfiguracija prek `.env`. |

**Frontend (TypeScript / Next.js):**

| Knjižnica | Zakaj / kateri problem rešuje |
|---|---|
| `next`, `react` | SSR/komponentno ogrodje (App Router, React 19). |
| `leaflet` + `react-leaflet` | Odprtokoden interaktivni zemljevid (brez plačljivih kvot). |
| `zustand` | Minimalen state management (4 storei po domeni). |
| `axios` | Enoten HTTP klient z interceptorji (JWT + samodejni refresh ob 401). |
| `tailwindcss` | Utility-first CSS za hiter razvoj UI. |

**Kotlin (JVM):**

| Knjižnica | Zakaj / kateri problem rešuje |
|---|---|
| `ktor-server` (+Netty) | Lahek async strežnik za scraping API. |
| `selenium-java` | Scraping JS-renderiranih turističnih strani (dinamična vsebina). |
| `ksoup` | Parsiranje HTML v Kotlinu (Jsoup-podoben). |
| `kotlinx-serialization-json` | Tipiziran JSON (`data class` ↔ JSON). |
| `okhttp` | HTTP klient admin aplikacije do backenda (s timeouti). |
| `kotlinx-coroutines` | Neblokirajoči klici — odziven UI. |
| `compose-multiplatform` / `material3` | Deklarativni namizni UI. |
| `kotlin-faker` | Generiranje realističnih testnih podatkov. |

## 3.4 Razredni diagrami

Ker projekt uporablja **tri programske jezike** (JavaScript, TypeScript, Kotlin),
podajamo ločen razredni diagram za vsakega. Diagrami so izpeljani neposredno iz
izvorne kode in prikazujejo dejanska polja, metode in relacije.

### 3.4.1 Backend (Node.js — Mongoose modeli, servisi, middleware)

Backend je modulski (CommonJS); servisi in middleware so prikazani kot razredi z
javnimi funkcijami, ki jih modul izvaža (`module.exports`).

```mermaid
classDiagram
    class User {
        <<Mongoose model>>
        +ObjectId _id
        +String email
        +String passwordHash
        +String displayName
        +String profilePicture
        +String refreshToken
        +String role
        +Boolean isActive
        +ObjectId[] trips
        +Date createdAt
        +Date updatedAt
        +comparePassword(candidatePassword) Boolean
        +findByEmail(email) User$
    }

    class Trip {
        <<Mongoose model>>
        +ObjectId _id
        +String title
        +String description
        +ObjectId owner
        +Stop[] stops
        +String[] tags
        +Boolean isPublic
        +Number viewCount
        +Number durationDays
        +String startDate
        +String endDate
        +ActiveSession activeSession
    }

    class Stop {
        <<podshema>>
        +ObjectId _id
        +String city
        +GeoPoint location
        +String description
        +Number dayNumber
        +Number order
        +String arrivalTime
        +String departureTime
        +Date visitedAt
        +String[] tags
    }

    class ActiveSession {
        +Boolean isActive
        +Date startedAt
        +Number currentStopIndex
        +String travelProfile
    }

    class GeoPoint {
        +String type
        +Number[] coordinates
    }

    class TokenService {
        <<servis>>
        +generateAccessToken(payload) String
        +generateRefreshToken(payload) String
        +verifyAccessToken(token) Object
        +verifyRefreshToken(token) Object
    }

    class OsrmService {
        <<servis>>
        +getRouteDirections(opts) Route
        +getTripNavigationDirections(opts) Navigation
        +getNextStop(trip, message) Stop
        +getOrderedStops(trip) Stop[]
        +parseCoordinate(location) Number[]
        +normalizeCoordinates(coordinates) Number[][]
        +normalizeProfile(profile) String
    }

    class NearbyPlacesService {
        <<servis>>
        +AMENITY_CATEGORIES Object
        +fetchNearbyPlaces(opts) Place[]
        +buildQuery(lat, lon, radius, categories) String
        +haversine(lat1, lon1, lat2, lon2) Number
        +parseNearbyConfig(raw) Config
        +createNearbyError(code, message) Error
    }

    class PlacesService {
        <<servis>>
        +resolvePlaceFromCoords(latLon) Resolved
    }

    class KotlinBridge {
        <<servis>>
        +getCity(slug) City
        +getPlaces(query) Places
        +getPlaceDetails(placeId) Details
        +getEvents(city) Event[]
    }

    class AuthMiddleware {
        <<middleware>>
        +protect(req, res, next)
        +adminOnly(req, res, next)
        +selfOrAdmin(req, res, next)
    }

    class WsHandlers {
        <<WebSocket>>
        +registerWebSocketHandlers(wss)
    }

    User "1" o-- "0..*" Trip : owner, trips
    Trip "1" *-- "1..*" Stop : stops
    Trip "1" *-- "1" ActiveSession : activeSession
    Stop "1" *-- "0..1" GeoPoint : location
    AuthMiddleware ..> TokenService : verifyAccessToken
    WsHandlers ..> OsrmService : navigacija in ETA
    WsHandlers ..> NearbyPlacesService : bližnji POI
    WsHandlers ..> Trip : upravlja živo sejo
    PlacesService ..> NearbyPlacesService : OSM razrešitev
    PlacesService ..> KotlinBridge : Google detajli
    NearbyPlacesService ..> GeoPoint : Haversine
```

### 3.4.2 Frontend (TypeScript — tipi, Zustand storei, klienta)

```mermaid
classDiagram
    class User {
        <<interface>>
        +string _id
        +string email
        +string displayName
        +string profilePicture
        +string role
        +boolean isActive
        +string[] trips
    }
    class Stop {
        <<interface>>
        +string _id
        +string city
        +GeoJSON location
        +string description
        +number dayNumber
        +number order
        +string arrivalTime
        +string departureTime
        +string visitedAt
        +string[] tags
        +boolean isVisited
        +boolean isNext
    }
    class ActiveSession {
        <<interface>>
        +boolean isActive
        +string startedAt
        +number currentStopIndex
        +TravelProfile travelProfile
    }
    class Trip {
        <<interface>>
        +string _id
        +string title
        +string description
        +User owner
        +Stop[] stops
        +string[] tags
        +boolean isPublic
        +number viewCount
        +number durationDays
        +string startDate
        +string endDate
        +ActiveSession activeSession
    }

    class AuthStore {
        <<Zustand>>
        +User user
        +setUser(user)
        +clearUser()
    }
    class MapStore {
        <<Zustand>>
        +LonLat clickedLocation
        +Trip activeTrip
        +Place selectedPlace
        +Poi selectedPoi
        +setClickedLocation(loc)
        +setActiveTrip(trip)
        +updateActiveTrip(trip)
        +clearActiveTrip()
        +setSelectedPlace(place)
        +setSelectedPoi(poi)
    }
    class LiveStore {
        <<Zustand>>
        +boolean isActive
        +string activeTripId
        +number currentStopIndex
        +string currentStopId
        +GpsSignal gpsSignal
        +number lastLat
        +number lastLon
        +TravelProfile travelProfile
        +setActive(tripId, profile)
        +setCurrentStop(index, stopId)
        +advanceStop(newIndex, newStopId)
        +setGpsSignal(signal)
        +setLocation(lat, lon)
        +clear()
    }
    class UiStore {
        <<Zustand>>
        +boolean leftPanelOpen
        +boolean rightPanelOpen
        +LeftTab activeTab
        +boolean isEditingTrip
        +string activeTripId
        +boolean pendingStopFromMap
        +toggleLeftPanel()
        +toggleRightPanel()
        +setActiveTab(tab)
        +setEditingTrip(id)
        +clearEditingTrip()
        +triggerAddStop()
        +clearPendingStop()
    }

    class LiveSessionClient {
        <<singleton wsClient>>
        -WebSocket ws
        -Map handlers
        -number reconnectAttempts
        +boolean isConnected
        +connect(token)
        +disconnect()
        +on(type, handler) unsubscribe
        +sendLocation(lat, lon)
        +sendTripUpdate(tripId, lon, lat)
        +startTrip(tripId, profile, fresh)
        +stopTrip()
        +markVisited(stopId)
    }
    class ApiClient {
        <<modul lib/api>>
        +AxiosInstance api
        +refreshAccessToken(client) string
        +readToken() string
        +writeToken(token)
    }
    class useLiveSession {
        <<hook>>
    }

    Trip "1" *-- "1..*" Stop : stops
    Trip "1" *-- "0..1" ActiveSession
    Trip "1" o-- "0..1" User : owner
    AuthStore o-- User : user
    MapStore o-- Trip : activeTrip
    AuthStore ..> ApiClient : nalaganje profila
    useLiveSession ..> LiveSessionClient : posluša in pošilja
    useLiveSession ..> LiveStore : advanceStop
    useLiveSession ..> MapStore : updateActiveTrip
    LiveSessionClient ..> ApiClient : JWT žeton
```

### 3.4.3 Kotlin (data modeli, repozitoriji, scraping, strežnik, UI)

```mermaid
classDiagram
    class Trip {
        <<data class>>
        +String id
        +String title
        +String description
        +OwnerSummary owner
        +List~Stop~ stops
        +List~String~ tags
        +Boolean isPublic
        +Int viewCount
        +Int durationDays
        +String startDate
        +String endDate
        +ActiveSession activeSession
    }
    class Stop {
        <<data class>>
        +String id
        +String city
        +GeoPoint location
        +String description
        +Int dayNumber
        +Int order
        +String arrivalTime
        +String departureTime
        +String visitedAt
        +List~String~ tags
        +Boolean isVisited
        +Boolean isNext
    }
    class GeoPoint {
        <<data class>>
        +String type
        +List~Double~ coordinates
    }
    class ActiveSession {
        <<data class>>
        +Boolean isActive
        +String startedAt
        +Int currentStopIndex
        +String travelProfile
    }
    class OwnerSummary {
        <<data class>>
        +String id
        +String displayName
        +String profilePicture
    }
    class User {
        <<data class>>
        +String id
        +String name
        +String email
        +String password
        +String role
        +Boolean isActive
        +LocalDateTime createdAt
        +LocalDateTime updatedAt
    }
    class Event {
        <<data class>>
        +String title
        +LocalDateTime startDate
        +LocalDateTime endDate
        +String url
    }
    class CityDescription {
        <<data class>>
        +String city
        +String description
        +String url
    }

    class ApiService {
        <<object>>
        +get(endpoint) Result~T~
        +post(endpoint, body) Result~Res~
        +put(endpoint, body) Result~Res~
        +delete(endpoint) Result~T~
        +deleteNoBody(endpoint) Result~Unit~
        -loginAndStoreToken() String
    }
    class UserRepository {
        <<object>>
        +getAll() Result
        +create(user) Result
        +update(id, user) Result
        +delete(id) Result
    }
    class TripRepository {
        <<object>>
        +getAll() Result
        +create(trip) Result
        +update(id, trip) Result
        +delete(id) Result
    }
    class ScraperRepository {
        <<object>>
        +getEvents(city) Result
        +getWikipedia(city) Result
    }
    class DataGenerator {
        <<object>>
        +generateUser(from, to) User
        +generateStop(cities, dayNumber) Stop
        +generateTrip(...) Trip
        +generateRandomDateTime(from, to) LocalDateTime
    }
    class GooglePlacesApi {
        +search(query)
        +searchAsString(query) String
        +getPlaceDetails(placeId) String
    }
    class MestniScraperji {
        <<funkcije mainScraper.kt>>
        +scrapeMaribor() List~Event~
        +scrapeLjubljana() List~Event~
        +scrapeKranj() List~Event~
        +scrapeKoper() List~Event~
        +scrapePostojna() List~Event~
        +scrapeMurskaSobota() List~Event~
    }
    class WikipediaScraper {
        <<funkcije>>
        +scrapeCityDescriptionAsObject(slug) CityDescription
    }
    class KtorServer {
        <<Application.kt port 8080>>
        +getCity(slug) Json
        +getPlaces(query) Json
        +getPlaceDetails(id) Json
        +getEvents(city) Json
    }
    class EnvReader {
        <<object>>
        +getGooglePlaces() String
        +getAuthEmail() String
        +getAuthPassword() String
    }
    class Screen {
        <<sealed class>>
        Users
        Trips
        Scraped
        Generator
    }

    Trip "1" *-- "0..*" Stop : stops
    Trip "1" *-- "1" ActiveSession
    Trip "1" o-- "0..1" OwnerSummary : owner
    Stop "1" *-- "1" GeoPoint : location
    UserRepository ..> ApiService : REST do backenda
    TripRepository ..> ApiService
    ScraperRepository ..> ApiService
    UserRepository ..> User
    TripRepository ..> Trip
    ScraperRepository ..> Event
    ScraperRepository ..> CityDescription
    ApiService ..> EnvReader : poverilnice
    GooglePlacesApi ..> EnvReader : API ključ
    KtorServer ..> GooglePlacesApi : /places
    KtorServer ..> MestniScraperji : /events
    KtorServer ..> WikipediaScraper : /city
    MestniScraperji ..> Event : ustvarja
    DataGenerator ..> User : faker
    DataGenerator ..> Trip : faker
    Screen ..> UserRepository : Users zaslon
    Screen ..> TripRepository : Trips zaslon
    Screen ..> ScraperRepository : Scraper zaslon
    Screen ..> DataGenerator : Generator zaslon
```

---
---

# 4. DevOps (CI/CD)

## 4.1 Pregled poteka dela

Vsak repozitorij ima **ločene GitHub Actions** poteke. Razvojni model temelji na vejah:
spremembe gredo najprej na razvojno vejo (`development` / `dev`), kjer se **samodejno
poženejo testi**; po združitvi v `main` se sproži **gradnja Docker slike, objava na
Docker Hub in samodejni deploy** prek **Webhook** protokola na produkcijsko VM.

Faze DevOps metodologije in pripadajoče akcije:

| Faza DevOps | Orodje / akcija v projektu |
|---|---|
| **Plan** | Jira (projekt SCRUM, epici + sprinti), veje (`development`, `dev`, `feature/*`) |
| **Code** | trije repozitoriji v org. DropInSlovenia, integracija GitHub ↔ Jira |
| **Build** | `docker/build-push-action` (Dockerfile za vsako komponento) |
| **Test** | GitHub Actions: `node --test` (backend), ESLint + `vitest` (frontend) |
| **Release** | objava slike na **Docker Hub** (`dropinslovenia_frontend`, `_backend`, `_kotlin-server`, tag `:latest`) |
| **Deploy** | **Webhook** (`distributhor/workflow-webhook`) → `webhook` (adnanh, Go) na **VM:9000** → `deploy.sh` |
| **Operate** | `webhook.service` (systemd, `Restart=always`), Docker `--restart unless-stopped`, log v `/var/log/deploy.log` |

## 4.2 Diagram poteka — CI (testiranje)

Velja za **push na `development`/`dev`** in **pull request** proti `development`/`main`.

```mermaid
flowchart TD
    A["Razvijalec: git push<br/>(veja development / dev)"] --> B{"GitHub Actions<br/>test workflow"}
    B --> C["actions/checkout"]
    C --> D["setup-node (Node 22) + cache npm"]
    D --> E["npm ci"]
    E --> F{"Zaženi teste"}
    F -->|Backend| G["node --test → junit.xml"]
    F -->|Frontend| H["npm run lint + vitest → junit.xml"]
    G --> I["dorny/test-reporter<br/>objavi poročilo"]
    H --> I
    I --> J{"Testi uspešni?"}
    J -->|Da| K["✅ Check zelen<br/>PR sme v main"]
    J -->|Ne| L["❌ Check rdeč<br/>PR blokiran"]
```

## 4.3 Diagram poteka — CD (gradnja + objava + deploy prek webhook)

Velja za **push / merge na `main`**. Deploy workflow (`deploy.yml`) imajo **vsi trije
repozitoriji** (frontend, backend, kotlin-server) — vsak gradi svojo Docker sliko, jo
naloži na Docker Hub in pokliče **svoj** webhook (`/hooks/deploy-frontend`,
`/hooks/deploy-backend`, `/hooks/deploy-kotlin`). Tako push v en repozitorij posodobi
le pripadajočo storitev, ostali dve ostaneta nedotaknjeni.

```mermaid
flowchart TD
    A["Merge v main"] --> B["GitHub Actions: deploy workflow"]
    B --> C["actions/checkout"]
    C --> D["docker/login-action<br/>(Docker Hub: secrets DOCKERHUB_*)"]
    D --> E["docker/build-push-action<br/>build + push :latest"]
    E --> F["Job: notify-server (needs build-and-push)"]
    F --> G["distributhor/workflow-webhook<br/>POST http://VM_HOST:9000/hooks/deploy-backend<br/>+ WEBHOOK_SECRET"]
    G --> H{"webhook na VM<br/>preveri HMAC podpis"}
    H -->|veljaven| I["deploy.sh: docker pull novejše slike"]
    I --> J["docker run -d --restart unless-stopped<br/>(zamenjava kontejnerja)"]
    J --> K["✅ Nova različica v produkciji"]
    H -->|neveljaven| L["❌ Zavrnjeno"]
```

**Konkretna konfiguracija na VM (iz poročila P3):**

- **Webhook strežnik:** paket `webhook` (adnanh, Go), posluša na **portu 9000**, teče kot
  **systemd** servis `/etc/systemd/system/webhook.service` (`Restart=always`, zagon ob bootu).
- **Hooki:** `/etc/webhook/hooks.json` definira tri hooke — `deploy-frontend`,
  `deploy-backend`, `deploy-kotlin`. Vsak ima `trigger-rule` z **HMAC-SHA256** preverjanjem
  (header `X-Hub-Signature-256`), tako da se skripta izvede **le** ob veljavnem podpisu.
- **Deploy skripta:** `/opt/deploy/deploy.sh` sprejme ime storitve (`$1`) iz JSON payloada
  (`{"service": "..."}`), nato: `docker stop` + `docker rm` star kontejner → `docker pull`
  sveže `:latest` slike z Docker Hub → `docker run -d --env-file <.env> --restart unless-stopped`.
  Vsak korak se beleži v `/var/log/deploy.log` z datumom in uro.
- **Skrivnosti:** `.env` (Mongo URI, JWT skrivnosti) se v kontejner naloži prek `--env-file`,
  nikoli ni zapisan v skripti ali logih.

> **Znane luknje (iz P3):** secret je trenutno v `hooks.json` v čistopisu, webhook
> teče prek HTTP (ne HTTPS), deploy user ima dostop do `.env` in Dockerja. Priporočene
> izboljšave (ločen `deployer` user, secret iz env spremenljivke, TLS prek nginx) so opisane
> v poglavju 5.6.

## 4.4 Kontejnerizacija

| Komponenta | Osnova slike | Posebnosti |
|---|---|---|
| Backend | `node:22-alpine` | `npm ci --omit=dev`, EXPOSE 3000 |
| Frontend | `node:22-alpine` (večstopenjski; različica za docker-compose v `frontend/Dockerfile` uporablja `node:20-alpine`) | Next.js `standalone`, EXPOSE 3001, `NEXT_PUBLIC_API_URL` kot build-arg |
| Kotlin | `gradle:8.5-jdk21` → `eclipse-temurin:21-jre-alpine` | namesti Chromium + ChromeDriver za Selenium, EXPOSE 8080 |

Orkestracija: `docker-compose.yml` (tri storitve, `depends_on`, `restart: unless-stopped`).
Vrstni red zagona: `kotlin-server` → `backend` → `frontend`. Backend dobi
`KOTLIN_SERVICE_URL=http://kotlin-server:8080`, frontend pa `API_URL=http://backend:3000`
(komunikacija po internem Docker omrežju, ne prek `localhost`).

`build.context` poti v `docker-compose.yml`: `./PrincipiProjekt/desktopApp`
(kotlin-server), `./backendProjekt` (backend) in `./webApp` z
`dockerfile: frontend/Dockerfile` (frontend). Frontend prejme javni IP VM kot build-arg:
`NEXT_PUBLIC_API_URL=http://68.210.138.63:3000` (Next.js ga vtisne v JS bundle že ob gradnji,
zato mora biti podan med `docker build`, ne ob zagonu).

> **Opomba o dveh načinih namestitve:** za **lokalni razvoj** se uporablja en
> `docker-compose.yml` z eno centralno `.env` (vse tri storitve hkrati). V **produkciji na VM**
> pa se vsaka storitev posodablja **neodvisno** prek CI/CD (GitHub Actions → Docker Hub →
> webhook → `deploy.sh` zažene posamezen `docker run`), ne prek `docker compose`.

## 4.5 Produkcijska infrastruktura in register slik

**Strežnik (Azure VM, iz P2):**

| Parameter | Vrednost |
|---|---|
| Ponudnik / naročnina | Microsoft Azure — **Azure for Students** |
| Resource group | `DropInSlovenia_group_05191709` |
| Ime VM | `dropinslovenia-vm` |
| Regija | **Austria East (Zone 2)** _(West Europe ni bil na voljo za to naročnino)_ |
| OS | **Ubuntu Server 24.04 LTS** |
| Velikost | **Standard B2ts v2** (2 vCPU, 1 GiB RAM) |
| Disk | **Premium SSD LRS, 30 GB** (3× replikacija v istem podatkovnem centru) |
| Javni IP | **68.210.138.63** |

> Ker ima VM le **1 GiB RAM**, vse tri storitve skupaj (frontend ~150–200 MB, backend ~100 MB,
> Kotlin JVM ~256 MB + OS) ob gradnji presežejo RAM. Zato je dodana **2 GB swap datoteka**
> (`/swapfile`, trajno v `/etc/fstab`), ki prepreči sesutje VM med `docker build`.

**Register slik (Docker Hub, iz P3):**

- Tri slike: `dropinslovenia_frontend`, `dropinslovenia_backend`, `dropinslovenia_kotlin-server`
  (tag `:latest`, ki ga GitHub Actions prepiše ob vsakem deployu).
- Prijava v CI prek **Access Tokena** (ne gesla) — token ima omejene pravice in ga je ob
  morebitnem uhajanju mogoče takoj preklicati.
- Možna nadgradnja: dodatno tagiranje s `${{ github.sha }}` za rollback na prejšnjo verzijo.

**GitHub Secrets (v vseh treh repozitorijih):** `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`,
`WEBHOOK_SECRET`, `VM_HOST` — šifrirani, nikoli vidni v logih, referencirani prek
`${{ secrets.IME }}`.

**Testna ogrodja (CI nad razvojno vejo):**

| Komponenta | Veja sprožilca | Ogrodje | Stanje |
|---|---|---|---|
| Backend | `development` (+ PR v `development`/`main`) | vgrajeni `node --test` | Testni *workflow* pripravljen; dejanski unit testi so v pripravi (kandidata: `utils/osrmToGeoJSON.js`, `utils/asyncHandler.js`) |
| Frontend | `dev` (+ PR v `dev`/`main`) | ESLint + `vitest` (`--passWithNoTests`) | Lint in testni okvir pripravljena; unit testi v pripravi (kandidat: `lib/utils/tokens.ts`) |
| Kotlin | `dev` (+ PR) | `kotlin.test` prek `./gradlew jvmTest` | Obstaja test `composeApp/src/jvmTest/.../ComposeAppDesktopTest.kt` |

Rezultati se objavijo prek `dorny/test-reporter` v zavihku **Checks** (JUnit XML).

---
---

# 5. Varnost programske rešitve

## 5.1 Varnostne vloge uporabnikov

Sistem pozna **dve vlogi** (`role` polje na uporabniku), ki ju uveljavljajo middleware funkcije:

| Vloga | Pravice |
|---|---|
| `user` | Dostop do svojega profila in svojih potovanj; ustvarjanje/urejanje/brisanje **lastnih** potovanj; ogled javnih potovanj. |
| `admin` | Vse pravice: upravljanje vseh uporabnikov, ogled in urejanje **vseh** potovanj (vključno z `?all=true`), prestavljanje lastništva. |

Uveljavljanje (v `backend/middleware/authMiddleware.js` + controllerji):

```mermaid
flowchart LR
    R["Zahteva na zaščiteni endpoint"] --> P{"protect:<br/>veljaven JWT?"}
    P -->|Ne| E401["401 NO_TOKEN /<br/>TOKEN_EXPIRED / TOKEN_INVALID"]
    P -->|Da| ROLE{"Tip zaščite"}
    ROLE -->|adminOnly| A{"role == admin?"}
    A -->|Ne| E403["403 FORBIDDEN"]
    A -->|Da| OK["✅ Dovoljeno"]
    ROLE -->|selfOrAdmin| S{"lastnik ali admin?"}
    S -->|Ne| E403
    S -->|Da| OK
    ROLE -->|owner-check| O{"owner == user.id<br/>ali admin?"}
    O -->|Ne| E403
    O -->|Da| OK
```

- **`protect`** — zahteva veljaven JWT (`Authorization: Bearer <token>`).
- **`adminOnly`** — samo admin (npr. seznam vseh uporabnikov, brisanje uporabnikov).
- **`selfOrAdmin`** — lastnik vira ali admin (dostop do svojega profila).
- **owner-check** — v `tripsController` (uredi/izbriši le svoje potovanje); vidljivost
  zasebnih potovanj (`isPublic:false`) se preverja v `getTripById`, `route/geojson` in WS `start_trip`.

## 5.2 Avtentikacija in žetoni (JWT)

- **Dva ločena žetona:** *access* (veljaven 15 min) in *refresh* (veljaven 7 dni),
  podpisana z **ločenima skrivnostma** (`ACCESS_TOKEN_SECRET`, `REFRESH_TOKEN_SECRET`).
- **Rotacija:** ob `POST /api/auth/refresh` se izda nov par; shranjeni refresh token v
  bazi se primerja in zamenja (omeji ponovno uporabo ukradenega žetona).
- **WebSocket:** JWT se preveri že ob HTTP *upgrade* zahtevi (`server.js`), pred
  vzpostavitvijo seje; ob neveljavnem žetonu je povezava zavrnjena (401). Ena aktivna
  seja na uporabnika (nova zapre staro).
- **Frontend:** axios interceptor samodejno osveži žeton ob `401` in ponovi zahtevo.

## 5.3 Varovanje podatkov

- **Gesla:** nikoli shranjena v čistopisu — **bcrypt** (salt + hash, cost 12).
  Polje `passwordHash` ima `select:false` in se **nikdar** ne vrne v API odgovorih.
- **Refresh token** v bazi je prav tako `select:false`.
- **Validacija vhoda:** Mongoose sheme (regex za email, dolžine nizov, formati
  `HH:mm` / `YYYY-MM-DD`, veljavnost GeoJSON koordinat); sanitizacija postaj
  (`sanitizeStops` odstrani nepopolne koordinate).
- **Tajnosti:** v `.env` (ni v repozitoriju), v CI/CD prek **GitHub Secrets**
  (`DOCKERHUB_*`, `WEBHOOK_SECRET`, `VM_HOST`, Mongo poverilnice, JWT skrivnosti).
- **Šifriran prenos:** povezave do MongoDB Atlas, OSRM, Google in Wikipedije potekajo
  prek TLS (HTTPS / `mongodb+srv`).

## 5.4 Omejitve uporabnikov in zaščita aplikacijskega sloja

| Mehanizem | Konfiguracija | Namen |
|---|---|---|
| **Rate limiting** (`express-rate-limit`) | privzeto **100 zahtev / 15 min** na `/api` | zaščita pred zlorabo in (D)DoS na aplikacijskem sloju |
| **helmet** | varni HTTP headerji | zaščita pred XSS, clickjacking, MIME-sniffing |
| **CORS** | whitelist iz `CORS_ORIGIN`, `credentials:true` | dovoli le znane izvore |
| **Omejitev velikosti telesa** | `express.json({ limit: '1mb' })` | preprečuje velike payload napade |
| **Centralni error handler** | enotni odgovori | ne razkriva *stack trace* / notranjosti |
| **Deaktivacija računa** | `isActive:false` | prijava blokirana (403) |

## 5.5 Požarni zid in omrežna varnost (dvonivojska zaščita)

Sistem uporablja **dva sloja požarnega zidu**: oblačni Azure NSG (na ravni omrežja Azure)
in sistemski UFW (na ravni operacijskega sistema VM). Oba sta dejansko konfigurirana (P2/P3).

### Sloj 1 — Azure Network Security Group (NSG)

NSG je virtualni požarni zid, ki ga vsak Azure VM dobi samodejno. Privzeto so **vsa vhodna
vrata zaprta razen 22 (SSH)** — načelo najmanjših privilegijev. Z *Inbound port rules*
(VM → Networking → Create port rule) so navzven odprta le nujna vrata:

| Vrata | Pravilo | Protokol | Namen |
|---|---|---|---|
| 22 | (privzeto) | TCP | SSH dostop |
| 3000 | `allow-backend` | TCP | Node.js/Express REST API + WebSocket |
| 3001 | `allow-frontend` | TCP | Next.js frontend |
| 8080 | `allow-kotlin` | TCP | Kotlin Ktor scraping API |
| 9000 | `allow-webhook` | TCP | Deploy webhook (CI/CD) |

### Sloj 2 — UFW na VM (dodatno zaklepanje porta 9000)

Na sami VM je nameščen **UFW** (Uncomplicated Firewall). Ključno: port **9000 (webhook) je
omejen samo na GitHub Actions IP obsege** — torej ga ne more klicati kdorkoli z interneta,
tudi če pozna URL:

```bash
sudo ufw allow 22
sudo ufw allow 3000
sudo ufw allow 3001
sudo ufw allow 8080
sudo ufw deny  9000                                   # zapri za vse
sudo ufw allow from 140.82.112.0/20  to any port 9000 # samo GitHub Actions
sudo ufw allow from 185.199.108.0/22 to any port 9000
sudo ufw allow from 192.30.252.0/22  to any port 9000
sudo ufw --force enable
```

> GitHub IP obsegi se občasno spremenijo; aktualni seznam je na
> <https://api.github.com/meta> pod ključem `actions`.

### Ostala omrežna varnost

- **SSH:** prijava prek **para ključev `ed25519`** (vsak član svoj javni ključ v
  `~/.ssh/authorized_keys`, pravice `700`/`600`) — varneje od gesla, odporno na brute-force.
- **MongoDB Atlas:** ločen bazni uporabnik (`dropinslovenia-user`) z geslom; povezava prek
  TLS (`mongodb+srv`). **IP allowlist je v razvoju nastavljen na `0.0.0.0/0`** (dostop iz
  lokalnih okolij in VM); za produkcijo je priporočeno zožiti na **samo IP VM (68.210.138.63)**.
- **Izolacija:** vsaka komponenta v ločenem Docker kontejnerju; `restart: unless-stopped`
  za samodejno okrevanje po napaki ali rebootu VM.

## 5.6 Varnost CI/CD in webhook poteka

Deploy prek webhooka je posebej zaščiten, saj sproži spremembe v produkciji:

- **HMAC-SHA256 podpis (implementirano):** GitHub Actions ob vsakem klicu podpiše payload z
  `WEBHOOK_SECRET` (header `X-Hub-Signature-256`); webhook strežnik na VM izračuna isti podpis
  in primerja. Brez poznavanja skrivnosti ni mogoče ustvariti veljavnega podpisa → deploy se
  ne izvede. Test: `curl http://<IP>:9000/hooks/deploy-backend` vrne *"Hook rules were not
  satisfied."* (zahteva veljaven podpis).
- **Docker Hub Access Token** namesto gesla (omejene pravice, takojšen preklic).
- **GitHub Secrets** za vse poverilnice (nikoli v YAML/git).

**Znane luknje in priporočene izboljšave (analiza iz P3):**

| Luknja | Tveganje | Predlagana rešitev |
|---|---|---|
| Webhook prek HTTP (ne HTTPS) | MITM vidi metapodatke deployev | TLS prek nginx + Let's Encrypt (`certbot`) |
| `WEBHOOK_SECRET` v `hooks.json` v čistopisu | viden vsem z SSH dostopom | branje iz env spremenljivke v `webhook.service` |
| Deploy user ima dostop do `.env` in Dockerja | ob zlorabi eskalacija do skrivnosti/root | ločen `deployer` user z minimalnimi pravicami |

---

## Priloga — Pretvorba diagramov v slike

Če mora biti oddaja v PDF/Wordu s slikami namesto Mermaid kode:

1. Odpri <https://mermaid.live>.
2. Prilepi kodo posameznega diagrama (npr. od `sequenceDiagram` do konca bloka).
3. Po potrebi popravi besedilo/datume.
4. **Actions → Export** → PNG ali SVG.
5. Vstavi sliko v končni dokument (Word/Google Docs/LaTeX).

Diagrami v tem dokumentu (za izvoz): Gantt (1), sekvenčni (3), arhitektura (1),
razredni (3), DevOps flowchart (2), varnost flowchart (1).
