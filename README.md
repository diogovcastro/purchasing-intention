# 🛒 Quem vai comprar? Prevendo a intenção de compra em uma loja on-line

> **Trabalho Final de Mineração de Dados** · Ciência da Computação · Universidade Federal de Uberlândia (UFU)
> Disciplina de Mineração de Dados · Profa. Elaine Ribeiro Faria
> Tema: **Classificação**

De cada 100 visitantes de uma loja on-line, uns 84 entram, olham algumas páginas e vão embora. Este trabalho é sobre
os outros 16: a partir do comportamento de navegação de uma sessão, **prever se ela vai terminar em compra**.

Parece simples até você perceber que um modelo que responde "ninguém compra" acerta 84,4% das vezes, sem fazer
absolutamente nada de útil. Boa parte do trabalho foi justamente não cair nessa armadilha.

---

## 🎯 O desafio em números

| | |
|---|---|
| Base | [Online Shoppers Purchasing Intention](https://archive.ics.uci.edu/dataset/468/online+shoppers+purchasing+intention+dataset) (UCI) |
| Sessões | 12.330 (12.205 depois de remover as duplicatas) |
| Atributos | 17 preditores (10 numéricos, 7 categóricos) + a classe `Revenue` |
| Desbalanceamento | 84,5% não compram · 15,5% compram |
| Algoritmos | KNN, Random Forest, Gradient Boosting e rede neural MLP |
| Combinações de parâmetros testadas | 72 (720 treinos com validação cruzada de 10 folds) |

## 🏆 Resultado final (conjunto de teste)

| Algoritmo | F1 | PR-AUC | Precisão | Revocação |
|---|---|---|---|---|
| 🥇 **Random Forest** | **0,683** | **0,739** | **0,633** | 0,741 |
| 🥈 Gradient Boosting | 0,666 | 0,727 | 0,588 | 0,767 |
| 🥉 MLP | 0,655 | 0,710 | 0,549 | **0,814** |
| KNN | 0,535 | 0,555 | 0,403 | 0,796 |

O **Random Forest** levou: melhor F1, melhor PR-AUC e o menor número de alarmes falsos. O Gradient Boosting chegou
colado (a diferença está dentro da variação entre os folds), e o MLP é o mais "otimista": encontra mais compradores,
mas também acusa compra onde não tem. O KNN sofreu com as 47 colunas de one-hot dominando o cálculo de distância.

## 📏 Por que não a acurácia?

Porque ela mente com classes desbalanceadas. As métricas do trabalho focam em quem compra:

- **F1** (principal): só é alto se o modelo encontra os compradores **e** acerta quando diz que alguém vai comprar;
- **PR-AUC**: avalia a ordenação das sessões em todos os limiares, sem ficar inflada pelo desbalanceamento;
- **Precisão e revocação**: mostram se o modelo erra mais por alarme falso ou por comprador perdido.

A acurácia e a ROC-AUC aparecem só na justificativa, como exemplo do que **não** usar (os números estão no
notebook de classificação e no relatório).

---

## 🗺️ Mapa do repositório

```
.
├── data/
│   ├── raw/                          # base original da UCI
│   └── processed/                    # base limpa (sem duplicatas, tipos ajustados)
├── 01_analise_exploratoria/          # o que tem nesses dados?
│   ├── analise_exploratoria.ipynb
│   ├── figuras/  ·  tabelas/
├── 02_preprocessamento/              # um pipeline sob medida para cada algoritmo
│   ├── preprocessamento.ipynb
│   └── pipelines/                    # pipelines salvos (sem treino) em .joblib
├── 03_classificacao/                 # busca em grade, escolha das métricas, teste final
│   ├── classificacao.ipynb
│   ├── figuras/  ·  tabelas/
├── relatorio_latex/                  # relatório final (LaTeX + PDF)
│   ├── trabalho_final.tex
│   ├── trabalho_final.pdf
│   └── figuras/
└── Trabalho-MD.pdf                   # enunciado do trabalho
```

Cada fase mora na sua pasta numerada, com um notebook que salva as próprias figuras e tabelas. A ordem importa:
a 02 lê a base limpa e salva os pipelines que a 03 usa.

## 🧪 Como rodar

O ambiente é gerenciado pelo [uv](https://docs.astral.sh/uv/) (Python 3.13).

```bash
uv sync                 # cria o .venv com as versões travadas no uv.lock
uv run jupyter lab      # abre os notebooks
```

Ou, para executar tudo de ponta a ponta sem abrir nada:

```bash
uv run jupyter nbconvert --to notebook --execute --inplace 01_analise_exploratoria/analise_exploratoria.ipynb
uv run jupyter nbconvert --to notebook --execute --inplace 02_preprocessamento/preprocessamento.ipynb
uv run jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=3000 03_classificacao/classificacao.ipynb
```

> ☕ O notebook de classificação leva cerca de **11 minutos**, quase todos gastos pelo MLP na busca em grade.
> Todos os passos aleatórios usam a semente 42, então os números saem iguais a cada execução.

## 🔬 O que acontece em cada fase

**1. Análise exploratória.** Desbalanceamento, 125 duplicatas (todas sessões de saída imediata), categorias raras,
outliers que na verdade são informativos (quem tem valores extremos compra mais), e a descoberta de que o
`PageValues` manda no jogo: com ele acima de zero, 56,3% das sessões compram; com ele zerado, só 3,9%.

**2. Pré-processamento.** Uma parte comum a todos (remover duplicatas, padronizar `June` → `Jun`, booleanos em 0/1)
e uma parte específica para cada algoritmo:

| | KNN | Random Forest | Gradient Boosting | MLP |
|---|---|---|---|---|
| Escala | log(1 + x) + min-max | — | — | z-score |
| Categóricos | one-hot | códigos inteiros | nativos | one-hot |
| Desbalanceamento | sobreamostragem | `class_weight` | `class_weight` | sobreamostragem |

Tudo que aprende com os dados roda dentro de um `Pipeline`, ajustado só na parte de treino de cada fold. Nada de
vazamento do teste para o treino.

**3. Classificação.** Divisão estratificada 80/20, busca em grade com validação cruzada estratificada de 10 folds
**só no treino** (mesmos folds para todo mundo), escolha da melhor parametrização pelo F1 e uma única avaliação final
no teste, que ficou guardado até o fim.

## 📄 Relatório

O relatório completo está em [`relatorio_latex/trabalho_final.pdf`](relatorio_latex/trabalho_final.pdf), com a
descrição dos algoritmos (e pseudocódigo do Random Forest, do Gradient Boosting e do MLP), as tabelas comparativas,
as matrizes de confusão e a discussão dos resultados. Para compilar de novo:

```bash
cd relatorio_latex && latexmk -pdf trabalho_final.tex
```

A pasta `relatorio_latex/` é autocontida e pode ser enviada direto para o Overleaf.

## 👥 Equipe

- Cassio Lima Silva
- Diogo Vieira de Castro
- Rodrigo Otávio Fernandes Rodrigues

## 📚 Créditos

Base de dados de Sakar, C. O. e Kastro, Y., disponibilizada no UCI Machine Learning Repository e descrita em:

> SAKAR, C. O.; POLAT, S. O.; KATIRCIOGLU, M.; KASTRO, Y. Real-time prediction of online shoppers' purchasing
> intention using multilayer perceptron and LSTM recurrent neural networks. *Neural Computing and Applications*,
> v. 31, p. 6893–6908, 2019.

Feito com pandas, NumPy, Matplotlib, scikit-learn e imbalanced-learn.
