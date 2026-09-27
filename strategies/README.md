# IC Markets Raw — Trend Pullback MTF (Pine Script v6)

Fichier : [`icmarkets_eurusd_trend_pullback.pine`](icmarkets_eurusd_trend_pullback.pine)

> ⚠️ Un backtest est une simulation historique, pas une garantie de résultats futurs.
> Le script n'a **pas** été backtesté ni compilé dans ce dépôt (aucun TradingView disponible dans l'environnement de génération). Compilez-le et testez-le vous-même (voir « Installation ») avant tout usage réel, d'abord sur compte **démo**.

---

## 1. Résumé du projet

| Élément | Choix |
|---|---|
| Broker | IC Markets — compte **Raw Spread** |
| Marché / actif | Forex — **EURUSD** (flux `ICMARKETS:EURUSD`) |
| Timeframe | **H1** (entrée) + **H4** (tendance) |
| Style | Suivi de tendance — achat/vente des replis (pullbacks) |
| Indicateurs | EMA 21 / 50 / 200, EMA 200 H4, RSI 14, ADX 14, ATR 14 |
| Capital initial | 10 000 USD (modifiable dans *Propriétés*) |
| Risque par trade | 0,5 % de l'equity (input) |
| Commission | 3,50 $ / lot / côté (7 $ aller-retour) |
| Slippage | 3 ticks = 0,3 pip (couvre spread Raw moyen + glissement) |
| Pyramiding | 0 (une seule position à la fois) |

**Pourquoi EURUSD H1 ?** C'est la paire au spread Raw le plus serré chez IC Markets (souvent 0,0–0,1 pip), la plus liquide, avec des coûts faibles par rapport à la taille d'un stop ATR en H1 (~10–20 pips). En M1–M5, la commission de 7 $/lot pèserait beaucoup plus lourd par rapport au gain moyen.

## 2. Architecture du script

```text
1. Configuration (strategy(): capital, commission, slippage, marge)
2. Inputs (groupés : direction, tendance, entrée, filtres, risque, sorties, protections, visuel)
3. Indicateurs (EMA, RSI, ATR, ADX)
4. MTF non-repaint (EMA 200 H4 sur bougie H4 clôturée)
5. Tendance (H4 + H1 + ADX)
6-7. Conditions Long / Short
8. Gestion du risque (sizing, perte journalière, trades/jour, série de pertes, cooldown)
9. Entrées (+ alertes JSON)
10-11. SL / TP1 / break-even / trailing / TP2 / sorties forcées
12. Alertes (alert_message des ordres)
13. Visualisation (EMA, SL/TP ancrés au prix, signaux)
14. Tableau de statistiques
```

## 3–4. Logique de trading

### Entrée Long
```
Clôture H4 > EMA200 H4 (bougie clôturée)
+ EMA50 > EMA200 et close > EMA200 (H1)
+ ADX ≥ 20 (marché directionnel)
+ Le low a touché l'EMA21 dans les 5 dernières bougies (pullback)
+ Close > EMA21, close > high précédent, bougie haussière (reprise)
+ 50 < RSI < 70 (momentum sans surachat)
+ Session 07:00–17:00 Londres, pas vendredi soir
+ Protections OK (perte du jour, trades/jour, pause, cooldown)
→ Achat à l'ouverture de la bougie suivante
```
Short : exact miroir.

### Sorties
| Sortie | Règle |
|---|---|
| Stop Loss | Close ± 1,5 × ATR(14) → définit **1R** |
| TP1 | 1R, ferme 50 % de la position |
| Break-even | Après TP1, stop remonté au prix d'entrée + 0,2 pip (couvre la commission) |
| Trailing | Après TP1, stop chandelier `high − 2 × ATR` (ne recule jamais) |
| TP2 | 2,5R sur le reste (désactivable → trailing seul) |
| Perte de tendance | EMA50 recroise l'EMA200 en H1 → clôture |
| Vendredi | Clôture forcée à 20:00 Londres (évite le gap du week-end) |

Les SL/TP sont calculés sur la clôture de la bougie signal — ce sont exactement les niveaux envoyés dans l'alerte, donc le backtest et le bot utilisent les mêmes niveaux.

## 5. Gestion du risque

- **Taille de position** : `unités = (equity × risque%) / (distance SL × valeur d'une unité)`, arrondie vers le bas au pas de 1 000 unités (0,01 lot). Conversion de devise automatique via `request.currency_rate`.
- **Plafond de levier** : 10× effectif par position (input), indépendamment du levier du compte.
- **Perte journalière max** : 2 % de l'equity du début de journée → plus de nouvelles entrées ce jour-là.
- **Trades max / jour** : 3.
- **Série de pertes** : après 4 positions perdantes consécutives → pause de 24 bougies.
- **Cooldown** : 3 bougies après chaque sortie.
- **R:R effectif** : 50 % à 1R + 50 % à 2,5R ≈ 1,75R moyen si tout est atteint ; le break-even réduit les pertes après TP1 mais transforme une partie des gagnants en trades nuls. Le win rate attendu d'un tel système est typiquement de 35–50 % : les séries de 5 à 8 pertes sont normales.

## 6. Backtest — réglages et lecture

Dans *Strategy Tester → Propriétés*, vérifiez : capital, commission (`0.000035` par contrat = par unité), slippage 3, marge 3,33 % (1:30 ESMA ; mettez 0,2 pour un compte 1:500).

À analyser, **pas seulement le Net Profit** : Profit Factor (> 1,3 souhaitable), Max Drawdown, nombre de trades (> 100 pour être significatif), Average Trade (doit rester nettement positif **après** commissions), Sharpe/Sortino, stabilité année par année.

Remarques :
- Le tableau et TradingView comptent chaque sortie partielle comme un « trade » : 1 position = jusqu'à 2 trades.
- Le flux TradingView est un prix **bid** ; les Long sont réellement exécutés à l'ask. Le slippage de 3 ticks compense le spread Raw moyen, mais pas les pics de spread (rollover 23h–00h serveur, news). Le filtre de session évite déjà le rollover.
- Swaps non modélisés (TradingView ne les gère pas) — impact faible car les positions durent généralement quelques heures et sont fermées le vendredi.
- Autres instruments : la commission est une constante de `strategy()`. Pour l'or (`XAUUSD`, 1 lot = 100 oz) utilisez `commission_value = 0.035` et un `lotStep` de 1 ; pour les indices, la commission IC Markets est nulle (spread seul) → `commission_value = 0` et augmentez le slippage.

## 7. Repainting / Lookahead

| Élément | Statut |
|---|---|
| `request.security` H4 | `expression[1]` + `lookahead_on` → dernière bougie H4 **clôturée**, identique historique / temps réel ✅ |
| Signaux | uniquement `barstate.isconfirmed`, exécution à l'ouverture suivante ✅ |
| `calc_on_every_tick` | `false` ✅ |
| Pivots / zigzag | aucun ✅ |
| Trailing | mis à jour sur bougie clôturée ✅ |
| Limite intrabar | En historique, si SL et TP sont dans la même bougie, TradingView suppose un ordre de parcours (open→plus proche extrême). Activez *Bar Magnifier* (plan Premium) pour plus de réalisme. ⚠️ |

**Signal historique = signal disponible en temps réel** : aucune donnée future n'est utilisée.

## 8. Analyse MTF

`H4 → tendance de fond (EMA200)` · `H1 → tendance + pullback + déclencheur`. Une bougie H1 de 13:00 utilise la bougie H4 clôturée à 12:00, jamais celle en cours. Le script refuse de tourner si le HTF est inférieur au timeframe du graphique.

## 9. Indicateurs — rôle et redondances

| Indicateur | Rôle | Redondance |
|---|---|---|
| EMA 200 H4 | Filtre de tendance majeure | Corrélé à EMA50/200 H1, mais sur une autre échelle (voulu) |
| EMA 50/200 H1 | Tendance locale + sortie | — |
| EMA 21 | Zone de pullback | — |
| RSI 14 | Momentum (50–70) + anti-surachat | Seul oscillateur : pas d'empilement RSI/Stoch/MACD |
| ADX 14 | Régime tendance vs range | Mesure la force, pas la direction (complémentaire) |
| ATR 14 | SL, trailing, sizing | — |

## 10. Conditions de marché

- ✅ **Tendance** H4 claire : régime cible.
- ⚠️ **Range** : l'ADX filtre une partie, mais les faux départs restent la principale source de pertes.
- ⚠️ **News (NFP, BCE, FOMC)** : spreads et slippage plus élevés que modélisés → envisagez de couper le bot ces jours-là.
- ⚠️ **Faible volatilité** (été, fêtes) : ATR faible → stop serré, sensible au bruit.

## 11. Optimisation — protocole recommandé

```
In-Sample 2019–2022 → optimisation légère → Out-of-Sample 2023–2024 → Forward test démo 2–3 mois
```
- N'optimisez que 2–3 paramètres (ex. `slAtrMult`, `tp2R`, `adxMin`) et gardez des valeurs rondes.
- Recherchez un **plateau** de paramètres rentables, pas le pic.
- Vérifiez la robustesse sur GBPUSD / AUDUSD avec les mêmes réglages.

## 12. Alertes / Webhook

1. Ajoutez la stratégie au graphique `ICMARKETS:EURUSD` H1.
2. *Alerte* → Condition : la stratégie → « Order fills only » (remplissage d'ordres).
3. Message : `{{strategy.order.alert_message}}`
4. Webhook : l'URL de votre bridge (ex. PineConnector, TradersPost, ou un bridge MT4/MT5 perso).

Messages envoyés (JSON) :
```json
{"action":"buy","symbol":"EURUSD","exchange":"ICMARKETS","units":25000,"lots":0.25,
 "price":1.08452,"sl":1.08251,"tp1":1.08653,"tp2":1.08954,"strategy":"ICM_TrendPB","time":"..."}
```
`action` ∈ `buy`, `sell`, `close_partial_buy`, `close_partial_sell`, `close_buy`, `close_sell`, `close_all`.
Chaque alerte correspond à un ordre **réellement exécuté** par la stratégie (pas à une condition théorique). Les SL/TP étant aussi gérés par la stratégie, configurez votre bridge pour soit poser SL/TP côté broker (recommandé, protection en cas de coupure), soit exécuter uniquement les messages de clôture — pas les deux.

## Installation

**Option A — manuelle** : ouvrez le Pine Editor, collez le contenu du fichier `.pine`, *Ajouter au graphique*.

**Option B — via ce MCP** (TradingView Desktop lancé avec CDP) :
```bash
cp strategies/icmarkets_eurusd_trend_pullback.pine scripts/current.pine
node scripts/pine_push.js        # injecte + compile + affiche les erreurs
```
ou avec Claude : `pine_set_source` → `pine_smart_compile` → `pine_get_errors`, puis `data_get_strategy_results`.

## Checklist de validation

- [ ] Compilation sans erreur dans TradingView (Pine v6)
- [ ] Graphique `ICMARKETS:EURUSD`, H1
- [ ] Propriétés : capital, commission 0.000035/contrat, slippage 3, marge adaptée à votre levier
- [ ] Entrées / sorties visibles, SL-TP correctement placés sur le prix
- [ ] Backtest ≥ 100 trades, analysé au-delà du Net Profit
- [ ] Out-of-Sample puis forward test en démo
- [ ] Alertes « Order fills only » + `{{strategy.order.alert_message}}` testées en démo
