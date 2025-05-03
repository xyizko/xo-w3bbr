<h1 align="center"><code> cffrpa002 </code></h1>
<h2 align="center"><i>(Medium) Arbitrary Reveal Timeout Griefing - Allows a player to lock opponent's funds/tokens for an arbitrary duration.</i></h2>
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

1. When creating a game, Player A specifies the \_timeoutInterval, which is the duration allowed for players to reveal their moves after both have committed. The contract enforces a minimum of 5 minutes but no maximum.
2. A malicious Player A can set this interval to an extremely large value (e.g., several years).
3. If Player B joins, both commit, and Player B fails to reveal their move, Player A (or anyone) cannot call timeoutReveal to resolve the game and potentially reclaim funds/tokens until this excessively long timeoutInterval has fully elapsed.
4. This griefs Player B by locking their funds/tokens for an arbitrary period dictated by Player A.

# Vulnerability Details

1. The `createGameWithEth` and `createGameWithToken` functions accept `_timeoutInterval` as an argument from the game creator (Player A).
2. This value is stored in the Game struct and used to calculate the `revealDeadline`.
3. The `revealMove` function requires `block.timestamp <= game.revealDeadline`, and `timeoutReveal` requires `block.timestamp > game.revealDeadline`.
4. The only check on `_timeoutInterval` is `require(_timeoutInterval >= 5 minutes`, "Timeout must be at least 5 minutes");.There is no upper bound validation.

If Player A creates a game with a very large \_timeoutInterval, and the game reaches the Committed state where both players have committed but Player B fails to reveal:

1. Player A cannot reveal for B.
2. Player A cannot call timeoutReveal until block.timestamp exceeds the very large revealDeadline (block.timestamp + very\_large\_timeout).
3. Player B's ETH (in ETH games) or Token (in Token games) remains locked in the contract until Player A (or someone else) can successfully call timeoutReveal after the long timeout period.
4. Player B is grieved as their funds are inaccessible for an unreasonable duration.

Code Location:

1. <https://github.com/CodeHawks-Contests/2025-04-rock-paper-scissors/blob/main/src/RockPaperScissors.sol#L96-L117>
2. <https://github.com/CodeHawks-Contests/2025-04-rock-paper-scissors/blob/main/src/RockPaperScissors.sol#L229-L235>
3. <https://github.com/CodeHawks-Contests/2025-04-rock-paper-scissors/blob/main/src/RockPaperScissors.sol#L262-L267>

```solidity 
function createGameWithEth(uint256 _totalTurns, uint256 _timeoutInterval) external payable returns (uint256) {
        require(msg.value >= minBet, "Bet amount too small");
        require(_totalTurns > 0, "Must have at least one turn");
        require(_totalTurns % 2 == 1, "Total turns must be odd");
        require(_timeoutInterval >= 5 minutes, "Timeout must be at least 5 minutes"); // @audit-issue No upper bound on timeoutInterval

        uint256 gameId = gameCounter++;

        Game storage game = games[gameId];
        game.playerA = msg.sender;
        game.bet = msg.value;
        game.timeoutInterval = _timeoutInterval; // @audit-issue Stored attacker-controlled interval
        game.creationTime = block.timestamp;
        game.joinDeadline = block.timestamp + joinTimeout;
        game.totalTurns = _totalTurns;
        game.currentTurn = 1;
        game.state = GameState.Created;

        emit GameCreated(gameId, msg.sender, msg.value, _totalTurns);

        return gameId;
    }

    function revealMove(uint256 _gameId, uint8 _move, bytes32 _salt) external {
        // ... validation ...
        require(block.timestamp <= game.revealDeadline, "Reveal phase timed out"); // @audit-info Depends on attacker-controlled deadline
        // ... logic ...
    }

    function timeoutReveal(uint256 _gameId) external {
        // ... validation ...
        require(block.timestamp > game.revealDeadline, "Reveal phase not timed out yet"); // @audit-info Can only be called after attacker-controlled deadline
        // ... logic ...
    }

```


## Exploitation Procedure 
Prerequisites:

Player A (attacker) wants to lock up Player B's funds/tokens.

Steps:

1. Player A calls createGameWithEth (or createGameWithToken), providing a valid \_totalTurns and an extremely large value for \_timeoutInterval (e.g., 100 years). Player A sends the required bet amount or transfers the token.
2. Player B joins the game by calling joinGameWithEth (or joinGameWithToken), sending the required bet amount or transferring the token. Player B's funds/tokens are now held by the contract.
3. Player A and Player B both call commitMove for the first turn. The game state transitions to Committed, and the revealDeadline is set to block.timestamp + extremely\_large\_timeoutInterval.
4. Player A calls revealMove.
5. Player B fails to call revealMove for the current turn.
6. Player A (or anyone attempting to resolve the game) tries to call timeoutReveal. This call will revert with "Reveal phase not timed out yet" because block.timestamp is not greater than the revealDeadline (which is set far in the future).
7. Player B's funds/tokens (and potentially Player A's, although A could win by timeout eventually) are now locked in the contract until the revealDeadline passes.


# Impact 

The vulnerability doesn't allow direct theft of funds, but it enables griefing and denial of access to funds/tokens for the opposing player (and potentially the creator if the opponent acts maliciously) for an arbitrarily long, attacker-defined duration. This makes the game impractical or unusable if a malicious player decides to lock up deposits.

# Recomendations

Recommendations Implement an upper bound for the _timeoutInterval parameter in createGameWithEth and createGameWithToken. The maximum value should be reasonable for a game (e.g., a few days or weeks, not years).

