# Code Review — `TelecomX_BR.ipynb`

## Objetivo do review
Avaliar o notebook com foco em preparar uma base confiável para um **modelo preditivo de churn**.

## Pontos positivos
- O fluxo macro está bem dividido em etapas de extração, transformação e análise.
- Já existe tratamento inicial de tipos e criação de uma feature derivada (`contas_diarias`).
- Há visualizações úteis para hipóteses iniciais de churn.

## Melhorias recomendadas (priorizadas)

## 1) Robustez de ingestão e reprodutibilidade
**Problema:** a coleta da API não valida status HTTP nem timeout.

**Risco para ML:** pipelines quebram silenciosamente em produção/execuções agendadas.

**Melhoria sugerida:**
- usar `requests.get(..., timeout=30)`;
- validar com `raise_for_status()`;
- opcional: salvar snapshot local em `Dados/` com timestamp para versionamento de dados.

---

## 2) Padronização de schema (bug de nomenclatura no rename)
**Problema:** após transformar colunas para minúsculas, algumas chaves do dicionário de rename mantêm camel case (`internet_onlineBackup`, `internet_StreamingMovies`) e outra tem typo (`internet_deviceoprotection`).

**Risco para ML:** colunas ficam sem renomear, gerando inconsistência entre EDA e etapa de modelagem/feature store.

**Melhoria sugerida:**
- garantir dicionário 100% em minúsculas e sem typos;
- validar colunas ausentes após rename com um check automático.

---

## 3) Tratamento de missing incompleto
**Problema:** `account_Charges_Total` é convertido com `errors='coerce'`, mas não há estratégia explícita para os NaNs resultantes.

**Risco para ML:** treino falha ou perde linhas relevantes; vazamento de decisão manual ad hoc.

**Melhoria sugerida:**
- definir política: remover linhas específicas (ex.: tenure 0) ou imputar (mediana por segmento);
- registrar em célula dedicada a quantidade removida/imputada.

---

## 4) Variáveis categóricas com valores semânticos ambíguos
**Problema:** várias colunas possuem categorias como `"No internet service"` / `"No phone service"`.

**Risco para ML:** one-hot ingênuo pode misturar ausência de serviço com resposta negativa real.

**Melhoria sugerida:**
- normalizar categorias para preservar semântica (ex.: `sem_internet`, `nao`);
- documentar mapeamentos e aplicar de forma vetorizada.

---

## 5) Codificação da variável alvo
**Problema:** o alvo `churn` permanece textual.

**Risco para ML:** inconsistências em avaliação/serialização de pipeline.

**Melhoria sugerida:**
- criar `target_churn = (churn == 'Yes').astype(int)`;
- manter coluna original para análise, mas padronizar a de treino.

---

## 6) Potencial vazamento e separação treino/teste
**Problema:** o notebook realiza transformações e EDA sem delimitar claramente a fronteira de treino/teste.

**Risco para ML:** data leakage ao calcular estatísticas globais antes de split.

**Melhoria sugerida:**
- separar em duas fases: `split` inicial e depois `fit` de transformadores só no treino;
- consolidar com `Pipeline`/`ColumnTransformer` do scikit-learn.

---

## 7) Engenharia de atributos adicionais
**Oportunidades de ganho preditivo:**
- `gasto_medio_mensal_por_tempo = gastos_totais / meses_contrato` (com proteção para divisão por zero);
- flags binárias: `contrato_mensal`, `pagamento_eletronico`, `fibra`;
- bucketização de `meses_contrato` para capturar não linearidade.

---

## 8) Avaliação estatística da EDA
**Problema:** EDA está majoritariamente descritiva visual.

**Risco para ML:** decisões de feature podem ficar subjetivas.

**Melhoria sugerida:**
- adicionar testes de associação (qui-quadrado para categóricas, Mann-Whitney para numéricas);
- incluir ranking simples de importância (mutual information ou árvore baseline).

---

## 9) Métricas e baseline de modelagem
**Problema:** não há baseline preditivo ainda.

**Melhoria sugerida:**
- começar com baseline de regressão logística + árvore;
- split estratificado (ex.: 80/20) e `cross-validation`;
- medir ROC-AUC, PR-AUC, F1, Recall (classe churn), matriz de confusão;
- ajustar threshold de decisão conforme custo de retenção.

---

## 10) Organização do notebook para produção
**Melhoria sugerida:**
- transformar blocos em funções (`carregar_dados`, `limpar_dados`, `criar_features`);
- mover regras para um script `.py` reutilizável;
- fixar semente (`random_state`) e versão de libs (`requirements.txt`).

## Checklist mínimo recomendado antes do 1º modelo
- [ ] Validar ingestão HTTP com tratamento de erro.
- [ ] Corrigir dicionário de rename e validar schema final.
- [ ] Definir política explícita para NaNs de `gastos_totais`.
- [ ] Criar `target_churn` binário.
- [ ] Separar treino/teste antes de ajustes baseados em distribuição.
- [ ] Treinar baseline com pipeline e métricas de classificação.
