# Dokumentacja Wdrożenia: Llamafile w kontenerze Docker (Zadanie 3)

Poniższa dokumentacja przedstawia proces konteneryzacji modelu językowego Llamafile. Środowisko oparto na silniku Docker, wykorzystując wieloetapowe budowanie obrazu (multi-stage build) oraz plik `compose.yaml`.

## 1\. Wybór sposobu dostarczenia wag modelu

**Uzasadnienie:** Zdecydowano się na włączenie wag modelu (pliku llamafile) bezpośrednio do obrazu kontenera na etapie budowania, rezygnując z montowania ich jako wolumen. Wybór ten gwarantuje pełną hermetyzację i przenośność (immutable infrastructure). Po pobraniu obrazu z rejestru na dowolną inną maszynę, aplikacja jest gotowa do natychmiastowego działania, bez konieczności ręcznego pobierania gigabajtowych plików przez administratora.

## 2\. Pliki konfiguracyjne

### Dockerfile

Zastosowano obraz bazowy ze ściśle określoną wersją. Model pobierany jest w pierwszym etapie i weryfikowany sumą kontrolną SHA-256. W docelowym obrazie uruchamiany jest przez dedykowanego użytkownika (nie-roota).

```dockerfile
# Etap 1: Budowanie i weryfikacja
FROM debian:bookworm-slim AS builder
ARG MODEL\_CHECKSUM
RUN apt-get update \&\& apt-get install -y curl ca-certificates \&\& rm -rf /var/lib/apt/lists/\*
WORKDIR /build
RUN curl -L -o model.llamafile "https://huggingface.co/jartine/TinyLlama-1.1B-Chat-v1.0-GGUF/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q4\_K\_M.llamafile?download=true"
RUN echo "${MODEL\_CHECKSUM}  model.llamafile" | sha256sum -c -
RUN chmod +x model.llamafile

# Etap 2: Produkcyjny obraz końcowy
FROM debian:bookworm-slim
RUN useradd -r -s /bin/false llamauser \&\&     mkdir -p /app \&\&     chown llamauser:llamauser /app
WORKDIR /app
COPY --from=builder --chown=llamauser:llamauser /build/model.llamafile /app/model.llamafile
USER llamauser
ENV HOME=/app
HEALTHCHECK --interval=15s --timeout=5s --start-period=45s --retries=3   CMD bash -c '</dev/tcp/127.0.0.1/${PORT}' || exit 1
CMD \["sh", "-c", "/bin/sh /app/${MODEL\_FILE} --server --host 0.0.0.0 --port ${PORT} -t ${THREADS} --nobrowser"]
```

### compose.yaml

```yaml
services:
  llamafile:
    image: bmaksymiuk/llamafile:1.0
    container\_name: llamafile\_docker
    restart: always
    env\_file:
      - .env
    ports:
      - "192.168.56.102:${PORT}:${PORT}"
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 1536M
```

### .env.example

```env
PORT=8080
MODEL\_FILE=model.llamafile
THREADS=2
```

## 3\. Weryfikacja działania środowiska

**Rozmiar zbudowanego obrazu:**

```text
IMAGE                      ID             DISK USAGE   CONTENT SIZE   EXTRA
bmaksymiuk/llamafile:1.0   f73f8568a0a1       1.91GB          918MB    U   
```

**Izolacja sieciowa (Bindowanie portu):**

```text
LISTEN 0      4096   192.168.56.102:8080      0.0.0.0:\*
```

**Odpytanie modelu (REST API) z gospodarza:**

```json
{"choices":\[{"finish\_reason":"stop","index":0,"message":{"content":"Czytam.</s>","role":"assistant"}}],"created":1793549723,"id":"chatcmpl-Ng2dDChWbc6ycdrBGsNa54UwRBWBuPyc","model":"LLaMA\_CPP","object":"chat.completion","usage":{"completion\_tokens":6,"prompt\_tokens":31,"total\_tokens":37}}
```

## 4\. Porównanie wdrożeń: systemd vs Docker

|Cecha|systemd (Zadanie 2)|Docker (Zadanie 3)|
|-|-|-|
|**Zajmowane miejsce na dysku**|Ok. 680 MB (tylko waga pliku llamafile)|Ok. 760 MB (llamafile + warstwa bazowa debian:bookworm-slim)|
|**Czas do pierwszej odpowiedzi**|Krótszy (bezpośrednie uruchomienie procesu)|Nieco dłuższy (narzut demona Dockera oraz czas inicjalizacji healthchecku)|
|**Zużycie pamięci (RAM)**|Ok. 820 MB|Ok. 820 MB + niewielki narzut izolacji kontenera|
|**Kroki do uruchomienia na nowej maszynie**|Wiele ręcznych kroków (dodanie użytkownika, pobranie z curl, chown, edycja .service, daemon-reload)|Zaledwie dwa kroki (instalacja silnika Docker, wpisanie `docker compose up -d`)|
||||



## 4\. Instrukcja uruchomienia(dla nowej maszyny)



Sklonuj repozytorium z plikami konfiguracyjnymi.



Zmień nazwę pliku .env.example na .env i w razie potrzeby dostosuj zmienne (np. port).



Mając zainstalowany silnik Docker, przejdź do katalogu z plikami i uruchom środowisko poleceniem:

docker compose up -d



Poczekaj na pobranie i zbudowanie obrazu. Weryfikację stanu usługi przeprowadzisz poleceniem docker compose ps (oczekiwany status: healthy).

