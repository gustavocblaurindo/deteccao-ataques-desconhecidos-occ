# Detecção de tráfego malicioso desconhecido com OCC

Este projeto investiga se é possível detectar ataques de rede que o modelo nunca viu durante o treino.

Para isso, são comparados modelos supervisionados (Regressão Logística, Random Forest e XGBoost), que aprendem com exemplos de ataque, e modelos de classificação de uma classe, o One-Class SVM e o Isolation Forest, que veem apenas tráfego normal no treino e tratam como suspeito tudo o que foge desse padrão. Ao final, é avaliado se combinar os dois tipos de modelo (ensemble) traz algum ganho.

## Os dados

O dataset utilizado é o NF-UNSW-NB15-v2, uma versão em NetFlow do UNSW-NB15, com cerca de 2,39 milhões de fluxos e 9 tipos de ataque além do tráfego benigno. O arquivo é grande e não está incluído no repositório: ele deve ser baixado e colocado em `dataset/NF-UNSW-NB15-v2.csv`.

O pré-processamento consiste em:

- remover as colunas que identificam origem e destino da comunicação (IPs, porta de origem e ID de DNS), para que o modelo aprenda o comportamento do tráfego e não decore endereços;
- checar nulos, infinitos e colunas constantes (nenhum caso encontrado);
- deduplicar os dados. Esse foi o passo que mais alterou a base: cerca de 70% dos fluxos eram idênticos a algum outro. Foram removidas as duplicatas e também os fluxos com atributos iguais e rótulos diferentes, tudo antes da divisão entre treino e teste, para não haver vazamento de cópias de um lado para o outro.

Como consequência, algumas classes pequenas encolheram bastante (Reconnaissance perdeu 98%, por exemplo). Isso é uma limitação importante na leitura dos resultados.

## Desenho do experimento

No split principal (o "A"), quatro tipos de ataque são considerados conhecidos e entram no treino: Exploits, Fuzzers, Generic e DoS. Os outros cinco (Reconnaissance, Shellcode, Backdoor, Analysis e Worms) são os desconhecidos e aparecem somente no teste. Nem a validação contém esses ataques, de modo que limiares e hiperparâmetros não usam nenhuma informação sobre o que deveria ser novidade.

| Conjunto | Benigno | Ataques conhecidos | Ataques desconhecidos |
|---|---|---|---|
| Treino | 414.333 | 20.360 | 0 |
| Validação | 138.111 | 6.787 | 0 |
| Teste | 138.111 | 6.787 | 2.251 |

Todos os modelos são comparados no mesmo ponto de operação, **1% de falsos positivos**, com o limiar escolhido no benigno da validação. As métricas por categoria incluem intervalo de confiança de 95% calculado por bootstrap.

Além do split A, o notebook traz outros experimentos:

- ablação de atributos (remoção de TTL, janela TCP e porta);
- leave-one-attack-out, em que cada tipo de ataque é escondido do XGBoost, um de cada vez;
- split B, que esconde justamente Exploits e Fuzzers, as categorias em que o XGBoost mais errava;
- cascata, uma alternativa à regra OU no ensemble;
- LOKO, outra forma de escolher os hiperparâmetros dos modelos OCC;
- curva que mostra quantos ataques rotulados o XGBoost precisa para superar o OCSVM.

## Resultados

Recall nos ataques desconhecidos, a 1% de falsos positivos (split A):

| Modelo | Recall |
|---|---|
| XGBoost | 0,985 |
| One-Class SVM | 0,761 |
| Isolation Forest | 0,086 |

Em resumo:

- Detectar ataques inéditos é possível, mas depende do tipo de ataque. Shellcode é fácil de separar do benigno. No split B, o recall do XGBoost em Fuzzers cai para 0,764 e o do OCSVM cai para 0,524.
- O ensemble não ajudou. Com o FPR total travado em 1%, o saldo de ataques detectados ficou negativo em todas as versões: −1.248 (OCSVM) e −966 (IF) na regra OU, e −2.450 (OCSVM) e −1.821 (IF) na cascata.
- O Isolation Forest é muito sensível ao pré-processamento. Apesar de ter AUC alta nos desconhecidos, seu recall é muito baixo quando o limiar fica em 1% de FPR.
- O OCSVM vai mal em Analysis (recall de 0,048), categoria que o XGBoost detecta com 0,9375 no leave-one-attack-out.
- Com cerca de 100 ataques rotulados no treino, o XGBoost já supera o OCSVM.
- A ablação produziu resultados confusos (remover a porta chegou a melhorar levemente o recall), então não é possível afirmar nada sobre a importância real de TTL, janela ou porta. Os modelos em árvore toleraram melhor a remoção de atributos do que a Regressão Logística.

Observação: algumas classes desconhecidas têm pouquíssimos exemplos no teste (apenas 45 fluxos de Worms), a deduplicação alterou a proporção natural dos ataques e todos os resultados vêm de um único dataset.

## Estrutura da pasta

```
v3/
├── attack-detectionv3.ipynb      notebook com o pipeline completo
├── scores_supervisionado.npz     scores de LogReg, RF e XGBoost (validação e teste)
├── resultados_fpr_grid.csv       recall por FPR (de 0,1% a 5%)
├── resultados_splitb.csv         métricas do split B
├── resultados_labels.csv         curva de eficiência de rótulos
├── R_cascata.csv                 resultados da cascata por tipo de ataque
└── imagens/                      gráficos gerados pelo notebook
```

Ao ser executado, o notebook também cria `nf_unsw_v2_clean.parquet`, os dois splits em parquet (`nf_unsw_v2_split_A.parquet` e `nf_unsw_v2_split_B.parquet`) e os arquivos `loko_if.csv` e `loko_occ.csv`. Nada disso está incluído no repositório.

## Como executar

São necessários Python 3.10 ou superior e as bibliotecas abaixo:

```bash
pip install pandas numpy scikit-learn xgboost matplotlib pyarrow jupyter
```

Em seguida, basta colocar o dataset em `dataset/`, abrir o notebook e executar as células **na ordem**, já que várias dependem de variáveis criadas anteriormente (a cascata, por exemplo, usa os modelos OCC e os resultados do leave-one-attack-out). O notebook localiza a raiz do projeto automaticamente e, no Google Colab, monta o Drive e procura em `MyDrive/AM2/dataset/`.

O One-Class SVM é a etapa mais lenta. Por isso, a busca de hiperparâmetros usa 30 mil fluxos benignos e o modelo final usa 120 mil dos 414 mil disponíveis. A semente aleatória é 42 em todos os experimentos.
