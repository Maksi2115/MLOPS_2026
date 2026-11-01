# Etap 1: Budowanie i weryfikacja
FROM debian:bookworm-slim AS builder

# Argument pobierający sumę kontrolną z docker-compose
ARG MODEL_CHECKSUM

RUN apt-get update && apt-get install -y curl ca-certificates && rm -rf /var/lib/apt/lists/*

WORKDIR /build
# Pobieranie modelu
RUN curl -L -o model.llamafile "https://huggingface.co/jartine/TinyLlama-1.1B-Chat-v1.0-GGUF/resolve/main/TinyLlama-1.1B-Chat-v1.0.Q4_K_M.llamafile?download=true"

# Weryfikacja pobranego pliku za pomocą SHA-256
RUN echo "${MODEL_CHECKSUM}  model.llamafile" | sha256sum -c -
RUN chmod +x model.llamafile

# Etap 2: Produkcyjny obraz końcowy (bez narzędzi deweloperskich)
FROM debian:bookworm-slim

RUN useradd -r -s /bin/false llamauser && \
	mkdir -p /app && \
	chown llamauser:llamauser /app

WORKDIR /app

# Kopiujemy wyłącznie zweryfikowany plik i zmieniamy właściciela
COPY --from=builder --chown=llamauser:llamauser /build/model.llamafile /app/model.llamafile

USER llamauser

ENV HOME=/app

# Healthcheck sprawdzający port bez konieczności instalowania polecenia curl (używa /dev/tcp basha)
HEALTHCHECK --interval=15s --timeout=5s --start-period=45s --retries=3 \
  CMD bash -c '</dev/tcp/127.0.0.1/${PORT}' || exit 1

# Uruchomienie korzystające ze zmiennych środowiskowych
CMD ["sh", "-c", "/bin/sh /app/${MODEL_FILE} --server --host 0.0.0.0 --port ${PORT} -t ${THREADS} --nobrowser"]
