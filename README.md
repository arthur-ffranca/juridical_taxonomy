# ZELLU — Documentação do DE-PARA de movimentações processuais

Este documento descreve, passo a passo, o trabalho realizado para construir uma inteligência de **DE-PARA** de movimentações entre tribunais brasileiros: fontes usadas, papel do **JEV**, equivalências, cobertura dos 91 tribunais, interface, esboço de **início → fim** e o que foi obtido ao final.

---

## 1. Objetivo

Responder, de forma operacional:

1. **Duas movimentações em tribunais distintos representam a mesma função processual?**
2. **Quais códigos TPU (CNJ) são compartilhados entre quais tribunais?**
3. **Onde a nomenclatura diverge** (mesmo sentido, nomes ou códigos diferentes)?
4. **Como esboçar início e fim** típicos do fluxo, com equivalentes cross-tribunal?

O produto final é um app web (consulta A→B, tabela, grafo, fluxo início→fim) mais artefatos JSON/CSV consultáveis.

---

## 2. Fontes de dados

| Fonte | Papel |
|---|---|
| **DataJud (CNJ)** | Processos e movimentos observados nos tribunais (códigos, nomes, datas, complementos). |
| **TPU / SGT (CNJ)** | Taxonomia oficial de movimentos (`codigo`, `nome`, hierarquia/`caminho`, glossário). |
| **JEV** | Modelo/serviço de julgamento de equivalência semântica entre pares (com e sem contexto). |

A coleta ampla DataJud cobriu **91 tribunais** (Justiça Estadual, Federal, Trabalho, Eleitoral, Superiores, etc.), com páginas JSON por tribunal em `zellu-benchmark/coleta_ampla/paginas/`.

A TPU local está em `zellu-benchmark/data/tpu_movimentos.json` (~911 itens).

---

## 3. Pipeline — passos realizados

### Passo A — Coleta e indexação

1. Consultas à API DataJud por alias de tribunal (incluindo TREs no formato `tre-{uf}`).
2. Persistência paginada (`pagina-*.json`) com processos, movimentos (`codigo`, `nome`, `dataHora`, `complementosTabelados`).
3. Agregação por tribunal: quais códigos TPU aparecem, com que frequência e com que nome local.
4. Enriquecimento TPU: caminho hierárquico oficial (ex.: `Serventuário › Distribuidor › Distribuição`).

**Cuidados técnicos relevantes:**

- Paginação/ordenação da API (cursores, datas nulas em alguns tribunais como TJMG).
- Aliases corretos dos TREs (`tre-sp`, não só `sp`).

### Passo B — DE-PARA determinístico (mesmo código TPU)

Regra central:

> Se o **mesmo código TPU** aparece em dois tribunais, isso é equivalência de alta confiança (mesma entrada da tabela unificada do CNJ).

A partir disso geramos:

- Lista de **conceitos compartilhados** (código presente em ≥ 2 tribunais).
- Matriz de arestas **par de tribunais × código**.
- Relatório de cobertura: quantos pares de tribunais têm pelo menos uma equivalência.

**Resultado principal deste passo:**

| Métrica | Valor |
|---|---:|
| Tribunais indexados | **91** |
| Códigos TPU observados | **557** |
| Conceitos em ≥ 2 tribunais | **458** |
| Arestas par×código | **192.674** |
| Pares de tribunais com ≥ 1 eq. | **4.095 / 4.095 (100%)** |
| Conceitos em ≥ 50 tribunais | **55** |

Arquivos:

- `resultados_jev_v4/relatorio_equivalencias_91.json` (+ `.md`)
- `resultados_jev_v4/equivalencias_tabela.json`
- App: `apps/taxonomia-grafo/public/equivalencias_tabela.json`

### Passo C — Casamento / consolidação do grafo DE-PARA

Scripts da linha V4 (`casar_depara_v4.py`, `consolidar_depara_v4.py`, etc.) unificaram:

- Observações DataJud ↔ nós de conceito TPU.
- Arestas `observa` (tribunal → conceito).
- Relações auxiliares (ex.: antônimos heurísticos — com correção de falso positivo por hífen).

Isso alimentou o grafo interativo (sigma.js / graphology).

### Passo D — Ataque a não-equivalentes e rematch

Nem todo par “parecido” é equivalente. Havia:

- Hard negatives (códigos/nomes próximos, funções distintas).
- Ausências pontuais (código visto em um lado, faltando no outro).

Ações:

1. **`atacar_nao_equivalentes.py`** — segundo passe sobre pares marcados como não equivalentes.
2. **Cobertura dos tribunais faltantes** — scripts `cobrir_9_tribunais.py` e `cobrir_tres.py` (incl. TREs).
3. **Rematch** dos hard negatives TRE (81 casos) e lote de não-equivalentes (~180).

Resultados típicos do rematch (consolidados no meta do grafo):

- Rematch ~180 casos: maioria reconciliada nos dois lados; pouquíssimas ausências abertas.
- Rematch TRE 81: alta taxa de recuperação.
- Absências residuais documentadas (ex.: códigos sem par em TJPE / TJAP / TREBA, conforme CSV de ausentes).

### Passo E — JEV sobre nomenclatura divergente

Além do “mesmo código”, há casos em que:

- os **nomes são parecidos / mesma função**, mas os **códigos diferem**; ou
- o contexto processual muda a leitura do par.

Aqui entra o **JEV** (detalhe na seção 4).

Scripts:

- `executar_jev_v4.py` / gerações V2–V4 (benchmarks controlados).
- `nomenclatura_divergente_jev.py` — amostra de pares nome-similar / código-distinto.
- `indice_depara_consulta.py` — índice A→B com confiança.

### Passo F — Índice consultável A→B

Gerado `depara_consulta.json` com:

- Movimentos por tribunal.
- Mapa de códigos → tribunais.
- Links JEV (incluindo **código distinto**).
- Metadados de fonte (`mesmo_codigo_tpu` vs `jev`).

Meta do índice:

| Campo | Valor |
|---|---:|
| Tribunais | 91 |
| Conceitos | 557 |
| Compartilhados | 458 |
| Links JEV | 508 |
| Links JEV com código distinto | **23** |

### Passo G — Interface web

App Next.js em `zellu-benchmark/apps/taxonomia-grafo`:

| Aba | Função |
|---|---|
| **Consultar DE-PARA** | Escolhe tribunal origem/destino + movimento; lista equivalentes com confiança. |
| **Tabela** | Filtra conceitos compartilhados por código/nome/cobertura. |
| **Fluxo início→fim** | Esboço empírico do primeiro/último movimento + equivalentes. |
| **Grafo** | Taxonomia interativa (conceitos × tribunais). |

Export estático (`output: 'export'`) para GitHub Pages com `basePath` `/juridical_taxonomy`.

Pacote pronto para upload manual: pasta `github-pages-upload/` e zip `zellu-github-pages.zip`.

### Passo H — Esboço início → fim

Pedido adicional: esboçar **início e fim** das movimentações e equivalentes entre tribunais (inclusive nomenclaturas diferentes).

Script: `ZELLU_BENCHMARK_JEV_V4/esbocar_fluxo_inicio_fim.py`

Método:

1. Amostra até 3 páginas × ~100 processos por tribunal (91 tribunais).
2. Ordena movimentos por `dataHora`.
3. Conta **primeiro** e **último** movimento de cada processo.
4. Cruza com TPU, DE-PARA (mesmo código), complementos locais e `jevDiff`.

Amostra obtida:

- **19.200** processos lidos  
- **18.734** com movimentos  
- **91** tribunais  

**Inícios mais comuns (amostra):**

| Código | Nome | Processos (1º movimento) |
|---:|---|---:|
| 26 | Distribuição | 13.622 |
| 85 | Petição | 693 |
| 132 | Recebimento | 597 |
| 981 | Recebimento | 539 |
| 51 | Conclusão | 516 |

**Fins mais comuns (amostra):**

| Código | Nome | Processos (último movimento) |
|---:|---|---:|
| 246 | Definitivo | 5.208 |
| 22 | Baixa Definitiva | 4.649 |
| 123 | Remessa | 1.700 |
| 85 | Petição | 979 |
| 11383 | Ato ordinatório | 698 |

**Arcos início→fim frequentes:**

- `26 Distribuição` → `246 Definitivo`
- `26 Distribuição` → `22 Baixa Definitiva`
- `26 Distribuição` → `123 Remessa`

**Aliases funcionais (mesmo nome, códigos distintos):**

- **Recebimento**: códigos `132` e `981` (ambos aparecem como início em tribunais diferentes).
- **Remessa**: códigos `123` / `982` (papéis empíricos distintos na amostra).

**Nomenclatura operacional:** no mesmo código TPU, os `complementosTabelados` (ex.: tipo de distribuição, tipo de documento) variam — isso captura “rótulos locais” sem mudar o código canônico.

Artefato: `fluxo_inicio_fim.json` (também em `apps/taxonomia-grafo/public/`).

> Aviso: o esboço é **estatístico**, não norma. “Início/fim” = primeiro/último evento na linha do tempo DataJud da amostra, não necessariamente a classificação jurídica formal do ato.

---

## 4. Papel do JEV

### O que é

**JEV** (serviço/modelo de julgamento de equivalência) avalia se dois movimentos — descritos por nome, código, tribunal e, quando houver, contexto — são **equivalentes na prática processual**.

Ele não substitui a TPU; ele **complementa** o casamento quando o código sozinho não basta.

### Quando usamos JEV

| Situação | Sem JEV | Com JEV |
|---|---|---|
| Mesmo código TPU em A e B | Equivalência determinística (confiança alta) | Confirmação opcional / contexto |
| Nomes parecidos, **códigos diferentes** | Fica ambíguo ou perdido | Classifica equivalente / não equivalente |
| Hard negatives | Risco de falso positivo por similaridade lexical | Distingue antônimos / funções opostas |
| Sequências / contexto | Só bag-of-codes | Julga pares no fluxo |

### Experimentos V4 (benchmark controlado)

Na linha `ZELLU_BENCHMARK_JEV_V4` (amostra histórica documentada no README V4):

| Experimento | equivalente | nao_equivalente | inconclusivo |
|---|---:|---:|---:|
| Nomenclatura (positive_clear) | 160 | 0 | 0 |
| Sequências | 120 | 0 | 0 |
| Hard negatives | 1* | 79 | 0 |
| Contexto | 17 | 101 | 2 |

\* falso hard-negative ligado a hifenização / rótulo de bucket (corrigido na heurística).

Modos tipicamente usados: **com contexto** e **sem contexto** (centenas de chamadas por lote).

### Nomenclatura divergente (pós-matriz 91)

Amostra de pares com similaridade de nome / mesmo ramo TPU, mas código distinto. Exemplos de equivalentes encontrados:

- TRT24 `241` “Conhecimento em Parte e Provimento em Parte…” ↔ TJPR `240` “Conhecimento em Parte e Provimento ou Concessão”
- STJ `11796` ↔ TJMG `900` — “Declaração de competência em conflito” (códigos diferentes, mesmo rótulo)
- Pares “Provimento em Parte” ↔ formulação composta no TJPR
- TRERJ `12648` “Determinação de Diligência” ↔ TJRJ `1013` “Determinação”

No índice final: **23** links JEV com **código distinto** entram na consulta A→B com confiança média (frente à confiança alta do mesmo código TPU).

### Como a confiança aparece na UI

| Confiança | Critério |
|---|---|
| **Alta** | Mesmo código TPU observado nos dois tribunais |
| **Média** | JEV marcou equivalente (nomenclatura/contexto), possivelmente código distinto |
| **Baixa / ambíguo** | Bucket JEV ambíguo ou evidência parcial |

---

## 5. O que foi obtido (entrega)

### Artefatos de dados

| Artefato | Conteúdo |
|---|---|
| `relatorio_equivalencias_91.*` | Cobertura 91 tribunais, histogramas, tops |
| `equivalencias_tabela.json` | Conceitos compartilhados filtráveis |
| `depara_consulta.json` | Índice A→B + `jevDiff` |
| `taxonomia_grafo.json` | Nós/arestas para o grafo |
| `fluxo_inicio_fim.json` | Esboço início/fim + aliases + complementos |
| `rematches_nao_equivalentes.*` / ausentes | Diagnóstico de buracos |
| CSVs JEV V4 | equivalentes / não equivalentes com contexto |

### Aplicação

- Next.js + TypeScript + Tailwind  
- Abas: Consultar · Tabela · Fluxo · Grafo  
- Dev local: porta **3467** (`npm run dev`)  
- Build estático com `GITHUB_PAGES=true` e `GITHUB_REPOSITORY=arthur-ffranca/juridical_taxonomy`

### Publicação

- Repo alvo: https://github.com/arthur-ffranca/juridical_taxonomy  
- Site alvo: https://arthur-ffranca.github.io/juridical_taxonomy/  
- Pacote para upload manual: `github-pages-upload/` (inclui `.nojekyll` e `_next`)  
- Script opcional com token: `scripts/publish-github-pages.sh`

### Números-resumo

1. **91/91** tribunais no índice.  
2. **458** conceitos compartilhados; **100%** dos pares de tribunais com ≥ 1 equivalência.  
3. **192.674** arestas determinísticas par×código.  
4. **508** links JEV no índice; **23** com código distinto.  
5. Esboço fluxo: **~18,7k** processos; início dominante **Distribuição (26)**; fins dominantes **Definitivo (246)** e **Baixa Definitiva (22)**.

---

## 6. Limitações (importantes)

1. **DataJud ≠ espelho completo** de todos os atos internos de cada sistema processual.  
2. **Início/fim empíricos** dependem do que foi publicado e da amostra (3 páginas/tribunal no esboço).  
3. **Mesmo nome oficial** em muitos códigos (a TPU já unifica rótulos); a divergência “rica” aparece sobretudo em **complementos**, **códigos distintos** e pares **JEV**.  
4. **JEV** é julgamento estatístico/modelo — pares ambíguos devem ser revisados juridicamente.  
5. Publicação no GitHub Pages depende de push autenticado no repo do usuário (token/`GH_TOKEN` ou upload manual do zip).

---

## 7. Como reproduzir os passos principais

```bash
# 1) Matriz de equivalências 91 tribunais
cd zellu-benchmark/ZELLU_BENCHMARK_JEV_V4
python3 matriz_equivalencias_91.py

# 2) Índice DE-PARA consultável
python3 indice_depara_consulta.py

# 3) Nomenclatura divergente + JEV (requer chave/API do JEV)
python3 nomenclatura_divergente_jev.py

# 4) Esboço início → fim
python3 esbocar_fluxo_inicio_fim.py

# 5) App
cd ../apps/taxonomia-grafo
npm install
npm run dev   # http://0.0.0.0:3467
# Atalho: http://127.0.0.1:3467/#fluxo
```

Variáveis típicas de ambiente na coleta/JEV: `KEY` / `DATAJUD_KEY`, e a chave do JEV conforme `.env` das pastas V2–V4.

---

## 8. Mapa rápido de pastas

```
zellu-benchmark/
  coleta_ampla/paginas/          # DataJud bruto
  data/tpu_movimentos.json       # TPU
  ZELLU_BENCHMARK_JEV_V4/        # pipeline DE-PARA + JEV + fluxo
    resultados_jev_v4/           # saídas
  apps/taxonomia-grafo/          # UI Next.js
github-pages-upload/             # site estático pronto para o Pages
scripts/publish-github-pages.sh  # publish com GH_TOKEN
DOCUMENTACAO_ZELLU_DEPARA.md     # este arquivo
```

---

## 9. Conclusão

Construímos um DE-PARA em camadas:

1. **Âncora determinística** — mesmo código TPU (base confiável e mensurável).  
2. **Camada JEV** — recupera equivalências onde nomenclatura/contexto divergem do código.  
3. **Cobertura nacional** — 91 tribunais, pares 100% cobertos por ao menos um conceito compartilhado.  
4. **Produto consultável** — lookup, tabela, grafo e esboço início→fim.  

O JEV não inventa a taxonomia: ele **liga o que a TPU e a coleta não alinhariam só pelo código**, especialmente nomenclaturas locais e pares de códigos distintos com a mesma função prática.
