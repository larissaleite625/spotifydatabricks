# Análise de Streaming do Spotify com Databricks e Enriquecimento de IA

Este projeto de engenharia de dados implementa um pipeline completo na plataforma **Databricks** para processar, analisar e visualizar o histórico de streaming do Spotify. Usaremos a arquitetura Medalhão (Bronze, Silver, Gold), culminando em um dashboard analítico no **Lakeview**.

O grande diferencial do projeto são os jobs automatizados que enriquecem os dados de duas formas:

  - Utilizando o DeepSeek, que é extremamente barato, para classificar e atribuir estilos musicais a cada faixa, permitindo análises de gênero mais específicas.
  - Consultando a API oficial do Spotify para obter metadados detalhados das músicas, como popularidade, data de lançamento do álbum e URLs das capas.

## 🏗️ Estrutura do Repositório

O projeto está organizado com uma estrutura clara para separar a lógica do pipeline, os artefatos do dashboard e as definições de workflow.

```

spotifydatabricks/
├── src/
│   ├── Silver 2014-2024.ipynb
│   ├── Silver 2025.ipynb
|   |── Get Album.ipynb
│   ├── Estilos com Deep Seek.ipynb
│   └── Validacao dados 2024.ipynb
├── dashs/
│   ├── Spotify.lvdash.json
│   └── \*.png
└── workflows/
└── Estilos.yaml
└── GetAlbum.yaml

```

- `src/spotify_project/`: Contém os notebooks que formam o pipeline ETL.
- `dashs/`: Armazena os ativos do dashboard, incluindo o arquivo de definição do Lakeview (`spotify.lvdash.json`) e as imagens de cada gráfico.
- `workflows/`: Contém a definição YAML do job do Databricks que automatiza o processo de enriquecimento.

---

## 🔎 Visão Geral Técnica

O pipeline foi desenhado para ser incremental e resiliente, processando um histórico de dados extenso e incorporando novas extrações de forma eficiente.

### 🥉 Camada Bronze: Ingestão de Dados Brutos

A camada Bronze serve como o repositório inicial dos dados extraídos do Spotify. Neste projeto, ela é composta por múltiplas fontes que representam diferentes períodos de extração:

- `bronze.spotify.spotify_analise`: Tabela principal contendo o histórico de streaming de 2014 a 2024. Esta fonte já passou por um tratamento inicial anteriormente.
- `bronze.spotify.streaming_history_audio_2024_2025_10` e `bronze.spotify.streaming_history_audio_2025_11`: Novas tabelas com dados brutos incrementais, solicitados recentemente ao Spotify, cobrindo o final de 2024 e o ano de 2025 até setembro.

### 🥈 Camada Silver: Limpeza e Padronização

Na camada Silver, os dados brutos são unificados, limpos e padronizados para criar uma fonte de verdade confiável.

- **Tabela Principal:** `silver.spotify_eng.streaming_history_2014_202509`
- **Processo:**
  1. O notebook **`Silver 2014-2024.ipynb`** processa a tabela histórica principal.
  2. O notebook **`Silver 2025.ipynb`** processa e une os novos arquivos de dados de 2025, garantindo a consistência do schema.
  3. As principais transformações incluem: conversão de timestamps, extração de features (ano, mês, hora), cálculo de métricas de tempo (`minutes_played`, `hours_played`) e limpeza de registros nulos ou inválidos.
  4. Finalmente, os dados históricos e os recentes são unidos para formar a tabela Silver consolidada.

Outras:
- **silver.spotify_eng.getalbum:** Tabela de dimensão enriquecida com metadados da API do Spotify. Orquestrado pelo job GetAlbum.yaml, enriquece os dados de forma incremental. Ele identifica músicas no histórico que ainda não possuem metadados detalhados, busca essas informações na API oficial do Spotify (como popularidade, detalhes do álbum, imagens) e as armazena na tabela silver.spotify_eng.getalbum.

### 🥇 Camada Gold: Modelagem e Análise

A camada Gold é onde os dados são transformados para atender às necessidades de negócio e análise. É aqui que o enriquecimento com IA acontece.

- **Tabelas Principais:**
  1. `gold.spotify.spotify_musicas`: Uma tabela de dimensão que serve como um **catálogo de músicas e seus respectivos estilos**. Ela é alimentada e enriquecida de forma incremental pelo job de IA.
  2. `gold.spotify.streaming_history_com_estilos`: A tabela de fatos final, que é a fonte principal para o dashboard. Ela é o resultado da junção da tabela Silver (`streaming_history_2014_202509`) com a tabela de estilos (`spotify_musicas`).

---

## 🧠 Enriquecimento de Estilos com IA (DeepSeek)

O coração da camada Gold é o notebook **`Estilos com Deep Seek.ipynb`**, orquestrado como um job recorrente (`workflows/Estilos.yaml`).

- **Lógica do Processo:**
  1. **Identificação:** O job primeiro compara o histórico da camada Silver com a tabela de estilos (`gold.spotify.spotify_musicas`) para encontrar músicas que ainda não foram classificadas.
  2. **Amostragem:** Se músicas não classificadas forem encontradas, o job seleciona um **lote aleatório de 10 faixas** para processar. Se nenhuma for encontrada, o job é encerrado com sucesso, economizando recursos.
  3. **Classificação via API:** Os metadados das 10 músicas são enviados para a API da **DeepSeek**, que retorna o estilo musical mais apropriado.
  4. **Atualização Inteligente (`MERGE`):** O resultado é inserido na tabela `gold.spotify.spotify_musicas` usando o comando `MERGE`. Isso garante que apenas as novas músicas sejam adicionadas, **evitando duplicatas** e enriquecendo a base de conhecimento de forma incremental.
  5. **Recriação da Tabela Final:** Por fim, a tabela `gold.spotify.streaming_history_com_estilos` é recriada do zero, garantindo que ela sempre reflita o estado mais atualizado dos estilos musicais.

---

## 📊 Dashboard e Visualizações

As análises são consolidadas em um dashboard no Databricks Lakeview, cujos gráficos são alimentados diretamente pela tabela `gold.spotify.streaming_history_com_estilos`.

| **Total de Horas Ouvidas por Mês** | **Artista Mais Escutado de Cada Ano** |
|------------------------------------|---------------------------------------|
| ![Total de horas ouvidas por mês](dashs/Total%20de%20horas%20ouvidas%20por%20m%C3%AAs.png) | ![Artista mais escutado de cada ano](dashs/Artista%20mais%20escutado%20de%20cada%20ano.png) |

| **Top 20 Artistas (Por Horas)** | **Top 20 Artistas (Por Plays)** |
|---------------------------------|---------------------------------|
| ![Top 20 Artistas (Soma de Horas)](dashs/Top%2020%20Artistas%20%28Soma%20de%20Horas%29.png) | ![Top 20 Artistas (Count Plays)](dashs/Top%2020%20Artistas%20%28Count%20Plays%29.png) |


| **Distribuição de Estilos Musicais** | **Top 20 Músicas (Por Horas)** |
|--------------------------------------|--------------------------------|
| ![Distribuição de Estilos Musicais](dashs/Distribui%C3%A7%C3%A3o%20de%20Estilos%20Musicais.png) | ![Top 10 músicas por soma de horas escutadas](dashs/Top%2010%20m%C3%BAsicas%20por%20soma%20de%20horas%20escutadas.png) |

![Área empilhada dos 5 estilos com mais minutos por mês](dashs/%C3%81rea%20empilhada%20dos%205%20estilos%20com%20mais%20minutos%20por%20m%C3%AAs.png)

---

## ▶️ Como Reproduzir no Databricks
1. **Solicite seus dados ao Spotify**:
    1. Acesse a página de privacidade da sua conta Spotify.
    2. Na seção "Baixar seus dados", solicite uma cópia do seu Histórico de streaming. O Spotify levará alguns dias para processar e enviar os arquivos para você.

2. **Ingestão para a Camada Bronze:** Você receberá vários arquivos JSON. No Databricks, utilize a funcionalidade de "Upload Data" na UI para carregar todos esses arquivos e criar tabelas. Você pode fazer todo o processo via interface mesmo, com poucos códigos.

3. **Adapte e Execute o Pipeline Silver:** Os notebooks já incluem o tratamento para o histórico de músicas. Você pode adaptá-los para incluir dados de podcasts, se desejar, ajustando os filtros e a lógica de tratamento.
4. **Configure e Execute a Camada Gold:** 
    1. Configure os secrets: Crie um Databricks Secret Scope chamado SPOTIFY e adicione uma chave chamada APIKEY_DEEPSEEK com sua chave da API da DeepSeek.
    2. Execute o Job de Enriquecimento: Rode o notebook Estilos com Deep Seek.ipynb. Ele irá automaticamente identificar as músicas sem estilo na sua camada Silver e começar a classificá-las em lotes.
5. **Importe o Dashboard:** No Databricks Lakeview, importe o arquivo `dashs/Spotify.lvdash.json`.
6. **Agende o Job (Opcional):** Em "Workflows", crie um novo job usando o arquivo `workflows/Estilos.yaml` para automatizar o enriquecimento.

---

## 🧪 Tecnologias Utilizadas

- **Plataforma:** Databricks (Notebooks, Workflows/Jobs, Lakeview, Delta Lake)
- **Linguagens:** Python, PySpark, SQL
- **IA / LLM:** API da DeepSeek para enriquecimento de dados
- **Arquitetura:** Medalhão (Bronze, Silver, Gold)
- **CI/CD:** Databricks Repos com integração GitHub

## 🚀 Futuro

- **Análise de Sentimentos com IBM Watson:**
    - Enviar as letras para a API do IBM Watson Natural Language Understanding para extrair o sentimento (positivo, negativo, neutro) e emoções (alegria, raiva, tristeza) de cada faixa.
    - Isso permitiria criar dashboards para analisar o "humor" musical ao longo do tempo.

- **Enriquecimento com a API Web do Spotify:**

    - Usar a API oficial do Spotify para enriquecer a tabela de estilos (gold.spotify.spotify_musicas) com features de áudio para cada faixa.
    - Atributos como danceability, energy, acousticness, valence (positividade) e tempo poderiam ser adicionados.
    - Isso possibilitaria análises técnicas sobre as características das músicas mais ouvidas e a criação de um sistema de recomendação pessoal.