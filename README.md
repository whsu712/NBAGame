# NBAGame
BaseNBAGame
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract BaseNBAGame is Ownable, ReentrancyGuard {

    uint256 public constant MIN_BET = 0.001 ether;
    uint256 public constant MAX_BET = 0.5 ether;
    uint256 public houseEdge = 400; // 4%

    uint256 public totalGames;

    enum Team { 
        Lakers, Warriors, Celtics, Knicks, Bulls, 
        Heat, Suns, Nuggets, Mavericks, Bucks 
    }

    struct NBAGame {
        address player1;
        address player2;
        uint256 betAmount;
        Team team1;
        Team team2;
        Team winner;
        bool isFinished;
    }

    mapping(uint256 => NBAGame) public games;
    mapping(address => uint256) public playerWins;
    mapping(address => uint256) public playerGamesPlayed;

    event GameCreated(uint256 indexed gameId, address indexed player1, Team team1, uint256 betAmount);
    event GameJoined(uint256 indexed gameId, address indexed player2, Team team2);
    event GameEnded(uint256 indexed gameId, Team winner, address winnerAddress, uint256 payout);

    constructor() Ownable(msg.sender) {}

    // Player 1 creates a game and picks a team
    function createGame(Team myTeam) external payable nonReentrant {
        require(msg.value >= MIN_BET && msg.value <= MAX_BET, "Bet out of range");
        require(uint256(myTeam) <= 9, "Invalid team");

        totalGames++;
        games[totalGames] = NBAGame({
            player1: msg.sender,
            player2: address(0),
            betAmount: msg.value,
            team1: myTeam,
            team2: Team.Lakers, // placeholder
            winner: Team.Lakers,
            isFinished: false
        });

        emit GameCreated(totalGames, msg.sender, myTeam, msg.value);
    }

    // Player 2 joins and picks their team
    function joinGame(uint256 gameId, Team myTeam) external payable nonReentrant {
        NBAGame storage game = games[gameId];
        
        require(!game.isFinished, "Game already finished");
        require(game.player2 == address(0), "Game is full");
        require(msg.value == game.betAmount, "Bet must match");
        require(uint256(myTeam) <= 9, "Invalid team");
        require(msg.sender != game.player1, "Cannot play yourself");

        game.player2 = msg.sender;
        game.team2 = myTeam;

        emit GameJoined(gameId, msg.sender, myTeam);

        _resolveGame(gameId);
    }

    function _resolveGame(uint256 gameId) internal {
        NBAGame storage game = games[gameId];
        
        Team winningTeam = _simulateGame(game.team1, game.team2);
        
        uint256 payout = (game.betAmount * 2 * (10000 - houseEdge)) / 10000;
        address winnerAddress;

        if (winningTeam == game.team1) {
            winnerAddress = game.player1;
            playerWins[game.player1]++;
        } else {
            winnerAddress = game.player2;
            playerWins[game.player2]++;
        }

        require(address(this).balance >= payout, "Insufficient balance");
        
        (bool success, ) = payable(winnerAddress).call{value: payout}("");
        require(success, "Payout failed");

        game.winner = winningTeam;
        game.isFinished = true;

        playerGamesPlayed[game.player1]++;
        playerGamesPlayed[game.player2]++;

        emit GameEnded(gameId, winningTeam, winnerAddress, payout);
    }

    // Simple but fun NBA simulation logic
    function _simulateGame(Team a, Team b) internal pure returns (Team) {
        if (a == b) {
            return a; // Same team pick = draw (refund in real version)
        }

        // Famous rivalries and strength bias
        if ((a == Team.Lakers && b == Team.Warriors) ||
            (a == Team.Celtics && b == Team.Knicks) ||
            (a == Team.Bucks && b == Team.Heat)) {
            return a;
        }

        // Random-like but deterministic based on enum order
        uint256 strengthA = (uint256(a) * 7 + 13) % 100;
        uint256 strengthB = (uint256(b) * 11 + 17) % 100;

        return strengthA >= strengthB ? a : b;
    }

    function setHouseEdge(uint256 newEdge) external onlyOwner {
        require(newEdge <= 600, "House edge too high");
        houseEdge = newEdge;
    }

    function withdrawFunds(uint256 amount) external onlyOwner nonReentrant {
        require(amount <= address(this).balance, "Insufficient balance");
        (bool success, ) = payable(owner()).call{value: amount}("");
        require(success, "Withdraw failed");
    }

    function getGame(uint256 gameId) external view returns (NBAGame memory) {
        return games[gameId];
    }

    function getPlayerStats(address player) external view returns (
        uint256 wins,
        uint256 gamesPlayed
    ) {
        return (playerWins[player], playerGamesPlayed[player]);
    }

    receive() external payable {}
}
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract BaseNBAGamePoints is Ownable, ReentrancyGuard {

    uint256 public constant STARTING_POINTS = 1000;
    uint256 public constant BET_POINTS = 100;        // 每场对战消耗100积分
    uint256 public constant WIN_REWARD = 180;        // 赢了获得180积分（净赚80）

    uint256 public totalGames;

    enum Team { 
        Lakers, Warriors, Celtics, Knicks, Bulls, 
        Heat, Suns, Nuggets, Mavericks, Bucks 
    }

    struct NBAGame {
        address player1;
        address player2;
        Team team1;
        Team team2;
        Team winner;
        bool isFinished;
    }

    mapping(address => uint256) public playerPoints;
    mapping(address => uint256) public playerWins;
    mapping(address => uint256) public playerGamesPlayed;
    mapping(uint256 => NBAGame) public games;

    event GameCreated(uint256 indexed gameId, address indexed player1, Team team1);
    event GameJoined(uint256 indexed gameId, address indexed player2, Team team2);
    event GameEnded(uint256 indexed gameId, Team winner, address winnerAddress);

    constructor() Ownable(msg.sender) {}

    // 首次玩自动领取初始积分
    function _initPlayer(address player) internal {
        if (playerPoints[player] == 0) {
            playerPoints[player] = STARTING_POINTS;
        }
    }

    function createGame(Team myTeam) external nonReentrant {
        _initPlayer(msg.sender);
        require(playerPoints[msg.sender] >= BET_POINTS, "Not enough points");
        require(uint256(myTeam) <= 9, "Invalid team");

        playerPoints[msg.sender] -= BET_POINTS;

        totalGames++;
        games[totalGames] = NBAGame({
            player1: msg.sender,
            player2: address(0),
            team1: myTeam,
            team2: Team.Lakers,
            winner: Team.Lakers,
            isFinished: false
        });

        emit GameCreated(totalGames, msg.sender, myTeam);
    }

    function joinGame(uint256 gameId, Team myTeam) external nonReentrant {
        NBAGame storage game = games[gameId];
        
        require(!game.isFinished, "Game already finished");
        require(game.player2 == address(0), "Game is full");
        require(msg.sender != game.player1, "Cannot play yourself");
        require(uint256(myTeam) <= 9, "Invalid team");

        _initPlayer(msg.sender);
        require(playerPoints[msg.sender] >= BET_POINTS, "Not enough points");

        playerPoints[msg.sender] -= BET_POINTS;
        game.player2 = msg.sender;
        game.team2 = myTeam;

        emit GameJoined(gameId, msg.sender, myTeam);

        _resolveGame(gameId);
    }

    function _resolveGame(uint256 gameId) internal {
        NBAGame storage game = games[gameId];
        
        Team winningTeam = _simulateGame(game.team1, game.team2);
        address winnerAddress;

        if (winningTeam == game.team1) {
            winnerAddress = game.player1;
            playerWins[game.player1]++;
            playerPoints[game.player1] += WIN_REWARD;
        } else {
            winnerAddress = game.player2;
            playerWins[game.player2]++;
            playerPoints[game.player2] += WIN_REWARD;
        }

        game.winner = winningTeam;
        game.isFinished = true;

        playerGamesPlayed[game.player1]++;
        playerGamesPlayed[game.player2]++;

        emit GameEnded(gameId, winningTeam, winnerAddress);
    }

    function _simulateGame(Team a, Team b) internal pure returns (Team) {
        if (a == b) return a;
        
        uint256 scoreA = (uint256(a) * 17 + 23) % 100;
        uint256 scoreB = (uint256(b) * 13 + 37) % 100;
        
        return scoreA >= scoreB ? a : b;
    }

    // 查询自己积分和战绩
    function getMyStats() external view returns (uint256 points, uint256 wins, uint256 gamesPlayed) {
        return (playerPoints[msg.sender], playerWins[msg.sender], playerGamesPlayed[msg.sender]);
    }

    function getGame(uint256 gameId) external view returns (NBAGame memory) {
        return games[gameId];
    }

    // Owner functions
    function givePoints(address player, uint256 amount) external onlyOwner {
        playerPoints[player] += amount;
    }
}
