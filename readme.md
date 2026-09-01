# Laboratório Prático — Departamento de Compras

## Construção de uma Infraestrutura de Rede com Docker

### Disciplina

Redes de Computadores / Desenvolvimento de Sistemas

### Tema

Criação de uma rede departamental utilizando containers Docker

### Departamento

**COMPRAS**

---

# 1. Objetivos do laboratório

Ao final da atividade, o aluno deverá ser capaz de:

* criar uma rede Docker;
* compreender o conceito de rede isolada;
* criar containers utilizando `docker run`;
* conectar containers à mesma rede;
* compreender a resolução de nomes entre containers;
* criar um servidor Web utilizando Nginx;
* criar um servidor FTP;
* criar um servidor de banco de dados;
* criar uma aplicação cliente para consultar o banco;
* testar comunicação entre serviços;
* identificar portas e protocolos utilizados pelos serviços;
* compreender a diferença entre um serviço interno e um serviço publicado;
* analisar quais serviços devem ou não ser disponibilizados externamente.

---

# 2. Cenário

A empresa possui um Departamento de Compras.

O departamento precisa possuir sua própria infraestrutura de rede:

```text
                     DEPARTAMENTO DE COMPRAS

                         REDE-COMPRAS
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
     compras-web         compras-ftp         compras-db
       NGINX                 FTP               MariaDB
          ▲                                       ▲
          │                                       │
          └───────────────┐               ┌───────┘
                          │               │
                          ▼               ▼
                    compras-client   compras-app
                    "Navegador"       Aplicação
```

Todos os servidores estarão inicialmente **isolados dentro da rede `rede-compras`**.

Nenhum serviço deverá ser publicado diretamente na Internet nesta primeira etapa.

---

# 3. Conceito importante

Antes de começar, observe a diferença entre:

```text
Container
```

e

```text
Rede Docker
```

Criar um container não significa automaticamente que ele poderá conversar com qualquer outro container.

Neste laboratório, vamos criar explicitamente uma rede:

```text
rede-compras
```

Os serviços que pertencerem a essa rede poderão se comunicar utilizando seus nomes.

Por exemplo:

```text
compras-web
compras-ftp
compras-db
compras-app
compras-client
```

---

# 4. Verificar o ambiente

Entre no `lab-server`.

Verifique se o Docker está funcionando:

```bash
docker version
```

Depois:

```bash
docker ps
```

Também podemos verificar as redes existentes:

```bash
docker network ls
```

---

# 5. Criar a rede do Departamento de Compras

Crie a rede:

```bash
docker network create rede-compras
```

Verifique:

```bash
docker network ls
```

Deverá aparecer uma rede semelhante a:

```text
rede-compras
```

Agora consulte suas características:

```bash
docker network inspect rede-compras
```

Neste momento a rede ainda não possui nossos servidores.

---

# 6. Criar o servidor Web

Vamos utilizar o Nginx.

Execute:

```bash
docker run -d \
  --name compras-web \
  --network rede-compras \
  nginx:alpine
```

Verifique:

```bash
docker ps
```

Deverá aparecer:

```text
compras-web
```

---

## 6.1 Testar o Nginx

Entre no próprio container:

```bash
docker exec -it compras-web sh
```

Dentro do container:

```bash
wget -qO- http://localhost
```

Deverá aparecer o HTML padrão do Nginx.

Saia:

```bash
exit
```

---

# 7. Criar uma página para o Departamento de Compras

Execute:

```bash
docker exec compras-web sh -c \
'echo "<h1>Departamento de Compras</h1><p>Servidor Web funcionando!</p>" > /usr/share/nginx/html/index.html'
```

Teste novamente:

```bash
docker exec compras-web wget -qO- http://localhost
```

Agora o servidor Web possui uma página personalizada.

---

# 8. Criar o servidor FTP

Vamos criar um servidor FTP para representar o serviço utilizado pelo departamento para transferência de arquivos.

Crie um volume:

```bash
docker volume create compras-arquivos
```

Agora crie o servidor:

```bash
docker run -d \
  --name compras-ftp \
  --network rede-compras \
  -v compras-arquivos:/home/vsftpd \
  -e FTP_USER=aluno \
  -e FTP_PASS=Laboratorio@123 \
  fauria/vsftpd
```

Verifique:

```bash
docker ps
```

E consulte os logs:

```bash
docker logs compras-ftp
```

---

# 9. Criar o servidor de banco de dados

Vamos utilizar MariaDB.

Primeiro crie um volume para preservar os dados:

```bash
docker volume create compras-db-data
```

Agora crie o banco:

```bash
docker run -d \
  --name compras-db \
  --network rede-compras \
  -v compras-db-data:/var/lib/mysql \
  -e MARIADB_ROOT_PASSWORD=Root@123 \
  -e MARIADB_DATABASE=compras \
  -e MARIADB_USER=aluno \
  -e MARIADB_PASSWORD=Aluno@123 \
  mariadb:11
```

Verifique:

```bash
docker ps
```

Aguarde alguns segundos para que o MariaDB seja inicializado.

Consulte os logs:

```bash
docker logs compras-db
```

---

# 10. Criar as tabelas do Departamento de Compras

Entre no banco:

```bash
docker exec -it compras-db mariadb \
  -u aluno \
  -pAluno@123 \
  compras
```

Agora crie a tabela de produtos:

```sql
CREATE TABLE produtos (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(100),
    preco DECIMAL(10,2),
    estoque INT
);
```

Insira alguns produtos:

```sql
INSERT INTO produtos (nome, preco, estoque)
VALUES
('Teclado', 89.90, 20),
('Mouse', 49.90, 35),
('Monitor', 899.90, 10),
('Notebook', 3500.00, 5);
```

Consulte:

```sql
SELECT * FROM produtos;
```

Observe o resultado.

Depois saia:

```sql
exit
```

---

# 11. Criar o aplicativo do Departamento de Compras

Agora criaremos um container que representará uma aplicação utilizada pelo departamento.

Execute:

```bash
docker run -it \
  --name compras-app \
  --network rede-compras \
  python:3.12-alpine \
  sh
```

Agora estamos dentro do container Python.

Instale o driver para MariaDB/MySQL:

```bash
pip install pymysql
```

---

# 12. Testar a comunicação com o banco

Ainda dentro do `compras-app`, teste:

```bash
ping compras-db
```

Observe que não estamos utilizando um endereço IP.

Estamos utilizando:

```text
compras-db
```

O Docker realiza a resolução do nome dentro da rede.

Agora teste a porta:

```bash
nc -zv compras-db 3306
```

Se aparecer uma mensagem indicando que a porta está aberta, temos comunicação TCP entre os containers.

A comunicação é:

```text
compras-app
     │
     │ TCP/3306
     ▼
compras-db
```

---

# 13. Criar o programa de consulta

Ainda dentro do `compras-app`, execute:

```bash
cat > app.py <<'PY'
import pymysql

conexao = pymysql.connect(
    host="compras-db",
    user="aluno",
    password="Aluno@123",
    database="compras"
)

cursor = conexao.cursor()

cursor.execute("SELECT * FROM produtos")

for produto in cursor.fetchall():
    print(produto)

cursor.close()
conexao.close()
PY
```

Execute:

```bash
python app.py
```

O resultado deverá apresentar os produtos cadastrados.

Exemplo:

```text
(1, 'Teclado', Decimal('89.90'), 20)
(2, 'Mouse', Decimal('49.90'), 35)
(3, 'Monitor', Decimal('899.90'), 10)
(4, 'Notebook', Decimal('3500.00'), 5)
```

Neste momento temos:

```text
compras-app
     │
     │ SQL
     ▼
compras-db
```

---

# 14. Criar o "navegador" do laboratório

Nesta aula não vamos publicar o Nginx na Internet.

Em vez disso, vamos criar um container que funcionará como um **cliente da rede**.

Saia do `compras-app`:

```bash
exit
```

Agora crie o cliente:

```bash
docker run -it \
  --name compras-client \
  --network rede-compras \
  curlimages/curl:latest \
  sh
```

Observe:

```text
compras-client
      │
      └── rede-compras
```

Esse container representa um computador dentro do Departamento de Compras.

---

# 15. Acessar o servidor Web

Dentro do `compras-client`:

```bash
curl http://compras-web
```

O resultado deverá ser:

```html
<h1>Departamento de Compras</h1>
<p>Servidor Web funcionando!</p>
```

Parabéns!

Temos comunicação HTTP entre dois containers:

```text
                 HTTP
compras-client ────────► compras-web
```

---

# 16. Observar o HTTP

Agora execute:

```bash
curl -v http://compras-web
```

Observe especialmente:

```text
Connected to compras-web
```

e:

```text
GET /
```

e:

```text
HTTP/1.1 200 OK
```

Pergunta:

**Por que conseguimos utilizar `compras-web` em vez de descobrir o endereço IP do container?**

Resposta esperada:

> Porque o Docker fornece resolução de nomes entre containers conectados à mesma rede.

---

# 17. Testar o FTP

Ainda no `compras-client`:

```bash
nc -zv compras-ftp 21
```

A porta padrão do FTP é:

```text
21
```

Portanto:

```text
compras-client
      │
      │ TCP/21
      ▼
compras-ftp
```

---

# 18. Testar o banco

Ainda no `compras-client`:

```bash
nc -zv compras-db 3306
```

A porta padrão do MariaDB é:

```text
3306
```

Temos:

```text
compras-client
      │
      │ TCP/3306
      ▼
compras-db
```

---

# 19. Testar todos os servidores pelo nome

Ainda dentro do `compras-client`, execute:

```bash
ping compras-web
```

Depois:

```bash
ping compras-ftp
```

Depois:

```bash
ping compras-db
```

Observe que os três nomes são resolvidos dentro da rede Docker.

---

# 20. Testar HTTP novamente

Execute:

```bash
curl http://compras-web
```

Depois:

```bash
curl -v http://compras-web
```

Compare os resultados.

---

# 21. Visualizar a rede

Saia do `compras-client`:

```bash
exit
```

Agora execute no `lab-server`:

```bash
docker network inspect rede-compras
```

Observe os containers conectados.

Você deverá encontrar:

```text
compras-web
compras-ftp
compras-db
compras-app
compras-client
```

Temos agora uma infraestrutura completa:

```text
                    rede-compras
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
 compras-web        compras-ftp       compras-db
       ▲                                   ▲
       │                                   │
       │ HTTP                         SQL  │
       │                                   │
       └────────── compras-client     compras-app
```

---

# 22. Identificando os protocolos

Preencha a tabela:

| Serviço   | Container      | Protocolo     | Porta |
| --------- | -------------- | ------------- | ----: |
| Web       | compras-web    | HTTP          |    80 |
| FTP       | compras-ftp    | FTP           |    21 |
| Banco     | compras-db     | MySQL/MariaDB |  3306 |
| Aplicação | compras-app    | TCP/SQL       |  3306 |
| Cliente   | compras-client | HTTP/FTP/TCP  |     — |

---

# 23. Conceito importante: portas internas

Até este momento, não utilizamos:

```bash
-p
```

Por exemplo, não fizemos:

```bash
docker run -p 8080:80 ...
```

Mesmo assim, o Nginx funciona.

Por quê?

Porque os containers estão na mesma rede Docker.

```text
rede-compras

compras-client
       │
       │ HTTP
       ▼
compras-web:80
```

A porta `80` está disponível **dentro da rede**.

Ela não precisa necessariamente estar publicada no servidor físico.

---

# 24. Desafio 1 — Descobrir o endereço IP

Descubra o IP do Nginx:

```bash
docker inspect compras-web
```

Localize:

```text
IPAddress
```

Depois compare com:

```bash
docker exec compras-client ping compras-web
```

Pergunta:

> O que acontece se o endereço IP de `compras-web` mudar?

---

# 25. Desafio 2 — Utilizar o nome em vez do IP

Teste:

```bash
curl http://compras-web
```

Agora tente descobrir o endereço IP:

```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' compras-web
```

Depois:

```bash
curl http://IP_DO_CONTAINER
```

Compare os dois métodos.

Pergunta:

> Qual abordagem é mais adequada para uma aplicação: utilizar o IP ou o nome do serviço?

---

# 26. Desafio 3 — Descobrir quem está conectado à rede

Execute:

```bash
docker network inspect rede-compras
```

Identifique:

* IP do `compras-web`;
* IP do `compras-ftp`;
* IP do `compras-db`;
* IP do `compras-app`;
* IP do `compras-client`.

Monte uma tabela:

| Container      | IP |
| -------------- | -- |
| compras-web    |    |
| compras-ftp    |    |
| compras-db     |    |
| compras-app    |    |
| compras-client |    |

---

# 27. Desafio 4 — Testar isolamento

Crie um novo container fora da rede:

```bash
docker run --rm \
  curlimages/curl:latest \
  curl http://compras-web
```

O que acontece?

Agora execute:

```bash
docker run --rm \
  --network rede-compras \
  curlimages/curl:latest \
  curl http://compras-web
```

Compare os resultados.

### Pergunta

Por que o segundo comando consegue acessar o Nginx e o primeiro não?

---

# 28. Desafio 5 — O que realmente está protegido?

Analise a situação:

```text
                 INTERNET
                    X
                    │
                    │
             ┌──────┴──────┐
             │ rede-compras│
             └──────┬──────┘
                    │
       ┌────────────┼────────────┐
       │            │            │
       ▼            ▼            ▼
      WEB          FTP           DB
```

Neste momento:

* o Nginx não está publicado;
* o FTP não está publicado;
* o banco não está publicado;
* a aplicação não está publicada;
* os serviços conseguem conversar internamente.

---

# 29. Desafio final — O problema da empresa

Agora imagine que a diretoria da empresa faça a seguinte solicitação:

> "Precisamos disponibilizar o site do Departamento de Compras para usuários externos."

O endereço desejado seria:

```text
https://compras.inetz.com.br
```

Mas existe uma regra:

> **O banco de dados NÃO pode ficar disponível na Internet.**

E outra:

> **O servidor FTP também NÃO deve ficar disponível publicamente.**

A arquitetura atual é:

```text
                    INTERNET
                       │
                       X
                       │
                 rede-compras
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
         WEB          FTP           DB
```

## Perguntas para discussão

### 1.

Como poderíamos permitir que usuários externos acessem somente:

```text
compras-web
```

sem expor:

```text
compras-db
```

e:

```text
compras-ftp
```

?

### 2.

Precisamos publicar todas as portas dos containers?

Explique.

### 3.

Seria seguro executar:

```bash
-p 80:80
```

para todos os serviços?

Por quê?

### 4.

Qual serviço deveria funcionar como ponto de entrada da rede?

### 5.

Como poderíamos fazer:

```text
Internet
   │
   ▼
HTTPS
   │
   ▼
Gateway / Proxy
   │
   ▼
compras-web
```

sem permitir:

```text
Internet ─────► compras-db
```

?

### 6.

Onde o Traefik poderia entrar nessa arquitetura?

---

# 30. Desafio de arquitetura

Desenhe uma nova arquitetura que permita:

```text
                 INTERNET
                     │
                     ▼
              compras.inetz.com.br
                     │
                     ▼
                  TRAEFIK
                     │
                     ▼
                compras-web
                     │
               rede-compras
          ┌──────────┼──────────┐
          │          │          │
          ▼          ▼          ▼
         FTP         DB        APP
```

Mas respeitando:

```text
Internet → WEB       PERMITIDO
Internet → FTP       BLOQUEADO
Internet → DB        BLOQUEADO
Internet → APP       BLOQUEADO
```

---

# 31. Reflexão final

Responda:

1. O que é uma rede Docker?
2. Por que os containers conseguem utilizar seus nomes para se comunicar?
3. Qual a função do DNS interno do Docker?
4. Qual a diferença entre uma porta interna e uma porta publicada?
5. Qual a função do parâmetro `-p`?
6. Por que não devemos publicar o banco de dados sem necessidade?
7. Por que não devemos publicar todos os serviços?
8. Qual seria a função de um proxy reverso?
9. Onde o Traefik poderia ser utilizado?
10. Como podemos disponibilizar apenas o servidor Web para a Internet?
11. Como podemos manter FTP e banco restritos à rede interna?
12. O que significa aplicar o princípio do menor privilégio nesse cenário?

---

# Resultado esperado

Ao terminar esta atividade, o aluno terá construído manualmente uma pequena infraestrutura empresarial:

```text
                    DEPARTAMENTO DE COMPRAS

                       rede-compras
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
          ▼                 ▼                 ▼
     compras-web       compras-ftp       compras-db
        Nginx              FTP              MariaDB
          ▲                                     ▲
          │                                     │
          │ HTTP                           SQL  │
          │                                     │
          └──────────────┐             ┌────────┘
                         ▼             ▼
                   compras-client  compras-app
```

O aluno terá utilizado:

```text
docker network
docker run
docker ps
docker exec
docker inspect
docker network inspect
docker volume
ping
curl
nc
Python
SQL
HTTP
FTP
```

A próxima etapa do laboratório será transformar essa infraestrutura interna em uma infraestrutura com **acesso externo controlado**, utilizando um **proxy reverso/Traefik**, mantendo os serviços internos protegidos.
