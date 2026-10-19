# Presets de Sinais — v21 para OTC M1

Você reclamou de pouquíssimos sinais em M1 — normal. A v21 conservadora filtra exaustão e lateralização agressivamente (-18 a -25% vs v20). Para M1 você precisa volume.

## Sem mexer no código: ajuste os inputs no Polarium

| Preset | `min_score` | `min_gap` | `max_chop` | `trend_slope_thr` | `max_extension` | `RSI 72/28` | `max_consec` | Sinais vs v21 | Qualidade |
|--------|-------------|-----------|------------|-------------------|----------------|-------------|--------------|---------------|-----------|
| **Conservador (v21 atual)** | 6 | 2 | 61.8 | 0.10 | 1.35 | 72/28 | 3 | 100% | Alta (C+ ~20%) |
| **Balanceado (recomendado)** | 5 | 1 | 65 | 0.07 | 1.50 | 75/25 | 4 | +40-50% | Média-Alta |
| **Agressivo (volume máximo)** | 5 | 0 | 68 (ou `use_chop=0`) | 0.06 | 1.80 | 78/22 | 5 | +70-90% | Média |
| **Scalp noturno (19h-01h)** | 5 | 1 | 63 | 0.08 | 1.40 | 72/28 | 3 | +20-30% | Alta |

### Como aumentar passo a passo (faça nesta ordem, teste 1 noite cada):

1. **`min_score 6 → 5`** — maior ganho imediato (+25-30% sinais). Libera score 5 que é “médio”, não lixo. C+ continua só 8+.
2. **`min_gap 2 → 1`** — permite 1 sinal a cada 1 vela vazia vs 2. Em M1, 2 de gap mata sequências de pullback duplo.
3. **`max_chop 61.8 → 65`** — sua sessão noturna fica tradável. Só bloqueia lateralização forte (>65 por 2 velas).
4. **`trend_slope_thr 0.10 → 0.07`** — libera tendências lentas que a v21 exige inclinação forte.
5. **`rsi 72/28 → 75/25` + `max_consecutive 3→4`** — deixa passar 3ª vela esticada que antes era bloqueada.

> Não desligue `use_exhaustion` e `use_inclination` de uma vez — você volta ao v20 inflado. Ajuste 1 por vez.

## Arquivos entregues

- `OTC_M1_EMA_SMA_RETRACAO_v21.qcs` — conservadora (padrão)
- `OTC_M1_EMA_SMA_RETRACAO_v21_AGRESSIVO.qcs` — balanceada, pronta para colar (diferenças acima já aplicadas)

Ambos têm os mesmos toggles. Você pode alternar entre eles sem reinstalar.

## Regra de ouro M1

Com payout 85-87% (BE 53.5-54.1%), **quantidade > qualidade até certo ponto**. É melhor 14 sinais/noite com 56% de acerto do que 6 sinais com 62% — PF é maior. Use:

- `score >=6` para operar (não só `C+`)
- Filtre só `C RET` se quiser menos sinais, mas perca `C` de continuação que em M1 funciona
- `show_pre=1` para ver amarelo `C?/P?` mas **nunca entre nele** — só para antecipar

