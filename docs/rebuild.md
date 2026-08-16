# Reestruturação da Infraestrutura do Faro

> Documento técnico de acompanhamento da reconstrução da infraestrutura
> de desenvolvimento e produção do Faro.

**Início:** 16/08/2026  
**Status:** Planejamento / Inventário inicial

---

# 1. Objetivo

Reestruturar a infraestrutura do Faro para que o sistema tenha uma
arquitetura simples, segura, reproduzível e independente de uma máquina
específica.

A infraestrutura deve ser reconstruível pela Morh a partir dos
repositórios Git, sem depender de configurações manuais esquecidas em
um servidor.

O objetivo principal é que, caso o computador de desenvolvimento ou o
servidor de produção seja perdido, seja possível reconstruir o ambiente
seguindo a documentação do projeto.

---

# 2. Motivações

A infraestrutura utilizada atualmente em produção tornou o processo de
deploy excessivamente manual e frágil.

Entre os problemas encontrados:

- alterações manuais diretamente no servidor;
- dependências instaladas manualmente no servidor;
- versões diferentes de Node entre ambientes;
- problemas de permissões em arquivos;
- diretórios `dist` sendo manipulados manualmente;
- backups manuais de `dist`;
- configuração de Nginx dentro do servidor;
- Docker parcialmente configurado;
- Docker Compose utilizado como infraestrutura geral;
- dificuldade para identificar exatamente qual estado do código está em produção;
- dificuldade para realizar rollback;
- dificuldade para reconstruir o servidor do zero.

A nova arquitetura deve eliminar, ou reduzir drasticamente, essas
dependências manuais.

---

# 3. Princípio principal

A infraestrutura deve pertencer ao código e à documentação da Morh,
e não ao servidor.

O servidor deve ser tratado como uma máquina substituível.

A regra desejada é:

    Código local
        ↓
    Git
        ↓
    Servidor
        ↓
    Docker
        ↓
    Aplicação

Se o servidor for destruído, devemos conseguir criar outro e reconstruir
o ambiente.

Se o computador da Morh for perdido, devemos conseguir baixar os
repositórios novamente e reconstruir o ambiente.

---

# 4. Estado atual conhecido

## 4.1 Frontend

Repositório local:

    ~/faro

O repositório está atualmente limpo e sincronizado com `origin/main`.

Commit atual no momento do inventário:

    b347aad ajusta exibições de coisass

Estrutura observada:

    dist/
    node_modules/
    public/
    src/
    .git/
    package.json
    package-lock.json
    vite.config.js
    tailwind.config.js
    README.md
    vercel.json
    .vscode/

O frontend utiliza:

- Vue 3
- Vite 7.3.1
- Tailwind CSS 4
- Axios
- Pinia
- Vue Router
- Leaflet
- QRCode Vue
- Zod
- Tabler Icons
- Fantasticon

Scripts atualmente definidos:

    npm run dev
    npm run build
    npm run preview
    npm run build:icons

O build de produção atualmente é realizado através de:

    npm run build

---

## 4.2 Backend

Repositório local:

    ~/faro-backend

O repositório está atualmente limpo e sincronizado com `origin/main`.

Commit atual no momento do inventário:

    2b1f063 salva todos os campos do evento

O backend utiliza:

- Laravel 13.8
- PHP ^8.3
- Laravel Sanctum
- Laravel Tinker
- Composer

Estrutura observada:

    app/
    bootstrap/
    config/
    database/
    public/
    resources/
    routes/
    storage/
    tests/
    vendor/
    artisan
    composer.json
    composer.lock
    .env
    .env.example

O `.env` está corretamente ignorado pelo Git.

---

## 4.3 Banco de dados

O projeto utiliza PostgreSQL em produção.

As migrations existentes fazem parte do código do backend e incluem,
entre outras:

- users
- cache
- jobs
- events
- event_managers
- event_attachments
- event_agents
- personal_access_tokens
- resources
- notifications
- alterações de schema de usuários e recursos

As migrations deverão continuar sendo a fonte versionada da estrutura
do banco.

O banco de produção não deverá ser armazenado dentro do repositório.

---

## 4.4 Storage

O backend possui a estrutura Laravel de `storage/`.

O repositório contém os `.gitignore` necessários para os diretórios de
storage, enquanto arquivos gerados pela aplicação não devem fazer parte
do código versionado.

A estratégia definitiva para persistência de uploads e arquivos ainda
será definida durante a reestruturação.

---

## 4.5 Infraestrutura local atual

No momento do inventário não existe uma infraestrutura Docker do Faro
configurada no ambiente local.

Não foi encontrado:

    docker-compose.yml
    docker-compose.yaml
    Dockerfile

dentro dos projetos atuais do Faro.

O único Dockerfile encontrado no diretório pessoal pertence ao ambiente
do NVM:

    ~/.nvm/Dockerfile

Portanto, a infraestrutura do Faro poderá ser reconstruída de forma
limpa, sem necessidade de preservar a arquitetura Docker existente no
servidor.

---

# 5. Arquitetura pretendida

A arquitetura planejada para produção terá quatro componentes
principais em containers:

    ┌───────────────────────────────┐
    │          Gateway              │
    │   HTTPS / domínio / routing   │
    └───────────────┬───────────────┘
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
    ┌─────────────┐     ┌─────────────┐
    │  Frontend   │     │   Backend   │
    │   Docker    │     │   Docker    │
    └─────────────┘     └──────┬──────┘
                               │
                               ▼
                        ┌─────────────┐
                        │ PostgreSQL  │
                        │   Docker    │
                        └─────────────┘

Containers planejados:

- faro-frontend
- faro-backend
- faro-postgres
- faro-gateway

---

# 6. Responsabilidade dos containers

## faro-frontend

Responsável exclusivamente pela aplicação frontend.

Deve permitir que o frontend seja construído e atualizado
independentemente do backend.

A ideia é que uma alteração exclusivamente visual possa ser publicada
sem reconstruir ou reiniciar o backend.

---

## faro-backend

Responsável exclusivamente pela aplicação Laravel.

Deve permitir que alterações no backend sejam publicadas
independentemente do frontend.

---

## faro-postgres

Responsável exclusivamente pelo banco de dados PostgreSQL.

O container não deverá ser recriado desnecessariamente durante um deploy
normal da aplicação.

Os dados deverão existir em armazenamento persistente independente do
ciclo de vida do container.

---

## faro-gateway

Responsável pela entrada pública do sistema.

Responsabilidades previstas:

- HTTPS
- certificados
- roteamento
- comunicação entre domínio e serviços
- exposição pública do frontend
- encaminhamento das requisições da API para o backend

A tecnologia exata do gateway ainda será definida.

---

# 7. Repositórios

A princípio, frontend e backend continuarão em repositórios separados.

Motivo:

- permitir deploy independente;
- separar responsabilidades;
- permitir alterar somente o frontend;
- permitir alterar somente o backend;
- reduzir acoplamento;
- facilitar manutenção.

Um terceiro repositório será utilizado para infraestrutura:

    faro-infra

Responsabilidade:

- Dockerfiles relacionados à infraestrutura;
- Docker Compose/orquestração;
- configuração do gateway;
- scripts de instalação e recuperação;
- documentação operacional;
- procedimentos de backup e restauração.

---

# 8. Fluxo de deploy pretendido

O objetivo é chegar a um fluxo simples.

## Alteração somente no frontend

Local:

    git add
    git commit
    git push

Servidor:

    git pull
    docker compose ...
    
O processo exato de reconstrução/reinício ainda será definido.

---

## Alteração somente no backend

Local:

    git add
    git commit
    git push

Servidor:

    git pull
    docker compose ...

O banco somente deverá ser alterado quando houver migrations ou outra
alteração que realmente exija mudança estrutural.

---

## Alteração de infraestrutura

Local:

    git add
    git commit
    git push

Servidor:

    git pull

Depois, os serviços necessários serão reconstruídos/reiniciados.

---

## Alteração do banco

As alterações de estrutura do banco deverão ser versionadas através das
migrations do Laravel.

O deploy deverá executar somente as migrations necessárias.

O PostgreSQL não deverá ser destruído e recriado em um deploy comum.

---

# 9. Filosofia de deploy

Não utilizar como requisito:

- GitHub Actions;
- GHCR;
- pipelines externos;
- serviços de CI/CD obrigatórios;
- ferramentas adicionais desnecessárias.

A primeira proposta é manter o fluxo simples:

    Git
      ↓
    Servidor
      ↓
    Docker
      ↓
    Aplicação

O servidor deverá obter o código dos repositórios e construir/executar
os containers localmente.

---

# 10. Segurança

A infraestrutura deverá ser segura sem depender da memória de uma
pessoa.

Segredos não deverão ser versionados.

Exemplos:

- senhas do PostgreSQL;
- APP_KEY;
- credenciais externas;
- tokens;
- certificados privados;
- credenciais de produção.

O repositório deverá conter apenas:

- código;
- configurações não sensíveis;
- templates;
- exemplos;
- documentação;
- scripts necessários para reconstrução.

---

# 11. Reprodutibilidade

A infraestrutura deverá permitir:

### Computador destruído

Comprar/usar outro computador → baixar os repositórios → seguir README
→ reconstruir ambiente.

### Servidor destruído

Criar novo servidor → instalar Docker → baixar `faro-infra` → recuperar
configurações/segredos → subir serviços → restaurar banco/arquivos.

### Servidor de testes

Criar futuramente um ambiente separado utilizando a mesma infraestrutura,
com configurações próprias.

A criação do servidor de testes não faz parte da primeira etapa da
reestruturação.

---

# 12. Ambiente de testes

Está planejado futuramente um ambiente separado, por exemplo:

    serv.brfaro.com

Esse ambiente deverá permitir testar:

- frontend;
- backend;
- migrations;
- uploads;
- autenticação;
- integração entre containers;
- gateway;
- alterações de infraestrutura.

O ambiente de testes deverá utilizar banco e configurações próprios.

Por enquanto, nenhum novo servidor será criado.

---

# 13. O que NÃO será feito neste momento

Antes da definição completa da nova arquitetura:

- não modificar produção;
- não executar comandos destrutivos no servidor atual;
- não tentar corrigir a infraestrutura antiga;
- não mover manualmente `dist`;
- não criar novos backups manuais de `dist`;
- não alterar permissões do servidor atual;
- não recriar o banco atual;
- não destruir o servidor atual;
- não criar servidor de testes ainda.

A infraestrutura atual de produção será considerada uma referência do
estado existente, não a base obrigatória da nova arquitetura.

---

# 14. Documentação operacional futura

Será criado um `README.md` no `faro-infra` contendo procedimentos para
operações comuns.

Cenários previstos:

- subir somente o frontend;
- subir somente o backend;
- subir toda a infraestrutura;
- atualizar frontend;
- atualizar backend;
- executar migrations;
- verificar status dos containers;
- visualizar logs;
- fazer rollback;
- criar servidor novo;
- reconstruir servidor destruído;
- criar ambiente de testes;
- fazer backup do banco;
- restaurar banco;
- destruir/restaurar banco;
- restaurar arquivos persistentes;
- renovar certificados;
- recuperar o sistema após falha.

---

# 15. Decisões tomadas até agora

## Decisão 01
Frontend e backend permanecerão em repositórios separados.

### Motivo
Permitir deploy independente e manter responsabilidades separadas.

---

## Decisão 02
A infraestrutura terá um repositório próprio: `faro-infra`.

### Motivo
Separar código da aplicação da infraestrutura necessária para executá-la.

---

## Decisão 03
A aplicação será executada através de Docker.

### Motivo
Padronizar ambiente, reduzir dependências instaladas manualmente no
servidor e tornar a infraestrutura reconstruível.

---

## Decisão 04
A produção utilizará quatro componentes principais:

- frontend;
- backend;
- PostgreSQL;
- gateway.

### Motivo
Separar responsabilidades e permitir operações independentes.

---

## Decisão 05
Não utilizar GHCR, GitHub Actions ou uma plataforma externa de CI/CD
como requisito da arquitetura inicial.

### Motivo
Manter o processo simples, controlável pela Morh e com poucas
dependências externas.

---

## Decisão 06
O servidor deverá ser considerado substituível.

### Motivo
A infraestrutura não pode depender de configurações manuais
existentes em uma máquina específica.

---

## Decisão 07
A reconstrução começará pelo ambiente local e pela infraestrutura
versionada, sem modificar a produção atual.

### Motivo
Evitar repetir os problemas encontrados durante o deploy anterior e
permitir validar a nova arquitetura antes da migração.

---

# 16. Próximas etapas

As próximas etapas ainda não foram executadas.

Sequência planejada:

1. finalizar inventário do frontend;
2. finalizar inventário do backend;
3. analisar configurações atuais;
4. definir arquitetura final do `faro-infra`;
5. criar documentação inicial do `faro-infra`;
6. criar Dockerfile do frontend;
7. criar Dockerfile do backend;
8. criar configuração do PostgreSQL;
9. criar gateway;
10. criar ambiente local completo;
11. testar frontend + backend + PostgreSQL + gateway;
12. documentar operações;
13. preparar ambiente de testes;
14. preparar nova produção;
15. migrar dados;
16. validar produção;
17. realizar corte definitivo.

---

# 17. Regra desta reestruturação

Nenhuma mudança de infraestrutura deve ser feita simplesmente porque
"funciona".

Toda mudança importante deverá responder:

- O que estamos fazendo?
- Por que estamos fazendo?
- O que acontece se der errado?
- Como desfazemos?
- Como reconstruímos isso em outra máquina?
- Está documentado?
