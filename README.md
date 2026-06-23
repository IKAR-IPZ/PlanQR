# PlanQR - dokumentacja dla utrzymujących

PlanQR to system do wyświetlania planu zajęć, obsługi tabletów przy salach,
komunikatów administracyjnych oraz list obecności. Ten plik jest punktem
startowym dla kolejnych osób rozwijających projekt.

Szczegółowe instrukcje uruchomienia są w dokumentacjach modułów:

- [planqr-backend/README.md](planqr-backend/README.md)
- [planqr-frontend/README.md](planqr-frontend/README.md)

## Najważniejsze repozytoria

To repozytorium spina główny stack aplikacji i korzysta z dwóch submodułów Git:

| Ścieżka | Repozytorium | Rola |
| --- | --- | --- |
| `planqr-backend` | [github.com/IKAR-IPZ/planqr-backend](https://github.com/IKAR-IPZ/planqr-backend) | API backendowe, baza danych, logowanie, tablety, obecność |
| `planqr-frontend` | [github.com/IKAR-IPZ/planqr-frontend](https://github.com/IKAR-IPZ/planqr-frontend) | Aplikacja webowa dla użytkowników, administratorów i tabletów |

Z projektem jest powiązane jeszcze jedno repozytorium:

| Repozytorium | Rola |
| --- | --- |
| [github.com/IKAR-IPZ/planqr-kantech](https://github.com/IKAR-IPZ/planqr-kantech) | Osobna usługa napisana w Pythonie. Obsługuje fizyczne czytniki kart / system Kantech i przekazuje zdarzenia obecności do API PlanQR. |

Przy klonowaniu projektu trzeba pamiętać o submodułach:

```bash
git clone --recurse-submodules <adres-tego-repozytorium>
```

Jeśli repo zostało już sklonowane bez submodułów:

```bash
git submodule update --init --recursive
```

## Architektura

Główne elementy systemu:

- `planqr-frontend` - React, TypeScript, Vite; w kontenerze serwowany przez `nginx`.
- `planqr-backend` - Express, TypeScript, Prisma; wystawia API pod `/api`.
- PostgreSQL - baza danych aplikacji, konfigurowana przez `DATABASE_URL`.
- LDAP - źródło logowania i danych użytkowników.
- Endpointy planu ZUT - źródło danych planu zajęć.
- `planqr-kantech` - zewnętrzny worker Python dla skanów kart.

Uproszczony przepływ:

```text
Przeglądarka / tablet
        |
        v
Frontend nginx
        |
        v
Backend PlanQR API
        |
        +--> PostgreSQL
        +--> LDAP
        +--> plan.zut.edu.pl
        ^
        |
Python service: planqr-kantech
czytniki kart / system Kantech
```

Rootowy `docker-compose.yml` buduje i uruchamia frontend oraz backend. Nie
uruchamia PostgreSQL, dlatego baza danych musi działać osobno i być dostępna
dla backendu przez `DATABASE_URL`.

## Środowisko produkcyjne

Aktualny podział produkcyjny zakłada dwa serwery:

| Serwer | Rola |
| --- | --- |
| `planqr-app` | Uruchamia aplikację: kontenery frontendu i backendu. Frontend kończy TLS przez `nginx` i proxyuje ruch do backendu. |
| `planqr-db` | Przechowuje PostgreSQL i dane aplikacji. To krytyczny element do backupów. |

Ważne zasady utrzymania:

- Na `planqr-app` powinien znajdować się aktualny kod aplikacji, plik `.env`
  oraz certyfikaty w `certs/cert.pem` i `certs/cert.key`, jeśli frontend działa
  w kontenerze.
- Na `planqr-db` należy pilnować backupów PostgreSQL. Rootowy Compose nie
  tworzy bazy, więc awaria tego serwera oznacza utratę danych, jeśli nie ma
  poprawnych kopii zapasowych.
- `.env` nie może być commitowany. Zawiera m.in. `DATABASE_URL`, `JWT_SECRET`,
  dane konta admina z env i token workera.
- Backend w Compose najprościej uruchamiać po HTTP (`DISABLE_HTTPS=true`), bo
  TLS kończy frontendowy `nginx`.
- `BACKEND_INTERNAL_URL` w kontenerze frontendu powinien wskazywać backend z
  perspektywy sieci Dockera, typowo `http://backend:9099`.

## Konfiguracja

Wspólny plik konfiguracyjny znajduje się w katalogu głównym repozytorium:

```text
.env
```

Szablon:

```text
.env.example
```

Najważniejsze zmienne:

| Zmienna | Znaczenie |
| --- | --- |
| `DATABASE_URL` | Connection string PostgreSQL dla Prisma i backendu. |
| `JWT_SECRET` | Sekret do podpisywania tokenów. Musi być unikalny poza developmentem. |
| `CORS_ORIGIN` | Lista adresów frontendu dozwolonych przez backend. |
| `BACKEND_PUBLIC_URL` | Publiczny adres backendu, bez `/api`. |
| `BACKEND_INTERNAL_URL` | Adres backendu widoczny z kontenera frontendu. |
| `FRONTEND_PUBLIC_URL` | Adres frontendu używany lokalnie przez Vite/HMR. |
| `FRONTEND_PORT` | Port hosta mapowany na `nginx:443` frontendu. |
| `LDAP_URL`, `LDAP_DN` | Konfiguracja logowania przez LDAP. |
| `WORKER_SECRET_TOKEN` | Opcjonalny Bearer token dla usługi `planqr-kantech`. |
| `ZUT_SCHEDULE_STUDENT_URL`, `ZUT_SCHEDULE_URL` | Endpointy planu zajęć ZUT. |
| `ZUT_PLAN_BASE_URL` | Bazowy adres używany przez linki i kody QR. |

Pełne opisy zmiennych są w:

- [planqr-backend/README.md](planqr-backend/README.md)
- [planqr-frontend/README.md](planqr-frontend/README.md)
- [.env.example](.env.example)

## Główne przepływy w aplikacji

### Logowanie

Backend obsługuje logowanie przez LDAP. Opcjonalne konto z
`ROOT_ADMIN_LOGIN` / `ROOT_ADMIN_PASSWORD` omija LDAP i zawsze ma dostęp
administracyjny. To konto jest przeznaczone awaryjnie i nie powinno zastąpić
normalnych kont administracyjnych.

W development można włączyć `DEV_AUTH_BYPASS=true`, ale tylko przy
`NODE_ENV=development`.

### Plan zajęć

Frontend i backend korzystają z endpointów planu ZUT:

- `ZUT_SCHEDULE_STUDENT_URL`
- `ZUT_SCHEDULE_URL`
- `ZUT_PLAN_BASE_URL`

Jeśli widoki sal, rezerwacji albo linki QR przestają działać, te zmienne są
jednym z pierwszych miejsc do sprawdzenia.

### Tablety i rejestracja urządzeń

Tablety łączą się z aplikacją frontendową i są rejestrowane w backendzie.
Backend przechowuje status urządzeń, przypisaną salę, ostatnie IP, profil
wyświetlacza, motyw, czarny ekran i ustawienia komunikatów priorytetowych.

Najważniejsze endpointy są pod:

```text
/api/devices
/api/registry
```

### Komunikaty priorytetowe

Komunikaty priorytetowe są zarządzane w panelu administracyjnym i zapisywane w
PostgreSQL. Backend uruchamia job harmonogramów komunikatów, który pilnuje
aktywnych przedziałów czasowych.

Uploady komunikatów są serwowane z:

```text
/priority-message-uploads
```

### Obecność i Kantech

Obecność jest zapisywana w backendzie PlanQR, ale zdarzenia z fizycznych
czytników kart obsługuje osobna usługa:

[github.com/IKAR-IPZ/planqr-kantech](https://github.com/IKAR-IPZ/planqr-kantech)

Ta usługa jest napisana w Pythonie i działa jako integracja z systemem
Kantech / czytnikami kart. Jej zadaniem jest odczyt zdarzeń ze skanerów i
wysłanie ich do backendu PlanQR.

Najważniejsze endpointy po stronie PlanQR:

| Endpoint | Rola |
| --- | --- |
| `POST /api/attendance/scan` | Zapisuje skan karty. Może otworzyć/zamknąć sesję lub dopisać osobę do obecności. |
| `GET /api/attendance/list` | Zwraca listę obecności dla sesji. Może być chroniony tokenem workera. |
| `POST /api/attendance/sessions/:id/send` | Zamyka i oznacza sesję jako wysłaną. Może być używany przez prowadzącego albo usługę z tokenem. |

Jeśli `WORKER_SECRET_TOKEN` jest ustawiony, usługa zewnętrzna powinna wysyłać:

```http
Authorization: Bearer <WORKER_SECRET_TOKEN>
```

## Uruchomienie lokalne

Najpierw przygotuj konfigurację w katalogu głównym:

```bash
cp .env.example .env
```

Potem uruchom backend:

```bash
cd planqr-backend
npm install
npm run prisma:generate
npm run prisma:push
npm run dev
```

W drugim terminalu uruchom frontend:

```bash
cd planqr-frontend
npm install
npm run dev
```

Pełny stack kontenerowy z katalogu głównego:

```bash
docker compose up --build
```

Pamiętaj: rootowy Compose nie startuje PostgreSQL. `DATABASE_URL` musi
wskazywać działającą bazę danych.

## Co sprawdzić po zmianach

Backend:

```bash
cd planqr-backend
npm run build
npm run lint
```

Frontend:

```bash
cd planqr-frontend
npm run build
npm run lint
```

Pełny stack:

```bash
docker compose up --build
```

Po starcie sprawdź:

- czy frontend ładuje się w przeglądarce,
- czy `/api` odpowiada przez proxy frontendu,
- czy backend ma połączenie z PostgreSQL,
- czy logowanie LDAP działa w docelowej sieci,
- czy tablety widzą aktualny frontend,
- czy usługa `planqr-kantech` może wysyłać skany do backendu.

W trybie development Swagger backendu jest dostępny pod:

```text
/api/docs
```

## Najczęstsze problemy

| Problem | Co sprawdzić |
| --- | --- |
| Backend nie startuje przez błąd konfiguracji | Brakujące lub błędne zmienne w `.env`. |
| Prisma nie łączy się z bazą | `DATABASE_URL`, dostępność `planqr-db`, firewall, port PostgreSQL. |
| Frontend dostaje błąd CORS | `CORS_ORIGIN` musi zawierać dokładny adres frontendu z protokołem i portem. |
| `502 Bad Gateway` pod `/api` | `BACKEND_INTERNAL_URL` wskazuje zły adres albo backend nie działa. |
| Frontend w Dockerze nie startuje | Brakuje `certs/cert.pem` albo `certs/cert.key`. |
| LDAP nie działa lokalnie | Brak dostępu do sieci uczelni albo VPN. |
| Skanery kart nie zapisują obecności | Sprawdź `planqr-kantech`, `WORKER_SECRET_TOKEN`, adres backendu i endpointy `/api/attendance/*`. |
| Plan sal się nie ładuje | Sprawdź `ZUT_SCHEDULE_STUDENT_URL`, `ZUT_SCHEDULE_URL` i dostęp do `plan.zut.edu.pl`. |

## Zasady dla przyszłych zmian

- Nie commituj `.env`, certyfikatów, tokenów, haseł ani dumpów bazy.
- Przy zmianach w modelu danych aktualizuj `planqr-backend/prisma/schema.prisma`
  i dopisz jasną procedurę migracji.
- Przy zmianach w API sprawdź, czy frontend i `planqr-kantech` dalej używają
  zgodnego kontraktu.
- Przy zmianach w konfiguracji dopisz nowe zmienne do `.env.example` oraz do
  README odpowiedniego modułu.
- Przy zmianach dotyczących tabletów testuj przynajmniej widok zwykły,
  rejestrację urządzenia i panel administratora.
- Przy zmianach dotyczących obecności testuj scenariusz z prowadzącym,
  studentem/użytkownikiem i zewnętrznym workerem Kantech.
