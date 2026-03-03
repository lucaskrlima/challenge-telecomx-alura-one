# Code Review — `challenge-telecomx-alura-one-pt2`

## Status de acesso
Não foi possível acessar o repositório remoto durante esta execução por bloqueio de rede (erro `403 Forbidden` no túnel HTTP ao tentar baixar arquivos do GitHub).

## O que foi tentado
- Clone via `git clone`.
- Download direto de arquivo bruto (`raw.githubusercontent.com`).

## Próximo passo recomendado
Quando o acesso ao repositório estiver disponível, revisar com foco em:
1. **Leakage**: split treino/teste antes de imputação/encoding.
2. **Pipeline**: uso de `Pipeline` + `ColumnTransformer` para reprodutibilidade.
3. **Métricas**: ROC-AUC, PR-AUC, Recall/F1 de churn e matriz de confusão.
4. **Threshold**: seleção orientada a custo de retenção.
5. **Validação**: `StratifiedKFold` e controle de desbalanceamento.
6. **Deployabilidade**: serialização com `joblib` e versão de dependências.

## Entrega complementar nesta branch
Como avanço prático, foi criada uma versão de modelagem completa no notebook `Notebooks/TelecomX_BR_modelagem.ipynb` com:
- pré-processamento,
- baseline com dois modelos,
- métricas em validação cruzada e teste,
- ajuste de threshold.
