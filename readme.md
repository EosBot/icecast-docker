# **Runbook: Configuração e Gestão do Docker Swarm com NGINX e Icecast**

## **Visão Geral**
Este runbook detalha as etapas necessárias para configurar e gerenciar um ambiente Docker Swarm, incluindo a criação de uma rede `nginx` e o deploy de uma stack com serviços de `nginx` e `icecast`, além de configurar HTTPS com certificados locais.

## **Pré-requisitos**
```bash
sudo apt-get update ; apt-get install -y apparmor-utils
hostnamectl set-hostname icecast.com
```
### Altere localhost em -> nano /etc/hosts para <127.0.0.1 icecast.com>

---

### Instale o Docker
```bash
curl -fsSL https://get.docker.com | bash
```
---

## **Etapa 1: Inicializar o Docker Swarm**

### **Descrição**
Inicialize o Docker Swarm no nó que atuará como o nó manager.

### **Comando**
```bash
docker swarm init
```

### **Resultado Esperado**
- Swarm iniciado com sucesso.
- O comando retornará um token de join que pode ser usado para adicionar nós ao cluster.

---

## **Etapa 2: Criar Certificados SSL/TLS**

### **Descrição**
Crie certificados SSL/TLS autoassinados para uso com o NGINX.

### **Comandos**
```bash
sudo mkdir -p /etc/ssl/icecast
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/icecast/private.key -out /etc/ssl/icecast/certificate.crt
```

### **Resultado Esperado**
- Certificados SSL/TLS gerados e armazenados em `/etc/ssl/icecast`.

---

## **Etapa 3: Criar a Rede NGINX**

### **Descrição**
Crie uma rede `overlay` chamada `nginx` para comunicação entre os containers dentro do Swarm.

### **Comando**
```bash
docker network create --driver=overlay nginx
```

### **Resultado Esperado**
- Rede `nginx` criada e disponível para uso pelos serviços da stack.

---

## **Etapa 4: Configurar o Arquivo `docker-compose.yml`**

### **Descrição**
Configure o arquivo `docker-compose.yml` para utilizar a rede `nginx`, além de configurar o NGINX para HTTPS.

### **Arquivo**
```yaml
version: '3.8'

services:
  icecast:
    image: libretime/icecast:2.4.4-debian
    volumes:
      - logs:/var/log/icecast2
      - /etc/localtime:/etc/localtime:ro
    environment:
      - ICECAST_PASSWORD=123@mudar
      - ICECAST_RELAY_PASSWORD=123@mudar
      - ICECAST_SOURCE_PASSWORD=123@mudar
      - ICECAST_ADMIN_PASSWORD=123@mudar
      - ICECAST_ADMIN_USERNAME=admin
      - ICECAST_ADMIN_EMAIL=teste@admin.com
      - ICECAST_LOCATION=America/Sao_Paulo
      - ICECAST_HOSTNAME=localhost
      - ICECAST_MAX_CLIENTS=100
      - ICECAST_MAX_SOURCES=100
    ports:
      - "8000:8000"
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    networks:
      - nginx

  nginx:
    image: nginx:latest
    depends_on:
      - icecast
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - /etc/ssl/icecast:/etc/ssl/icecast:ro
    ports:
      - "80:80"
      - "443:443"
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    networks:
      - nginx

volumes:
  logs:
    external: true
    name: icecast-logs

networks:
  nginx:
    external: true
    name: nginx
```

### **Resultado Esperado**
- O arquivo `docker-compose.yml` está configurado para usar a rede `nginx`, além de permitir a configuração de HTTPS.

---

## **Etapa 5: Configurar o Arquivo `nginx.conf`**

### **Descrição**
Configure o NGINX para suportar HTTPS e redirecionar o tráfego HTTP para HTTPS.

### **Arquivo**
```nginx
events {}

http {
    server {
        listen 80;
        server_name localhost; # Substitua pelo seu domínio ou IP

        # Redirecionar HTTP para HTTPS
        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name localhost; # Substitua pelo seu domínio ou IP

        ssl_certificate /etc/ssl/icecast/certificate.crt; # Caminho para o certificado
        ssl_certificate_key /etc/ssl/icecast/private.key; # Caminho para a chave privada

        location / {
            proxy_pass http://icecast:8000; # Nome do serviço e porta no Docker
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### **Resultado Esperado**
- O arquivo `nginx.conf` está configurado para suportar HTTPS e redirecionar tráfego HTTP para HTTPS.

---

## **Etapa 6: Realizar o Deploy da Stack**

### **Descrição**
Faça o deploy da stack configurada usando o Docker Swarm.

### **Comando**
```bash
docker stack deploy --prune --resolve-image always --detach=false -c <filename> <stack_name>
```

- Substitua `<filename>` pelo nome do arquivo YAML.
- Substitua `<stack_name>` pelo nome desejado para a stack.

### **Resultado Esperado**
- A stack é implantada no Swarm, e os serviços `icecast` e `nginx` são executados com segurança.

---

## **Etapa 7: Verificação**

### **Descrição**
Verifique se os serviços estão em execução e funcionando corretamente.

### **Comandos**
- Verificar o status dos serviços:
  ```bash
  docker stack services <stack_name>
  ```
- Verificar a rede:
  ```bash
  docker network ls
  ```
- Verificar os logs dos containers:
  ```bash
  docker service logs <service_name>
  ```

### **Resultado Esperado**
- Todos os serviços estão `Running (1/1)` e sem falhas.
- A rede `nginx` está presente e atribuída corretamente.

### **Instale o BUTT (Broadcast Using This Tool) em sua máquina local.**

**Configuração do BUTT:**

- Servidor: localhost
- Porta: 8000
- Senha: source_password (conforme configurado no Docker)
- Formato de áudio: Escolha o formato compatível com o Icecast (e.g., MP3, OGG)
- Ponto de montagem: /stream (ou o que você configurou)
- Transmitir um áudio de teste:
- Selecione um arquivo de áudio (.mp3 ou .wav) e comece a transmissão.

---

## **Etapa 8: Gerenciamento e Monitoramento**

### **Escalar Serviços**
Para escalar o número de réplicas de um serviço:

```bash
docker service scale <stack_name>_<service_name>=<replicas>
```

### **Atualizar Serviços**
Para atualizar a imagem de um serviço:

```bash
docker service update --image <new_image> <stack_name>_<service_name>
```
