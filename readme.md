Descobrindo o Endereço IP
Execute:

getent hosts lab-server
Anote o endereço IP retornado.

Exemplo:

10.x.x.x    lab-server
Depois execute:

ping lab-server
Observe que o nome:

lab-server
é convertido para um endereço IP.

Conceito
Isso demonstra o funcionamento da resolução de nomes dentro da rede Docker.

7. Acesso SSH
Agora vamos acessar o servidor.

Execute:

ssh aluno@lab-server
Na primeira conexão será apresentada uma mensagem semelhante a:

The authenticity of host 'lab-server' can't be established.
Digite:

yes
Quando solicitado, informe a senha fornecida pelo professor.

Após o login, deverá aparecer:

aluno@lab-server:~$
8. Identificando o Servidor
Execute:

hostname
Resultado esperado:

lab-server
Agora:

whoami
Resultado:

# Roteiro de Atividades Práticas: Infraestrutura do Setor de Compras
## Módulo: Provisionamento de Microsserviços e Sub-redes Didáticas no `lab-server`

---

### 📋 Topologia da Sub-rede de Compras (Sub-rede 8)

| Parâmetro | Valor |
| :--- | :--- |
| **Nome da Rede Docker** | `rede_compras` |
| **Sub-rede (CIDR)** | `192.168.1.224/27` |
| **Máscara de Sub-rede** | `255.255.255.224` |
| **Gateway da Rede** | `192.168.1.225` |

#### 🛸 Mapeamento de Serviços do Departamento de Compras

| Serviço | Container Name | Imagem Docker | IP Fixo na Sub-rede | Porta | Função Didática |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Servidor Web** | `compras-web` | `nginx:alpine` | `192.168.1.230` | `80` | Frontend do Portal de Compras |
| **Banco de Dados** | `compras-db` | `postgres:15-alpine` | `192.168.1.231` | `5432` | Base de Fornecedores e Cotações |
| **API de Cotações** | `compras-api` | `python:3.11-slim` | `192.168.1.232` | `8000` | Processamento de Pedidos |
| **Servidor de Cache** | `compras-cache` | `redis:alpine` | `192.168.1.233` | `6379` | Cache de Sessão de Cotações |

---

## 🎯 Objetivos da Atividade
1. Conectar via SSH ao servidor `lab-server`.
2. Criar a rede isolada do setor de compras (`rede_compras` - `192.168.1.224/27`).
3. Subir e provisionar os 4 serviços da infraestrutura de Compras com endereçamento IP estático.
4. Testar a conectividade de rede (ping e portas) entre os microsserviços.
5. Criar o arquivo `docker-compose.yml` local do Setor de Compras para automação da infraestrutura.

---

## 🚀 Passo a Passo das Atividades

### **Etapa 1: Acesso ao Servidor `lab-server`**
Conecte-se ao servidor central:
```bash
ssh aluno@lab-server
```
*(Senha: `Laboratorio@123`)*

---

### **Etapa 2: Criação da Sub-rede do Setor de Compras**
Crie a rede virtual no Docker para isolar os serviços de compras:
```bash
docker network create \
  --driver bridge \
  --subnet 192.168.1.224/27 \
  --gateway 192.168.1.225 \
  rede_compras
```

---

### **Etapa 3: Subindo os Serviços do Departamento de Compras**

#### 1. Servidor de Banco de Dados (`compras-db`)
```bash
docker run -d \
  --name compras-db \
  --network rede_compras \
  --ip 192.168.1.231 \
  -e POSTGRES_DB=compras_db \
  -e POSTGRES_USER=gestor_compras \
  -e POSTGRES_PASSWORD=SenhaDBCompras123 \
  postgres:15-alpine
```

#### 2. Servidor de Cache de Cotações (`compras-cache`)
```bash
docker run -d \
  --name compras-cache \
  --network rede_compras \
  --ip 192.168.1.233 \
  redis:alpine
```

#### 3. Servidor Web / Portal de Compras (`compras-web`)
```bash
docker run -d \
  --name compras-web \
  --network rede_compras \
  --ip 192.168.1.230 \
  -p 8080:80 \
  nginx:alpine
```

#### 4. API de Cotações (`compras-api`)
```bash
docker run -d \
  --name compras-api \
  --network rede_compras \
  --ip 192.168.1.232 \
  python:3.11-slim python3 -m http.server 8000
```

---

### **Etapa 4: Validação e Testes de Conectividade**

1. **Listar os containers do setor de compras rodando:**
   ```bash
   docker ps
   ```

2. **Verificar os IPs atribuídos na rede `rede_compras`:**
   ```bash
   docker network inspect rede_compras
   ```

3. **Testar comunicação entre a API e o Banco de Dados:**
   ```bash
   docker exec -it compras-api ping -c 3 192.168.1.231
   ```

4. **Testar porta do Banco de Dados PostgreSQL (5432):**
   ```bash
   docker exec -it compras-web nc -zv 192.168.1.231 5432
   ```

5. **Testar porta do Redis Cache (6379):**
   ```bash
   docker exec -it compras-web nc -zv 192.168.1.233 6379
   ```

---

### **Etapa 5: Automatizando a Infraestrutura com Docker Compose**
Para consolidar o aprendizado, crie a pasta da aplicação e monte o arquivo `docker-compose.yml` da arquitetura de Compras:

1. Crie o diretório do projeto:
   ```bash
   mkdir -p ~/infra-compras && cd ~/infra-compras
   ```

2. Crie o arquivo `docker-compose.yml`:
   ```yaml
   version: '3.8'

   services:
     compras-web:
       image: nginx:alpine
       container_name: compras-web
       networks:
         rede_compras:
           ipv4_address: 192.168.1.230
       ports:
         - "8080:80"

     compras-db:
       image: postgres:15-alpine
       container_name: compras-db
       environment:
         POSTGRES_DB: compras_db
         POSTGRES_USER: gestor
         POSTGRES_PASSWORD: SenhaDBCompras123
       networks:
         rede_compras:
           ipv4_address: 192.168.1.231

     compras-cache:
       image: redis:alpine
       container_name: compras-cache
       networks:
         rede_compras:
           ipv4_address: 192.168.1.233

   networks:
     rede_compras:
       driver: bridge
       ipam:
         config:
           - subnet: 192.168.1.224/27
             gateway: 192.168.1.225
   ```

3. Suba toda a infraestrutura com um único comando:
   ```bash
   docker-compose up -d
   ```

---

### ✅ Conclusão
Você construiu a arquitetura completa do **Setor de Compras**, contendo Servidor Web, Banco de Dados PostgreSQL, API e Cache Redis, todos devidamente alocados e isolados na sub-rede `192.168.1.224/27`.


aluno
