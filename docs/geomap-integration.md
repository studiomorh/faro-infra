# Habilitação da integração visual GeoMap

Esta extensão não modifica o Compose-base, APP_ENV, banco, volumes ou gateways. Adiciona somente as variáveis do consumidor GeoMap ao serviço backend. O código consumidor precisa estar na imagem backend antes da ativação. Não há cálculo de rotas ou importador GPX produtivo nesta entrega.

No .env **do servidor**, configurar a origem HTTPS, origens permitidas de assets, credencial de serviço e flag inicialmente falsa. A credencial não entra no Git. A lista exata de nomes está em `deploy/geomap.override.yml`.

Usar explicitamente os dois arquivos em comandos de configuração/atualização:

```
docker compose -f docker-compose.yml -f deploy/geomap.override.yml config --quiet
docker compose -f docker-compose.yml -f deploy/geomap.override.yml up -d --no-deps backend
```

O segundo comando recria o backend para receber as variáveis; executar somente depois de revisar a versão de imagem e o pacote aprovado. Se precisar construir código, fazer o build no fluxo normal e registrar SHA/imagem anterior. Não copiar dist manualmente. Não executar build ou up só para testar a configuração em produção.

Antes de ligar GEOMAP_PUBLISHED_BASE_ENABLED, conferir no GeoMap o workspace FARO, grant restrito de serviço, publicação Operacional, descritor e todos os assets versionados (Range/CORS/ícones/glyphs). Não usar origem localhost no servidor nem mudar APP_ENV para habilitar exceções locais.

A configuração efetivamente instalada pode conter ajustes que o Git do servidor não registra. Conferir estado Git e imagem executada em leitura; não sobrescrever o Compose ou Dockerfile instalado com a cópia do notebook. Manter este overlay nos próximos comandos de operação, pois o Compose padrão não o inclui automaticamente.

Para recuperar, guardar imagens e configuração anteriores; restaurar esses valores e recriar apenas serviços necessários. Flag falsa usa a base legada; bundle anterior usa a integração e exige preservar seus assets. Não limpar/restaurar o banco por um problema visual. Conferir `/up`, autenticação e mapa após qualquer troca.
