# Faro Infra

Infraestrutura oficial do projeto Faro.

Este repositório contém exclusivamente os arquivos necessários para
executar, configurar e operar a infraestrutura do Faro.

A infraestrutura é projetada para ser:

- simples;
- segura;
- reproduzível;
- versionada;
- independente de um servidor específico;
- reconstruível pela Morh.

O servidor é considerado descartável.

---

# 1. Arquitetura

A produção do Faro é composta por quatro componentes principais:

- `faro-frontend`
- `faro-backend`
- `faro-postgres`
- `faro-gateway`

Arquitetura:

```text
                         INTERNET
                            │
                            ▼
                     ┌──────────────┐
                     │ FARO GATEWAY │
                     │   HTTP/HTTPS │
                     └──────┬───────┘
                            │
                  ┌─────────┴─────────┐
                  │                   │
                  ▼                   ▼
           ┌─────────────┐     ┌─────────────┐
           │  FRONTEND   │     │   BACKEND   │
           │  Vue/Vite   │     │   Laravel   │
           └─────────────┘     └──────┬──────┘
                                      │
                                      ▼
                               ┌─────────────┐
                               │ PostgreSQL  │
                               └─────────────┘
