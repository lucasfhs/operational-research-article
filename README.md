# Localização capacitada de facilidades — Otimização 1

**Todo o código está em `main.ipynb`:** dados, heurísticas, solver, exportação,
auditoria, CSV e gráficos. Não há scripts Python separados nem importações locais.

```text
main.ipynb                       # único arquivo de código
src/data/
├── saiwa.parquet
├── holmberg.parquet
├── beasley.parquet
└── origens.json                 # fontes, datas, DOIs, hashes e exclusões
README.md
references.bib
Issue.pdf / OverleafTemplate/    # materiais do professor, preservados
```

## Executar no Kaggle

1. Criar um Dataset privado com **somente os quatro arquivos de `src/data/`**,
   todos juntos. Não enviar código no Dataset.
2. Importar **`main.ipynb`** e anexar o Dataset em **Add Input / Add Data**.
3. Executar as células em ordem. A pasta dos dados é localizada automaticamente;
   deve aparecer `Instâncias disponíveis: 106`. CPU basta.
4. Na configuração, habilitar apenas o que deseja executar:

| Controle | Função | Padrão |
| --- | --- | --- |
| `RUN_HEURISTICS` | Gulosa, VND, GRASP, ILS, Tabu e SA | `False` |
| `RUN_SOLVER` | Modelo LP/MIP pelo SCIP | `False` |
| `RUN_PLOTS` | Auditoria, CSV e quatro tipos de gráfico | `False` |

Para começar, habilitar **somente `RUN_HEURISTICS`**. O padrão usa `p1`, uma
semente e 5 s por método: aproximadamente 30 s de busca. É teste da plataforma,
não a avaliação científica final. Ajustar `NAMES`, `METHODS_TO_RUN`, `SEEDS`,
`SEARCH`, `LABEL` e `STAGE` para lotes posteriores.

Python 3.11+ e PyArrow são necessários. Gráficos usam Matplotlib; o solver usa
PySCIPOpt/SCIP. Se faltar alguma dependência, executar no notebook:

```python
%pip install pyarrow matplotlib
# Somente se for usar o solver:
%pip install pyscipopt==6.3.0
```

Instalações no Kaggle exigem Internet; ler os Parquet não exige downloads. Se
houver mais de um Dataset com `origens.json`, informar a pasta em `DATA_PATH`.

## Solver, gráficos e retomada no mesmo notebook

- **Solver:** habilitar `RUN_SOLVER`, definir `SOLVER_NAMES`, `SOLVER_SECONDS`,
  `LP_SECONDS` e `SOLVER_SEED`. LP/MIP têm limites separados. O ZIP guarda
  parâmetros, logs, solução, limites, gap, nós, iterações e status. Arquivos
  existentes não são sobrescritos; usar outro `LABEL` para nova execução.
- **Gráficos:** habilitar `RUN_PLOTS`. Usa resultados do lote atual ou um ZIP
  informado em `RESULTS_ZIP`. `SOLVER_ZIPS` permite fornecer referências ótimas
  anteriores. Gera qualidade, tempo, status e convergência por instância, mais CSV.
- **Retomada:** anexar o ZIP anterior e preencher `RESUME_ZIP`. Execuções completas
  com a mesma identidade são puladas; uma busca interrompida precisa recomeçar.
- **Localmente:** abrir este mesmo `main.ipynb` em Jupyter. Para só analisar, deixar
  heurísticas e solver desligados, indicar `RESULTS_ZIP` e habilitar gráficos.
  Baterias demoradas das heurísticas continuam destinadas ao Kaggle.

Com `LABEL='teste-01'`, baixar as saídas habilitadas: `teste-01.zip`,
`teste-01-solver-<instancia>.zip` e `teste-01-graficos.zip`. No Kaggle ficam em
`/kaggle/working`; localmente, em `resultados/`. **Salvar uma versão com saídas e
baixar os ZIPs**: a sessão temporária não garante persistência.

## Dados e validade experimental

Fonte: [CommaLAB — Universidade de Pisa](https://commalab.di.unipi.it/datasets/mex/),
download em 07/11/2026. A proveniência está em `src/data/origens.json`.

| Parquet | Origem | Instâncias |
| --- | --- | ---: |
| `saiwa.parquet` | Real: fábrica italiana, 23 instalações × 104 clientes | 1 |
| `holmberg.parquet` | Sintético: 71 originais, dois em quarentena | 69 |
| `beasley.parquet` | Benchmarks, incluindo 100 instalações × 1000 clientes | 36 |
| **Total utilizável** | | **106** |

`p26` e `p47` foram excluídas por inconsistências, registradas no JSON, sem correção
presumida. Vetores e matriz de custos são colunas Parquet numéricas float64;
os 106 hashes semânticos foram preservados na conversão. Os Parquet ficam
ignorados pelo Git; o JSON é versionável. A função `download_data(DATA_DIR)` pode
recriar dados ausentes localmente a partir das URLs/hashes, sem armazenar raw na
pasta; requer escrita nessa pasta e gzip para o original histórico Holmberg.
Não ocorre download automático. Conferir condições antes de publicar os dados.

Ainda é necessário revisar custos/unidades de cada família para a avaliação
final. O status `final` bloqueia dados sem revisão. Não há resultados científicos
finais coletados. Os métodos são versões próprias de referência, não reproduções
integrais dos artigos. A semente controla aleatoriedade, mas parada por relógio
não garante saída idêntica. Tempos de ambientes distintos não são speedup
controlado. Falhas e reexecuções são preservadas; não escolher só a melhor semente.

## Modelo e artigo

SSCFLP: escolher instalações com capacidade e atribuir cada cliente integralmente
a uma delas, minimizando abertura + atendimento:

$$\min \sum_i f_i y_i+\sum_i\sum_j c_{ij}x_{ij}$$
$$\sum_i x_{ij}=1;\quad \sum_j d_jx_{ij}\leq Q_i y_i;\quad x_{ij}\leq y_i;\quad x_{ij},y_i\in\{0,1\}.$$

$c_{ij}$ é custo total por cliente, sem multiplicar a demanda duas vezes. A ligação
explícita protege também demanda zero. Incumbente de timeout não é ótimo; falha
da gulosa não prova inviabilidade. LP original e limite dual final são distintos.

Roteiro: revisar fontes → separar desenvolvimento/teste → piloto e calibração →
congelar finalistas/parâmetros → repetições, comparação com solver e ablação →
artigo de **no mínimo 10 páginas no template fornecido**. Reservar, se possível,
`saiwa` como caso real. O artigo precisa de resumo/abstract, introdução,
relacionados, modelo, heurística, resultados, conclusão e referências.

Bibliografia única: `references.bib`, com 12 entradas. Modelo/dados: Holmberg,
Ahuja, Guastaroba/Speranza e Kong; GRASP/Tabu: Delmaire e Ho; ILS: Lourenço e
Almeida; SA: Kirkpatrick. Kong é preprint. Completar leituras/fichamentos antes
de atribuir detalhes técnicos: a consulta inicial foi parcial para vários textos.
