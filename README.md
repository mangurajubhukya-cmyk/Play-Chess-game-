https://mangurajubhukya-cmyk.github.io/Play-Chess-game-/

# Play-Chess-game-
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Chess Training Center</title>
    <!-- Load Tailwind CSS from CDN for reliable styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom styles for piece colors and responsiveness */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #111827; /* Gray-900 background */
        }
        #app-container {
            min-height: 100vh;
            display: flex;
            flex-direction: column;
            align-items: center;
            padding-bottom: 3rem;
        }
        .piece {
            transition: transform 0.1s ease-out;
            text-shadow: 1px 1px 3px rgba(0,0,0,0.5);
            font-size: clamp(30px, 10vw, 60px); /* Responsive font size for pieces */
        }
        .text-white-piece { color: #fefefe; } /* Light pieces */
        .text-black-piece { color: #1f2937; } /* Dark pieces */
        #chess-board {
            box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.4), 0 4px 6px -2px rgba(0, 0, 0, 0.2);
            width: 100%;
            max-width: 500px;
            aspect-ratio: 1 / 1;
        }
        .cell {
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            user-select: none;
            transition: background-color 0.1s;
        }
        .cell:hover {
            opacity: 0.9;
        }
    </style>
</head>
<body>

    <div id="app-container">
        <!-- Header -->
        <header id="app-header" class="bg-gray-900 shadow-xl p-4 flex justify-between items-center w-full sticky top-0 z-10 border-b-4 border-indigo-700 max-w-7xl">
            <div class="text-white text-2xl font-extrabold cursor-pointer" onclick="navigate('HOME')">
                <span class="text-indigo-400">Chess</span>Dev
            </div>
            <button
                onclick="navigate('HOME')"
                class="px-4 py-2 bg-indigo-600 text-white rounded-lg text-sm font-semibold shadow-md hover:bg-indigo-700 transition"
            >
                Main Menu
            </button>
        </header>

        <!-- Main Content Area -->
        <main id="main-content" class="pt-8 pb-12 w-full flex justify-center flex-1"></main>
    </div>

    <script>
        // --- Global State ---
        let currentPage = 'HOME';
        let aiDifficulty = 'easy';
        let rankData = loadRankData();
        
        // Game State Variables
        let board = [];
        let turn = 'white';
        let selectedId = null;
        let validMoves = [];
        let captured = [];
        let gameOver = null;
        let message = '';

        // --- Core Game Logic and Utilities (Same as before) ---
        const BOARD_SIZE = 8;
        const PIECES = {
            'R': '♜', 'N': '♞', 'B': '♝', 'Q': '♛', 'K': '♚', 'P': '♟', // Black (Uppercase in array)
            'r': '♖', 'n': '♘', 'b': '♗', 'q': '♕', 'k': '♔', 'p': '♙'  // White (Lowercase in array)
        };
        const INITIAL_BOARD_FEN = 'rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR';

        const PIECE_VALUES = {
            'P': 10, 'N': 30, 'B': 30, 'R': 50, 'Q': 90, 'K': 900,
            'p': 10, 'n': 30, 'b': 30, 'r': 50, 'q': 90, 'k': 900
        };

        const INITIAL_BOARD = [
            ['R', 'N', 'B', 'Q', 'K', 'B', 'N', 'R'],
            ['P', 'P', 'P', 'P', 'P', 'P', 'P', 'P'],
            ['', '', '', '', '', '', '', ''],
            ['', '', '', '', '', '', '', ''],
            ['', '', '', '', '', '', '', ''],
            ['', '', '', '', '', '', '', ''],
            ['p', 'p', 'p', 'p', 'p', 'p', 'p', 'p'],
            ['r', 'n', 'b', 'q', 'k', 'b', 'n', 'r']
        ];
        
        // --- Board Coordinate Helpers ---
        const idToCoords = (id) => {
            const file = id.charAt(0).toUpperCase();
            const rank = parseInt(id.charAt(1));
            const col = file.charCodeAt(0) - 'A'.charCodeAt(0);
            const row = BOARD_SIZE - rank;
            return [row, col];
        };

        const coordsToId = (row, col) => {
            const file = String.fromCharCode('A'.charCodeAt(0) + col);
            const rank = BOARD_SIZE - row;
            return `${file}${rank}`;
        };

        const getPieceColor = (piece) => {
            if (!piece) return null;
            return piece === piece.toLowerCase() ? 'white' : 'black';
        };

        const isPathBlocked = (r1, c1, r2, c2, board) => {
            const dr = Math.sign(r2 - r1);
            const dc = Math.sign(c2 - c1);
            let r = r1 + dr;
            let c = c1 + dc;
            while (r !== r2 || c !== c2) {
                if (board[r][c]) {
                    return true;
                }
                r += dr;
                c += dc;
            }
            return false;
        };

        const getValidMoves = (r, c, board) => {
            const piece = board[r][c];
            if (!piece) return [];

            const pieceType = piece.toUpperCase();
            const color = getPieceColor(piece);
            const potentialMoves = [];

            const addSlidingMoves = (directions) => {
                for (const [dr, dc] of directions) {
                    let nextR = r + dr;
                    let nextC = c + dc;
                    while (nextR >= 0 && nextR < BOARD_SIZE && nextC >= 0 && nextC < BOARD_SIZE) {
                        const targetPiece = board[nextR][nextC];
                        if (targetPiece) {
                            if (getPieceColor(targetPiece) !== color) {
                                potentialMoves.push(coordsToId(nextR, nextC));
                            }
                            break;
                        }
                        potentialMoves.push(coordsToId(nextR, nextC));
                        nextR += dr;
                        nextC += dc;
                    }
                }
            };

            const addStepMoves = (deltas) => {
                for (const [dr, dc] of deltas) {
                    const nextR = r + dr;
                    const nextC = c + dc;
                    if (nextR >= 0 && nextR < BOARD_SIZE && nextC >= 0 && nextC < BOARD_SIZE) {
                        const targetPiece = board[nextR][nextC];
                        if (!targetPiece || getPieceColor(targetPiece) !== color) {
                            potentialMoves.push(coordsToId(nextR, nextC));
                        }
                    }
                }
            };

            switch (pieceType) {
                case 'P': // Pawn
                    const direction = (color === 'white') ? -1 : 1;
                    const startRank = (color === 'white') ? 6 : 1;
                    let moveR = r + direction;
                    if (moveR >= 0 && moveR < BOARD_SIZE && !board[moveR][c]) {
                        potentialMoves.push(coordsToId(moveR, c));
                        if (r === startRank) {
                            let doubleR = r + 2 * direction;
                            if (!board[doubleR][c] && !board[moveR][c]) { // Check for blockage on single step
                                potentialMoves.push(coordsToId(doubleR, c));
                            }
                        }
                    }
                    for (const dc of [-1, 1]) {
                        const capR = r + direction;
                        const capC = c + dc;
                        if (capR >= 0 && capR < BOARD_SIZE && capC >= 0 && capC < BOARD_SIZE) {
                            const targetPiece = board[capR][capC];
                            if (targetPiece && getPieceColor(targetPiece) !== color) {
                                potentialMoves.push(coordsToId(capR, capC));
                            }
                        }
                    }
                    break;
                case 'R': addSlidingMoves([[0, 1], [0, -1], [1, 0], [-1, 0]]); break;
                case 'B': addSlidingMoves([[1, 1], [1, -1], [-1, 1], [-1, -1]]); break;
                case 'Q': addSlidingMoves([[0, 1], [0, -1], [1, 0], [-1, 0], [1, 1], [1, -1], [-1, 1], [-1, -1]]); break;
                case 'K': addStepMoves([[0, 1], [0, -1], [1, 0], [-1, 0], [1, 1], [1, -1], [-1, 1], [-1, -1]]); break;
                case 'N': addStepMoves([[2, 1], [2, -1], [-2, 1], [-2, -1], [1, 2], [1, -2], [-1, 2], [-1, -2]]); break;
            }

            return potentialMoves.filter(targetId => {
                const [targetR, targetC] = idToCoords(targetId);
                const pieceType = piece.toUpperCase();
                if (['P', 'N', 'K'].includes(pieceType)) return true;
                return !isPathBlocked(r, c, targetR, targetC, board);
            });
        };

        const getAllMoves = (color, board) => {
            const allMoves = [];
            for (let r = 0; r < BOARD_SIZE; r++) {
                for (let c = 0; c < BOARD_SIZE; c++) {
                    const piece = board[r][c];
                    if (getPieceColor(piece) === color) {
                        const moves = getValidMoves(r, c, board);
                        if (moves.length > 0) {
                            allMoves.push({
                                from: coordsToId(r, c),
                                moves: moves
                            });
                        }
                    }
                }
            }
            return allMoves;
        };

        // --- AI Logic (MiniMax Implementation - simplified for performance) ---
        const simulateMove = (currentBoard, fromId, toId) => {
            const [fromR, fromC] = idToCoords(fromId);
            const [toR, toC] = idToCoords(toId);

            // Deep copy the board
            const newBoard = currentBoard.map(row => [...row]);
            const movedPiece = newBoard[fromR][fromC];

            newBoard[toR][toC] = movedPiece;
            newBoard[fromR][fromC] = '';

            // Pawn Promotion (Auto-Queen)
            if (movedPiece.toUpperCase() === 'P') {
                if (getPieceColor(movedPiece) === 'white' && toR === 0) {
                    newBoard[toR][toC] = 'q';
                } else if (getPieceColor(movedPiece) === 'black' && toR === 7) {
                    newBoard[toR][toC] = 'Q';
                }
            }

            return newBoard;
        };

        const evaluateBoard = (currentBoard) => {
            let score = 0;
            for (let r = 0; r < BOARD_SIZE; r++) {
                for (let c = 0; c < BOARD_SIZE; c++) {
                    const piece = currentBoard[r][c];
                    if (piece) {
                        const value = PIECE_VALUES[piece.toUpperCase()] || 0;
                        if (getPieceColor(piece) === 'black') {
                            score += value; // Black (Maximizer)
                        } else {
                            score -= value; // White (Minimizer)
                        }
                    }
                }
            }
            return score;
        };

        // MiniMax with Alpha-Beta Pruning (Simplified for fast execution)
        const minimax = (currentBoard, depth, isMaximizingPlayer, alpha, beta) => {
            if (depth === 0) {
                return evaluateBoard(currentBoard);
            }

            const color = isMaximizingPlayer ? 'black' : 'white';
            const possibleMoves = getAllMoves(color, currentBoard);

            if (possibleMoves.length === 0) {
                // Return a large score indicating checkmate/stalemate
                return isMaximizingPlayer ? -9999 : 9999;
            }

            if (isMaximizingPlayer) {
                let maxEval = -Infinity;
                for (const { from, moves } of possibleMoves) {
                    for (const to of moves) {
                        const newBoard = simulateMove(currentBoard, from, to);
                        const evaluation = minimax(newBoard, depth - 1, false, alpha, beta);
                        maxEval = Math.max(maxEval, evaluation);
                        alpha = Math.max(alpha, evaluation);
                        if (beta <= alpha) break;
                    }
                }
                return maxEval;
            } else { // Minimizing Player (White)
                let minEval = Infinity;
                for (const { from, moves } of possibleMoves) {
                    for (const to of moves) {
                        const newBoard = simulateMove(currentBoard, from, to);
                        const evaluation = minimax(newBoard, depth - 1, true, alpha, beta);
                        minEval = Math.min(minEval, evaluation);
                        beta = Math.min(beta, evaluation);
                        if (beta <= alpha) break;
                    }
                }
                return minEval;
            }
        };

        const getAIMove = (currentBoard, difficulty) => {
            // Difficulty mapping: Easy=1, Medium/Hard=2 (2-ply is good balance for browser performance)
            const MAX_DEPTH = (difficulty === 'easy') ? 1 : 2;

            const color = 'black';
            const possibleMoves = getAllMoves(color, currentBoard);

            if (possibleMoves.length === 0) return null;

            let maxEval = -Infinity;
            const bestMoves = [];

            for (const { from: fromId, moves } of possibleMoves) {
                for (const toId of moves) {
                    const newBoard = simulateMove(currentBoard, fromId, toId);
                    // Start search for the opponent (minimizing player)
                    const evaluation = minimax(newBoard, MAX_DEPTH - 1, false, -Infinity, Infinity);

                    if (evaluation > maxEval) {
                        maxEval = evaluation;
                        bestMoves.length = 0;
                        bestMoves.push({ fromId, toId });
                    } else if (evaluation === maxEval) {
                        bestMoves.push({ fromId, toId });
                    }
                }
            }

            if (bestMoves.length > 0) {
                // Select a move randomly among the best moves to add variety
                const randomIndex = Math.floor(Math.random() * bestMoves.length);
                return bestMoves[randomIndex];
            }

            return null;
        };

        // --- Storage for Wins/Ranks (Uses localStorage) ---
        function loadRankData() {
            try {
                const data = localStorage.getItem('chessRanks');
                return data ? JSON.parse(data) : { wins: 0, losses: 0 };
            } catch (e) {
                console.error("Error loading rank data:", e);
                return { wins: 0, losses: 0 };
            }
        }

        function updateRankData(result) {
            const data = loadRankData();
            if (result === 'win') {
                data.wins += 1;
            } else if (result === 'loss') {
                data.losses += 1;
            }
            localStorage.setItem('chessRanks', JSON.stringify(data));
            rankData = data; // Update global state
        }
        
        // --- Game State Management ---
        function resetGame() {
            board = INITIAL_BOARD.map(row => [...row]);
            turn = 'white';
            selectedId = null;
            validMoves = [];
            captured = [];
            gameOver = null;
            message = '';
        }

        function performMove(fromId, toId, gameMode) {
            const [fromR, fromC] = idToCoords(fromId);
            const [toR, toC] = idToCoords(toId);

            const movedPiece = board[fromR][fromC];
            const capturedPiece = board[toR][toC];

            if (capturedPiece) {
                captured.push(capturedPiece);
            }

            board[toR][toC] = movedPiece;
            board[fromR][fromC] = '';

            let promoted = false;
            if (movedPiece.toUpperCase() === 'P') {
                if (getPieceColor(movedPiece) === 'white' && toR === 0) {
                    board[toR][toC] = 'q';
                    promoted = true;
                } else if (getPieceColor(movedPiece) === 'black' && toR === 7) {
                    board[toR][toC] = 'Q';
                    promoted = true;
                }
            }

            if (capturedPiece && capturedPiece.toUpperCase() === 'K') {
                gameOver = getPieceColor(movedPiece);
                message = `Game Over! ${gameOver.toUpperCase()} wins by King Capture!`;
                if (gameMode === 'ai') handleGameOver(gameOver);
            }

            if (!gameOver) {
                turn = (turn === 'white' ? 'black' : 'white');
            }
            selectedId = null;
            validMoves = [];
            message = promoted ? `${getPieceColor(movedPiece).toUpperCase()} Pawn Promoted!` : '';
            
            renderPage(); // Rerender the current view
            
            if (!gameOver && turn === 'black' && gameMode === 'ai') {
                setTimeout(handleAITurn, 500);
            }
        }
        
        function handleGameOver(winner) {
            if (winner === 'white') {
                updateRankData('win');
            } else if (winner === 'black') {
                updateRankData('loss');
            }
            rankData = loadRankData(); // Refresh data for the home view
        }

        function handleAITurn() {
            message = "Computer is thinking...";
            renderBoardContainer(); // Update message while thinking

            setTimeout(() => {
                const move = getAIMove(board, aiDifficulty);

                if (move) {
                    performMove(move.fromId, move.toId, 'ai');
                } else {
                    gameOver = 'draw';
                    message = "Stalemate! (No legal moves for the computer)";
                    handleGameOver('draw');
                    renderBoardContainer();
                }
            }, 500);
        }

        function handleCellClick(id, gameMode) {
            if (gameOver || (gameMode === 'ai' && turn === 'black')) return;

            const [r, c] = idToCoords(id);
            const piece = board[r][c];
            const pieceColor = getPieceColor(piece);

            if (selectedId === null) {
                if (piece && pieceColor === turn) {
                    selectedId = id;
                    validMoves = getValidMoves(r, c, board);
                } else if (piece) {
                    message = `That's the ${pieceColor}'s piece! It's your turn (${turn}).`;
                }
            } else if (selectedId === id) {
                selectedId = null;
                validMoves = [];
            } else if (validMoves.includes(id)) {
                performMove(selectedId, id, gameMode);
                return; // Stop here, renderPage will be called after move
            } else if (piece && pieceColor === turn) {
                selectedId = id;
                validMoves = getValidMoves(r, c, board);
            } else {
                message = "Invalid move or target.";
            }
            
            renderBoardContainer();
        }

        // --- Render Functions (Pure HTML generation) ---

        function getLevelDescription(wins) {
            if (wins >= 20) return "International Master";
            if (wins >= 10) return "National Champion";
            if (wins >= 5) return "State Level";
            return "Local Club Player";
        }

        function renderHomeView() {
            const level = getLevelDescription(rankData.wins);

            return `
                <div class="flex flex-col items-center p-6 bg-gray-800 rounded-xl shadow-2xl m-4 md:m-8 w-full max-w-xl mx-auto">
                    <h1 class="text-3xl font-extrabold text-white mb-8 text-center">Chess Training Center</h1>

                    <div class="w-full p-6 bg-gray-700 rounded-xl shadow-lg mb-8 text-center">
                        <h3 class="text-xl font-bold text-yellow-400">Your Current Rank</h3>
                        <p class="text-3xl font-mono text-white mt-2">${level}</p>
                        <p class="text-sm text-gray-300 mt-2">Wins: ${rankData.wins} | Losses: ${rankData.losses}</p>
                    </div>

                    <div class="flex flex-col gap-4 w-full">
                        <button onclick="navigate('AI_SELECT')" class="py-4 bg-green-600 text-white text-xl font-bold rounded-xl shadow-md hover:bg-green-700 transition duration-300">
                            Practice with AI (Computer)
                        </button>
                        <button onclick="navigate('SELF_PLAY')" class="py-4 bg-blue-600 text-white text-xl font-bold rounded-xl shadow-md hover:bg-blue-700 transition duration-300">
                            Self-Practice (2-Player)
                        </button>
                        <button onclick="navigate('COMPETITION_LEVELS')" class="py-4 bg-indigo-600 text-white text-xl font-bold rounded-xl shadow-md hover:bg-indigo-700 transition duration-300">
                            Competition Levels & Rewards
                        </button>
                    </div>
                </div>
            `;
        }
        
        function renderCapturedPieces() {
            const whitePieces = captured.filter(p => getPieceColor(p) === 'white');
            const blackPieces = captured.filter(p => getPieceColor(p) === 'black');

            const renderSymbols = (pieces) => pieces.map(p => `<span class="mx-0.5">${PIECES[p]}</span>`).join('');

            return `
                <div class="flex flex-col gap-3 p-4 bg-gray-700 rounded-xl shadow-inner w-full">
                    <div class="flex flex-col">
                        <p class="text-sm font-bold text-gray-300">White's Captures (Black Pieces)</p>
                        <div class="text-3xl text-gray-800 flex flex-wrap min-h-8">
                            ${renderSymbols(blackPieces)}
                        </div>
                    </div>
                    <div class="flex flex-col">
                        <p class="text-sm font-bold text-gray-300">Black's Captures (White Pieces)</p>
                        <div class="text-3xl text-white flex flex-wrap min-h-8">
                            ${renderSymbols(whitePieces)}
                        </div>
                    </div>
                </div>
            `;
        }

        function renderBoardContainer() {
            const gameMode = currentPage === 'SELF_PLAY' ? 'self' : 'ai';
            const boardHTML = renderChessBoard(gameMode);
            const statusClass = gameOver ? 'bg-red-700 text-white' : (turn === 'white' ? 'bg-white text-gray-800' : 'bg-gray-800 text-white');
            const messageClass = message ? 'animate-pulse' : '';
            const isAITurn = gameMode === 'ai' && turn === 'black' && !gameOver;
            
            document.getElementById('main-content').innerHTML = `
                <div class="p-4 flex flex-col items-center w-full max-w-6xl">
                    <h2 class="text-3xl font-bold text-white mb-6 text-center">${currentPage === 'SELF_PLAY' ? 'Self-Practice Mode' : `AI Practice: ${aiDifficulty.toUpperCase()}`}</h2>
                    
                    <div class="flex flex-col items-center gap-6 w-full md:flex-row md:items-start md:justify-center md:gap-10">
                        <!-- Left Column (Info/Controls) -->
                        <div class="flex flex-col items-center gap-4 w-full md:max-w-xs lg:max-w-md order-2 md:order-1">
                            <h2 class="text-2xl font-extrabold p-3 rounded-xl shadow-lg w-full text-center transition-colors duration-300 ${statusClass}">
                                ${gameOver ? `${gameOver.toUpperCase()} WINS!` : (isAITurn ? 'COMPUTER IS THINKING...' : `${turn.toUpperCase()}'s Turn`)}
                            </h2>
                            <p id="game-message" class="text-md font-semibold text-yellow-400 ${messageClass}">${message}</p>

                            ${renderCapturedPieces()}

                            <div class="flex gap-4 mt-2 w-full">
                                <button
                                    onclick="restartGame('${gameMode}')"
                                    class="flex-1 px-4 py-3 bg-indigo-600 text-white text-lg font-bold rounded-xl shadow-lg hover:bg-indigo-700 transition duration-200"
                                >
                                    Restart
                                </button>
                            </div>
                            <button onclick="navigate('${gameMode === 'self' ? 'HOME' : 'AI_SELECT'}')" class="mt-4 px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700">
                                &larr; Back
                            </button>
                        </div>

                        <!-- Right Column (Board) -->
                        <div class="w-full max-w-md order-1 md:order-2">
                            ${boardHTML}
                        </div>
                    </div>
                </div>
            `;
            // Attach event listeners after rendering
            attachBoardListeners(gameMode);
        }
        
        function attachBoardListeners(gameMode) {
            const boardEl = document.getElementById('chess-board');
            if (boardEl) {
                boardEl.querySelectorAll('.cell').forEach(cell => {
                    cell.onclick = () => handleCellClick(cell.id, gameMode);
                });
            }
        }

        function renderChessBoard(gameMode) {
            const cells = [];
            const flipped = gameMode === 'self' && turn === 'white';
            const rows = flipped ? [7, 6, 5, 4, 3, 2, 1, 0] : [0, 1, 2, 3, 4, 5, 6, 7];
            const cols = flipped ? [7, 6, 5, 4, 5, 6, 7, 0] : [0, 1, 2, 3, 4, 5, 6, 7]; // Fixed col order for rendering

            rows.forEach(r => {
                cols.forEach(c => {
                    const id = coordsToId(r, c);
                    const pieceKey = board[r][c];
                    const pieceSymbol = PIECES[pieceKey] || '';
                    const isWhiteSquare = (r + c) % 2 === 0;
                    
                    let cellClasses = `cell flex items-center justify-center text-5xl select-none h-full w-full ${
                        isWhiteSquare ? 'bg-gray-200 hover:bg-gray-300' : 'bg-gray-500 hover:bg-gray-600'
                    }`;

                    let pieceClasses = `piece ${pieceKey === pieceKey.toLowerCase() ? 'text-white-piece' : 'text-black-piece'}`;

                    if (id === selectedId) {
                        cellClasses = cellClasses.replace(/bg-gray-200|bg-gray-500|hover:bg-gray-300|hover:bg-gray-600/, 'bg-emerald-500');
                        cellClasses += ' ring-4 ring-emerald-700 scale-[1.03] shadow-inner';
                    }

                    if (validMoves.includes(id)) {
                        const isCapture = pieceKey !== '';
                        cellClasses = cellClasses.replace(/bg-gray-200|bg-gray-500|hover:bg-gray-300|hover:bg-gray-600|bg-emerald-500/, isCapture ? 'bg-red-400' : 'bg-yellow-400');
                        if (isCapture) cellClasses += ' ring-4 ring-red-700';
                        else cellClasses += ' ring-4 ring-yellow-700';
                    }

                    cells.push(
                        `<div id="${id}" class="${cellClasses}">
                            <span class="${pieceClasses}">${pieceSymbol}</span>
                        </div>`
                    );
                });
            });

            return `
                <div
                    id="chess-board"
                    class="grid grid-cols-8 grid-rows-8 w-full max-w-lg border-8 border-gray-700 shadow-2xl rounded-lg overflow-hidden"
                >
                    ${cells.join('')}
                </div>
            `;
        }

        function renderAISelectView() {
            const difficulties = [
                { level: 'Beginner', desc: '1-Ply search (Easy to beat).', color: 'bg-green-600', difficulty: 'easy' },
                { level: 'Intermediate', desc: '2-Ply search (A decent challenge).', color: 'bg-yellow-600', difficulty: 'medium' },
                { level: 'Advanced', desc: '2-Ply search (Play carefully!).', color: 'bg-red-600', difficulty: 'hard' },
            ];
            
            document.getElementById('main-content').innerHTML = `
                <div class="flex flex-col items-center p-6 bg-gray-800 rounded-xl shadow-2xl m-4 md:m-8 w-full max-w-xl mx-auto">
                    <h2 class="text-3xl font-bold text-white mb-8 text-center">Select AI Difficulty</h2>
                    <div class="flex flex-col gap-6 w-full">
                        ${difficulties.map(d => `
                            <div 
                                class="p-5 rounded-xl shadow-lg text-center cursor-pointer transform hover:scale-[1.02] transition duration-200 ${d.color}"
                                onclick="selectDifficulty('${d.difficulty}')">
                                <h3 class="text-2xl font-bold text-white">${d.level}</h3>
                                <p class="text-sm text-gray-100 mt-1">${d.desc}</p>
                            </div>
                        `).join('')}
                    </div>
                    <button onclick="navigate('HOME')" class="mt-10 px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700">
                        &larr; Back to Main Menu
                    </button>
                </div>
            `;
        }
        
        function renderCompetitionLevelsView() {
            const levels = [
                { name: "Local Club Player", threshold: 0, reward: "Master basic openings and checkmate patterns." },
                { name: "State Level", threshold: 5, reward: "Access to advanced game analysis tools." },
                { name: "National Champion", threshold: 10, reward: "Exclusive Gold board skin and title." },
                { name: "International Master", threshold: 20, reward: "Permanently unlock all future features and skins." },
            ];

            const currentWins = rankData.wins;
            
            document.getElementById('main-content').innerHTML = `
                <div class="flex flex-col items-center p-6 bg-gray-800 rounded-xl shadow-2xl m-4 md:m-8 w-full max-w-xl mx-auto">
                    <h2 class="text-3xl font-bold text-white mb-8 text-center">Competition Levels & Rewards</h2>
                    <div class="space-y-6 w-full">
                        ${levels.map((level, index) => {
                            const isUnlocked = currentWins >= level.threshold;
                            const nextThreshold = levels[index + 1] ? levels[index + 1].threshold : null;
                            const progress = nextThreshold ? Math.min(100, (currentWins / nextThreshold) * 100) : 100;
                            const levelClass = isUnlocked ? 'bg-indigo-600' : 'bg-gray-700';

                            let progressHTML = '';
                            if (isUnlocked && nextThreshold) {
                                progressHTML = `
                                    <div class="mt-3">
                                        <p class="text-xs text-white mb-1">Progress to Next Rank (${nextThreshold} Wins):</p>
                                        <div class="w-full bg-gray-500 rounded-full h-2.5">
                                            <div
                                                class="bg-yellow-400 h-2.5 rounded-full"
                                                style="width: ${progress}%"
                                            ></div>
                                        </div>
                                    </div>
                                `;
                            }

                            return `
                                <div class="p-5 rounded-xl shadow-xl transition-all duration-300 ${levelClass}">
                                    <div class="flex justify-between items-center mb-2">
                                        <h3 class="text-2xl font-extrabold ${isUnlocked ? 'text-yellow-300' : 'text-white'}">${level.name}</h3>
                                        <p class="text-sm font-bold ${isUnlocked ? 'text-white' : 'text-gray-300'}">
                                            ${isUnlocked ? 'UNLOCKED' : `Requires ${level.threshold} Wins`}
                                        </p>
                                    </div>
                                    <p class="text-md ${isUnlocked ? 'text-gray-100' : 'text-gray-400'}">
                                        **Reward:** ${level.reward}
                                    </p>
                                    ${progressHTML}
                                </div>
                            `;
                        }).join('')}
                    </div>
                    <button onclick="navigate('HOME')" class="mt-10 px-6 py-2 bg-gray-600 text-white rounded-lg hover:bg-gray-700">
                        &larr; Back to Main Menu
                    </button>
                </div>
            `;
        }

        // --- Navigation and Restart Handlers ---
        
        function restartGame(mode) {
            resetGame();
            if (mode === 'self') {
                navigate('SELF_PLAY');
            } else {
                navigate('AI_PLAY');
            }
        }

        function selectDifficulty(difficulty) {
            aiDifficulty = difficulty;
            resetGame();
            navigate('AI_PLAY');
        }

        function navigate(page) {
            currentPage = page;
            const mainContent = document.getElementById('main-content');
            
            switch (currentPage) {
                case 'HOME':
                    mainContent.innerHTML = renderHomeView();
                    break;
                case 'SELF_PLAY':
                case 'AI_PLAY':
                    // Reset game state when entering a new game mode
                    if (board.length === 0 || gameOver !== null) resetGame();
                    renderBoardContainer();
                    break;
                case 'AI_SELECT':
                    renderAISelectView();
                    break;
                case 'COMPETITION_LEVELS':
                    // Reload rank data before rendering
                    rankData = loadRankData(); 
                    renderCompetitionLevelsView();
                    break;
                default:
                    mainContent.innerHTML = renderHomeView();
            }
        }

        // --- Initialization ---
        window.onload = () => {
            navigate('HOME');
        };
    </script>
</body>
</html>

