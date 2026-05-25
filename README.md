# Metodologia — Identificação de Rotação via HPS

**Autor:** Jarbas Malta  
**Instituição:** UFMT — Pós-Graduação em Ciência de Dados  
**Contexto:** TCC / Plataforma Melvin Preditiva

---

## Sumário

1. [Motivação](#1-motivação)
2. [Estrutura do repositório](#2-estrutura-do-repositório)
3. [Coleta de dados — coletor.py](#3-coleta-de-dados)
4. [Implementação do HPS](#4-implementação-do-hps)
5. [Decisões algorítmicas e validações](#5-decisões-algorítmicas-e-validações)
6. [Filtros de qualidade](#6-filtros-de-qualidade)
7. [Pipeline completo de execução](#7-pipeline-completo-de-execução)
8. [Bancada de testes controlados](#8-bancada-de-testes-controlados)
9. [Limitações identificadas](#9-limitações-identificadas)

---

## 1. Motivação

O motor de diagnóstico vibracional da plataforma Melvin depende da rotação nominal (`rotacaoNominal`) cadastrada no sistema para localizar as harmônicas de falha. Esse dado é frequentemente incorreto (erro de cadastro) ou variável (equipamentos com inversor de frequência — VFD), comprometendo todo o diagnóstico.

O **Harmonic Product Spectrum (HPS)** estima a frequência de rotação dominante diretamente do espectro FFT, sem depender de cadastro, tornando o diagnóstico autônomo.

---

## 2. Estrutura do repositório

```
TCC/
├── coletor.py            # Coleta FFTs da API Melvin/Influx e salva em JSON
├── hps_analysis.ipynb    # Notebook de análise: HPS, visualizações, tabelas
├── README.md             # Este arquivo
├── requirements.txt      # Dependências Python
└── dados/
    ├── equipamentos.json              # Catálogo de equipamentos com sensores
    ├── ffts/{serialNumber}.json       # Melhor medição selecionada por sensor (campo geral)
    └── bancada/{serialNumber}.json    # Medições da bancada de testes (22/05/2026)
```

---

## 3. Coleta de dados

### 3.1 Fonte

- **API Melvin** (`api-novo.oimelvin.com.br`): catálogo de equipamentos, tags e `rotacaoNominal`
- **API Influx** (`api-sensores-prd.oimelvin.com.br`): histórico de medições FFT por sensor, paginadas em blocos de 100

Credenciais em `src/tendencias/secrets.yaml` (chave `influxdb.api_key`).

### 3.2 Estratégia de seleção da melhor medição

O HPS opera sobre um único espectro (snapshot). Para cada sensor, o coletor:

1. Varre **todas** as medições dos últimos `--janela` dias via paginação com cursor (`temMais` / `proximoCursor`)
2. Para cada medição, calcula o **RMS na faixa de rotação útil** `[0.5×f_rot, 3×f_rot]` Hz
3. Seleciona e salva a medição com **maior RMS nessa faixa**

**Por que paginar em vez de pegar as N mais recentes?**  
Com `--max-scan 20` (padrão antigo), todos os snapshots selecionados podiam ser do mesmo horário do dia — coincidindo com o turno de descanso da máquina. Ao varrer ~900 medições (≈300/dia × 3 dias), cobrimos todos os turnos de operação.

### 3.3 Parâmetros do coletor

| Parâmetro | Padrão | Descrição |
|---|---|---|
| `--janela` | 3 | Dias de janela para busca |
| `--max-scan` | 900 | Máximo de medições varridas por sensor |
| `--max-equip` | 100 | Limite de equipamentos para coleta FFT |
| `--serie` | — | Modo série histórica (para análise temporal) |
| `--dias` | 30 | Dias de histórico no modo `--serie` |
| `--serials` | — | Lista de serialNumbers separados por vírgula — coleta apenas esses sensores (sem consultar `--max-equip`) |
| `--data-inicio` | — | Início da janela em ISO 8601 UTC; substitui `--janela`/`--dias` e ativa modo série |
| `--data-fim` | — | Fim da janela em ISO 8601 UTC (par obrigatório de `--data-inicio`) |
| `--output-dir` | `dados/ffts` | Diretório de saída para os arquivos JSON coletados |

### 3.4 Execução

```bash
# Ativar ambiente virtual
source /media/jarbas/Arquivos/CienciaDeDados/venvs/hps/bin/activate

# Coleta padrão — melhor snapshot dos últimos 3 dias
python coletor.py

# Ampliar janela
python coletor.py --janela 7

# Série histórica para análise temporal
python coletor.py --serie --dias 30

# Coleta direcionada — sensores específicos em janela temporal precisa (bancada de testes)
python coletor.py \
    --serie \
    --serials 24097747,23253979,23253865 \
    --data-inicio 2026-05-22T19:00:00Z \
    --data-fim    2026-05-22T22:00:00Z \
    --max-docs 500 \
    --output-dir dados/bancada
```

---

## 4. Implementação do HPS

### 4.1 Variante adotada: soma linear

O HPS clássico usa produto logarítmico:

```
HPS[f] = ∏ S[h·f],  h = 1..H
```

Espectros IoT são **esparsos** — a maioria dos bins é zero ou próxima de zero. Um único bin nulo zeraria todo o produto. Por isso adotamos a **variante de soma linear**:

```
HPS_soma[f] = Σ S[h·f],  h = 1..H
```

Bins nulos contribuem 0 (elemento neutro), e a coincidência harmônica em f₀ ainda produz pico proeminente. O argmax é idêntico ao do produto nos casos típicos.

### 4.2 Número de harmônicas: n_harmonics = 3

Testamos H = 5 (padrão de referências bibliográficas) e H = 3. Para espectros IoT industriais:

- **H = 5**: as compressões h=4 e h=5 frequentemente caem em bins sem energia, acumulando zeros e distorcendo o resultado
- **H = 3**: cobre 1×, 2× e 3× — as harmônicas mais confiáveis e sempre presentes em máquinas rotativas em carga

**Decisão: `n_harmonics = 3`**

### 4.3 Normalização espectral (whitening): removida

Testou-se whitening local (divisão pelo mediano em janela de ±5 Hz) antes do HPS, com a ideia de achatar o perfil 1/f e realçar picos de rotação. O resultado foi negativo:

- Picos de ruído (de amplitude absoluta baixa) ficam tão altos quanto picos reais de rotação após normalização
- O HPS opera melhor sobre amplitudes **absolutas**: em máquinas com carga suficiente, o pico de rotação domina naturalmente o espectro bruto

**Decisão: `normalizar=False` (padrão)**

A função `_normalizar_espectro` está mantida no código para uso diagnóstico opcional, mas não faz parte do pipeline padrão.

### 4.4 Pré-filtro passa-alta: F_CUTOFF = 10 Hz

Bins abaixo de 10 Hz contêm:
- Componente DC
- Vibração de estrutura (mesas, bases, pisos)
- Artefatos sub-síncronos de instalação

Esses componentes não representam rotação da máquina e produzem falsos positivos no HPS. Todos os bins com `f < 10 Hz` são zerados antes do cálculo.

**Decisão: `F_CUTOFF = 10.0 Hz`**

**Atenção (bug corrigido):** a versão inicial aplicava whitening *após* o corte, criando janelas assimétricas nos bins logo acima de 5 Hz → esses bins ficavam com valores normalizados artificialmente altos → HPS sempre apontava para o primeiro bin da faixa útil (~328 RPM). A correção: aplicar whitening primeiro (espectro completo), depois aplicar o corte.

### 4.5 Limite inferior da busca: f_min ≥ F_CUTOFF

O parâmetro `rpm_min` define o início da janela de busca do argmax. Mesmo com `rpm_min = 300 RPM (5 Hz)`, os bins abaixo de 10 Hz foram zerados pelo corte. Porém, as compressões h=2 e h=3 ainda trazem energia de frequências superiores para esses bins zerados, tornando `spec_h1[f] ≈ 0` mas `hps[f] > 0`.

Isso faz com que qualquer método baseado em divisão por h=1 (como normalização por espectro) exploda nesses bins. A correção:

```python
f_min = max(rpm_min / 60.0, f_cutoff_hz)
```

**Decisão: a busca nunca começa abaixo de F_CUTOFF.**

---

## 5. Decisões algorítmicas e validações

### 5.1 Correção de oitava por razão de amplitude

**Problema identificado:** quando há ruído estrutural de baixa frequência em `f₀/2`, o HPS em `f₀/2` recebe:
- h=1: energia estrutural (não-rotacional)
- h=2: pico de rotação real em `f₀` mapeado para `f₀/2`

Essa soma pode superar o HPS em `f₀`, fazendo o método identificar o **sub-harmônico** em vez do fundamental.

**Exemplo observado:** sensor YBR-ST05-RASP (1715 RPM cadastrado). Sem correção: estimativa 891 RPM (metade de 1782 RPM). Causa: ruído estrutural em ~14 Hz + pico de rotação em 28.6 Hz mapeado para 14.3 Hz via h=2.

**Solução implementada — correção de oitava por razão espectral:**

Após o argmax, verifica-se se o candidato é um sub-harmônico comparando a amplitude espectral do candidato com a do seu dobro:

```python
for mult in [2, 3]:
    f_mult  = freqs_hps[idx_peak] * mult
    amp_cand = spec[idx_peak]
    amp_mult = spec[idx_mult]
    if amp_mult / amp_cand >= 2.0:
        idx_peak = idx_mult  # promove para o dobro
        break
```

**Lógica:** se `spec[2f]` tem amplitude ≥ 2× a de `spec[f]`, então `f` provavelmente não é o fundamental — é um sub-harmônico com ruído. O fundamental real está em `2f`.

**Limiar 2.0:** conservador o suficiente para não promover casos ambíguos, mas suficiente para capturar ruído estrutural fraco vs. pico de rotação forte.

### 5.2 Abordagens testadas e descartadas

#### Normalização HPS por h=1 (descartada)

Tentou-se `hps_norm[f] = hps[f] / (spec[f] + ε)` para penalizar frequências onde apenas h=1 contribui. Falhou: espectros esparsos têm `spec[f] ≈ 0` na maioria dos bins → `hps_norm` explode para valores da ordem de bilhões nesses bins → seleção completamente errada.

#### Busca por h≥2 (hps_h2plus, descartada)

Tentou-se usar apenas as compressões h=2 e h=3 para a busca, ignorando h=1. O objetivo era penalizar picos isolados de h=1 (harmônico 2× dominante). Problema: para espectros com múltiplos picos densos (ex.: sensor YBR-ST05-EXST03), bins em frequências ainda menores recebem contribuições de h=2 e h=3 de múltiplos picos → a seleção migra para frequências ainda mais baixas (~800 RPM) do que o argmax original.

#### Correção de oitava via h2plus (descartada)

Tentou-se substituir o critério de promoção por `hps_h2plus[2f] ≥ 0.70 × hps_h2plus[f]`. Não resolve o caso de sub-harmônico de máquina com espectro limpo (1× dominante), pois `hps_h2plus[2f]` é quase zero quando não há 4× ou 6× harmônicos.

### 5.3 Caso não resolvido: harmônico 2× dominante

**Sensor YBR-ST05-EXST03:** espectro com picos em ~13, 20, 27, 40, 47, 57 Hz. O usuário identificou visualmente ~20 Hz como o fundamental (série harmônica 20→40→60 Hz). Porém o HPS estima 40 Hz (2×) porque:
- `spec[40]` é o pico de maior amplitude (2× harmônico dominante — padrão típico de desalinhamento)
- `hps[40]` supera `hps[20]` no argmax

Nenhuma variação testada do HPS conseguiu resolver esse caso sem degradar outros. Documentado como **limitação do método**: quando o 2× harmônico domina o espectro, o HPS pode encontrar o 2× em vez do fundamental. Isso ocorre porque a amplitude absoluta do 2× supera a soma harmônica em torno do fundamental.

---

## 6. Filtros de qualidade

### 6.1 Filtro de atividade — VEL_PICO_MINIMO

Sensores em máquinas paradas (entressafra, manutenção) produzem apenas ruído 1/f. Esses espectros não contêm informação de rotação e contaminam as estatísticas de concordância.

**Critério de exclusão:** sensor descartado se o pico máximo do espectro **acima de F_CUTOFF** for menor que `VEL_PICO_MINIMO = 2.0 mm/s`.

```python
mask_util = freqs >= F_CUTOFF
if espectro[mask_util].max() < VEL_PICO_MINIMO:
    return None  # descarta
```

**Por que verificar acima do corte?** O pico global `espectro.max()` é sempre o bin de menor frequência (perfil 1/f tem máximo no DC). Uma máquina parada pode ter `espectro.max() >> 2 mm/s` por conta do componente DC/sub-síncrono, mas zero energia útil acima de 10 Hz.

### 6.2 Filtro de RPM cadastrado

Na análise multi-equipamento, sensores com `rotacaoNominal = 0` ou `NaN` são excluídos do cálculo de concordância (não há referência para calcular erro relativo).

### 6.3 Constantes configuráveis

```python
F_CUTOFF        = 10.0   # Hz — corte passa-alta
F_PLOT_MIN      = 10.0   # Hz — início dos gráficos
VEL_PICO_MINIMO = 2.0    # mm/s — limiar de atividade da máquina
RMS_MINIMO      = 0.05   # mm/s — aviso de sinal fraco no título do gráfico
```

---

## 7. Pipeline completo de execução

```
1. Coletar dados
   └─ python coletor.py --janela 3 --max-scan 900 --max-equip 100
      ├─ Busca equipamentos na API Melvin
      ├─ Para cada sensor: pagina API Influx, seleciona melhor RMS
      └─ Salva dados/equipamentos.json + dados/ffts/{serial}.json

2. Abrir notebook
   └─ jupyter lab hps_analysis.ipynb
      (ou VS Code com extensão Jupyter)

3. Executar células em ordem:
   ├─ cell0003  — importações e paleta de cores
   ├─ cell0007  — geração do sinal sintético
   ├─ cell0009  — definição de estimar_rpm_hps (algoritmo HPS completo)
   ├─ cell0010  — validação com sinal sintético (erro: 0.00%)
   ├─ cell0012  — carregamento dos dados coletados
   ├─ cell0015  — funções auxiliares de extração
   ├─ cell0017  — funções de análise e plot individual
   ├─ cell0018  — análise de sensor individual (configurar SERIAL_SELECIONADO)
   ├─ cell0021  — loop multi-sensor (aplica filtros + HPS em todos)
   ├─ cell0022  — tabela de concordância
   ├─ cell0023  — gráfico RPM cadastrado × estimado
   └─ cell0024  — distribuição do erro relativo
```

---

## 8. Bancada de testes controlados

### 8.1 Configuração

Em 22/05/2026, três sensores Melvin foram instalados em equipamentos de características conhecidas para validação controlada:

| Serial | Equipamento | Tipo | RPM nominal | Observações |
|---|---|---|---|---|
| **24097747** | Motor WEG W22 IR3 Premium | Motor elétrico | 1780 | 0,37 kW, 4 polos, 60 Hz; rolamentos 6202 ZZ; com inversor Allen-Bradley PowerFlex 40 |
| **23253979** | Bomba piscina | Bomba centrífuga | 1780 | Velocidade fixa; sem inversor |
| **23253865** | Exaustor F02-531-MB001 | Ventilador | — (não cadastrado) | Velocidade fixa; RPM real desconhecido |

O motor estava conectado a um inversor de frequência configurado em **39,3 Hz** e **60,0 Hz** em diferentes momentos do teste (observado no display do VFD). Isso equivale a rotações teóricas de ~2358 RPM (a 39,3 Hz) e ~1700 RPM (a 60 Hz, com deslizamento típico para 4 polos).

Janela de operação planejada: **22/05/2026 16:00–19:00 BRT (19:00–22:00 UTC).**

### 8.2 Dados coletados

A coleta foi feita com os novos parâmetros `--serials` e `--data-inicio`/`--data-fim` do coletor:

```bash
python coletor.py \
    --serie \
    --serials 24097747,23253979,23253865 \
    --data-inicio 2026-05-22T19:00:00Z \
    --data-fim    2026-05-22T22:00:00Z \
    --max-docs 500 \
    --output-dir dados/bancada
```

Resultados da coleta na janela de teste:

| Serial | Docs coletados | Observação |
|---|---|---|
| 24097747 | 0 | Sensor sem transmissões na janela; primeiro dado registrado a partir de 23/05/2026 02:14 UTC |
| 23253979 | 3 | Baixa frequência de amostragem; dados presentes na janela |
| 23253865 | 88 | Sensor ativo; cobertura completa da janela |

### 8.3 Análise HPS — resultados

O filtro `VEL_PICO_MINIMO = 2,0 mm/s` foi aplicado verificando o pico **acima de F_CUTOFF = 10 Hz**:

| Serial | Equipamento | Docs aprovados | Pico > 10 Hz | RPM estimado (HPS) | Desvio padrão | Confiança |
|---|---|---|---|---|---|---|
| 24097747 | Motor WEG + VFD | — | n/a (sem dados) | n/a | — | — |
| 23253979 | Bomba piscina | 0/3 | 0,97 mm/s máx | — (reprovado) | — | — |
| 23253865 | Exaustor | 80/88 | 6,27 mm/s | **2671,9 RPM (44,53 Hz)** | 0,0 RPM | 7,14 |

### 8.4 Interpretação

**Motor (24097747):** O sensor não transmitiu dados durante a janela de teste planejada. Dados posteriores (23–24/05) mostram perfil 1/f puro, com pico global em 0,781 Hz e energia acima de 10 Hz inferior a 0,12 mm/s. Isso indica que o motor estava operando em carga muito leve — insuficiente para gerar vibração rotacional detectável pelo sensor. O filtro de atividade funciona corretamente: rejeita o sensor quando não há informação de rotação utilizável.

**Bomba piscina (23253979):** Apenas 3 medições na janela. O maior pico acima de 10 Hz foi 0,97 mm/s — abaixo do limiar. Perfil com picos em ~172 Hz e ~409 Hz, sem harmônica clara de rotação identificável. O sensor pode estar em posição de montagem desfavorável (acoplamento de vibração fraco) ou a bomba gera vibração predominantemente abaixo de 10 Hz.

**Exaustor (23253865):** O resultado mais expressivo do teste. O HPS identificou o mesmo pico fundamental (44,531 Hz = 2671,9 RPM) em **100% dos 80 documentos aprovados**, com desvio padrão de 0 RPM. Isso demonstra a consistência e repetibilidade do algoritmo em condições reais de operação estacionária. A estrutura harmônica do espectro é: 1× = 44,53 Hz (pico dominante, 6,3 mm/s) e 4× = 178,1 Hz (segundo pico, 1,1 mm/s), padrão típico de ventilador com 4 pás (frequência de passagem de pá = 4× a rotação).

### 8.5 Parâmetros do coletor para coleta direcionada

A versão atual do coletor suporta coleta direcionada por serial e janela temporal precisa:

| Parâmetro | Descrição |
|---|---|
| `--serials` | Lista de serialNumbers separados por vírgula |
| `--data-inicio` | Início em ISO 8601 UTC (ex: `2026-05-22T19:00:00Z`) |
| `--data-fim` | Fim em ISO 8601 UTC |
| `--output-dir` | Diretório de saída (padrão: `dados/ffts`) |

Quando `--data-inicio` e `--data-fim` são fornecidos, substituem `--janela`/`--dias` e o modo série é ativado automaticamente.

---

## 9. Limitações identificadas

| Limitação | Descrição | Impacto |
|---|---|---|
| Ambiguidade de oitava (f₀/2) | Ruído estrutural em f₀/2 + mapeamento h=2 pode superar o fundamental | Parcialmente corrigida pela razão de amplitude |
| 2× harmônico dominante | Quando o 2× tem maior amplitude que o 1×, HPS encontra 2×f₀ | Caso não resolvido; documentado como limitação |
| Resolução espectral | ΔF ≈ 0.78 Hz → incerteza de ±0.39 Hz = ±23 RPM a 1800 RPM | Inerente ao hardware do sensor |
| Espectro esparso | >80% dos bins em zero; métodos baseados em divisão por h=1 explodem | Motivo da adoção da soma linear |
| Entressafra / máquinas paradas | Maioria dos sensores instalados pode estar sem operação | Mitigado pelo filtro VEL_PICO_MINIMO |
| VFD sem telemetria | Rotação real diverge do cadastro; HPS encontra a real, cadastro está errado | Funcionalidade desejada — HPS é a solução |

---

## Referências

- Noll, A. M. (1967). *Cepstrum Pitch Determination.* JASA.
- Schroeder, M. R. (1968). *Period Histogram and Product Spectrum.* JASA.
- Randall, R. B. (2011). *Vibration-based Condition Monitoring.* Wiley.
- ISO 10816-3:2009 — *Mechanical Vibration — Evaluation of Machine Vibration.*
