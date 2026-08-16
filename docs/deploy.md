1. Trabalha normalmente em ~/faro
cd ~/faro

Testa localmente:

npm run dev

Quando estiver tudo funcionando:

git status
git add .
git commit -m "descrição da alteração"
git push origin main
2. Gerar o build de produção LOCALMENTE

Depois do push:

npm run build

Isso gera:

~/faro/dist/

⚠️ Não subimos src/ para o Nginx.

O que o servidor serve é o conteúdo compilado de:

/opt/faro/frontend/dist
3. Compactar o dist

Na máquina local:

cd ~/faro
tar -czf /tmp/faro-dist.tar.gz dist
4. Mandar o build para o servidor
scp /tmp/faro-dist.tar.gz morh@206.189.230.209:/tmp/
🖥️ No servidor

Entramos:

ssh morh@206.189.230.209

E:

cd /opt/faro/frontend
5. Fazer backup do build atual

Eu manteria essa prática porque é ótima para produção:

sudo mv dist dist.backup-AAAA-MM-DD

Por exemplo:

sudo mv dist dist.backup-2026-08-10
6. Extrair o novo build
tar -xzf /tmp/faro-dist.tar.gz -C /opt/faro/frontend

E conferir:

ls -lah /opt/faro/frontend/dist
7. Reiniciar o Nginx
cd /opt/faro
docker compose restart nginx

Conferir:

docker compose ps

Esperamos:

faro-nginx       Up
faro-backend     Up
faro-postgres    Up
