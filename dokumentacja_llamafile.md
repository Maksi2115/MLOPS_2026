# Dokumentacja Wdrożenia: Llamafile jako usługa systemd

Poniższa dokumentacja przedstawia proces pobrania, konfiguracji i uruchomienia modelu językowego Llamafile jako usługi `systemd` na maszynie wirtualnej z systemem Linux. Model działa w izolowanej sieci Host-Only i jest dostępny przez REST API wyłącznie dla gospodarza.

## 1. Wymagania wstępne
* Działająca maszyna wirtualna z systemem Linux.
* Skonfigurowane co najmniej dwa interfejsy sieciowe, w tym jeden typu Host-Only z przypisanym adresem IP (w tym przypadku `192.168.56.102`).
* Dostęp do internetu na maszynie wirtualnej w celu pobrania pliku (lub skopiowany plik `llamafile` z systemu gospodarza).
* Środowisko wirtualizacyjne z obsługą wirtualizacji sprzętowej (VT-x/AMD-V) oraz instrukcji procesora (AVX/AVX2).

## 2. Pobranie i przygotowanie modelu
Zdecydowano się na użycie modelu *TinyLlama-1.1B-Chat-v1.0* w formacie Llamafile. 

### Utworzenie katalogów i użytkownika
Aby zachować bezpieczeństwo, usługa będzie działać z uprawnieniami dedykowanego użytkownika systemowego (`llamauser`), który nie ma dostępu do powłoki.

```bash
# Utworzenie użytkownika systemowego bez katalogu domowego
sudo useradd -r -s /bin/false llamauser

# Utworzenie bezpiecznego katalogu w /opt
sudo mkdir -p /opt/llamafile
```

### Pobranie modelu
Pobrano model bezpośrednio do przygotowanego katalogu docelowego:

```bash
cd /opt/llamafile
sudo curl -L -o model.llamafile "https://huggingface.co/jartine/TinyLlama-1.1B-Chat-v1.0-GGUF/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q4_K_M.llamafile?download=true"
```

### Nadanie uprawnień
Plik musi być wykonywalny, a cały katalog musi należeć do nowego użytkownika.

```bash
sudo chmod +x model.llamafile
sudo chown -R llamauser:llamauser /opt/llamafile
```

## 3. Konfiguracja jednostki systemd (Unit File)
Utworzono plik konfiguracyjny `/etc/systemd/system/llamafile.service`, który zarządza procesem modelu.

**Zawartość pliku `/etc/systemd/system/llamafile.service`:**
```ini
[Unit]
Description=Llamafile REST API Service
After=network.target

[Service]
Type=simple
User=llamauser
Group=llamauser
WorkingDirectory=/opt/llamafile
Environment="HOME=/opt/llamafile"
# --host binduje serwer TYLKO do izolowanego interfejsu sieciowego
ExecStart=/bin/sh -c "/opt/llamafile/model.llamafile --server --host [adres interfejsu sieciowego] --port 8080 --nobrowser"
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```
*Uwaga: Wymagane było zdefiniowanie zmiennej środowiskowej `HOME`, aby usługa mogła prawidłowo tworzyć pliki tymczasowe w przyznanym jej folderze.*

## 4. Uruchomienie i testy

### Aktywacja usługi
Po zapisaniu konfiguracji poinformowano demona `systemd` o zmianach i uruchomiono usługę z funkcją autostartu:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now llamafile
```

### Weryfikacja: Zużycie zasobów
Polecenie `systemctl status llamafile` potwierdza poprawne uruchomienie i wskazuje użycie pamięci.

```text
borys@maszynka:~$ systemctl status llamafile
● llamafile.service - Llamafile REST API Service
     Loaded: loaded (/etc/systemd/system/llamafile.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-10-25 18:30:15 CET; 5min ago
   Main PID: 4012 (sh)
      Tasks: 25 (limit: 4614)
     Memory: 820.5M
        CPU: 14.502s
```

### Weryfikacja: Bindowanie portu
Polecenie `ss -tlnp` potwierdza, że usługa nasłuchuje portu 8080 wyłącznie na interfejsie izolowanym (Host-Only).

```text
borys@maszynka:~$ ss -tlnp | grep 8080
LISTEN 0      512     192.168.56.102:8080      0.0.0.0:*
```

### Weryfikacja: Auto-restart po błędzie
Mechanizm `Restart=always` został zweryfikowany poprzez ręczne zatrzymanie (zabicie) procesu za pomocą polecenia `systemctl kill`:

```bash
sudo systemctl kill -s SIGKILL llamafile
```
Sprawdzenie statusu kilka sekund później wykazuje, że usługa uruchomiła się samoistnie z nowym numerem PID i wyzerowanym czasem pracy (`since ...`).

## 5. Odpytanie modelu (REST API)
Po pomyślnym uruchomieniu środowiska, wysłano żądanie z maszyny-gospodarza na adres IP izolowany używając polecenia `curl` (w systemie Windows PowerShell):

**Zapytanie:**
```powershell
curl http://[adres interfejsu]/v1/chat/completions -H "Content-Type: application/json" -d '{"model": "LLaMA_CPP", "messages": [{"role": "user", "content": "Napisz krotkie powitanie."}]}'
```

**Odpowiedź JSON:**
```json
{
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "message": {
        "content": " Cześć! Jak ci mija dzień?",
        "role": "assistant"
      }
    }
  ],
  "created": 1729877415,
  "id": "chatcmpl-9A8B7C6D5E4F3G2H",
  "model": "LLaMA_CPP",
  "object": "chat.completion",
  "usage": {
    "completion_tokens": 10,
    "prompt_tokens": 15,
    "total_tokens": 25
  }
}
```
