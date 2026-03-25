# GUIA PASSO A PASSO - PROVA DE PROJETO DE SOFTWARE

## Visão Geral do que precisa ser feito

Criar uma API REST em Python (Flask) que se conecta ao MongoDB, com deploy automático via Docker + GitHub Actions na AWS.

---

## PARTE 1 — ESTRUTURA DO PROJETO (Arquivos que você precisa criar)

```
meu-projeto/
├── main.py                        ← Código da API (rotas)
├── requirements.txt               ← Dependências Python
├── Dockerfile                     ← Para criar imagem Docker
├── .gitignore                     ← Arquivos para o Git ignorar
└── .github/
    └── workflows/
        └── deploy.yml             ← Pipeline CI/CD do GitHub Actions
```

---

## PARTE 2 — CRIAÇÃO DA API (main.py)

### Passo 1: Imports e configuração
- Importar Flask, request, jsonify do flask
- Importar MongoClient do pymongo
- Importar datetime, uuid, requests, os
- Pegar variáveis de ambiente com `os.environ.get()` para MONGO_URL e USER_API_URL
- Criar o app Flask: `app = Flask(__name__)`
- Conectar ao MongoDB: `client = MongoClient(mongo_url)`
- Acessar banco e coleção: `db = client['posts_db']` e `posts_collection = db['posts']`

### Passo 2: Rota GET /post (listar posts)
- Decorador: `@app.route("/post", methods=["GET"])`
- Buscar todos os posts: `posts_collection.find({}, {"_id": 0})`
  - O `{"_id": 0}` remove o campo _id do MongoDB da resposta
- Converter para lista e retornar com `jsonify(posts), 200`

### Passo 3: Rota POST /post (criar post)
- Decorador: `@app.route("/post", methods=["POST"])`
- Pegar o ID do usuário do HEADER: `request.headers.get("usuario")`
- Se não veio o header, retornar erro 400
- **VALIDAR O USUÁRIO**: fazer um GET na API de usuários:
  - `requests.get(f"{user_api_url}/users/{user_id}")`
  - Se o status_code NÃO for 200/201, retornar erro 404
  - Colocar dentro de try/except para tratar erros de conexão
- Pegar dados do body: `data = request.json`
- Montar o dicionário do post com:
  - `id`: gerar com `str(uuid.uuid4())`
  - `titulo`: vem do body (`data["titulo"]`)
  - `mensagem`: vem do body (`data["mensagem"]`)
  - `data`: preencher no servidor com `datetime.now().isoformat()`
  - `usuario`: vem do header
- Inserir no MongoDB: `posts_collection.insert_one(post)`
- Remover o `_id` que o MongoDB adiciona: `post.pop("_id", None)`
- Retornar o post com `jsonify(post), 201`

### Passo 4: Bloco main
- `if __name__ == "__main__": app.run(debug=True, port=5002)`

---

## PARTE 3 — DEPENDÊNCIAS (requirements.txt)

Criar arquivo com:
```
Flask==2.3.3
pymongo==4.6.1
requests==2.31.0
```

- **Flask** → Framework web
- **pymongo** → Driver do MongoDB para Python
- **requests** → Para fazer chamadas HTTP (validar usuário)

---

## PARTE 4 — DOCKERFILE

Passo a passo do Dockerfile:

1. `FROM python:3.11-slim` → Imagem base do Python
2. `WORKDIR /app` → Define diretório de trabalho
3. `COPY requirements.txt .` → Copia o arquivo de dependências
4. `RUN pip install --no-cache-dir -r requirements.txt` → Instala dependências
5. `COPY . .` → Copia todo o código para dentro do container
6. `EXPOSE 5000` → Expõe a porta 5000
7. `ENV FLASK_APP=main.py` → Diz qual é o arquivo principal
8. `ENV FLASK_RUN_HOST=0.0.0.0` → Aceita conexões de qualquer IP
9. `CMD ["flask", "run"]` → Comando para executar a aplicação

---

## PARTE 5 — GITHUB ACTIONS (deploy.yml)

### Localização do arquivo
- DEVE ficar em `.github/workflows/deploy.yml` (exatamente esse caminho)

### Estrutura do workflow

1. **Trigger**: executa no push para branch `main`
   ```yaml
   on:
     push:
       branches: ["main"]
   ```

2. **Checkout do código**: `actions/checkout@v4`

3. **Setup Python**: `actions/setup-python@v3` com python-version "3.10"

4. **Instalar dependências**: `pip install -r requirements.txt`

5. **Login no Docker Hub**: `docker/login-action@v3`
   - username: `${{ secrets.DOCKER_USER }}`
   - password: `${{ secrets.DOCKERHUB_TOKEN }}`

6. **Setup Docker Buildx**: `docker/setup-buildx-action@v3`

7. **Build e Push da imagem**: `docker/build-push-action@v5`
   - context: `.`
   - file: `./Dockerfile`
   - push: `true`
   - tags: `${{ secrets.DOCKER_USER }}/posts-api:${{ github.sha }}`

8. **Deploy na AWS via SSH**: `appleboy/ssh-action@master`
   - host: `${{ secrets.HOST_TEST }}`
   - username: `"ubuntu"`
   - key: `${{ secrets.KEY_TEST }}`
   - Script que roda na AWS:
     - Para e remove container antigo (`docker stop` + `docker rm`)
     - Baixa a imagem nova (`docker pull`)
     - Roda o container novo (`docker run`) com variáveis de ambiente

---

## PARTE 6 — SECRETS DO GITHUB (Configurar no repositório)

Ir em: **Settings → Secrets and variables → Actions → New repository secret**

| Secret Name      | Valor                                            | Como descobrir                         |
|------------------|--------------------------------------------------|----------------------------------------|
| DOCKER_USER      | seu usuário do Docker Hub                        | É o teu login do Docker Hub            |
| DOCKERHUB_TOKEN  | token de acesso do Docker Hub                    | Docker Hub → Account Settings → Security → New Access Token |
| HOST_TEST        | IP da AWS (ex: 18.230.76.235)                    | Professor fornece                      |
| KEY_TEST         | chave SSH privada (.pem) para acessar a AWS      | Professor fornece o arquivo .pem, colar o conteúdo inteiro |
| MONGO_URL        | mongodb://NOME-CONTAINER-MONGO:27017             | Entrar na AWS, rodar `docker ps`, olhar coluna NAMES do container mongo |
| USER_API_URL     | http://NOME-CONTAINER-USERS:PORTA-INTERNA        | Entrar na AWS, rodar `docker ps`, olhar coluna NAMES do container da API de users |

**IMPORTANTE:** Os valores de MONGO_URL e USER_API_URL dependem dos NOMES dos containers que **já estão rodando na AWS dos outros projetos**. A tua API de posts vai usar o MongoDB e a API de usuários que já existem lá. Você PRECISA entrar na AWS com SSH e rodar `docker ps` para ver os nomes certos.

Na prova, entra na AWS, roda `docker ps`, e usa os nomes que aparecerem lá.

Se colocar o nome errado, a API vai dar erro 500. Para corrigir: atualizar o secret no GitHub e re-rodar o pipeline (Actions → clicar no run → Re-run all jobs).

---

## PARTE 7 — ENTRAR NA AWS E VERIFICAR TUDO

### Como entrar na AWS
```bash
ssh -i "caminho/da/chave.pem" ubuntu@IP_DA_AWS
```
No Windows, se der erro de permissão no .pem: botão direito no arquivo → Propriedades → Segurança → Avançado → Desabilitar herança → remover todos exceto o seu usuário.

### PRIMEIRO PASSO ao entrar: rodar `docker ps`
Isso mostra todos os containers rodando. Você precisa anotar:
1. O NOME do container do MongoDB → vai no secret MONGO_URL
2. O NOME do container da API de usuários → vai no secret USER_API_URL

### Comandos essenciais dentro da AWS

**Ver todos os containers rodando (MAIS IMPORTANTE):**
```bash
docker ps
```
Mostra NAMES (nome dos containers), PORTS (portas) e IMAGE (imagem). Usar os NAMES para configurar os secrets MONGO_URL e USER_API_URL.

**Ver containers parados também:**
```bash
docker ps -a
```

**Ver logs de um container (para debugar erros):**
```bash
docker logs posts-api
docker logs posts-api --tail 50
```

**Ver quais containers estão numa rede:**
```bash
docker network inspect rede
```

**Ver redes Docker que existem:**
```bash
docker network ls
```

**Criar a rede Docker (se não existe):**
```bash
docker network create -d bridge rede
```

**Subir o MongoDB (se não existe):**
```bash
docker run -d --network=rede --name mongo-connections -p 27017:27017 mongo:7
```

**Parar e remover um container:**
```bash
docker stop nome-do-container
docker rm nome-do-container
```

**Rodar um container manualmente (para testar sem pipeline):**
```bash
docker run -d -p 8081:5000 -e MONGO_URL=mongodb://NOME-MONGO:27017 -e USER_API_URL=http://NOME-USERS:5000 --network=rede --name posts-api IMAGEM
```

### Como descobrir os valores certos para os Secrets

1. Entrar na AWS com SSH
2. Rodar `docker ps`
3. Olhar a coluna NAMES:
   - O container do MongoDB → usar o nome no MONGO_URL: `mongodb://NOME:27017`
   - O container da API de usuários → usar o nome no USER_API_URL: `http://NOME:5000`
4. Esses nomes funcionam como DNS dentro da rede Docker

### TROUBLESHOOTING - Se der erro 500

1. Entrar na AWS com SSH
2. Rodar `docker logs posts-api` para ver o erro
3. Se o erro for **"name resolution"** ou **"ServerSelectionTimeoutError"**: o MONGO_URL está com nome errado. Corrigir o secret no GitHub.
4. Se o erro for **"Connection refused"** no requests: o USER_API_URL está com nome ou porta errada. Corrigir o secret.
5. Depois de corrigir o secret: ir em Actions no GitHub → clicar no último run → **Re-run all jobs**
6. Esperar o pipeline rodar e testar de novo com curl

---

## PARTE 8 — DESCOBRIR PORTAS LIVRES

### Pelo terminal do Windows (SEM entrar na AWS)
Testar portas com curl. O tipo de resposta diz tudo:
- **Timeout** (demora e falha) = porta BLOQUEADA no Security Group
- **Connection refused** (falha rápido) = porta ABERTA e LIVRE → essa serve!
- **Responde HTML/JSON** = porta aberta mas JÁ OCUPADA

```bash
curl.exe -m 3 http://IP:PORTA
```

Testar as portas comuns: 80, 8080, 8081, 8082, 3000, 5000, 5001, 5002

### Pela AWS (se tiver SSH)
```bash
docker ps
```
Mostra quais portas já estão mapeadas. Escolher uma que não aparece.

### Se nenhuma porta livre estiver aberta no Security Group
Abrir no console da AWS:
1. EC2 → Instances → selecionar instância
2. Aba Security → clicar no Security Group
3. Edit inbound rules → Add rule
4. Type: Custom TCP → Port: a porta que quer → Source: 0.0.0.0/0 → Save

---

## PARTE 9 — TESTAR A API

### Criar um post (com curl ou Postman):
```bash
curl.exe -X POST http://IP:PORTA/post -H "Content-Type: application/json" -H "usuario: ID-DO-USUARIO-AQUI" -d "{\"titulo\": \"Meu post\", \"mensagem\": \"Ola mundo\"}"
```

### Listar posts:
```bash
curl.exe http://IP:PORTA/post
```

### Testar com usuário inválido (deve retornar erro 404):
```bash
curl.exe -X POST http://IP:PORTA/post -H "Content-Type: application/json" -H "usuario: id-falso" -d "{\"titulo\": \"Teste\", \"mensagem\": \"Nao deveria funcionar\"}"
```

**NOTA Windows:** usar `curl.exe` (não `curl` sozinho) e aspas duplas com `\"` para JSON

---

## PARTE 10 — FLUXO COMPLETO DE DEPLOY (Resumão)

1. Escrever o código (main.py, requirements.txt, Dockerfile)
2. Criar o workflow em `.github/workflows/deploy.yml`
3. Configurar os Secrets no GitHub
4. Na AWS, garantir que a rede Docker e o MongoDB existem
5. Fazer `git add .` → `git commit` → `git push`
6. O GitHub Actions automaticamente:
   - Faz build da imagem Docker
   - Envia para o Docker Hub
   - Conecta via SSH na AWS
   - Para o container antigo
   - Sobe o container novo

---

## DICAS RÁPIDAS PARA A PROVA

- **Porta da API de posts**: escolher uma livre (testar com curl), interna no container sempre 5000
- **MongoDB não precisa criar tabela**: ele cria automaticamente quando você insere
- **Header do usuário**: o campo `usuario` vai no HEADER HTTP, NÃO no body
- **Validação do usuário**: é um GET simples na API de users passando o ID
- **Data do post**: preencher no SERVIDOR com `datetime.now()`, não vem do cliente
- **UUID**: usar `uuid.uuid4()` para gerar ID único
- **Variáveis de ambiente**: usar `os.environ.get('NOME', 'valor_padrao')`
- **Docker network**: todos os containers precisam estar na mesma rede (`rede`)
- **Nomes DNS no Docker**: dentro da rede Docker, containers se comunicam pelo NOME do container (não por localhost)
- **Se o curl dá timeout**: a porta NÃO está liberada no Security Group da AWS. Ir em EC2 → Instances → Security → Security Group → Edit inbound rules → Add rule → Custom TCP, porta desejada, Source 0.0.0.0/0
- **Portas comuns já abertas na AWS**: 22 (SSH), 80 (HTTP), 8080. Usar porta 80 se possível para evitar ter que abrir no Security Group
- **Porta 80 é padrão HTTP**: se mapear `-p 80:5000`, acessa sem colocar porta na URL (ex: `http://IP/post`)

---

## MAPA MENTAL - Como as peças se conectam

```
[Usuário/Postman]
       |
       | HTTP Request (Header: usuario)
       v
[API de Posts (Flask/Python)] ──── porta livre (ex: 8081)
       |                |
       |                | GET /users/{id} (valida usuário)
       |                v
       |         [API de Usuários] ──── porta 5000/80
       |
       | Salva/Busca posts
       v
[MongoDB] ──── porta 27017
```

Todos os containers conectados na rede Docker "rede" e se comunicam pelos nomes dos containers.
