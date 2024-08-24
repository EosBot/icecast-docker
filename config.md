# **Documentação: Adicionando um Ponto de Montagem e um Relay no Icecast**

## **Visão Geral**

Esta documentação descreve os passos necessários para adicionar um ponto de montagem e um relay na configuração do Icecast, e como carregar um arquivo XML personalizado para o Icecast usando Docker Compose. As alterações devem ser feitas diretamente no arquivo de configuração `icecast.xml` dentro do container ou carregando um arquivo personalizado.

## **Pré-requisitos**

- Acesso ao container do Icecast.
- Permissões para editar o arquivo `/etc/icecast.xml` dentro do container.
- Conhecimento básico sobre Docker Compose.

## **Método 1: Editar o Arquivo XML Dentro do Container**

### **Passos para Adicionar um Ponto de Montagem**

1. **Acesse o Container do Icecast**

   Primeiro, você precisa acessar o shell do container do Icecast:

   ```bash
   docker exec -it <nome_do_container> /bin/bash
   ```

2. **Abra o Arquivo de Configuração**

   Navegue até o diretório onde o arquivo de configuração `icecast.xml` está localizado e abra-o com um editor de texto:

   ```bash
   cd /etc
   nano icecast.xml
   ```

3. **Adicione um Novo Ponto de Montagem**

   Encontre a seção `<mount>` no arquivo `icecast.xml`. Se você não encontrar uma seção existente, você pode adicioná-la abaixo das outras seções `<mount>`.

   Adicione a seguinte configuração para criar um novo ponto de montagem `/stream`:

   ```xml
   <mount>
       <mount-name>/stream</mount-name>
       <public-name>Stream</public-name>
       <description>Exemplo de ponto de montagem</description>
       <genre>Rock</genre>
       <url>http://example.com</url>
       <public>1</public>
       <bitrate>128</bitrate>
       <format>mp3</format>
       <sample-rate>44100</sample-rate>
       <max-listeners>100</max-listeners>
       <admin-name>Admin</admin-name>
       <admin-email>admin@example.com</admin-email>
       <relay>0</relay>
       <fallback-mount>/fallback</fallback-mount>
       <fallback-when-no-source>1</fallback-when-no-source>
       <fallback-archive>1</fallback-archive>
   </mount>
   ```

   Ajuste os valores conforme necessário para seu ambiente.

4. **Salve e Feche o Arquivo**

   Salve as alterações e feche o editor. No `nano`, você pode fazer isso pressionando `Ctrl+O` para salvar e `Ctrl+X` para sair.

### **Passos para Adicionar um Relay**

1. **Abra o Arquivo de Configuração**

   Já estando dentro do container e com o arquivo `icecast.xml` aberto, localize a seção onde os relays são definidos. Se não houver uma seção para relays, adicione uma nova.

2. **Adicione a Configuração do Relay**

   Adicione a seguinte configuração na seção apropriada do arquivo `icecast.xml` para definir um relay:

   ```xml
   <relay>
       <relay-name>Example Relay</relay-name>
       <mount>/stream</mount>
       <hostname>relay.example.com</hostname>
       <port>8000</port>
       <password>123@mudar</password>
       <public>1</public>
       <description>Relay de exemplo</description>
   </relay>
   ```

   Ajuste os valores conforme necessário:

   - `<relay-name>`: Nome do relay.
   - `<mount>`: O ponto de montagem do relay.
   - `<hostname>`: O hostname do servidor relay.
   - `<port>`: Porta do servidor relay.
   - `<password>`: Senha de autenticação para o relay.
   - `<public>`: Se o relay deve ser público (1) ou privado (0).
   - `<description>`: Descrição do relay.

3. **Salve e Feche o Arquivo**

   Salve as alterações e feche o editor.

### **Reiniciar o Serviço Icecast**

Após realizar as alterações, reinicie o serviço Icecast para aplicar as novas configurações:

```bash
docker restart <nome_do_container>
```

### **Verificação**

- Verifique se o ponto de montagem e o relay foram configurados corretamente acessando a interface de administração do Icecast ou revisando os logs do serviço.
- Teste o ponto de montagem `/stream` e o relay para garantir que estão funcionando conforme esperado.

---

## **Método 2: Carregar um Arquivo XML Personalizado com Docker Compose**

### **Passos para Usar um Arquivo XML Personalizado**

1. **Prepare o Arquivo XML**

   Crie ou edite o arquivo `icecast.xml` em seu diretório de trabalho local com as configurações desejadas para o ponto de montagem e o relay.

2. **Atualize o Arquivo `docker-compose.yml`**

   No seu arquivo `docker-compose.yml`, adicione um volume para mapear seu arquivo `icecast.xml` local para o local de configuração do Icecast dentro do container. Aqui está um exemplo de configuração:

   ```yaml
   version: '3.8'

   services:
     icecast:
       image: libretime/icecast:2.4.4-debian
       volumes:
         - ./icecast.xml:/etc/icecast.xml
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

   volumes:
     logs:
       external: true
       name: icecast-logs

   networks:
     nginx:
       external: true
       name: nginx
   ```

3. **Faça o Deploy da Stack**

   Com o arquivo `docker-compose.yml` atualizado, execute o deploy da stack:

   ```bash
   docker stack deploy --prune --resolve-image always --detach=false -c docker-compose.yml <stack_name>
   ```

   Substitua `<stack_name>` pelo nome desejado para a stack.

### **Verificação**

- Verifique se o Icecast está utilizando o arquivo `icecast.xml` personalizado acessando a interface de administração do Icecast ou revisando os logs do serviço.
- Teste o ponto de montagem `/stream` e o relay para garantir que estão funcionando conforme esperado.
