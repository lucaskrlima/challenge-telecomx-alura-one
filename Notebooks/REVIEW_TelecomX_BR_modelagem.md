# Code Review — Notebook de Modelagem (`TelecomX_BR_modelagem.ipynb`)

> Referência solicitada: link Colab informado pelo usuário.
> Nesta execução, o acesso direto ao Colab não foi possível por bloqueio de rede (403). A revisão abaixo foi feita sobre o notebook equivalente presente no repositório local: `Notebooks/TelecomX_BR_modelagem.ipynb`.

## Resumo executivo
O notebook já está **bem acima de um baseline inicial**: possui pipeline com `ColumnTransformer`, validação cruzada estratificada, avaliação holdout e ajuste de threshold.

As melhorias prioritárias para elevar robustez e evitar viés otimista são:
1. Evitar **retreino no conjunto de teste** durante busca de threshold.
2. Adicionar **conjunto de validação** (ou CV interno) para seleção de threshold e hiperparâmetros.
3. Tratar explicitamente categorias semânticas (`No internet service`/`No phone service`).
4. Incluir **análise de calibragem** e métricas de negócio (ex.: top-k recall).
5. Preparar o notebook para reuso em produção (funções, persistência do pipeline, tracking).

---

## Pontos fortes
- Ingestão robusta com `timeout` e `raise_for_status`.
- Padronização de schema e checagem de colunas esperadas no rename.
- Separação treino/teste estratificada e uso de pipeline para evitar leakage de transformação.
- Métricas relevantes para churn: ROC-AUC, PR-AUC, Recall/F1/Precision.
- Primeiro ajuste de threshold orientado por objetivo.

---

## Melhorias recomendadas (priorizadas)

## 1) Evitar vazamento por seleção de threshold no teste
**Achado:** A função `avaliar_modelo` faz `fit` e avalia no teste; depois ela é reutilizada em loop de thresholds com o mesmo `X_test`, o que transforma o teste em conjunto de tuning.

**Risco:** métrica final otimista e baixa generalização.

**Sugestão:**
- dividir treino em `train/valid` (ou usar CV interno) para escolher threshold;
- manter `test` apenas para avaliação única final.

---

## 2) Separar função de treino e função de avaliação
**Achado:** `avaliar_modelo` chama `pipe.fit(...)` toda vez.

**Risco:** retreino desnecessário, custo computacional maior e risco de uso incorreto em etapas de seleção.

**Sugestão:**
- `treinar_modelo(pipe, X_train, y_train)` retorna pipeline fitado;
- `avaliar_modelo(pipe_fitado, X, y, threshold)` apenas mede desempenho.

---

## 3) Categorias semânticas ambíguas
**Achado:** O dataset clássico de churn traz valores como `No internet service` e `No phone service`.

**Risco:** one-hot pode capturar semântica inconsistente entre variáveis relacionadas.

**Sugestão:**
- padronizar categorias com mapeamento explícito (`sem_internet`, `sem_telefone`, `nao`, `sim`);
- registrar esse mapeamento em uma célula de regras de negócio.

---

## 4) Tratar desbalanceamento de forma mais orientada a negócio
**Achado:** `RandomForest` usa `class_weight='balanced_subsample'`, mas `LogisticRegression` não usa balanceamento.

**Sugestão:**
- testar `class_weight='balanced'` também na regressão logística;
- comparar impacto em Recall e PR-AUC;
- escolher threshold com base em custo de retenção (FN x FP), não apenas melhor F1.

---

## 5) Adicionar calibração de probabilidade
**Achado:** threshold depende da qualidade das probabilidades estimadas.

**Sugestão:**
- comparar modelo calibrado (`CalibratedClassifierCV`) vs não calibrado;
- medir Brier score e curva de calibração.

---

## 6) Expandir avaliação além de métricas médias
**Achado:** CV reporta média apenas.

**Sugestão:**
- incluir desvio-padrão e intervalo (mín/máx) por métrica;
- mostrar matriz de confusão normalizada e curva PR/ROC no holdout.

---

## 7) Engenharia de atributos e consistência temporal
**Achado:** boas features derivadas já existem, mas faltam verificações de estabilidade.

**Sugestão:**
- testar bucketização de `meses_contrato`;
- monitorar drift de distribuição por período (quando houver snapshots);
- validar importância de features por permutation importance/SHAP (baseline).

---

## 8) Reprodutibilidade e entrega para produção
**Sugestão:**
- encapsular em funções (`carregar`, `preparar`, `treinar`, `avaliar`);
- exportar pipeline final com `joblib`;
- registrar versão de dependências (`requirements.txt`/`pyproject`).

## Checklist prático (próxima iteração)
- [ ] Criar split `train/valid/test`.
- [ ] Selecionar threshold no `valid` e congelar para `test`.
- [ ] Adicionar `class_weight='balanced'` para Logistic Regression e comparar.
- [ ] Adicionar curvas ROC/PR + calibração.
- [ ] Persistir pipeline vencedor e gerar artefato de inferência.
