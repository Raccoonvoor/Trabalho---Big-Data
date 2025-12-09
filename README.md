# Trabalho Big Data  

**1. Estrutura de Pastas do Repositório**
```
steam-bigdata-pipeline/
│
├── data/                   # (Ignorado pelo Git) Onde os dados brutos ficam
│   └── kaggle_cache/       # Mapeado para o container Jupyter
│
├── notebooks/              # Seus scripts de análise e ETL
│   └── SteamData.ipynb  # O código que você forneceu vai aqui
│
├── docker-compose.yml      # A orquestração de toda a infraestrutura
├── requirements.txt        # Dependências do Python
├── README.md               # A documentação principal do projeto
└── .gitignore              # Para não subir arquivos pesados (CSV)
```
**2. O Arquivo docker-compose.yml (A Infraestrutura)**

Este é o coração do projeto. Ele sobe o Jupyter, o Mongo, o MinIO e o Metabase, conectando todos na mesma rede.
Crie um arquivo chamado docker-compose.yml na raiz:

version: '3.3'

services:
**# 1. JUPYTER LAB (Onde roda o código Python)**

**docker-compose.yml do Jupyter (ETL e Processamento).**
```
version: '3.3'

services:

  jupyter:
    build:
      context: . # Manda o Docker construir a imagem a partir de um Dockerfile (não mostrado) na pasta atual.
    ports:
      - "8888:8888" # Acessa o Jupyter Notebook em http://localhost:8888.
      # - "4040:4040" # Comentado, mas pronto para Spark UI, indicando escalabilidade futura.
    volumes:
      - ./notebooks:/home/jovyan/work # **CRÍTICO:** Mapeia seus scripts locais para o ambiente de trabalho do Jupyter.
      - ./data:/home/jovyan/data # Mapeamento para o seu código salvar ou buscar arquivos.
      - ./config:/home/jovyan/.jupyter # Persistência de configurações do Jupyter.
    networks:
      - mybridge # Conecta o Jupyter à rede comum para se comunicar com o MongoDB.

networks:
  mybridge: # O Jupyter depende dessa rede.
    external: # **CHAVE:** Declara que a rede já deve existir (criada separadamente ou por outro docker-compose).
      name: mybridge

volumes: # Declaração dos volumes, embora o mapeamento esteja usando caminhos relativos (./notebooks)
  notebooks:
  data:
  config:
```
Função: Execução do seu pipeline de ingestão e processamento.

Conexão: Ele se conecta ao MongoDB usando o nome do serviço mongo (conforme definido no arquivo do MongoDB) através da rede mybridge.

 **2. MONGODB (Banco NoSQL)**
 ```
version: '3.3'

services:
  mongo:
    image: mongo:4.4-bionic # Versão específica, garantindo estabilidade.
    container_name: mongo_service # Nome que você usa no seu código Python para a URI: 'mongo_service'.
    environment: # Credenciais de acesso administrativo.
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: mongo
    ports:
      - "27017:27017" # Porta para acesso externo (opcionalmente).
    volumes:
      - dbdata:/data/db # **CRÍTICO:** Volume nomeado para persistir o banco de dados.
      - ./db-seed:/db-seed # Volume para scripts de inicialização de banco de dados (se houver).
      - ./datasets:/datasets # Volume mapeado para dados (não usado pelo seu código Jupyter, mas boa prática).
    networks:
      - mybridge # O serviço de banco de dados precisa estar na rede.

  mongo-express:
    image: mongo-express:latest # UI para visualizar o MongoDB (muito útil para ver as coleções).
    container_name: mongo_express_service
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: admin # Variáveis de configuração do Mongo Express.
      ME_CONFIG_MONGODB_ADMINPASSWORD: pass
      ME_CONFIG_MONGODB_URL: mongodb://root:mongo@mongo:27017/ # URI de conexão.
    ports:
      - "8081:8081" # Acessa a UI do Mongo Express em http://localhost:8081.
    networks:
      - mybridge
    depends_on:
      - mongo # Garante que o Mongo suba antes do Mongo Express.
    # O comando wait-for-it.sh é uma técnica avançada para garantir que o Mongo esteja pronto antes de iniciar o Express.

networks:
  mybridge: # A rede que hospeda os serviços de dados.
    external: true # Redundância na declaração da rede.

volumes:
  dbdata: # O volume nomeado para persistência do Mongo.
  db-seed:
```
Função: Armazenamento e fornecimento dos dados.

Conexão: Expõe o nome de serviço mongo_service e o host mongo (no Mongo Express) para os outros containers.

**3. METABASE (Visualização e BI)**
```
version: '3.3'

services:
  metabase:
    image: metabase/metabase:latest # Imagem oficial do Metabase.
    container_name: metabase
    ports:
      - "3000:3000" # Acessa o Metabase em http://localhost:3000.
    environment:
      MB_DB_TYPE: h2 # Configuração do banco de dados interno (Metabase armazena suas próprias configs aqui).
      MB_DB_FILE: /metabase-data/metabase.db
      MB_JETTY_PORT: 3000
    volumes:
      - ./metabase-data:/metabase-data # Persistência do banco de dados interno do Metabase.
    restart: unless-stopped
    networks:
      - mybridge # **CHAVE:** Conecta o Metabase à rede comum.

networks:
  mybridge: # O Metabase precisa dessa rede para se conectar ao MongoDB.
    external:
      name: mybridge
```
Função: Catálogo e exploração dos dados modelados.

Conexão: Você configurará o Metabase manualmente (após iniciar) para usar o host mongo (ou mongo_service, dependendo de como você o nomeou) na porta 27017 dentro da rede mybridge.

  **Steam Recommendations Big Data Pipeline:**
  
O projeto implementa um pipeline de Engenharia de Dados ponta-a-ponta para processar, armazenar e visualizar dados de recomendações de jogos da Steam. O objetivo é ingerir grandes volumes de dados brutos e transformá-los em insights visuais.

**Arquitetura do Projeto:**

O projeto utiliza uma arquitetura baseada em microsserviços via Docker:

Ingestão & Processamento (Jupyter): Scripts Python baixam dados do Kaggle, realizam a limpeza e o chunking (processamento em lote) para otimização de memória.

**Data Lake/Storage (MongoDB):**

MongoDB: Atua como nosso Data Warehouse NoSQL, armazenando os dados estruturados de Usuários, Jogos e Recomendações.

Visualização (Metabase): Conectado ao MongoDB para criação de Dashboards e exploração de dados.

