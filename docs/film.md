# Film prezentujący działanie systemu Akademik

Film pokazuje, jak działa system **Akademik**, z jakich usług się składa i w jaki sposób te usługi ze sobą współpracują. Przechodzimy przez najważniejsze funkcje aplikacji, a przy okazji prezentujemy zastosowane mechanizmy integracji oraz zachowanie systemu w razie awarii.

1. **Architektura systemu** - na początku pokazujemy uruchomione usługi i diagram architektury. Wyjaśniamy, jak system jest podzielony na niezależne komponenty (SOA) i jak się one komunikują.

2. **Import wydarzeń z pliku CSV** – importujemy wydarzenia z pliku, a następnie sprawdzamy, czy pojawiły się w aplikacji. W ten sposób pokazujemy, jak system korzysta z danych pochodzących z zewnętrznego źródła.

3. **Obsługa skarg i powiadomień** – mieszkaniec wysyła skargę, a system tworzy powiadomienia dla niego i administratora. Pokazujemy, jak wiadomości przechodzą między usługami oraz jak mechanizm *Content-Based Router* kieruje je do odpowiednich kolejek.

4. **Zamykanie zgłoszenia i wysyłanie e-maila** – administrator zamyka zgłoszenie, po czym mieszkaniec otrzymuje wiadomość e-mail. Na tym przykładzie pokazujemy asynchroniczne przetwarzanie zadań z wykorzystaniem RabbitMQ i Celery.

5. **Awaria workera e-mail** – zatrzymujemy usługę wysyłającą wiadomości i sprawdzamy, co dzieje się z oczekującym e-mailem. RabbitMQ przechowuje zadanie w kolejce, a po ponownym uruchomieniu workera wiadomość zostaje wysłana. Pokazuje to, że chwilowa awaria workera nie musi oznaczać utraty wiadomości.

6. **Awaria usługi core-api** – tym razem wyłączamy jedną z usług, z której korzysta worker. Nieudane wywołania są ponawiane z coraz dłuższymi przerwami (*exponential backoff*). Po przywróceniu usługi wiadomość może zostać wysłana, co ilustruje sposób obsługi chwilowych problemów z komunikacją.

7. **Autoryzacja i synchronizacja danych** – pokazujemy, jak usługi zabezpieczają komunikację za pomocą klucza API. Następnie usuwamy użytkownika i sprawdzamy, czy jego kod QR został unieważniony również w drugiej usłudze. To przykład synchronizacji danych między komponentami systemu.

8. **Testy integracyjne** – na koniec uruchamiamy testy automatyczne i pokazujemy ich wyniki. Dzięki nim można sprawdzić nie tylko pojedyncze funkcje, ale też poprawność współpracy między usługami.

**Celem filmu** jest przedstawienie zarówno możliwości aplikacji Akademik, jak i rozwiązań stojących za jej działaniem: komunikacji między usługami, kolejek wiadomości, zabezpieczeń oraz mechanizmów pozwalających radzić sobie z awariami.
