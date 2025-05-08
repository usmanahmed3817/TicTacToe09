/**
 * Futuristic Tic-Tac-Toe Game
 * 
 * A modern implementation of the classic game with pause/resume functionality
 * and futuristic UI design with background music
 */

document.addEventListener('DOMContentLoaded', () => {
    // Initialize AdMob
    let interstitialAd = null;
    
    // Initialize banner ad
    (adsbygoogle = window.adsbygoogle || []).push({});
    
    // Setup interstitial ad
    function initializeInterstitialAd() {
        if (typeof google !== 'undefined' && google.ima) {
            const adContainer = document.createElement('div');
            adContainer.style.position = 'absolute';
            adContainer.style.display = 'none';
            document.body.appendChild(adContainer);
            
            const adDisplayContainer = new google.ima.AdDisplayContainer(adContainer);
            adDisplayContainer.initialize();
            
            const adsLoader = new google.ima.AdsLoader(adDisplayContainer);
            const adsRequest = new google.ima.AdsRequest();
            adsRequest.adTagUrl = 'https://googleads.g.doubleclick.net/pagead/ads?ad_type=video&client=ca-pub-9474678817392803&slotname=3267246419';
            
            adsLoader.requestAds(adsRequest);
            
            adsLoader.addEventListener(
                google.ima.AdsManagerLoadedEvent.Type.ADS_MANAGER_LOADED,
                function(adsManagerLoadedEvent) {
                    interstitialAd = adsManagerLoadedEvent.getAdsManager(adContainer);
                }
            );
        }
    }
    
    // Show interstitial ad
    function showInterstitialAd() {
        if (interstitialAd) {
            try {
                interstitialAd.init(window.innerWidth, window.innerHeight, google.ima.ViewMode.NORMAL);
                interstitialAd.start();
            } catch (adError) {
                console.error("AdError:", adError);
            }
        }
    }
    
    // Try to initialize ads after user interaction
    setTimeout(() => {
        initializeInterstitialAd();
    }, 1000);
    
    // Game audio setup using Tone.js
    const synth = new Tone.Synth({
        oscillator: {
            type: 'sine'
        },
        envelope: {
            attack: 0.005,
            decay: 0.1,
            sustain: 0.3,
            release: 0.8
        }
    }).toDestination();

    const fxSynth = new Tone.FMSynth({
        harmonicity: 3,
        modulationIndex: 10,
        oscillator: {
            type: 'sine'
        },
        envelope: {
            attack: 0.001,
            decay: 0.1,
            sustain: 0.1,
            release: 0.5
        },
        modulation: {
            type: 'sine'
        },
        modulationEnvelope: {
            attack: 0.001,
            decay: 0.5,
            sustain: 0.2,
            release: 0.1
        }
    }).toDestination();
    
    // Background music setup
    const backgroundMusic = new Tone.PolySynth(Tone.Synth, {
        oscillator: {
            type: 'sine'
        },
        envelope: {
            attack: 0.02,
            decay: 0.1,
            sustain: 0.3,
            release: 1
        }
    }).toDestination();
    
    // Create reverb effect for ambient sound
    const reverb = new Tone.Reverb({
        decay: 5,
        wet: 0.6
    }).toDestination();
    
    // Connect background music to reverb
    backgroundMusic.connect(reverb);
    
    // Set volume for background music (lower than game sounds)
    backgroundMusic.volume.value = -20;

    // Game elements
    const cells = document.querySelectorAll('.grid-cell');
    const statusMessage = document.getElementById('status-message');
    const pauseBtn = document.getElementById('pause-btn');
    const resumeBtn = document.getElementById('resume-btn');
    const resetBtn = document.getElementById('reset-btn');
    const pauseOverlay = document.getElementById('game-pause-overlay');
    const winOverlay = document.getElementById('win-overlay');
    const winText = document.getElementById('win-text');
    const newGameBtn = document.getElementById('new-game-btn');
    const musicVolumeSlider = document.getElementById('music-volume');
    
    // Game mode elements
    const onePlayerBtn = document.getElementById('one-player-btn');
    const twoPlayerBtn = document.getElementById('two-player-btn');
    const difficultySelection = document.getElementById('difficulty-selection');
    const easyBtn = document.getElementById('easy-btn');
    const mediumBtn = document.getElementById('medium-btn');
    const hardBtn = document.getElementById('hard-btn');

    // Game state
    let gameState = {
        board: Array(9).fill(''),
        currentPlayer: 'x',
        isGameOver: false,
        isPaused: false,
        winner: null,
        winningCombination: null,
        isOnePlayerMode: true, // Default to one player mode
        difficulty: 'medium', // Default difficulty level (easy, medium, hard)
        // For the AI, player 'x' is human, 'o' is AI
        humanPlayer: 'x',
        aiPlayer: 'o'
    };

    // Winning combinations (indices of the board array)
    const winningCombinations = [
        [0, 1, 2], [3, 4, 5], [6, 7, 8], // Rows
        [0, 3, 6], [1, 4, 7], [2, 5, 8], // Columns
        [0, 4, 8], [2, 4, 6]             // Diagonals
    ];

    // Audio note mappings for game sounds
    const notes = {
        x: 'C5',
        o: 'E5',
        win: ['C4', 'E4', 'G4', 'C5'],
        draw: ['E4', 'D4', 'C4'],
        pause: 'G3',
        resume: 'G4',
        reset: 'C3'
    };

    // Background music pattern
    let bgMusicPattern = [];
    
    // Initialize and start background music
    function initBackgroundMusic() {
        // Create a ambient futuristic music pattern
        const chords = [
            ["C4", "E4", "G4"],      // C major
            ["A3", "C4", "E4"],      // A minor
            ["F3", "A3", "C4"],      // F major
            ["G3", "B3", "D4"]       // G major
        ];
        
        // Create a repeating pattern with timing
        const now = Tone.now();
        let time = now;
        
        // Play a sequence of chords with timing to create ambient feeling
        for (let i = 0; i < 16; i++) {
            const chordIndex = i % chords.length;
            // Schedule chord to play
            backgroundMusic.triggerAttackRelease(chords[chordIndex], "2n", time);
            time += 2;
        }
        
        // Set up a loop to keep music going
        Tone.Transport.scheduleRepeat((time) => {
            // Randomly select chords for variation
            const randomChord = chords[Math.floor(Math.random() * chords.length)];
            backgroundMusic.triggerAttackRelease(randomChord, "2n", time);
        }, "2n", now + 32); // Start after initial sequence
        
        // Start the Transport
        Tone.Transport.start();
    }
    
    // Control music volume
    function handleVolumeChange(e) {
        const volume = e.target.value;
        // Convert slider value (0-100) to decibels (-60 to 0)
        // -60dB is very quiet, 0dB is full volume
        const db = volume === '0' ? -Infinity : -60 + (volume / 100 * 60);
        backgroundMusic.volume.value = db;
    }
    
    // Initialize the game
    function initGame() {
        updateBoard();
        cells.forEach(cell => {
            cell.addEventListener('click', handleCellClick);
        });
        
        pauseBtn.addEventListener('click', pauseGame);
        resumeBtn.addEventListener('click', resumeGame);
        resetBtn.addEventListener('click', resetGame);
        newGameBtn.addEventListener('click', resetGame);
        
        // Volume slider control
        musicVolumeSlider.addEventListener('input', handleVolumeChange);
        // Initialize volume from slider's default value
        handleVolumeChange({ target: { value: musicVolumeSlider.value } });
        
        // Start background music when user interacts
        document.body.addEventListener('click', function startAudio() {
            // Initialize audio context on first click (browser requirement)
            if (Tone.context.state !== 'running') {
                Tone.context.resume();
                initBackgroundMusic();
                document.body.removeEventListener('click', startAudio);
            }
        }, { once: false });
        
        updateStatusMessage();
    }

    // Handle cell click
    function handleCellClick(e) {
        if (gameState.isGameOver || gameState.isPaused) return;
        
        const cellIndex = parseInt(e.target.getAttribute('data-index'));
        
        // Check if cell is already filled
        if (gameState.board[cellIndex] !== '') return;
        
        // Update the game state
        gameState.board[cellIndex] = gameState.currentPlayer;
        
        // Play sound for move
        playSound(gameState.currentPlayer);
        
        // Check for winner
        checkGameStatus();
        
        // Switch player if game is not over
        if (!gameState.isGameOver) {
            gameState.currentPlayer = gameState.currentPlayer === 'x' ? 'o' : 'x';
        }
        
        // Update UI
        updateBoard();
        updateStatusMessage();
    }

    // Update the game board UI based on game state
    function updateBoard() {
        cells.forEach((cell, index) => {
            // Clear existing classes
            cell.classList.remove('x', 'o');
            
            // Add class based on current state
            if (gameState.board[index] !== '') {
                cell.classList.add(gameState.board[index]);
            }
            
            // Add highlight to winning cells
            if (gameState.winningCombination && 
                gameState.winningCombination.includes(index)) {
                cell.classList.add('winning-cell');
            } else {
                cell.classList.remove('winning-cell');
            }
        });
    }

    // Check if there's a winner or draw
    function checkGameStatus() {
        // Check for winner
        for (const combo of winningCombinations) {
            const [a, b, c] = combo;
            if (gameState.board[a] && 
                gameState.board[a] === gameState.board[b] && 
                gameState.board[a] === gameState.board[c]) {
                
                gameState.isGameOver = true;
                gameState.winner = gameState.currentPlayer;
                gameState.winningCombination = combo;
                
                // Play win sound
                playWinSound();
                
                // Show win animation
                showWinScreen();
                
                return;
            }
        }
        
        // Check for draw
        if (!gameState.board.includes('')) {
            gameState.isGameOver = true;
            gameState.winner = null; // Draw
            
            // Play draw sound
            playDrawSound();
            
            // Update UI for draw
            showDrawScreen();
        }
    }

    // Update status message
    function updateStatusMessage() {
        if (gameState.isGameOver) {
            if (gameState.winner) {
                if (gameState.isOnePlayerMode) {
                    // In one-player mode, show "You Win!" or "AI Wins!"
                    if (gameState.winner === gameState.humanPlayer) {
                        statusMessage.innerHTML = `<span class="player-${gameState.winner}">You Win!</span>`;
                    } else {
                        statusMessage.innerHTML = `<span class="player-${gameState.winner}">AI Wins!</span>`;
                    }
                } else {
                    // In two-player mode, show "Player X/O Wins!"
                    statusMessage.innerHTML = `Player <span class="player-${gameState.winner}">${gameState.winner.toUpperCase()}</span> wins!`;
                }
            } else {
                statusMessage.textContent = "It's a draw!";
            }
        } else {
            if (gameState.isOnePlayerMode) {
                // In one-player mode, show "Your Turn" or "AI's Turn"
                if (gameState.currentPlayer === gameState.humanPlayer) {
                    statusMessage.innerHTML = `<span class="player-${gameState.currentPlayer}">Your Turn</span>`;
                } else {
                    statusMessage.innerHTML = `<span class="player-${gameState.currentPlayer}">AI's Turn</span>`;
                }
            } else {
                // In two-player mode, show "Player X/O's Turn"
                statusMessage.innerHTML = `Player <span class="player-${gameState.currentPlayer}">${gameState.currentPlayer.toUpperCase()}</span>'s turn`;
            }
        }
    }

    // Play sound for moves
    function playSound(player) {
        synth.triggerAttackRelease(notes[player], '0.1');
    }

    // Play win sound sequence
    function playWinSound() {
        const now = Tone.now();
        notes.win.forEach((note, i) => {
            fxSynth.triggerAttackRelease(note, '0.2', now + i * 0.2);
        });
    }

    // Play draw sound sequence
    function playDrawSound() {
        const now = Tone.now();
        notes.draw.forEach((note, i) => {
            synth.triggerAttackRelease(note, '0.2', now + i * 0.2);
        });
    }

    // Show win animation
    function showWinScreen() {
        if (gameState.isOnePlayerMode) {
            // In one-player mode, show "YOU WIN!" or "AI WINS!"
            if (gameState.winner === gameState.humanPlayer) {
                winText.innerHTML = `<span class="player-${gameState.winner}">YOU WIN!</span>`;
                
                // Show interstitial ad when human wins (50% chance)
                if (Math.random() < 0.5) {
                    showInterstitialAd();
                }
            } else {
                winText.innerHTML = `<span class="player-${gameState.winner}">AI WINS!</span>`;
            }
        } else {
            // In two-player mode, show "PLAYER X/O WINS!"
            winText.innerHTML = `PLAYER <span class="player-${gameState.winner}">${gameState.winner.toUpperCase()}</span> WINS!`;
            
            // Show interstitial ad in two-player mode when game ends (40% chance)
            if (Math.random() < 0.4) {
                showInterstitialAd();
            }
        }
        winOverlay.classList.remove('d-none');
    }

    // Show draw animation
    function showDrawScreen() {
        winText.textContent = "IT'S A DRAW!";
        winOverlay.classList.remove('d-none');
        
        // Show interstitial ad when game ends in a draw (40% chance)
        if (Math.random() < 0.4) {
            showInterstitialAd();
        }
    }

    // Pause the game
    function pauseGame() {
        if (gameState.isGameOver) return;
        
        gameState.isPaused = true;
        pauseOverlay.classList.remove('d-none');
        pauseBtn.disabled = true;
        resumeBtn.disabled = false;
        
        // Play pause sound
        synth.triggerAttackRelease(notes.pause, '0.2');
        
        // Pause background music
        Tone.Transport.pause();
        
        // Lower volume for background effect
        backgroundMusic.volume.rampTo(-30, 0.5);
    }

    // Resume the game
    function resumeGame() {
        gameState.isPaused = false;
        pauseOverlay.classList.add('d-none');
        pauseBtn.disabled = false;
        resumeBtn.disabled = true;
        
        // Play resume sound
        synth.triggerAttackRelease(notes.resume, '0.2');
        
        // Resume background music
        Tone.Transport.start();
        
        // Restore volume
        backgroundMusic.volume.rampTo(-20, 0.5);
    }

    // AI Move Functions
    
    // Make an AI move after a delay to make it seem like the AI is thinking
    function makeAIMove() {
        // If it's not AI's turn or game is over, don't make a move
        if (gameState.currentPlayer !== gameState.aiPlayer || gameState.isGameOver || gameState.isPaused) return;
        
        // Add a slight delay to make it seem more natural (500-1500ms)
        const thinkingTime = 500 + Math.random() * 1000;
        
        setTimeout(() => {
            const bestMove = getBestMove();
            
            // Make the move
            if (bestMove !== -1) {
                gameState.board[bestMove] = gameState.aiPlayer;
                
                // Play sound for AI move
                playSound(gameState.aiPlayer);
                
                // Check for winner
                checkGameStatus();
                
                // Switch player if game is not over
                if (!gameState.isGameOver) {
                    gameState.currentPlayer = gameState.humanPlayer;
                }
                
                // Update UI
                updateBoard();
                updateStatusMessage();
            }
        }, thinkingTime);
    }
    
    // Get the best move for AI based on difficulty
    function getBestMove() {
        switch(gameState.difficulty) {
            case 'easy':
                return getEasyMove();
            case 'medium':
                return getMediumMove();
            case 'hard':
                return getHardMove();
            default:
                return getMediumMove();
        }
    }
    
    // Easy: Random available move
    function getEasyMove() {
        const availableMoves = [];
        
        // Find all empty cells
        gameState.board.forEach((cell, index) => {
            if (cell === '') {
                availableMoves.push(index);
            }
        });
        
        // Return a random available move
        if (availableMoves.length > 0) {
            return availableMoves[Math.floor(Math.random() * availableMoves.length)];
        }
        
        return -1; // No available moves
    }
    
    // Medium: Block player from winning or make random move
    function getMediumMove() {
        // 70% chance to use the hard AI, 30% to use easy AI
        if (Math.random() < 0.7) {
            return getHardMove();
        } else {
            return getEasyMove();
        }
    }
    
    // Hard: Minimax algorithm for optimal play
    function getHardMove() {
        // Try to take center first if available
        if (gameState.board[4] === '') {
            return 4;
        }
        
        // Try to find a winning move
        for (let i = 0; i < 9; i++) {
            if (gameState.board[i] === '') {
                gameState.board[i] = gameState.aiPlayer;
                
                if (checkWin(gameState.board, gameState.aiPlayer)) {
                    gameState.board[i] = ''; // Reset the move
                    return i;
                }
                
                gameState.board[i] = ''; // Reset the move
            }
        }
        
        // Try to block player's winning move
        for (let i = 0; i < 9; i++) {
            if (gameState.board[i] === '') {
                gameState.board[i] = gameState.humanPlayer;
                
                if (checkWin(gameState.board, gameState.humanPlayer)) {
                    gameState.board[i] = ''; // Reset the move
                    return i;
                }
                
                gameState.board[i] = ''; // Reset the move
            }
        }
        
        // Take corners if available
        const corners = [0, 2, 6, 8];
        const availableCorners = corners.filter(corner => gameState.board[corner] === '');
        
        if (availableCorners.length > 0) {
            return availableCorners[Math.floor(Math.random() * availableCorners.length)];
        }
        
        // Take any available edge
        const edges = [1, 3, 5, 7];
        const availableEdges = edges.filter(edge => gameState.board[edge] === '');
        
        if (availableEdges.length > 0) {
            return availableEdges[Math.floor(Math.random() * availableEdges.length)];
        }
        
        // If no strategic moves, return any available move
        return getEasyMove();
    }
    
    // Helper function to check if a player has won
    function checkWin(board, player) {
        for (const combo of winningCombinations) {
            const [a, b, c] = combo;
            if (board[a] === player && board[a] === board[b] && board[a] === board[c]) {
                return true;
            }
        }
        return false;
    }
    
    // Handle Game Mode Selection
    function setGameMode(mode) {
        gameState.isOnePlayerMode = mode === 'one-player';
        
        // Update button styles
        if (gameState.isOnePlayerMode) {
            onePlayerBtn.classList.add('selected-mode');
            twoPlayerBtn.classList.remove('selected-mode');
            difficultySelection.style.display = 'flex';
        } else {
            onePlayerBtn.classList.remove('selected-mode');
            twoPlayerBtn.classList.add('selected-mode');
            difficultySelection.style.display = 'none';
            
            // Show interstitial ad when switching to two-player mode (25% chance)
            if (Math.random() < 0.25) {
                showInterstitialAd();
            }
        }
        
        // Reset the game with new mode
        resetGame();
    }
    
    // Handle Difficulty Selection
    function setDifficulty(level) {
        gameState.difficulty = level;
        
        // Update button styles
        easyBtn.classList.remove('selected-difficulty');
        mediumBtn.classList.remove('selected-difficulty');
        hardBtn.classList.remove('selected-difficulty');
        
        switch(level) {
            case 'easy':
                easyBtn.classList.add('selected-difficulty');
                break;
            case 'medium':
                mediumBtn.classList.add('selected-difficulty');
                break;
            case 'hard':
                hardBtn.classList.add('selected-difficulty');
                // Show interstitial ad when player selects hard difficulty (30% chance)
                if (Math.random() < 0.3) {
                    showInterstitialAd();
                }
                break;
        }
        
        // Reset the game with new difficulty
        resetGame();
    }
    
    // Update handleCellClick to accommodate one-player mode
    function handleCellClick(e) {
        if (gameState.isGameOver || gameState.isPaused) return;
        
        // In one-player mode, only allow clicks if it's human's turn
        if (gameState.isOnePlayerMode && gameState.currentPlayer !== gameState.humanPlayer) return;
        
        const cellIndex = parseInt(e.target.getAttribute('data-index'));
        
        // Check if cell is already filled
        if (gameState.board[cellIndex] !== '') return;
        
        // Update the game state
        gameState.board[cellIndex] = gameState.currentPlayer;
        
        // Play sound for move
        playSound(gameState.currentPlayer);
        
        // Check for winner
        checkGameStatus();
        
        // Switch player if game is not over
        if (!gameState.isGameOver) {
            gameState.currentPlayer = gameState.currentPlayer === 'x' ? 'o' : 'x';
            
            // If it's one-player mode and AI's turn, make AI move
            if (gameState.isOnePlayerMode && gameState.currentPlayer === gameState.aiPlayer) {
                updateBoard();
                updateStatusMessage();
                makeAIMove();
                return;
            }
        }
        
        // Update UI
        updateBoard();
        updateStatusMessage();
    }
    
    // Reset the game
    function resetGame() {
        // Show interstitial ad randomly (about 30% of the time)
        if (Math.random() < 0.3) {
            showInterstitialAd();
        }
        
        // Play reset sound
        synth.triggerAttackRelease(notes.reset, '0.2');
        
        // Reset game state while preserving mode and difficulty settings
        const prevMode = gameState.isOnePlayerMode;
        const prevDifficulty = gameState.difficulty;
        
        gameState = {
            board: Array(9).fill(''),
            currentPlayer: 'x',
            isGameOver: false,
            isPaused: false,
            winner: null,
            winningCombination: null,
            isOnePlayerMode: prevMode,
            difficulty: prevDifficulty,
            humanPlayer: 'x',
            aiPlayer: 'o'
        };
        
        // Reset UI
        pauseOverlay.classList.add('d-none');
        winOverlay.classList.add('d-none');
        pauseBtn.disabled = false;
        resumeBtn.disabled = true;
        
        // Update the board and status
        updateBoard();
        updateStatusMessage();
        
        // If one-player mode and AI starts first (not implemented here, both modes start with X)
        // If we want to implement AI starting first, we would add that logic here
    }
    
    // Initialize the game
    function initGame() {
        updateBoard();
        
        // Game board event listeners
        cells.forEach(cell => {
            cell.addEventListener('click', handleCellClick);
        });
        
        // Game control event listeners
        pauseBtn.addEventListener('click', pauseGame);
        resumeBtn.addEventListener('click', resumeGame);
        resetBtn.addEventListener('click', resetGame);
        newGameBtn.addEventListener('click', resetGame);
        
        // Game mode event listeners
        onePlayerBtn.addEventListener('click', () => setGameMode('one-player'));
        twoPlayerBtn.addEventListener('click', () => setGameMode('two-player'));
        
        // Difficulty selection event listeners
        easyBtn.addEventListener('click', () => setDifficulty('easy'));
        mediumBtn.addEventListener('click', () => setDifficulty('medium'));
        hardBtn.addEventListener('click', () => setDifficulty('hard'));
        
        // Set initial mode and difficulty UI state
        setGameMode('one-player');
        setDifficulty('medium');
        
        // Volume slider control
        musicVolumeSlider.addEventListener('input', handleVolumeChange);
        // Initialize volume from slider's default value
        handleVolumeChange({ target: { value: musicVolumeSlider.value } });
        
        // Start background music when user interacts
        document.body.addEventListener('click', function startAudio() {
            // Initialize audio context on first click (browser requirement)
            if (Tone.context.state !== 'running') {
                Tone.context.resume();
                initBackgroundMusic();
                document.body.removeEventListener('click', startAudio);
            }
        }, { once: false });
        
        updateStatusMessage();
    }
    
    // Initialize the game
    initGame();
});
