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

**NOTEBOOK JUPYTER**

**Configuração e Conexão (Código Essencial)**

Como a aplicação se conecta à infraestrutura Docker e configura o ambiente para lidar com arquivos grandes:

**1. IMPORTAÇÕES E CONFIGURAÇÃO (Executar Primeiro)**
```
import os
import pandas as pd
import kagglehub
from pymongo import MongoClient
```

**Configuração Crítica de Disco (Volume Mapping)**

O bloco de código aborda um problema comum ao lidar com datasets grandes em ambientes conteinerizados: a falta de espaço.
```
# 📂 CONFIGURAÇÃO CRÍTICA DE DISCO
# Aponta o cache de download para a pasta mapeada no Docker (seu HD físico).
# Isso evita o erro "No space left on device" da VM.
CACHE_DIR = '/home/jovyan/datasets/kaggle_cache'
os.environ['KAGGLEHUB_CACHE'] = CACHE_DIR
```
**Configuração do MongoDB (Conexão de Serviço)**

Este bloco define como o seu script (rodando no contêiner Jupyter) encontra e se autentica no serviço de banco de dados (rodando no contêiner MongoDB).
```
# 🔌 CONFIGURAÇÃO DO MONGODB
# Usa o nome do serviço 'mongo_service' definido no docker-compose
MONGO_URI = "mongodb://root:mongo@mongo_service:27017/?authSource=admin"
DB_NAME = "SteamRecommendations"

print(f"⚙ Configuração:")
print(f"   -> Cache Temporário: {CACHE_DIR}")
print(f"   -> Destino MongoDB: {MONGO_URI.split('@')[1]}") # Mostra apenas o host para confirmar
```
**2. Função de Ingestão Otimizada (Chunking)**

A função abaixo é a chave para a eficiência, pois lê os arquivos CSV em pedaços de 10.000 linhas (chunk_size), prevenindo a sobrecarga da memória do contêiner Jupyter:
```
def ingest_file_to_mongo(file_path, collection_name):
    """Lê CSV em pedaços e envia para o Mongo para economizar RAM."""
    # ... inicializa cliente
    chunk_size = 10000 # Processa 10 mil linhas por vez
    
    try:
        # Lê o arquivo CSV em pedaços
        with pd.read_csv(file_path, chunksize=chunk_size, low_memory=False) as reader:
            for chunk in reader:
                data = chunk.to_dict('records')
                if data:
                    collection.insert_many(data)
                    # ... contagem e feedback
    # ... tratamento de erros
```
**Inicialização e Limpeza (Pré-Carga)**
```
client = MongoClient(MONGO_URI)
db = client[DB_NAME]
collection = db[collection_name]

# Limpa dados antigos para evitar duplicação se rodar 2x
collection.drop()
print(f"\n🔄 Processando '{collection_name}'...")
```
**Otimização de Memória (Chunking)**

Esta é a parte mais importante para um projeto de Big Data, pois resolve o problema de RAM limitada.
```
chunk_size = 10000 # Processa 10 mil linhas por vez
# ...
with pd.read_csv(file_path, chunksize=chunk_size, low_memory=False) as reader:
    for chunk in reader:
        # ... processamento do chunk
```
**Transformação e Carga (ETL)**

Dentro do loop de iteração (o coração da função), o código transforma o pedaço lido em um formato compatível com o MongoDB e o insere.
```
for chunk in reader:
                # Converte para formato JSON (lista de dicionários)
                data = chunk.to_dict('records')
                if data:
                    collection.insert_many(data)
                    total_inserted += len(data)
                    print(f"   -> Inseridos: {total_inserted} registros...", end='\r')
```
**Conclusão e Robustez do Pipeline**

Este bloco finaliza o processamento de cada coleção, garantindo que o pipeline seja robusto (tratamento de falhas) e limpo (liberação de recursos).
```
print(f"\n✅ Concluído: {total_inserted} documentos na coleção '{collection_name}'.")
        
    except Exception as e:
        print(f"\n❌ Erro ao processar {collection_name}: {e}")
    finally:
        client.close()
```
orchestrator Lógica da Função run_etl()

A função run_etl() coordena o fluxo de dados desde a origem (Kaggle) até o destino (MongoDB), garantindo que os dados sejam baixados, processados e verificados.

**Extração e Verificação**

Esta é a fase inicial, responsável por obter os dados brutos da fonte externa (Kaggle).

```
print("⬇ 1. Baixando dados do Kaggle (Isso pode levar alguns minutos)...")
try:
    path = kagglehub.dataset_download("antonkozyriev/game-recommendations-on-steam") 
    print(f"📂 Dados baixados em: {path}")
except Exception as e:
    # ... tratamento de erro
    return
```
**Carga Otimizada**

Esta fase é a iteração principal, onde a Carga de dados (utilizando a lógica de chunking da função ingest_file_to_mongo) é disparada para cada arquivo CSV.
```
files_map = { 
    'games.csv': 'games',
    'users.csv': 'users',
    'recommendations.csv': 'recommendations'
}
print("\n🚀 2. Enviando para o MongoDB...")

for csv_file, col_name in files_map.items():
    full_path = os.path.join(path, csv_file)
    if os.path.exists(full_path):
        ingest_file_to_mongo(full_path, col_name)
    else:
        print(f"⚠ Arquivo {csv_file} não encontrado.")
```
**Validação Final**

Esta etapa garante a qualidade e a completude do processo ELT.
```
print("\n📊 3. Verificação Final do Banco de Dados:")
client = MongoClient(MONGO_URI)
db = client[DB_NAME]
for col in db.list_collection_names():
    count = db[col].count_documents({})
    print(f"   📁 {col}: {count:,} documentos")
client.close()
```
**Comando de Execução do Pipeline**

A linha final do script é a chamada da função run_etl(), que é o ponto de entrada principal para iniciar todo o processo de Extração, Carga e Validação.
```
run_etl()

```
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

Utilizei o comando: **docker ps** para poder listar todos os containers na minha aplicação.
<img width="1312" height="111" alt="image" src="https://github.com/user-attachments/assets/bd53fbe4-3162-4336-99d7-7ab523f62fa3" />

Utilizei o comando: **docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' <CONTAINER_NAME_OR_ID>**
Para obter **<CONTAINER_NAME_OR_ID>**. Para no fim utilizar **docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'** mongo_service para encontrar o endereço ip
<img width="971" height="35" alt="image" src="https://github.com/user-attachments/assets/6fea60d7-325e-4c79-bd3d-750358cccb31" />

Como fiz no metabase:

<img width="1918" height="967" alt="image" src="https://github.com/user-attachments/assets/d4206c50-0a72-4e38-9966-ee19e6377c96" />




