# Twitch Gift Sub Allocation: Reverse-Engineered Optimization Model

## 1. Abstract
The allocation of community gift subscriptions on Twitch is often mistakenly assumed to be an absolute engagement reward or a random distribution. However, based on platform incentives, the algorithm solves an **incremental uplift optimization problem**. The objective is to maximize Expected Incremental Net Platform Margin while sustaining the social utility required to drive repeat purchases from "whale" gifters. 

Distributing gifts to completely disengaged users destroys platform value (zero conversion, zero social utility). Conversely, giving gifts to highly engaged, high-ad-revenue users without ad-blockers results in ad revenue cannibalization and organic subscription cannibalization. The algorithm targets the "Persuadable Middle."

---

## 2. The Objective Function

The algorithm maximizes the Expected Incremental Net Platform Margin ($\Delta \mathbb{E}[\Pi_i]$) over a rolling time horizon $T$ (e.g., 30 days) by ranking every eligible non-subscribed user $i \in \mathcal{U}_{\text{eligible}}$.

$$ \arg\max_{i \in \mathcal{U}_{\text{eligible}}} \Delta \mathbb{E}[\Pi_i] = \Delta \mathbb{E}[\text{SubMargin}_i] - \Delta \mathbb{E}[\text{AdLoss}_i] + \Delta \mathbb{E}[\text{RetAdLift}_i] + \mathbb{E}[\text{GifterMultiplier}_i] $$

---

## 3. Component Sub-Functions & Variables

### A. Incremental Subscription Margin ($\Delta \mathbb{E}[\text{SubMargin}_i]$)
Captures the financial gain of converting a recipient into a paying subscriber, minus the cannibalization of baseline organic purchases.

$$ \Delta \mathbb{E}[\text{SubMargin}_i] = \Big( P_i(\text{resub} \mid \text{gift}) - P_i(\text{organic} \mid \text{no gift}) \Big) \cdot \text{LTV}_{\text{sub}} \cdot (1 - s_c) $$

*   $P_i(\text{resub} \mid \text{gift})$: Probability user $i$ converts to a self-paid recurring subscription after the 30-day gift expires.
*   $P_i(\text{organic} \mid \text{no gift})$: Baseline probability user $i$ would have purchased a subscription organically.
*   $\text{LTV}_{\text{sub}}$: Expected gross cumulative subscription revenue over the user's expected continuous retention cycle.
*   $s_c$: The streamer's contractual revenue split (e.g., $0.50$, $0.70$).

### B. Ad Cannibalization Loss ($\Delta \mathbb{E}[\text{AdLoss}_i]$)
Because gifted subs grant ad-free viewing, Twitch forfeits 30 days of ad margin unless the user blocks ads.

$$ \Delta \mathbb{E}[\text{AdLoss}_i] = (1 - \alpha_i) \cdot H_{i,c} \cdot \delta_{\text{ads}} \cdot \left(\frac{\text{CPM}_{i,c}}{1000}\right) \cdot (1 - s_{\text{ad}}) $$

*   $\alpha_i \in \{0, 1\}$: Ad-block indicator (If the user blocks ads, $\alpha_i = 1$, zeroing out the loss).
*   $H_{i,c}$: Expected hours user $i$ will watch channel $c$ over the next 30 days.
*   $\delta_{\text{ads}}$: Ad impressions run per hour.
*   $\text{CPM}_{i,c}$: Effective cost-per-mille for channel $c$ in user $i$'s geographic market.

### C. Gifter Social Utility Multiplier ($\mathbb{E}[\text{GifterMultiplier}_i]$)
Gifters require immediate psychological reinforcement. Giving subs to inactive lurkers kills the momentum of "Hype Trains."

$$ \mathbb{E}[\text{GifterMultiplier}_i] = \lambda_g \cdot \Big[ w_1 \cdot \text{InChatNow}_i + w_2 \cdot \mathbb{E}[\text{ChatReactivity}_i] + w_3 \cdot (1 - \text{IsBotOrAlt}_i) \Big] $$

*   $\lambda_g$: Gifter whale weight (historical propensity to continue gifting based on chat reaction).
*   $\text{InChatNow}_i$: Binary flag ($1$ if connected to chat, $0$ if offline).
*   $\mathbb{E}[\text{ChatReactivity}_i]$: Likelihood the recipient types an immediate thank-you or spams emotes.

---

## 4. Execution Flow & Logic

When a user purchases a community gift of size $N$ (e.g., 50 subs), the algorithm executes the following sequence:

1.  **Filter Pool:** Generate $\mathcal{U}_{\text{eligible}}$ containing all non-subscribed followers of the channel.
2.  **Live Priority:** Segment the pool into $\mathcal{U}_{\text{live}}$ (currently connected to IRC/WebSocket) and $\mathcal{U}_{\text{offline}}$.
3.  **Score Generation:** Run the objective function for all $i \in \mathcal{U}_{\text{live}}$.
    *   *Ad-Blocker Bonus:* Users with $\alpha_i = 1$ receive higher net scores due to zero ad cannibalization.
    *   *Cannibalization Penalty:* Users with very high $P_i(\text{organic})$ receive heavy penalties.
4.  **Ranking:** Sort $\mathcal{U}_{\text{live}}$ descending by $\Delta \mathbb{E}[\Pi_i]$.
5.  **Allocation:** Distribute gifts to the top $N$ users.
6.  **Fallback:** If $N > |\mathcal{U}_{\text{live}}|$, calculate scores for $\mathcal{U}_{\text{offline}}$ (where $\mathbb{E}[\text{GifterMultiplier}_i] = 0$) and distribute the remainder to accounts with the highest historical $P_i(\text{resub} \mid \text{gift})$.

---

## 5. Key System Overrides & Edge Cases

*   **Prime Token Block:** The algorithm deprioritizes users who frequently use Amazon Prime tokens on the platform. A gifted sub locks them out of using Prime on the channel for 30 days, which can disrupt established behavior loops.
*   **Purchasing Power Parity (PPP) Geographic Arbitrage:** Due to localized subscription pricing, a sub gifted by a US viewer ($4.99) directed to a Tier-3 economy viewer (where a sub costs equivalent to $1.30) yields lower LTV upon conversion. The algorithm scores recipients in absolute target currency to preserve high-dollar conversion potential.