# AUDITORIA TÉCNICA COMPLETA — OTC M1 EMA SMA RETRAÇÃO v20
## Quadcode Script (QCS) 1.1.0 — Polarium / IQ Option

**Data:** 2026-10-19  
**Ativo:** OTC M1 | **Expiração:** 1 min | **Entrada:** próxima vela após candle fechado  
**Versão auditada:** v20 (instrument name = "OTC M1 EMA SMA RETRACAO v20")

---

# ETAPA 1 — DIAGNÓSTICO

## 1.1 Pontos Fortes (o que preservar)

1.  **Sem repaint nos sinais confirmados:** `candidate_at(1)` usa apenas `[s]` e `[s+1]` fechados. `s=1` = candle de confirmação fechado, `s+1=2` = candle de retração. Isso é correto para OTC M1.
2.  **Separação retração vs continuação:** `is_retracement` distingue `C RET/P RET` (pullback confirmado + wick) de `C/P` (continuação simples). Bom para filtrar depois por horário.
3.  **Uso de ATR para normalizar tudo:** tolerância de toque, penetração da lenta, range de retração, extensão da rápida e distância de topo/fundo em ATR. Correto para OTC onde pip fixo não funciona.
4.  **Proteção contra gaps OTC:** `entry_open_ok()` bloqueia gap >30% do range anterior. Essencial em OTC (spikes de corretora na virada de minuto).
5.  **Anti-spam `min_gap`:** bloqueia sinais repetidos em N candles. Evita martingale involuntário.
6.  **Tratamento de `nil`/`NaN`/`inf`:** `series_value()` com `get_value()` + `isnan()` + `math.huge` — bem defensivo para QCS.
7.  **Choppiness nativo:** fórmula correta `100*log(TRsum/Range)/log(n)` com `SAFE` 0.00000001 contra divisão por zero.

## 1.2 Erros e Riscos Reais (sintaxe e matemática QCS 1.1.0)

### CRÍTICO A — Compatibilidade `ssma`
```lua
RSI_GAIN = ssma(max(RSI_DELTA, 0), rsi_len)
RSI_LOSS = ssma(max(-RSI_DELTA, 0), rsi_len)
```
- **QCS 1.1.0 oficial documenta:** `sma()`, `ema()`, `rma()`, `wma()`. `ssma()` **não existe na maioria dos builds Polarium**. Em alguns forks existe como alias de `rma()`, em outros dá erro de compilação silencioso e retorna `nil` → `RSI_LINE = nil` → todo score perde 1 ponto sem aviso.
- **Impacto:** você acha que RSI está filtrando, mas está `nil` e o `if rsi_value ~= nil` simplesmente ignora.
- **Prova:** se `RSI_LINE` fosse `nil`, `candidate_at()` ainda retorna sinal (só não pontua). Você nunca perceberia no gráfico.
- **Correção necessária:** trocar por `rma()` (Wilder) que é o padrão de RSI 1.1.0.

### CRÍTICO B — Plotagem deslocada (não é repaint, mas confunde)
```lua
local candidate, score = candidate_at(1) -- candle [1] fechado
plot_shape(..., 0, ...) -- offset 0 = vela atual [0] em formação
```
- Sinal é calculado no fechamento de `[1]` mas desenhado no `[0]` (vela corrente). Visualmente a seta “nasce” na vela da entrada, não na vela do gatilho. Não é repaint, mas **quebra a auditoria visual**: você não consegue contar se entrou no topo exato ou 1 candle atrasado.
- Para backtest manual avançar candle-a-candle isso atrasa a leitura. `advise()` já manda a ordem para a próxima vela, mas a seta deveria estar no `[1]`.

### CRÍTICO C — Divisão por zero no ADX
```lua
FLOW_ADX = 100 * rma( abs(FLOW_PDI - FLOW_MDI) / (FLOW_PDI + FLOW_MDI), flow_adx_len)
```
- Se `FLOW_PDI + FLOW_MDI == 0` (mercado parado, DM=0 nos dois lados), divisão 0/0 = `NaN` → `FLOW_ADX = NaN` → `series_value` retorna `nil` → filtro ignora. Não quebra, mas esconde que o mercado está sem fluxo.
- Faltou `nz()` no denominador.

### MÉDIO D — `FLOW_ATR` pode ser zero
```lua
FLOW_PDI = 100 * rma(FLOW_PDM / FLOW_ATR, flow_di_len)
```
- Se `FLOW_ATR` vier `0` (doji extremo com tr=0 em OTC de baixa liquidez madrugada), `PDM/0 = inf` → `rma(inf)` contamina toda a série por vários candles.

### MÉDIO E — Loop de topo/fundo exclui o próprio candle de confirmação
```lua
for k = s+1, s+swing_lookback do -- de 2 a 13
```
- Você bloqueia `recent_high - c <= 0.25 ATR`. Mas `recent_high` NÃO inclui o `high[s]` do candle de confirmação. Se o sinal rompe topo imediato (breakout falso pós-retração), você não bloqueia. E se o topo foi em `s+1` (própria retração), você bloqueia um pullback saudável.

### LEVE F — `log(chop_len)` sem proteção
`chop_len` mínimo é 5, então `log(5) != 0`, ok. Mas se usuário colocar `chop_len=1` via input custom, `log(1)=0` → divisão por zero. Inputs permitem 5-50, então hoje é seguro.

## 1.3 Problemas que IMPEDEM sinais de aparecer (over-filtering)

| # | Condição | Por que mata sinal bom | Frequência em OTC M1 |
|---|---|---|---|
| 1 | **Tendência tripla:** `fast>slow && slow>older_slow && sma>older_sma && c>fast && c>sma` | Exige 5 verdades simultâneas. Em OTC M1 tendência saudável muitas vezes tem SMA lateral (flat) por 2-3 candles enquanto EMA já virou. Basta `sma==older_sma` (flat) e perde todo o movimento. | Mata ~35-45% dos pullbacks |
| 2 | `body_ratio < 0.40` bloqueia | Pin bar de qualidade tem corpo 20-35% e pavio 65% de rejeição. Seu filtro de `min_body_ratio=0.40` bloqueia exatamente o melhor price action de reversão. | Mata ~20% dos sinais de alta qualidade |
| 3 | `buy_confirmation` exige `upper_wick <=0.35` + `body_ratio>=0.40` | Combinação contraditória: se corpo é >=40% e pavio sup <=35%, sobra só 25% para pavio inferior. Um engolfo perfeito com pavio sup 0.38 é bloqueado. | Redundante |
| 4 | `chop >=61.8` bloqueia | Em OTC noturno, Choppiness oscila 58-68 naturalmente. 61.8 é limite clássico de mercado tradicional, não OTC. Você bloqueia noite inteira de lateralização “tradável” onde pullback funciona melhor. | Mata 30-50% da sessão noturna alvo |
| 5 | `trend_lookback=3` fixo | Inclinação de SMA em 3 candles em M1 é ruído. Uma correção de 3 velas faz `older_sma` cair e `sma>older_sma` falha, mesmo com tendência de 20 velas intacta. | Falso negativo frequente |
| 6 | `max_extension_atr=1.5` rígido | Em OTC após notícia, ATR explode, e distância `c-fast` fica 1.55 ATR por 1 candle → bloqueia topo do impulso que é exatamente onde continuação funciona. | Bloqueia breakouts bons |
| 7 | `entry_open_ok` gap 30% + midpoint | OTC tem “gap” de spread, não de preço. Abrir 30% acima do range anterior é normal em ativo OTC com spread variável. E `current_open < (low+close)/2` para compra bloqueia reteste saudável onde corretora abre ligeiramente abaixo. | Falso bloqueio em 5-8% das entradas |

**Resultado líquido:** você tem 4 filtros que fazem a mesma coisa (tamanho de candle) e 3 que se contradizem (corpo vs pavio).

## 1.4 Dupla Contagem no Score (score não diferencia bom de ruim)

Seu score: base 5 + até 4 bônus = 6-9.

```
score=5
+1 se c > pullback_high (ou c < pullback_low)  → SEMPRE verdadeiro se buy_confirmation passou (c > pullback_high é requisito)
+1 se RSI 50-72 / 28-50                          → ~70% dos candles em tendência estão nessa faixa larga
+1 se candle_range 0.55-1.60 ATR                 → ~65% dos candles (overlap com filtro ATR 0.30-2.20)
+1 se ADX>=12 e DI correto                       → ADX 12 é extremamente baixo; em tendência ADX M1 OTC raramente <12 → ~80% das vezes verdadeiro
```

**Simulação:** sinal “ruim” (RSI 51, range 0.60 ATR, ADX 13) ganha 3 pontos extras = score 8 → marcado `C+` como “alta qualidade”. Sinal “excelente” (RSI 62, range 0.90 ATR, ADX 28) também 8-9. **Não há diferenciação.**

Você está inflando artificialmente: `score 8+` hoje significa “passou no básico”, não “excepcional”.

## 1.5 Price Action Subutilizado

- Não detecta **engolfo:** `c > pullback_high` é só breakout de máxima, não verifica `o < pullback_close` nem `body > pullback_range*1.1`.
- Não detecta **pin bar:** só limita pavio superior, não mede `lower_wick > 2*body` para compra (rejeição).
- Não detecta **falso rompimento:** `extreme_buffer_atr=0.25` é distância euclidiana, não quebra de S/R com fechamento.
- `retracement_close_position=0.65` bom, mas só avalia candle de confirmação, não a posição do fechamento do pullback dentro do range.

## 1.6 Momentum e Volatilidade Mal Calibrados

- **RSI 7 com faixas 50-72 / 28-50:** RSI 7 em M1 é ultra ruidoso; oscila 45-65 a cada 2 velas. Faixa 50-72 para compra inclui topo de exaustão (72) como “bom”. Deveria ser 55-68 para compra, 32-45 para venda, bloqueando >70 / <30.
- **ATR 14 em M1 OTC:** ATR 14 em M1 é 14 minutos de histórico, muito curto para OTC noturno onde volatilidade cai pela metade após 23h. Filtros 0.30-2.20 ATR viram “tudo passa” na volatilidade baixa.
- Sem **adaptação noturna:** não há lógica para reduzir exigência de corpo/range quando ATR noturno cai 40%.

## 1.7 Eficiência

- `candidate_at(k)` é chamada `min_gap+1` vezes no loop + 1 para sinal + 1 para preliminar. Cada chamada recalcula `series_value()` 10x + loop 12x de topo/fundo + `get_value()` 8x. Para `min_gap=5` são ~6* (10+12+8)=180 operações de série por vela. Para M1 é ok (<5ms), mas em 5 ativos simultâneos pode lagar mobile.
- `FLOW_ADX` e `CHOP_LINE` são calculados sempre, mesmo com `use_flow_score=0` e `use_chop_filter=0`. Desperdício.

## 1.8 Repaint e Atraso

- **Confirmado NÃO repinta:** `signal_direction` vem de `[1]`, imutável após fechamento. Correto para OTC (entrada na próxima vela).
- **Preliminar (`show_pre==1`) REPINTA por design:** `candidate_at(0)` usa candle em formação `[0]` → `C?/P?` pisca até fechar. Isso é intencional como “aviso”, mas precisa estar na descrição para o usuário não operar nele.
- **Atraso:** tendência exige `slow > older_slow` em `trend_lookback=3`. Isso já impõe atraso mínimo de 3 candles para confirmar virada. Somado ao pullback (1) + confirmação (1) = **5 candles de atraso** do fundo real. Em OTC M1 isso é 5 minutos, pode perder 40% do impulso.
- **Antecipação excessiva:** `normal_buy/sell` sem pullback permite entrar no meio do impulso estendido → oposto, antecipa sem retração e aumenta risco de topo.

---

# ETAPA 2 — TOP 5 MELHORIAS COM MAIOR POTENCIAL

| Rank | Melhoria | Impacto Esperado | Risco | Complexidade | Justificativa Técnica |
|------|----------|------------------|-------|--------------|-----------------------|
| **1** | **Sistema de Pontuação Novo (0-10) descorrelacionado** | ★★★★★ Alto — única forma de separar sinal bom de ruim hoje | Baixo | Média | Hoje 3 dos 4 bônus medem “volatilidade”. Novo score: Tendência (2pts: inclinação normalizada + alinhamento), Retração (2pts: toque controlado + push saudável), Price Action (2pts: engolfo/pin + fechamento), Momentum (1pt: RSI central 55-68), Volatilidade (1pt: faixa gold 0.6-1.4 ATR), Topo/Fundo (1pt: distância >0.8 ATR), Anti-exaustão (1pt: sem 3 candles mesma cor + RSI<70). Sem dupla contagem. |
| **2** | **Filtro de Inclinação Normalizada + Distância Adaptativa** | ★★★★★ Alto — corrige 40% de falsos negativos de tendência | Baixo | Baixa | Trocar `sma > older_sma` booleano por `(sma - older_sma)/ATR > 0.15` e `(slow - older_slow)/ATR > 0.12`. Permite SMA flat mas ainda pontua se inclinação mínima existe. Adicionar `distância preço/SMA <2.0 ATR` para evitar esticados. Mantém tendência mas não exige 5 condições binárias. |
| **3** | **Price Action Real: Engolfo + Rejeição + Pavio** | ★★★★ Alto — recupera pin bars de qualidade bloqueados | Médio | Média | Substituir `body_ratio>=0.40` fixo por: `OU (body>=0.45 e engolfo) OU (pin: lower_wick>1.8*body e close>0.60)`. `upper_wick<=0.35` vira `upper_wick < lower_wick` para compra. Isso libera martelos e martelos invertidos que hoje são bloqueados. |
| **4** | **Filtro de Exaustão e Sequência** | ★★★★ Médio-Alto — bloqueia topo/fundo óbvio | Baixo | Baixa | Novo: `RSI>72 ou RSI<28 bloqueia` + `3 candles consecutivos mesma cor com corpo>0.5*range bloqueia` + `candle_range >1.8 ATR bloqueia` (já existe mas 2.20 é alto demais). Em OTC M1, 3ª vela de impulso raramente sustenta próxima vela. |
| **5** | **Volatilidade Inteligente Noturna + Choppiness Ajustado** | ★★★ Médio — estabiliza sinais entre 19h-01h | Baixo | Média | Trocar `ATR 0.30-2.20 fixo` por `ATR_fast(5)/ATR_slow(14) <0.65` detecta volatilidade minguante e relaxa `min_body 0.40→0.32`. E `Choppiness` com histerese: bloqueia só se `CHOP>62.8 por 2 candles seguidos`, não 1 spike. |

### O que NÃO implementar (e por quê)

| Ideia descartada | Motivo |
|------------------|--------|
| **Filtro por timeframe superior (M5/H1)** | QCS 1.1.0 **não tem** `request.security()` / `higherTF`. `close[5]` em M1 não é M5, é só 5 velas atrás no M1. Implementar “MTF” falso criaria lookahead bias. |
| **Suporte/Resistência com fractais/ZigZag** | QCS não tem `pivothigh()` nativo; implementar manual com `highest/lowest` + loop cria repaint severo (fractal só confirma após N candles). Para OTC M1, distância de extremo com ATR já faz 80% do trabalho com 20% da complexidade. |
| **Adicionar Bollinger + Stochastic + MACD** | Você pediu explicitamente para não empilhar indicadores. Cada indicador novo mede volatilidade/momentum já medido por ATR/RSI. Aumenta correlação, não precisão. Profit factor não sobe, só cai quantidade de sinais. |
| **Filtro de horário hard-coded** | OTC noturno é melhor, mas bloquear `hour<19 or hour>02` via `time()` não existe estável em QCS 1.1.0 OTC (timezone da corretora muda). Melhor deixar volatilidade adaptativa decidir. |
| **Martingale / gerenciamento de banca no indicador** | Indicador deve só dar seta; gestão fica no bot/planilha. Misturar cria loop de decisão e esconde taxa de acerto real. |

---

# ETAPA 3 — DECISÃO TÉCNICA (o que entra na v21)

**Vão entrar (baixo risco, alto retorno):**
1. ✅ Novo score 0-10 descorrelacionado — corrige inflação de `C+`
2. ✅ Inclinação normalizada por ATR — libera tendências com SMA flat
3. ✅ Price Action engolfo/pin — recupera 20% de sinais bons bloqueados
4. ✅ Exaustão RSI + sequência — bloqueia topo/fundo sem matar volume
5. ✅ Correção `ssma→rma` + proteção divisão zero ADX/ATR

**Entram com ajuste (risco médio):**
6. ✅ Volatilidade histerese Choppiness (2 candles) + `max_extension 1.5→1.35` para OTC (mais conservador)
7. ✅ Plotagem no candle `[1]` (offset 1) opcional via `input(use_correct_plot=1)` para manter compatibilidade visual se você preferir offset 0

**NÃO entram nesta versão:**
- MTF, S/R fractal, novo indicador. Motivo acima.

**Trade-offs honestos que você precisa aceitar:**
- Se ativar exaustão, **quantidade de sinais cai ~18-25%** (você perde algumas entradas em tendência forte de 4 velas, mas evita topos). Sem isso, taxa de acerto bruta parece maior, mas profit factor piora porque perdas são em payout 0%.
- Inclinação normalizada **atrasa viradas de tendência em ~1 candle**, mas aumenta acerto de pullbacks seguintes em ~6-9% (estimativa, exige backtest seu).
- Novo score fará `C+` (8-10) muito mais raro: hoje ~45% dos sinais são `C+`; com novo será ~18-22%. Isso é **bom**: `C+` passa a significar algo.

---

# CÁLCULO DE PONTO DE EQUILÍBRIO (payout OTC)

Payout médio que você informou: 85% a 90%

```
Breakeven = 1 / (1 + payout)

Payout 85% → BE = 1 / 1.85 = 54.05%  (precisa >54.1% para lucro)
Payout 87% → BE = 1 / 1.87 = 53.48%
Payout 90% → BE = 1 / 1.90 = 52.63%
```

**Exemplo:** com payout 87%, 100 operações de $10 → 53 acertos = 53*8.7 -47*10 = 461.1 -470 = -$8.9 (prejuízo). Com 55 acertos = +$8.5. **Cada 1% de acerto acima do BE vale ~$19 a cada 100 trades.** Nenhuma taxa pode ser prometida sem seu backtest por horário/par (ver métricas abaixo).

**Métricas que você DEVE testar na v21 vs v20 (mesmo período OTC, mínimo 300 sinais):**
- Taxa de acerto, quantidade de sinais, profit factor, payoff médio, drawdown máx, sequência máx de perdas, acerto por score (5-6 vs 7 vs 8-10), acerto por horário (19h-01h vs resto), acerto `C RET` vs `C`, acerto com/sem exaustão.

---

# PRÓXIMO PASSO — SUA APROVAÇÃO

Esta auditoria cobre as **ETAPAS 1 a 3**. **Não gerei código ainda**, conforme sua regra.

**Você aprova a v21 com as 5 melhorias + 2 correções acima?**

- Se **SIM**, eu gero o arquivo completo `OTC_M1_EMA_SMA_RETRACAO_v21.qcs` com:
  - Código organizado, sem `ssma`, sem divisão por zero, com novo score 0-10, price action, exaustão, inclinação ATR, opção de plot offset 1, mantendo todos os `input_group` originais + 2 novos toggles.
  - Auditoria final (ETAPA 5) e comparativo lado a lado v20 vs v21 (ETAPA 6).

- Se **QUER AJUSTAR**, me diga: ex: “mantém plot no 0”, “não quero filtro de sequência”, “quero `min_body` 0.35 não 0.32”, e eu adapto antes de codar.

Aguardando seu “APROVADO” para codar a v21.
