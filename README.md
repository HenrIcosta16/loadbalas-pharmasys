# nesse guia eu irei mostrar o passo a passo de como se utilizar o balanceador de carga loadbalas para uma api frontend utilizando NGINX com 5 containers de nos no Docker.


  # 📋 Pré-requisitos

   Antes de começar, verifique se você possui:

   1.NGINX instalado para gerenciar o balanceamento de carga

   2.Docker ou Docker Compose instalado

     As configurações variam entre eles, mas a escolha é sua

   3.Seu projeto frontend pronto para deploy



  # 🚀 Passo 1: Build da Aplicação para o Nginx

  Opção 1 - Se já tiver as dependências instaladas

         npm run build

  Opção 2 - Para instalar dependências e fazer build

         npm i && npm run build

  # 📂 Passo 2: Preparação dos Arquivos 

   1.de um ls no caminho da pasta pra ver se foi buildado. Copie a pasta 'dist' para o diretório do load balancer, a pasta ja é criada automaticamente como padrao com o nome dist, basta renomear o nome dela para build e todos os caminhos onde ela se encontra, veja como fazer isso na etapa 2 e 3 desse desse passo.

      ls /home/henrique/Downloads/PharmaSys-Front-main

   2.apos o 1 comando do 2 passo,se caso for buildado você ira recebera uma saida parecida com essa 

      db.json  dist  eslint.config.js  index.html  node_modules  package.json  package-lock.json  public  server.js  src  vite.config.js

   
   3.em seguida use o comando cp para copiar a construção da pagina do front end para a pasta do loadbalaspharmasys 

      cp -r /home/henrique/Downloads/PharmaSys-Front-main/dist /home/henrique/loadbalaspharmasys/
      

  # ⚙️ Passo 3: Configuração do Load Balancer

   crie o arquivo default.conf

      #  Define um grupo de nos de servidores upstream chamado 'nodes' para realizar balanceamento de carga

      upstream nodes {

         # Lista de servidores que receberão o tráfego atraves de nomes de serviços Docker
         server node1:80;
         server node2:80;
         server node3:80;
         server node4:80;
         server node5:80;
      }

      # Configuração do servidor principal
      server {

         # Escuta na porta 80 (HTTP padrão)
         listen 80;

         # Nome do servidor em produção
         server_name localhost;

         # Configuração para todas as requisições na raiz
         location / {

            # Redireciona o tráfego para o grupo 'nodes' definido acima
            proxy_pass http://nodes;

            # Headers importantes para manter informações originais da requisição:
            proxy_set_header Host $host;                  # Preserva o domínio da porta host
            proxy_set_header X-Real-IP $remote_addr;     # IP real do cliente
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;   # define uma Cadeia de proxies
            proxy_set_header X-Forwarded-Proto $scheme;              # Protocolo usado (http/https)
         }

         # Logs de acesso (registra todas as requisições)
         access_log /var/log/nginx/access.log;

         # Logs de erro (registra problemas)

         error_log /var/log/nginx/error.log;

          # Endpoint para monitoramento do Nginx (acessível via http://localhost/status)
         location /status { stub_status on; }   # Ativa a localização do status básico
      }


   #  🔧 Passo 4: Configuração Básica do NGINX

   Crie o arquivo nginx.conf:

         user  nginx;            # O Nginx vai rodar como usuário 'nginx' por segurança
         worker_processes  auto;             # Define automaticamente o número de workers baseado nos núcleos da CPU

         error_log  /var/log/nginx/error.log notice;      # Local do arquivo de log de erros (com nível 'notice')
         pid        /var/run/nginx.pid;                      # Arquivo que armazenará o ID do processo principal

         events {
            worker_connections  1024;                # Número máximo de conexões simultâneas por worker
         }

         http {
            include       /etc/nginx/mime.types;         # Inclui definições de tipos MIME para extensões de arquivos
            default_type  application/octet-stream;         # Tipo padrão para arquivos não especificados

            # Formato do log de acesso:
            log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                              '$status $body_bytes_sent "$http_referer" '
                              '"$http_user_agent" "$http_x_forwarded_for"';

            access_log  /var/log/nginx/access.log  main;    # Local do arquivo de log de acesso

            sendfile        on;                  # Habilita o método sendfile para transferência de arquivos mais eficiente
            keepalive_timeout  65;                   # Tempo que a conexão fica aberta após a resposta (em segundos)

            server {
               listen       80;                 # Escuta na porta 80 (HTTP padrão)
               server_name  localhost;             # Nome do servidor (neste caso, localhost)

               location / {                             # Configuração para a rota raiz "/"
                     root   /usr/share/nginx/html;            # Diretório onde estão os arquivos do site
                     index  index.html index.htm;                # Arquivos de índice procurados nesta ordem          
               }
            }
         }

   
   # 🐳 Passo 5: Configuração do Docker Compose

   aqui voce ira criar o arquivo de configuração do docker compose para rodar os containers do nginx com docker no front end



            version: '3.8'       # Versão do formato do arquivo docker-compose

            services:         # serviço a ser executado
            node1:
               image: nginx:1.25.2-alpine                # Usa a imagem oficial do Nginx com versão específica (Alpine é leve)
               container_name: pharmasys-node1           # nome personalizado para o container  
               volumes:                          #volume
                  - "/home/henrique/loadbalaspharmasys/build:/usr/share/nginx/html"  # Monta o diretório local no contêiner
               ports:
                  - "8081:80"     # Expõe a porta 80 do contêiner como 8081 na máquina host
               networks:
                  - mynetwork    # Conecta à rede personalizada


            # Os nodes 2 a 5 seguem a mesma configuração do node1, mas com portas diferentes
            node2:
               image: nginx:1.25.2-alpine
               container_name: pharmasys-node2 
               volumes:
                  - "/home/henrique/loadbalaspharmasys/build:/usr/share/nginx/html" 
               ports:
                  - "8082:80"
               networks:
                  - mynetwork

            # mesma coisa so muda a porta
            node3:
               image: nginx:1.25.2-alpine
               container_name: pharmasys-node3  
               volumes:
                  - "/home/henrique/loadbalaspharmasys/build:/usr/share/nginx/html" 
               ports:
                  - "8083:80"
               networks:
                  - mynetwork

            # mesma coisa so muda a porta
            node4:
               image: nginx:1.25.2-alpine
               container_name: pharmasys-node4  
               volumes:
                  - "/home/henrique/loadbalaspharmasys/build:/usr/share/nginx/html" 
               ports:
                  - "8084:80"
               networks:
                  - mynetwork

            # mesma coisa so muda a porta
            node5:
               image: nginx:1.25.2-alpine
               container_name: pharmasys-node5  
               volumes:
                  - "/home/henrique/loadbalaspharmasys/build:/usr/share/nginx/html" 
               ports:
                  - "8085:80"
               networks:
                  - mynetwork

            loadbalancer:           # o cara do balanceador
               image: nginx:1.25.2-alpine       # Mesma imagem do Nginx
               container_name: pharmasys-loadbalancer   # Nome do contêiner do balanceador
               ports:
                  - "80:80"          # Expõe a porta 80 padrão HTTP para o host
               volumes:
                  - ./default.conf:/etc/nginx/conf.d/default.conf     # Monta o arquivo de configuração personalizado

               depends_on:       
                  - node1       # Garante que os nodes iniciem primeiro
                  - node2
                  - node3
                  - node4
                  - node5
               networks:
                  - mynetwork     # Mesma rede dos nodes

            networks:
            mynetwork:
               driver: bridge        # Cria uma rede bridge para comunicação entre os contêineres


   # ▶️ Como Executar de duas formas

   1️⃣ opção.  Inicie os containers:

         docker-compose up -d

   
   2️⃣ opção. rodando todos os serviços ou rodando um de cada tanto faz

   ![alt text](<Captura de tela de 2025-06-13 05-45-06.png>)

   

   # 🚀 acessando a minha aplicação com base na porta configurada:


   🗾 Load Balancer:             
         
      👉   http://localhost


   ↪️ Nodes individuais:         
         
      👉    http://localhost:8081 a http://localhost:8085


   ✔ Verificar o status:       
   
      👉  curl http://localhost/status

   ✔ Verifique os logs: os ips apareceram no log dos nos. para ver 
  digite o seguinte comando:

       👉    docker-compose logs node1 | grep "X-Real-IP"
          

   
   
   
   # ✅ aqui esta uma imagem rodando

   ![alt text](<Captura de tela de 2025-06-13 05-20-15-1.png>)





   # 💡 Dicas

         Para parar os containers:           docker-compose down

         Verifique logs:                     docker logs pharmasys-loadbalancer


   # Este setup cria um ambiente robusto para desenvolvimento e teste com balanceamento de carga entre 5 nos de instâncias NGINX.




   # 👋 Adeus,ate a proxima!
