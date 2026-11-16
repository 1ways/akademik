# Akademik

System zarządzania akademikiem przeznaczony dla mieszkańców i administracji. Umożliwia obsługę zgłoszeń usterek i skarg, zarządzanie mieszkańcami i pokojami, prowadzenie kalendarza wydarzeń, korzystanie z czasowych kodów QR oraz odbieranie powiadomień.

Projekt edukacyjny. Aplikacja jest przeznaczona przede wszystkim do uruchamiania i demonstracji w środowisku lokalnym.

## Funkcjonalności

### Mieszkaniec

- Logowanie oraz odświeżanie tokenów uwierzytelniających.
- Przeglądanie danych własnego konta i współlokatorów.
- Zgłaszanie usterek i sprawdzanie statusu własnych zgłoszeń.
- Składanie skarg, również anonimowych.
- Przeglądanie powiadomień w aplikacji.
- Generowanie czasowych kodów QR oraz przeglądanie ich historii.
- Przeglądanie, dodawanie, edytowanie i usuwanie własnych wydarzeń w kalendarzu.

### Administrator

- Rejestrowanie mieszkańców oraz zarządzanie ich danymi i kontami.
- Zarządzanie pokojami i przydziałami mieszkańców do pokoi.
- Przeglądanie zgłoszeń i zmiana statusów usterek oraz skarg.
- Dostęp do powiadomień kierowanych do administratorów, również dotyczących skarg anonimowych.
- Unieważnianie kodów QR.
- Masowe tworzenie wydarzeń związanych z inspekcjami pokoi.

### Automatyzacje i integracje

- **Powiadomienia e-mail:** po zamknięciu usterki zadanie trafia do kolejki Celery; osobny worker pobiera adres mieszkańca z `core-api` i wysyła wiadomość przez SMTP. W środowisku lokalnym e-maile można przeglądać w Mailpit.
- **Powiadomienia APP i SMS:** osobne kolejki przetwarzają powiadomienia w aplikacji i rekordy dotyczące SMS. Integracja z zewnętrzną bramką SMS nie jest potwierdzona; nie należy traktować tego jako rzeczywistej wysyłki SMS.
- **Zadania okresowe:** Celery Beat obsługuje przypomnienia o płatnościach (codziennie o 08:00 UTC) i ponawianie obsługi powiadomień (co 30 minut).
- **Usunięcie użytkownika:** `core-api` informuje `service-api`, które unieważnia aktywne kody QR usuniętej osoby.
- **Import CSV:** skrypt pozwala zaimportować wydarzenia z pliku CSV.

> **Zakres kodów QR:** system udostępnia generowanie, unieważnianie i jednorazową walidację kodów po stronie API. Integracja z rzeczywistymi czytnikami lub zamkami drzwi nie jest częścią potwierdzonego zakresu projektu.

## Technologie

| Obszar | Technologie |
| --- | --- |
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS, shadcn/ui, Zustand, TanStack Query |
| API użytkowników (`core-api`) | ASP.NET Core 9, C#, Entity Framework Core |
| API funkcjonalne (`service-api`) | Python 3.12, FastAPI, SQLAlchemy async, Alembic |
| Zadania asynchroniczne | Celery 5, Celery Beat, RabbitMQ 3 |
| Baza danych | PostgreSQL 16 |
| Lokalna obsługa e-maili | Mailpit |
| Uruchamianie i routing | Docker Compose, Nginx |
| Testy | pytest, xUnit |

## Architektura

```mermaid
flowchart TD
    Browser[Przeglądarka] --> Nginx[Nginx :80]
    Nginx --> Web[Web / Next.js]
    Nginx --> Core[core-api / ASP.NET Core]
    Nginx --> Service[service-api / FastAPI]
    Core --> DB[(PostgreSQL)]
    Service --> DB
    Core <-->|HTTP + X-Api-Key| Service
    Service --> MQ[RabbitMQ]
    Beat[Celery Beat] --> MQ
    MQ --> Worker[Celery worker / APP, SMS, zadania okresowe]
    MQ --> EmailWorker[Celery e-mail worker]
    Worker --> DB
    EmailWorker --> Core
    EmailWorker --> Mailpit[Mailpit / SMTP]
    EmailWorker --> DB
```

- `core-api` odpowiada za logowanie, tokeny JWT, użytkowników, pokoje i przydziały.
- `service-api` obsługuje usterki, skargi, wydarzenia, kody QR i powiadomienia.
- Obie usługi korzystają z PostgreSQL, ale utrzymują własne zestawy tabel.
- `core-api` wystawia tokeny JWT, które `service-api` weryfikuje przy użyciu wspólnego sekretu.
- Endpointy integracyjne pomiędzy usługami wymagają nagłówka `X-Api-Key`.
- Nginx przekazuje `/` do frontendu, `/api/core/` do `core-api` i `/api/service/` do `service-api`.

Szczegółowe przepływy, diagramy sekwencji, obsługa błędów i ograniczenia: [docs/integration.md](docs/integration.md).

## Szybki start

### Wymagania

- Git
- Docker z obsługą `docker compose`

### Uruchomienie

```bash
git clone https://github.com/1ways/akademik.git
cd akademik
cp .env.example .env
```

W pliku `.env` ustaw własne hasła i klucze. W szczególności **zmień `POSTGRES_PASSWORD`, `RABBITMQ_PASS`, `JWT_SECRET` i `INTERNAL_API_KEY`**. Sekret JWT powinien mieć co najmniej 32 bajty; można go wygenerować poleceniem:

```bash
openssl rand -base64 48
```

Następnie uruchom:

```bash
docker compose up -d --build
```

Usługa `migrate` wykonuje migracje Alembic przed startem `service-api`; `core-api` stosuje migracje Entity Framework Core podczas uruchamiania.

| Adres | Zastosowanie |
| --- | --- |
| http://localhost | Aplikacja webowa przez Nginx |
| http://localhost:3001 | `core-api` (Swagger w środowisku Development) |
| http://localhost:3002/docs | Dokumentacja Swagger `service-api` |
| http://localhost:8025 | Mailpit: podgląd wiadomości e-mail |
| http://localhost:15673 | Panel zarządzania RabbitMQ |

Aby wyświetlić logi:

```bash
docker compose logs -f core-api service-api celery-worker celery-worker-email
```

Aby zatrzymać środowisko:

```bash
docker compose down
```

### Dane demonstracyjne

W środowisku `Development` dokumentacja projektu opisuje inicjalizowanie następujących kont:

| Rola | E-mail | Hasło |
| --- | --- | --- |
| Administrator | `admin@akademik.com` | `admin123` |
| Mieszkaniec | `i.kublytskyi@student.com` | `student123` |

**Uwaga:** są to konta demonstracyjne. Nie uruchamiaj konfiguracji Development z tymi danymi w środowisku publicznym.

### Konfiguracja środowiska

Przykładowe wartości znajdują się w [`.env.example`](.env.example).

| Zmienna | Przeznaczenie |
| --- | --- |
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | Połączenie z PostgreSQL |
| `RABBITMQ_USER`, `RABBITMQ_PASS` | Uwierzytelnianie w RabbitMQ |
| `JWT_SECRET` | Wspólny sekret JWT dla obu API |
| `INTERNAL_API_KEY` | Uwierzytelnianie wywołań integracyjnych przez `X-Api-Key` |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_FROM` | SMTP dla powiadomień e-mail (domyślnie Mailpit) |

## API

API są rozdzielone na dwie grupy.

**`core-api` (`/api/core/`):** logowanie, rejestracja użytkowników przez administratora, odświeżanie tokenów, profil, użytkownicy, pokoje i przydziały; dodatkowo wewnętrzny endpoint pobierania adresu e-mail użytkownika.

**`service-api` (`/api/service/`):** usterki, skargi, wydarzenia, kody QR, powiadomienia i endpoint integracyjny unieważniający kody po usunięciu konta.

Wybrane operacje:

| Metoda | Endpoint | Funkcja |
| --- | --- | --- |
| `POST` | `/api/core/auth/login` | Logowanie |
| `POST` | `/api/core/auth/register` | Rejestracja przez administratora |
| `GET` | `/api/core/me` | Profil użytkownika |
| `GET/POST` | `/api/service/v1/issues` | Lista i tworzenie zgłoszeń usterek |
| `PATCH` | `/api/service/v1/issues/{id}/status` | Zmiana statusu usterki przez administratora |
| `GET/POST` | `/api/service/v1/complaints` | Lista i tworzenie skarg |
| `GET/POST` | `/api/service/v1/qr-codes` | Lista i generowanie kodów QR |
| `POST` | `/api/service/v1/qr-codes/validate` | Jednorazowa walidacja kodu QR |
| `GET/POST` | `/api/service/v1/events` | Lista i tworzenie wydarzeń |
| `GET` | `/api/service/v1/notifications` | Powiadomienia użytkownika |

Szczegóły parametrów i schematów odpowiedzi są dostępne w Swaggerze po uruchomieniu usług. Większość operacji wymaga JWT, a operacje administracyjne odpowiedniej roli.

## Import wydarzeń CSV

```bash
docker compose exec service-api python -m scripts.import_events scripts/sample_events.csv
```

Plik CSV musi zawierać kolumny `user_id`, `title`, `description`, `start_at`, `end_at`, `type`. Daty należy podawać w formacie ISO 8601; obsługiwane typy to `ROOM_INSPECTION`, `PAYMENT`, `OTHER`. Skrypt informuje o liczbie poprawnie i niepoprawnie zaimportowanych wierszy.

## Testowanie

Repozytorium zawiera automatyczne testy obu usług backendowych. **Testy Python wykonują operacje czyszczące na tabelach bazy wskazanej przez `DATABASE_URL`; nie należy ich uruchamiać na bazie zawierającej dane aplikacji.**

### Python / service-api

Najpierw utwórz osobną bazę testową:

```bash
docker compose exec postgres createdb -U akademik akademik_test
```

Następnie, z katalogu głównego repozytorium:

```bash
cd apps/service-api
poetry install
DATABASE_URL=postgresql+asyncpg://akademik:secret@localhost:5432/akademik_test poetry run pytest -q
```

W przykładzie użyto wartości z domyślnego `.env.example`. Zmień dane połączenia, jeśli zmieniono użytkownika lub hasło PostgreSQL. Polecenie tworzenia bazy zakłada domyślnego użytkownika `akademik`.

### C# / core-api

```bash
cd apps/core-api/src
dotnet test Akademik.IntegrationTests/Akademik.IntegrationTests.csproj
```

Testy C# używają bazy w pamięci. Dodatkowe scenariusze manualne znajdują się w [docs/tests/scenarios.md](docs/tests/scenarios.md). README repozytorium podaje 47 testów Python i 9 testów C#; liczby mogą się zmieniać wraz z rozwojem projektu.

## Struktura repozytorium

```text
apps/
  core-api/                ASP.NET Core, EF Core, testy xUnit
  service-api/             FastAPI, modele, Alembic, zadania Celery, testy pytest
  web/                     Next.js i React
docs/
  integration.md           Architektura i scenariusze integracji
  diagrams/                Diagram architektury
  tests/scenarios.md       Scenariusze manualne
nginx/                      Konfiguracja reverse proxy
.github/workflows/         CI
.env.example               Przykładowa konfiguracja
docker-compose.yml          Usługi środowiska lokalnego
```

## Ograniczenia projektu

- Integracja z czytnikami lub elektronicznymi zamkami drzwi nie została potwierdzona; QR są walidowane przez API.
- Zewnętrzna bramka do rzeczywistej wysyłki SMS nie jest potwierdzona; wiadomości SMS są obsługiwane przez wewnętrzne zadania i bazę danych.
- Mailpit służy do lokalnego testowania e-maili, a nie do produkcyjnej dostawy wiadomości.
- Środowisko Docker Compose jest skonfigurowane do developmentu i demonstracji; nie należy wdrażać go publicznie bez przeglądu bezpieczeństwa i konfiguracji produkcyjnej.
- Status działania wszystkich ekranów webowych i pełnego scenariusza end-to-end wymaga dodatkowego sprawdzenia przez uruchomienie aplikacji.

## Autorzy

- **Katsiaryna Kozhych** — `service-api`
- **Ihor Kublytskyi** — `core-api`
- **Anton Hryshchuk** — `web`

## Dokumentacja

- [Integracja usług i architektura](docs/integration.md)
- [Scenariusze testowe](docs/tests/scenarios.md)

## Licencja

Projekt edukacyjny. W repozytorium nie wskazano osobnego pliku licencji; nie oznacza to automatycznie udzielenia licencji na dalsze wykorzystanie kodu.
