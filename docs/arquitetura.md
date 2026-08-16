# Arquitetura do Faro

**Versão:** 1.0  
**Data:** 16/08/2026  
**Status:** Proposta inicial  
**Projeto:** Faro  
**Organização:** Morh

---

# 1. Objetivo

Este documento define a arquitetura de infraestrutura planejada para o
Faro.

A arquitetura tem como objetivos:

- simplificar o deploy;
- separar frontend, backend, banco e gateway;
- permitir atualização independente dos componentes;
- reduzir configurações manuais no servidor;
- manter dados persistentes separados dos containers;
- permitir reconstrução completa da infraestrutura;
- permitir recuperação após perda do servidor;
- permitir que a Morh opere a infraestrutura sem depender de uma máquina
  específica.

Este documento representa a arquitetura desejada antes da implementação.

---

# 2. Estrutura dos projetos

O Faro será organizado localmente da seguinte forma:

    ~/Documentos/GitHub/faro/

    ├── faro-frontend/
    ├── faro-backend/
    └── faro-infra/

Cada diretório representa um projeto/repositório independente.

## faro-frontend

Contém exclusivamente o código da aplicação frontend.

Tecnologias atuais:

- Vue 3
- Vite
- Tailwind CSS
- JavaScript

Repositório:

    github.com/studiomorh/faro-frontend

---

## faro-backend

Contém exclusivamente o código da API/backend.

Tecnologias atuais:

- PHP 8.3+
- Laravel 13
- Laravel Sanctum
- Composer

Repositório:

    github.com/studiomorh/faro-backend

---

## faro-infra

Contém exclusivamente a infraestrutura necessária para executar o Faro.

Responsabilidades:

- Docker Compose;
- configuração dos serviços;
- gateway;
- redes;
- volumes;
- configurações de infraestrutura;
- scripts operacionais;
- documentação;
- procedimentos de recuperação.

O `faro-infra` não conterá cópias do código do frontend ou backend.

---

# 3. Arquitetura geral

A produção será composta inicialmente por quatro containers:

1. `faro-gateway`
2. `faro-frontend`
3. `faro-backend`
4. `faro-postgres`

O armazenamento persistente será fornecido através de volumes,
não através de um container adicional.

Arquitetura:

                        INTERNET
                           │
                           │ HTTPS
                           ▼
                  ┌─────────────────┐
                  │  FARO-GATEWAY   │
                  │                 │
                  │ HTTP / HTTPS    │
                  │ Routing         │
                  └────────┬────────┘
                           │
                 ┌─────────┴─────────┐
                 │                   │
                 │ /                 │ /api/*
                 ▼                   ▼
          ┌──────────────┐     ┌──────────────┐
          │   FRONTEND   │     │   BACKEND    │
          │              │     │              │
          │ Vue + Vite   │     │ Laravel/PHP  │
          └──────────────┘     └──────┬───────┘
                                      │
                                      │
                                      ▼
                               ┌──────────────┐
                               │  POSTGRESQL  │
                               │              │
                               │    Dados     │
                               └──────────────┘

                              +
                              │
                              ▼
                       STORAGE PERSISTENTE

---

# 4. Containers

## 4.1 faro-gateway

O gateway será o único componente diretamente exposto à Internet.

Responsabilidades:

- receber conexões HTTP/HTTPS;
- terminar TLS;
- direcionar requisições para o frontend;
- direcionar requisições `/api` para o backend;
- disponibilizar arquivos públicos quando necessário;
- controlar a entrada externa da aplicação.

O gateway não deverá conter código da aplicação.

---

## 4.2 faro-frontend

Responsável pela aplicação Vue.

Responsabilidades:

- executar/buildar o frontend;
- disponibilizar os arquivos estáticos de produção;
- permanecer independente do ciclo de atualização do backend.

O frontend não deverá precisar expor uma porta diretamente à Internet.

A comunicação externa ocorrerá através do gateway.

---

## 4.3 faro-backend

Responsável pela aplicação Laravel.

Responsabilidades:

- API;
- autenticação;
- regras de negócio;
- acesso ao banco;
- migrations;
- uploads;
- arquivos persistentes;
- demais funcionalidades do Laravel.

O backend não deverá ser diretamente exposto à Internet.

Sua comunicação externa ocorrerá através do gateway.

---

## 4.4 faro-postgres

Responsável pelo banco de dados PostgreSQL.

Responsabilidades:

- armazenar dados da aplicação;
- armazenar sessões;
- armazenar cache quando configurado para database;
- armazenar filas quando configuradas para database.

O PostgreSQL não deverá possuir porta pública.

Seu acesso deverá ocorrer exclusivamente através da rede interna
Docker.

---

# 5. Rede Docker

Os containers deverão compartilhar uma rede privada do Faro.

Exemplo conceitual:

    faro-gateway
          │
          ├──────── faro-frontend
          │
          └──────── faro-backend
                         │
                         └──────── faro-postgres

A comunicação entre serviços deverá utilizar os nomes dos serviços
Docker, e não `localhost`.

Exemplo:

    Backend → PostgreSQL

deverá utilizar conceitualmente:

    DB_HOST=postgres

e não:

    DB_HOST=127.0.0.1

---

# 6. Exposição de portas

Apenas o gateway deverá possuir portas públicas.

Arquitetura pretendida:

    Internet
       │
       ├── :80  → Gateway
       └── :443 → Gateway

As portas internas dos demais containers não deverão ser publicadas
diretamente no host sem necessidade.

Especialmente:

    PostgreSQL :5432

não deverá ser exposto publicamente.

---

# 7. Gateway e roteamento

O gateway deverá implementar o seguinte comportamento:

    https://app.brfaro.com/
        ↓
    frontend

    https://app.brfaro.com/api/*
        ↓
    backend

    https://app.brfaro.com/storage/*
        ↓
    arquivos públicos

O frontend já utiliza `/api` como base da API em produção.

Portanto, não é necessário expor uma URL pública independente para a API.

---

# 8. Domínios

O domínio principal planejado para a aplicação é:

    app.brfaro.com

Também existe atualmente o ambiente:

    crbv.brfaro.com

A utilização definitiva de cada domínio deverá ser definida durante a
implementação do gateway.

A arquitetura deverá permitir múltiplos domínios sem exigir alteração
no código da aplicação.

---

# 9. HTTPS

O acesso público deverá utilizar HTTPS.

O gateway será responsável pelo tratamento dos certificados TLS.

Certificados e chaves privadas não deverão ser versionados no Git.

---

# 10. Frontend e API

O frontend possui atualmente a seguinte lógica:

    produção:
        /api

    desenvolvimento:
        http://127.0.0.1:8000/api

Essa separação deverá ser preservada.

Em produção, frontend e backend deverão compartilhar o mesmo domínio
através do gateway.

Isso evita a necessidade de configurar uma URL pública diferente para
a API.

---

# 11. PostgreSQL

O PostgreSQL deverá utilizar armazenamento persistente.

O ciclo de vida do container não deverá determinar o ciclo de vida dos
dados.

Conceitualmente:

    faro-postgres
         │
         ▼
    faro_postgres_data

Remover/recriar o container não deverá apagar os dados.

A destruição do volume deverá ser considerada uma operação destrutiva e
exigir procedimento específico.

---

# 12. Storage da aplicação

O Laravel utiliza:

    storage/app/public

para arquivos públicos, incluindo atualmente avatars.

Esse diretório deverá possuir armazenamento persistente.

O código da aplicação poderá ser substituído durante um deploy sem
destruir os arquivos enviados pelos usuários.

Conceito:

    faro-backend
          │
          ▼
    storage/app/public
          │
          ▼
    volume persistente

A estratégia definitiva de volume será definida durante a implementação.

---

# 13. Migrations

A estrutura do banco será controlada pelas migrations do Laravel.

O código das migrations deverá permanecer no repositório do backend.

Durante um deploy que contenha alteração estrutural do banco:

    código atualizado
          ↓
    container atualizado
          ↓
    php artisan migrate --force

Migrations deverão ser executadas somente quando necessário.

Um deploy comum do frontend não deverá executar migrations.

---

# 14. Cache, sessões e filas

A configuração atual utiliza PostgreSQL para:

- sessões;
- cache;
- filas.

Configuração atual:

    SESSION_DRIVER=database
    CACHE_STORE=database
    QUEUE_CONNECTION=database

Não será adicionado Redis inicialmente.

Redis poderá ser introduzido futuramente caso exista necessidade
técnica real.

A arquitetura deverá evitar serviços adicionais sem necessidade.

---

# 15. Jobs e processamento assíncrono

No estado atual do projeto não foi identificada necessidade de um
worker dedicado.

Portanto, a arquitetura inicial não terá um container separado para:

- Redis;
- worker;
- Supervisor;
- WebSocket;
- scheduler dedicado.

Caso o sistema passe a exigir processamento assíncrono real, a arquitetura
poderá ser expandida.

---

# 16. E-mail

O backend possui suporte à configuração de mailers do Laravel.

E-mail não será tratado como um container próprio.

Quando necessário, o backend utilizará um serviço SMTP ou outro mailer
externo através de variáveis de ambiente.

Credenciais de e-mail não deverão ser versionadas.

---

# 17. S3 e armazenamento externo

O Laravel possui suporte configurado para S3.

A arquitetura inicial não depende de S3.

O armazenamento local persistente será utilizado inicialmente.

A migração para armazenamento externo poderá ser feita futuramente sem
alterar a arquitetura principal dos containers.

---

# 18. Segurança

A arquitetura seguirá o princípio de menor exposição possível.

Exposição pública:

    Gateway
       │
       └── HTTP/HTTPS

Acesso privado:

    Frontend
    Backend
    PostgreSQL

O PostgreSQL não será acessível diretamente pela Internet.

O backend não será acessível diretamente pela Internet.

O frontend não precisará publicar sua porta diretamente no host.

Segredos serão fornecidos através de configuração externa ao código.

Nunca deverão ser versionados:

- senhas;
- tokens;
- APP_KEY;
- credenciais de banco;
- credenciais SMTP;
- chaves privadas;
- certificados privados;
- credenciais AWS;
- outros segredos de produção.

---

# 19. Persistência

A infraestrutura terá pelo menos dois tipos de dados persistentes:

## Banco

    PostgreSQL
        ↓
    volume persistente

## Arquivos da aplicação

    Laravel storage/app/public
        ↓
    volume persistente

O código da aplicação deverá ser considerado descartável.

Os dados deverão ser considerados persistentes.

---

# 20. Deploy

O objetivo é que o deploy normal seja simples.

## Frontend

    local:
        git add
        git commit
        git push

    servidor:
        git pull
        atualizar/reconstruir frontend

## Backend

    local:
        git add
        git commit
        git push

    servidor:
        git pull
        atualizar/reconstruir backend

## Infraestrutura

    local:
        git add
        git commit
        git push

    servidor:
        git pull
        aplicar alterações da infraestrutura

## Banco

Somente quando houver migrations:

    atualizar backend
        ↓
    executar migrations

---

# 21. Deploy completo

Quando houver alterações em vários componentes:

    git pull
         ↓
    atualizar infraestrutura
         ↓
    construir containers necessários
         ↓
    iniciar serviços
         ↓
    executar migrations, se necessário
         ↓
    verificar saúde dos serviços

O PostgreSQL não deverá ser destruído durante um deploy normal.

---

# 22. Rollback

A arquitetura deverá permitir rollback através das versões dos
repositórios.

O procedimento deverá permitir retornar:

- frontend;
- backend;
- infraestrutura;

para uma versão anterior conhecida.

Rollback de banco deverá ser tratado separadamente, pois migrations
podem alterar dados de forma irreversível.

O procedimento definitivo de rollback será documentado após a
implementação e testes.

---

# 23. Backup

O PostgreSQL deverá possuir procedimento documentado de backup.

O backup deverá existir fora do ciclo de vida do container.

A infraestrutura deverá permitir:

    PostgreSQL
        ↓
    backup
        ↓
    armazenamento externo

A localização definitiva dos backups será definida posteriormente.

Arquivos persistentes do Laravel também deverão possuir estratégia de
backup.

---

# 24. Recuperação de servidor

O servidor deverá ser considerado substituível.

Em caso de perda completa do servidor:

    novo servidor
         ↓
    instalar Docker
         ↓
    obter faro-infra
         ↓
    obter faro-frontend
         ↓
    obter faro-backend
         ↓
    restaurar configurações/segredos
         ↓
    restaurar banco
         ↓
    restaurar arquivos persistentes
         ↓
    subir infraestrutura

A documentação operacional deverá permitir executar esse processo sem
depender da configuração manual do servidor antigo.

---

# 25. Ambiente de testes

Está planejado futuramente um ambiente separado, inicialmente previsto
como:

    serv.brfaro.com

Esse ambiente deverá utilizar:

- containers próprios;
- banco próprio;
- configurações próprias;
- arquivos próprios;
- certificados próprios.

O ambiente de testes não deverá compartilhar dados de produção.

A infraestrutura deverá permitir criar esse ambiente utilizando a mesma
arquitetura base.

---

# 26. Princípios arquiteturais

A infraestrutura do Faro deverá seguir os seguintes princípios:

1. Simplicidade antes de complexidade.
2. Segurança antes de conveniência.
3. Persistência separada dos containers.
4. Código versionado.
5. Infraestrutura versionada.
6. Segredos fora do Git.
7. Serviços não utilizados não devem existir.
8. Containers devem ter responsabilidades claras.
9. O servidor deve ser substituível.
10. Produção não deve depender de alterações manuais esquecidas.
11. Deploy deve ser reproduzível.
12. Recuperação deve ser documentada.
13. Frontend e backend devem poder ser atualizados separadamente.
14. O banco não deve ser destruído durante deploys normais.
15. Toda decisão estrutural importante deve ser documentada.

---

# 27. Estado da arquitetura

Este documento representa a arquitetura proposta.

Os seguintes pontos ainda precisam ser definidos durante a implementação:

- tecnologia final do gateway;
- estratégia de build do frontend;
- estratégia de imagem do backend;
- estratégia definitiva de volumes;
- gerenciamento de secrets;
- emissão/renovação de certificados;
- estratégia de backup;
- estratégia de restore;
- estratégia de rollback;
- health checks;
- política de logs;
- política de atualização dos containers.

Essas decisões deverão ser implementadas, testadas e posteriormente
registradas neste documento.

---

# 28. Regra de mudança

Nenhuma alteração estrutural importante deverá ser feita sem atualizar
este documento ou o `REESTRUTURACAO.md`.

A arquitetura documentada deve representar a infraestrutura realmente
utilizada.

Se a implementação divergir deste documento, a divergência deverá ser
registrada e justificada.
