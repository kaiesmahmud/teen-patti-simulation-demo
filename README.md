# teen-patti-simulation-demo

A browser-based 4-player Teen Patti (3-card poker) simulation. No real money, no backend — pure frontend JavaScript logic running in a single HTML file.

---

## How the Game Works (Overview)

- 4 players sit at a table
- Game plays **3 rounds**
- Each round: a fresh deck is shuffled and each player gets **3 cards**
- After all 3 rounds, the player who won the **most rounds** wins the pot
- Pot = ৳400 (৳100 bet × 4 players), winner takes ৳300 (25% room fee deducted)

---

## 1. The Deck

A standard 52-card deck is created from 4 suits × 13 values:

```
Suits:  S (Spades ♠)  H (Hearts ♥)  D (Diamonds ♦)  C (Clubs ♣)
Values: 2 3 4 5 6 7 8 9 10 J Q K A
```

Each card is stored as a string like `"K-H"` (King of Hearts) or `"10-S"` (Ten of Spades).

---

## 2. Card Values (Rank)

Every card has a numeric rank used internally for comparisons:

| Card | Rank |
|------|------|
| 2    | 2    |
| 3    | 3    |
| ...  | ...  |
| 9    | 9    |
| 10   | 10   |
| J    | 11   |
| Q    | 12   |
| K    | 13   |
| A    | 14   |

Higher rank = stronger card. Ace is always the highest single card.

---

## 3. Shuffle

The deck is shuffled using **Fisher-Yates algorithm**, run 3 passes for stronger randomness:

```
For each pass (×3):
  Walk the deck from the last card down to the second
  At each position i, pick a random position j between 0 and i
  Swap card[i] with card[j]
```

This guarantees every possible ordering of 52 cards is equally likely.

---

## 4. Dealing Cards

Cards are dealt **in rotation** (like a real dealer), one card at a time per player, 3 rounds of dealing:

```
Deal pass 1: P1 gets card, P2 gets card, P3 gets card, P4 gets card
Deal pass 2: P1 gets card, P2 gets card, P3 gets card, P4 gets card
Deal pass 3: P1 gets card, P2 gets card, P3 gets card, P4 gets card
```

Result: each player holds exactly 3 cards, all from the same shuffled deck (no repeats).

---

## 5. Hand Rankings

After receiving 3 cards, each player's hand is classified. Rankings from highest to lowest:

| Type Code | Hand Name      | Description                                              | Example        |
|-----------|----------------|----------------------------------------------------------|----------------|
| 6         | Trail 🔥        | All 3 cards same value (3-of-a-kind)                     | K K K          |
| 5         | Straight Flush | 3 consecutive cards, all same suit                       | 9♠ 8♠ 7♠       |
| 4         | Straight       | 3 consecutive cards, any suit                            | J♥ 10♦ 9♣      |
| 3         | Flush          | All 3 cards same suit, not consecutive                   | A♦ 7♦ 3♦       |
| 2         | Pair           | 2 cards of the same value                                | Q♠ Q♣ 5♥       |
| 1         | High Card      | None of the above — judged by highest card               | A♠ 9♥ 4♦       |

**Special rule — A-2-3 Straight:** In Teen Patti, Ace-2-3 is the **highest** straight (ranked above A-K-Q). It is detected as a special case since after sorting by rank descending the cards appear as [14, 3, 2] which doesn't follow the normal consecutive pattern.

---

## 6. Scoring Engine

### Step 1 — Sort cards by rank (highest first)

The 3 cards are sorted descending so comparisons are always in the same order.

### Step 2 — Detect hand type

```
isTrail     = all 3 ranks equal
isFlush     = all 3 suits equal
isNormalSeq = ranks go: top, top-1, top-2 (consecutive descending)
isA23       = ranks are exactly [14, 3, 2]  ← special Ace-2-3 case
isSeq       = isNormalSeq OR isA23
isPair      = any two adjacent cards (after sort) share a rank
```

### Step 3 — Assign internal score

The internal score is a single large integer that encodes both the hand type and the card values. It is **never shown to players** — it is only used by the system to decide who wins.

```
Trail         →  600,000 + (rank × 1,000)
Straight Flush→  500,000 + (seqTop × 1,000)
Straight      →  400,000 + (seqTop × 1,000)
Flush         →  300,000 + (rank1 × 10,000) + (rank2 × 100) + rank3
Pair          →  200,000 + (pairRank × 1,000) + kickerRank
High Card     →  100,000 + (rank1 × 10,000) + (rank2 × 100) + rank3
```

The 100,000-step gaps between hand types ensure that even the worst Trail (2-2-2 = 602,000) always beats the best Straight Flush (A-K-Q suited = 515,000). Within the same hand type, card ranks break the tie cleanly.

For A-2-3, `seqTop = 15` (higher than Ace = 14) so it correctly ranks above all other straights.

### Step 4 — Visible score (shown to user)

The visible score is a human-readable label combining the hand type code and the top two card values:

```
Format:  TypeCode · Card1Card2
Example: "2 · QQ"  → Pair of Queens
Example: "1 · AK"  → High Card, Ace-King
Example: "6 · KK"  → Trail of Kings
```

This is decorative — the **internal score** decides the winner, not the visible score.

---

## 7. Round Winner Logic

After all 4 players are scored in a round:

```
Find the highest internalScore among all 4 players
The player(s) with that score = round winner(s)
```

If two players have an identical internal score (extremely rare, means identical hand strength), they **both** earn the round point.

---

## 8. Final Winner Logic

The game winner is **not** determined by adding up raw scores across 3 rounds. Instead it uses a **points system**:

```
Step 1: Count round wins per player
        (each round win = 1 point, max 3 points)

Step 2: Player with most points = winner

Step 3: If tied on points → compare cumulative internalScore as tiebreaker

Step 4: If still tied → split the pot equally
```

This means a player who loses Round 1 and Round 2 can still win the game by winning Round 3 — keeping the match competitive until the end.

---

## 9. Payout

```
Each player bets:  ৳100
Total pot:         ৳400  (4 × ৳100)
Room fee (25%):    ৳100
Winner receives:   ৳300

If tied:           ৳300 ÷ number of winners  (split equally)
```

---

## 10. Score Table Columns Explained

The round score table shows these columns for each player:

| Column   | What it means                                                               |
|----------|-----------------------------------------------------------------------------|
| Player   | Player 1 through Player 4 — always in fixed order, never reordered         |
| Cards    | The 3 cards the player received this round (e.g. `K♠ 9♥ 4♦`)              |
| Hand     | The detected hand type (Trail, Flush, Pair, etc.)                          |
| Score    | **Visible score** — hand type code + top two card labels (for user display) |
| Internal | **Internal score** — the actual number used to decide the winner            |
| Pts      | Cumulative round wins for this player up to the currently viewed round      |

The row of the **round winner** is highlighted in gold.

---

## 11. UI Behaviour Summary

| Situation                        | Visual indicator                                      |
|----------------------------------|-------------------------------------------------------|
| Player wins a round              | Cards glow gold + "ROUND WIN" badge on their slot     |
| Player wins the overall game     | Slot gets gold pulsing border + 👑 crown              |
| Player loses the overall game    | Slot fades to 55% opacity                             |
| Clicking Round 1 / 2 / 3 tab    | Table AND cards both update to show that round's data |
| Round winner row in score table  | Entire row highlighted gold                           |
| Points tracker (★ pips)          | Shows how many rounds each player has won so far      |

---

## 12. Why Different Players Win Each Game

The original implementation added raw internal scores across all 3 rounds, which caused the player who got lucky in round 1 (e.g. a Flush) to build an insurmountable lead. The fixed version uses per-round points so that:

- Round 1 winner gets 1 point
- Round 2 winner gets 1 point  
- Round 3 winner gets 1 point
- Any player can take 2 of 3 rounds and win regardless of round 1 outcome

Combined with a 3-pass Fisher-Yates shuffle and rotation-style dealing (no one always gets the "top" cards), each player has an equal chance every game.