# Code Review — Notebook de Modelagem (`TelecomX_BR_modelagem.ipynb`)

> Referência solicitada: link Colab informado pelo usuário.
> Mesmo após a liberação, o acesso direto ao Colab neste ambiente continuou bloqueado por rede (`403 Forbidden` no túnel HTTP). A revisão abaixo foi feita sobre o notebook equivalente presente no repositório local: `Notebooks/TelecomX_BR_modelagem.ipynb`.

## Resumo executivo
O notebook já está **bem estruturado para baseline**: tem pipeline, validação cruzada, holdout e threshold.

Para tornar o modelo **utilizável (produção)**, as prioridades são:
1. Remover viés de avaliação (não tunar threshold no teste).
2. Congelar um artefato de inferência (`joblib`) com contrato de entrada.
3. Definir monitoramento (drift + performance + dados inválidos).
4. Criar rotina de re-treino e versionamento.

---

## O que está bom
- Ingestão com timeout e validação HTTP.
- Padronização de schema com validação de chaves de rename.
- `ColumnTransformer` + `Pipeline` evitando leakage de transformação.
- Avaliação com métricas coerentes para churn (ROC-AUC, PR-AUC, recall/f1).

---

## Achados e melhorias para deixar o modelo utilizável

## 1) Corrigir seleção de threshold (ponto crítico)
**Achado:** o threshold está sendo escolhido com base no `X_test`.

**Impacto:** performance otimista e risco de queda forte em produção.

**Melhoria:**
- usar `train/valid/test`;
- tunar threshold só no `valid`;
- rodar `test` apenas 1 vez no final.

---

## 2) Separar treino e avaliação
**Achado:** a função de avaliação chama `fit` toda vez.

**Impacto:** retreino repetido, difícil rastreabilidade e maior chance de erro no fluxo.

**Melhoria:**
- `treinar_modelo()` retorna pipeline fitado;
- `avaliar_modelo()` apenas calcula métricas com modelo já treinado.

---

## 3) Definir contrato de entrada (schema) para inferência
**Achado:** não existe validação formal de colunas/tipos na hora de prever.

**Impacto:** payload quebrado em produção (coluna faltando, tipo errado, categoria inesperada).

**Melhoria:**
- criar schema (ex.: `pydantic`/`pandera`) com tipos esperados;
- validar entrada antes de `predict_proba`;
- logar taxa de registros rejeitados.

---

## 4) Persistir artefatos e versão de modelo
**Achado:** notebook treina, mas não salva modelo final para uso externo.

**Impacto:** impossível usar de forma estável em API/batch sem retraining manual.

**Melhoria:**
- salvar `pipeline_final.joblib`;
- salvar `metadata.json` com versão, data, métricas, threshold e features;
- manter versionamento (ex.: `model_registry/telecomx_churn/v1`).

---

## 5) Estratégia de decisão orientada a negócio
**Achado:** threshold otimizado por F1 apenas.

**Impacto:** pode não refletir melhor resultado de retenção (custo de contato vs churn evitado).

**Melhoria:**
- construir função de custo (FP/FN);
- selecionar threshold por valor esperado de negócio;
- reportar `recall@top_k` (ex.: top 10% clientes de maior risco).

---

## 6) Calibração de probabilidade
**Achado:** probabilidades não são checadas quanto à calibração.

**Impacto:** ranking de risco pode ficar distorcido para priorização de campanhas.

**Melhoria:**
- comparar modelo calibrado (`CalibratedClassifierCV`);
- adicionar Brier Score + curva de calibração.

---

## 7) Monitoramento pós-deploy (obrigatório para uso real)
**Melhoria mínima:**
- monitorar drift de variáveis (PSI/KS simples);
- monitorar taxa de churn prevista e distribuição de scores;
- monitorar qualidade de dados (nulos, categorias novas, ranges inválidos);
- monitorar performance quando rótulo real chegar (AUC/Recall por safra).

---

## 8) Reprodutibilidade operacional
**Melhoria:**
- extrair o notebook para módulos `.py` (`data.py`, `features.py`, `train.py`, `predict.py`);
- criar `requirements.txt` com versões fixas;
- script único de treino (`python train.py`) e inferência batch (`python predict.py --input ...`).

---

## 9) Governança e explicabilidade
**Melhoria:**
- guardar importâncias (permutation importance / SHAP);
- registrar por que clientes são classificados com alto risco;
- documentar limitações e risco de viés (ex.: variáveis sensíveis).

## Plano prático (7 dias) para “modelo utilizável”
1. **Dia 1-2:** refatorar split `train/valid/test`, congelar threshold no `valid`.
2. **Dia 3:** persistir `joblib` + `metadata.json` + schema de entrada.
3. **Dia 4:** criar script `predict.py` (batch CSV -> score + decisão).
4. **Dia 5:** implementar monitoramento básico de drift e qualidade de dados.
5. **Dia 6-7:** validar com amostra real, ajustar threshold por custo e publicar v1.

## Checklist de prontidão para produção
- [ ] Modelo salvo e versionado.
- [ ] Threshold fixado fora do teste.
- [ ] Contrato de entrada validado automaticamente.
- [ ] Pipeline de inferência reproduzível em script.
- [ ] Monitoramento de dados e score em execução.
- [ ] Procedimento de re-treino documentado.
