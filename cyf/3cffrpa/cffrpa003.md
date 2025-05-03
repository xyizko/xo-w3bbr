<h1 align="center"><code> cffrpa003 </code></h1>
<h2 align="center"><i> (Medium) Stranded Wei in Tie Games</i></h2>
<h3 align="center"><i> ⚠️ Severity Medium</i></h3>

1. [Bugs Found](#bugs-found)
2. [Summary](#summary)
3. [Vulnerability Details](#vulnerability-details)
   1. [Exploitation Procedure](#exploitation-procedure)
4. [Impact](#impact)
5. [Recomendations](#recomendations)


# Bugs Found

[![](../../gfx/cffrpa_bf.jpg)](https://profiles.cyfrin.io/u/xyizko)

# Summary 
Due to integer division, a small amount (1 wei) of ETH can get permanently stuck in the contract balance during specific tie games.

# Vulnerability Details
In the case of a tie game using ETH bets, the contract calculates the total pot, deducts a 10% protocol fee, and then divides the remaining amount equally between the two players. The division (totalPot - fee) / 2 uses integer division. If the value totalPot - fee is an odd number of wei, the division will round down, leaving 1 wei remaining in the contract's balance. This stranded wei is not tracked or accounted for and cannot be withdrawn by the admin or players.

<https://github.com/CodeHawks-Contests/2025-04-rock-paper-scissors/blob/main/src/RockPaperScissors.sol#L511-L531>

```solidity
function _handleTie(uint256 _gameId) internal {
    Game storage game = games[_gameId];
    game.state = GameState.Finished;
    // Return ETH bets to both players, minus protocol fee
    if (game.bet > 0) {
        // Calculate protocol fee (10% of total pot)
        uint256 totalPot = game.bet * 2;
        uint256 fee = (totalPot * PROTOCOL_FEE_PERCENT) / 100; // PROTOCOL_FEE_PERCENT = 10
     uint256 refundPerPlayer = (totalPot - fee) / 2;       // PROBLEMATIC: Integer division if (totalPot - fee) is odd

     // Accumulate fees for admin
     accumulatedFees += fee;
     emit FeeCollected(_gameId, fee);
     // Refund both players
     (bool successA,) = game.playerA.call{value: refundPerPlayer}("");
     (bool successB,) = game.playerB.call{value: refundPerPlayer}("");
     require(successA && successB, "Transfer failed"); // (Note: DoS vulnerability also present here)
}
```

1. The calculation `(game.bet * 2 * 90) / 100` simplifies to `(game.bet * 9) / 5.` This result is then divided by 2.
2. The final amount to be refunded to each player is `((game.bet * 9) / 5) / 2.` If `(game.bet * 9) / 5` is an odd number (which can happen if game.bet is not divisible by 5, e.g., 6, 7, 11, 12 wei), the integer division by 2 will result in a remainder of 1 wei. This 1 wei is not sent to either player and remains in the contract.


## Exploitation Procedure 
1. Determine a BET\_AMOUNT (in wei) such that ((BET\_AMOUNT \* 2 \* 90) / 100) is an odd number.
2. For example, a bet of 6 wei: (6 \* 2 \* 90) / 100 = (12 \* 90) / 100 = 1080 / 100 = 10. 10 / 2 = 5. No stranded wei here.
3. Let's try 7 wei: (7 \* 2 \* 90) / 100 = (14 \* 90) / 100 = 1260 / 100 = 12. 12 / 2 = 6. Still no stranded wei. Hmm, (bet \* 9) / 5 needs to be odd. This happens if bet is not divisible by 5. Let's test bet = 1 wei: (1 \* 9) / 5 = 1 (integer division). 1 / 2 = 0. Total pot 2 wei, fee 0.
4. Refunds 0. 2 wei stranded. Let's test bet = 6 wei: (6 \* 9) / 5 = 10. 10 / 2 = 5. Total pot 12, fee 1. Refund 5 per player. 12 - 1 - (5+5) = 1 wei stranded. A bet of 6 wei works.
5. Create an ETH game with the identified BET\_AMOUNT (e.g., 6 wei).
   Join the game with a second account using the same bet amount.

Play the game to result in a tie (e.g., both play Rock in a 1-turn game).
Observe the contract's ETH balance after the game finishes. It will retain 1 wei from the tie.


# Impact 

A minimal amount (1 wei) of ETH becomes effectively lost within the contract's balance for each tie game where the net pot is an odd number of wei. Over time, across many such tie games, this could accumulate, but the amount per game is negligible. It is not a loss for the contract owner, but a minor loss for the players involved in that specific game.

# Recomendations

Adjust the calculation to handle the remainder explicitly or ensure the remainder is zero. For example, send the calculated fee to the admin pool, and divide the remaining amount. If the remaining amount is odd, add the 1 wei remainder to the admin fee or send it to one of the players (e.g., the first player). A clearer approach might be to add the remainder to the fee:

