<!DOCTYPE html>
<html lang="mr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gokul Race - Accessible Audio Racing Game</title>
    <style>
        * {
            box-sizing: border-box;
            user-select: none;
            -webkit-user-select: none;
        }
        body {
            margin: 0;
            padding: 16px;
            background-color: #121212;
            color: #ffffff;
            font-family: system-ui, -apple-system, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: space-between;
            min-height: 100vh;
        }
        header {
            text-align: center;
            margin-top: 10px;
        }
        h1 {
            font-size: 26px;
            margin: 0 0 8px 0;
            color: #ffcc00;
        }
        #status {
            font-size: 20px;
            font-weight: bold;
            min-height: 30px;
            color: #00ffcc;
        }
        #score-board {
            font-size: 18px;
            color: #eeeeee;
        }
        .controls-grid {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 16px;
            width: 100%;
            max-width: 420px;
            margin: 20px 0;
        }
        button {
            border: none;
            border-radius: 12px;
            font-weight: bold;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            text-align: center;
        }
        .main-btn {
            background-color: #ffcc00;
            color: #121212;
            font-size: 24px;
            padding: 20px;
            width: 100%;
            max-width: 420px;
            margin-bottom: 12px;
        }
        .steer-btn {
            background-color: #2c3e50;
            color: #ffffff;
            font-size: 24px;
            padding: 30px 10px;
        }
        .action-btn {
            background-color: #e74c3c;
            color: #ffffff;
            font-size: 22px;
            padding: 24px 10px;
        }
        .accel-btn {
            background-color: #27ae60;
            color: #ffffff;
            font-size: 22px;
            padding: 24px 10px;
        }
        button:active {
            transform: scale(0.96);
            opacity: 0.9;
        }
    </style>
</head>
<body>

    <header>
        <h1>गोकुळ रेस (Gokul Race)</h1>
        <div id="status" aria-live="polite">गेम सुरू करण्यासाठी खालील बटण दाबा</div>
        <div id="score-board">स्कोअर: <span id="score">0</span> | लेन: <span id="current-lane">मध्यम</span></div>
    </header>

    <div style="width: 100%; display: flex; flex-direction: column; align-items: center;">
        <button id="start-btn" class="main-btn" onclick="toggleGame()">गेम सुरू करा</button>

        <div class="controls-grid">
            <button class="steer-btn" onclick="steerLeft()" aria-label="डावीकडे वळा">⬅️ डावीकडे</button>
            <button class="steer-btn" onclick="steerRight()" aria-label="उजवीकडे वळा">उजवीकडे ➡️</button>
            <button class="action-btn" onclick="applyBrake()" aria-label="ब्रेक दाबा">🛑 ब्रेक</button>
            <button class="accel-btn" onclick="accelerateCar()" aria-label="ॲक्सिलेटर दाबा">⚡ रेस द्या</button>
        </div>
    </div>

    <script>
        let audioCtx = null;
        let engineGain = null;
        let engineOsc = null;
        let isPlaying = false;
        let score = 0;
        let playerLane = 0; // -1: Left, 0: Center, 1: Right
        let obstacleLane = 0;
        let gameLoopTimer = null;
        let obstacleTimer = null;
        let speed = 1.0;

        const lanes = ["डावी", "मध्यम", "उजवी"];

        function speak(text) {
            if ('speechSynthesis' in window) {
                window.speechSynthesis.cancel();
                const utter = new SpeechSynthesisUtterance(text);
                utter.lang = 'mr-IN';
                utter.rate = 1.1;
                window.speechSynthesis.speak(utter);
            }
        }

        function initAudio() {
            if (!audioCtx) {
                audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            }
            if (audioCtx.state === 'suspended') {
                audioCtx.resume();
            }
        }

        function startEngineSound() {
            engineOsc = audioCtx.createOscillator();
            engineGain = audioCtx.createGain();
            engineOsc.type = 'sawtooth';
            engineOsc.frequency.setValueAtTime(65, audioCtx.currentTime);

            const filter = audioCtx.createBiquadFilter();
            filter.type = 'lowpass';
            filter.frequency.setValueAtTime(250, audioCtx.currentTime);

            engineGain.gain.setValueAtTime(0.15, audioCtx.currentTime);

            engineOsc.connect(filter);
            filter.connect(engineGain);
            engineGain.connect(audioCtx.destination);
            engineOsc.start();
        }

        function playBrakeSound() {
            if (!audioCtx) return;
            initAudio();
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.type = 'triangle';
            osc.frequency.setValueAtTime(800, audioCtx.currentTime);
            osc.frequency.exponentialRampToValueAtTime(120, audioCtx.currentTime + 0.4);

            gain.gain.setValueAtTime(0.3, audioCtx.currentTime);
            gain.gain.linearRampToValueAtTime(0.01, audioCtx.currentTime + 0.4);

            osc.connect(gain);
            gain.connect(audioCtx.destination);
            osc.start();
            osc.stop(audioCtx.currentTime + 0.4);

            if (navigator.vibrate) navigator.vibrate(80);
        }

        function playHornSound(panValue) {
            if (!audioCtx) return;
            const panner = audioCtx.createStereoPanner ? audioCtx.createStereoPanner() : null;
            const osc1 = audioCtx.createOscillator();
            const osc2 = audioCtx.createOscillator();
            const gain = audioCtx.createGain();

            osc1.type = 'sawtooth';
            osc2.type = 'sawtooth';
            osc1.frequency.setValueAtTime(420, audioCtx.currentTime);
            osc2.frequency.setValueAtTime(530, audioCtx.currentTime);

            gain.gain.setValueAtTime(0.2, audioCtx.currentTime);
            gain.gain.linearRampToValueAtTime(0.01, audioCtx.currentTime + 0.6);

            if (panner) {
                panner.pan.setValueAtTime(panValue, audioCtx.currentTime);
                osc1.connect(panner);
                osc2.connect(panner);
                panner.connect(gain);
            } else {
                osc1.connect(gain);
                osc2.connect(gain);
            }
            gain.connect(audioCtx.destination);

            osc1.start();
            osc2.start();
            osc1.stop(audioCtx.currentTime + 0.6);
            osc2.stop(audioCtx.currentTime + 0.6);
        }

        function toggleGame() {
            initAudio();
            const btn = document.getElementById('start-btn');
            if (!isPlaying) {
                isPlaying = true;
                score = 0;
                speed = 1.0;
                playerLane = 0;
                updateDisplay();
                btn.textContent = "गेम थांबवा";
                speak("रेसिंग सुरू झाली! लक्ष देऊन ऐका.");
                startEngineSound();
                startGameLoops();
            } else {
                stopGame("गेम थांबवला");
            }
        }

        function startGameLoops() {
            scheduleNextObstacle();
            gameLoopTimer = setInterval(() => {
                if (!isPlaying) return;
                score += 1;
                document.getElementById('score').textContent = score;
                if (score % 10 === 0 && engineOsc) {
                    engineOsc.frequency.setValueAtTime(65 + Math.min(score, 100), audioCtx.currentTime);
                }
            }, 1000);
        }

        function scheduleNextObstacle() {
            if (!isPlaying) return;
            const delay = Math.max(2200, 4500 - (score * 50));
            obstacleTimer = setTimeout(() => {
                if (!isPlaying) return;
                spawnObstacle();
            }, delay);
        }

        function spawnObstacle() {
            const laneChoices = [-1, 0, 1];
            obstacleLane = laneChoices[Math.floor(Math.random() * laneChoices.length)];

            let voiceMsg = "";
            let pan = 0;

            if (obstacleLane === -1) {
                voiceMsg = "गाडी डावीकडून येत आहे, उजवीकडे वळा!";
                pan = -0.9;
            } else if (obstacleLane === 1) {
                voiceMsg = "गाडी उजवीकडून येत आहे, डावीकडे वळा!";
                pan = 0.9;
            } else {
                voiceMsg = "गाडी थेट समोरून येत आहे, बाजूला व्हा!";
                pan = 0;
            }

            speak(voiceMsg);
            playHornSound(pan);
            if (navigator.vibrate) navigator.vibrate([100, 50, 100]);

            setTimeout(() => {
                if (!isPlaying) return;
                if (playerLane === obstacleLane) {
                    crash();
                } else {
                    speak("छान वाचलात!");
                    scheduleNextObstacle();
                }
            }, 2000);
        }

        function steerLeft() {
            if (!isPlaying) return;
            if (playerLane > -1) {
                playerLane--;
                updateDisplay();
                speak(lanes[playerLane + 1]);
                if (navigator.vibrate) navigator.vibrate(40);
            } else {
                speak("रस्त्याचा शेवट!");
            }
        }

        function steerRight() {
            if (!isPlaying) return;
            if (playerLane < 1) {
                playerLane++;
                updateDisplay();
                speak(lanes[playerLane + 1]);
                if (navigator.vibrate) navigator.vibrate(40);
            } else {
                speak("रस्त्याचा शेवट!");
            }
        }

        function applyBrake() {
            if (!isPlaying) return;
            playBrakeSound();
            speak("ब्रेक!");
        }

        function accelerateCar() {
            if (!isPlaying) return;
            if (engineOsc) {
                const currentFreq = engineOsc.frequency.value;
                engineOsc.frequency.setValueAtTime(currentFreq + 25, audioCtx.currentTime);
                setTimeout(() => {
                    if (engineOsc) engineOsc.frequency.setValueAtTime(currentFreq, audioCtx.currentTime);
                }, 500);
            }
            speak("रेसिंग!");
        }

        function updateDisplay() {
            document.getElementById('current-lane').textContent = lanes[playerLane + 1];
        }

        function crash() {
            if (navigator.vibrate) navigator.vibrate([300, 100, 300, 100, 500]);
            speak("अपघात झाला! गेम संपला. तुमचा स्कोअर " + score);
            stopGame("गाडी धडकली! गेम संपला.");
        }

        function stopGame(msg) {
            isPlaying = false;
            clearTimeout(obstacleTimer);
            clearInterval(gameLoopTimer);
            if (engineOsc) {
                try { engineOsc.stop(); } catch(e){}
                engineOsc = null;
            }
            document.getElementById('status').textContent = msg;
            document.getElementById('start-btn').textContent = "पुन्हा खेळा";
        }
    </script>
</body>
</html>

