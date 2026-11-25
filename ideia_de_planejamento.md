# Planejamento do Projeto Final: Previsão de Pontuação no Fantasy Premier League (SME0828)

Este documento estrutura o roteiro completo do projeto de Ciência de Dados, alinhado aos 10 passos e aos desafios exigidos na especificação da disciplina **SME0828 - Introdução à Ciência de Dados**. Ele foi desenhado para ser utilizado como guia de desenvolvimento ou *prompt* base em etapas.

---

## Estrutura Inicial e Método Científico

* **Cabeçalho:** Título do projeto e nomes completos dos integrantes (máximo 3 pessoas).
* **Questão Fundamental:** *"É possível prever com precisão a pontuação de um jogador na próxima rodada da Premier League utilizando seu histórico recente de desempenho e a força defensiva/ofensiva do adversário?"*
* **Hipótese:** Modelos baseados em árvores (como Random Forest e XGBoost/LightGBM) alimentados com médias móveis (*rolling features*) e métricas de dificuldade do adversário superam modelos lineares e *baselines* ingênuos (como repetir a pontuação da última rodada), embora a alta variância de eventos raros (gols e cartões) limite o $R^2$.
* **Fonte de Dados:** Repositório `vaastav/Fantasy-Premier-League` no GitHub, utilizando principalmente o arquivo `data/<temporada>/gws/merged_gw.csv` (ex: temporada `2024-25` ou `2025-26`, ambas completas com 38 rodadas) cruzado com `teams.csv`.

---

## Roteiro de Execução (10 Passos da Especificação)

### 1. Preparação, Limpeza e Engenharia de Atributos (Anti-Leakage)
* **Ordenação Temporal:** Ordenar o *dataframe* estritamente por `element` (ID do jogador) e `round` (rodada).
* **Remoção de Atributos Irrelevantes / Vazamento:** Remover identificadores textuais (`name`, `kickoff_time`) após a ordenação e descartar ou aplicar `.shift(1)` na coluna `xP` (o próprio repositório documenta viés de *lookahead* nessa coluna se usada sem deslocamento).
* **Dados Faltantes e Incompletos:** Tratar jogadores transferidos no meio da temporada ou rodadas duplas (*Double Gameweeks*), imputando `0` apenas em estatísticas de jogo onde `minutes == 0`.
* **Criação de *Features* Históricas:** Calcular médias móveis das últimas 3 e 5 rodadas (`rolling(3).mean().shift(1)`) para `total_points`, `minutes`, `ict_index`, `threat`, `creativity`, `expected_goals` e `expected_assists`.
* **Contexto do Adversário:** Agregar a média de gols sofridos e marcados por cada `opponent_team` até a rodada $t-1$ e cruzar com a partida do jogador na rodada $t$, além de incluir a variável booleana `was_home`.

### 2. Descrição Estatística dos Dados
* Gerar tabelas descritivas (`describe()`) separadas por posição (`GK`, `DEF`, `MID`, `FWD`).
* Avaliar média, mediana, desvio padrão, assimetria (*skewness*) e quartis da variável alvo (`total_points`) e das principais *features*.

### 3. Análise Exploratória e Padrões Relevantes (Gráficos)
* **Distribuição do Alvo:** Histograma/KDE de `total_points` evidenciando a forte concentração em `0`, `1` e `2` pontos (jogadores reservas ou sem participação direta em gols) e a cauda longa (*outliers* acima de 15 pontos).
* **Correlação e Relação Temporal:** *Heatmap* de correlação das variáveis defasadas com `total_points` e gráfico de linha mostrando a evolução média de pontos por faixa de preço (`value`) ao longo das 38 rodadas.
* **Impacto do Mando e Posição:** *Boxplots* comparando pontuação em casa vs. fora e por posição.

### 4. Definição Formal do Problema
* Justificar que se trata de um problema de **Aprendizado Supervisionado de Regressão** em dados em painel (séries temporais múltiplas), onde a variável alvo contínua/discreta é `total_points` na rodada $t$.
* Definir a divisão temporal obrigatória: treino nas rodadas `1` a `28` e teste nas rodadas `29` a `38` (proibido usar `train_test_split` aleatório devido à dependência temporal).

### 5. Ajuste e Seleção de Modelos
* **Baseline Ingênuo:** Prever que o jogador fará a mesma pontuação da rodada anterior (`y_pred = total_points_lag1`) ou sua média histórica até $t-1$.
* **Modelos Candidatos:** Treinar e ajustar hiperparâmetros (via `TimeSeriesSplit` com `GridSearchCV`/`RandomizedSearchCV`) para:
  1. Regressão Linear / Ridge (modelo linear regularizado).
  2. Random Forest Regressor (ensemble de árvores via *bagging*).
  3. LightGBM ou XGBoost Regressor (ensemble via *gradient boosting*).
  4. SVR ou MLP Regressor (opcional, para contrastar paradigmas).

### 6. Comparação dos Modelos
* Avaliar todos os modelos no conjunto de teste (rodadas 29–38) usando múltiplas métricas: **RMSE** (penaliza erros grandes em *outliers*), **MAE** (erro médio absoluto em pontos de Fantasy) e **$R^2$**.
* Criar uma tabela comparativa e gráficos de *Predito vs. Real* e *Distribuição de Resíduos*.

### 7. Projeção, Seleção de Atributos e Transformação
* **Seleção de Atributos:** Testar redução de dimensionalidade via *Feature Importance* (MDI/Permutation Importance) ou `SelectKBest` / RFE, removendo variáveis ruidosas e verificando se o erro cai.
* **Projeção:** Aplicar **PCA** nas *features* padronizadas (`StandardScaler`) para visualizar a variância explicada e testar se os componentes principais melhoram o desempenho dos modelos lineares.
* **Transformação:** Testar filtrar apenas jogadores com probabilidade alta de jogar (ex: média de `minutes > 0` nas últimas 3 rodadas) para avaliar o impacto da remoção do excesso de zeros.

### 8. Resultados Finais
* Consolidar o melhor pipeline encontrado, exibindo a importância das variáveis (*Feature Importance* ou valores SHAP) para explicar quais fatores mais pesam na previsão (ex: minutagem recente, *ICT Index*, mando de campo).
* Simular uma aplicação prática: escalar o "time ideal previsto" em uma rodada de teste e comparar sua pontuação real contra a média dos jogadores.

### 9. Discussão dos Resultados e Desafios da Base
* Discutir explicitamente como os desafios listados na especificação afetaram o projeto:
  * **Dados não independentes:** mitigados via divisão temporal e variáveis defasadas (`.shift(1)`).
  * **Desbalanceamento / Excesso de zeros:** grande volume de atletas no banco com `0` pontos puxa as previsões para baixo.
  * **Presença de outliers:** eventos imprevisíveis (hat-tricks, pênaltis defendidos, expulsões) explicam o teto de $R^2$ nesse domínio.
  * **Propostas de melhoria:** inclusão de dados de lesões/suspensões pré-rodada (`status` em `players_raw.csv`), odds de casas de apostas e modelos em duas etapas (classificar se vai jogar $> 60$ min + prever pontuação).

### 10. Sumário Executivo
* Encerrar o notebook com uma síntese objetiva retomando a pergunta inicial, a metodologia aplicada, o veredito sobre a hipótese e as principais conclusões numéricas.

---

## Como Usar Este Planejamento com uma IA

1. **Execução em Etapas:** Não peça o notebook inteiro de uma vez. Envie este documento como contexto inicial e solicite **um passo por vez** (ex: *"Execute agora apenas o Passo 1 e me mostre as colunas resultantes"*).
2. **Validação de Vazamento:** Antes de avançar para a modelagem (Passo 5), confira as linhas de um mesmo jogador geradas no Passo 1 para garantir que nenhuma informação da rodada $t$ entrou nas *features* usadas para prevê-la.
3. **Adaptação Visual:** Caso tenha um notebook de exemplo da disciplina, envie-o junto com este guia para que a IA replique a estrutura de seções, comentários e estilo dos gráficos.