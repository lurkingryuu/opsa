FROM rustlang/rust:nightly-slim AS builder

WORKDIR /usr/src/opsa

# Preserve the repository layout so compile-time paths match local builds.
COPY excretor ./excretor
COPY migrations ./migrations

WORKDIR /usr/src/opsa/excretor

# Build the binary with SQLX offline mode
ENV SQLX_OFFLINE=true
RUN cargo build --release

FROM debian:bookworm-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY --from=builder /usr/src/opsa/excretor/target/release/excretor /app/excretor

RUN mkdir -p /app/assets
ENV STATIC_ASSETS_DIR=/app/assets

EXPOSE 3000
CMD ["/app/excretor"]
