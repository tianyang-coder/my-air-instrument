<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FC 风格音乐盒</title>
    <!-- 引入 Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- 引入 Inter 字体 -->
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;700&display=swap" rel="stylesheet">
    <!-- 引入 Tone.js 用于音频合成 -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.8.49/Tone.min.js"></script>
    <style>
        /* 基础的 FC 风格样式 */
        body {
            font-family: 'Inter', sans-serif;
            background-color: #21216e; /* FC 蓝色背景 */
            color: #ffffff;
            user-select: none;
            cursor: default;
        }

        .pixel-font {
            /* 模拟像素字体效果 */
            text-shadow: 2px 2px 0px #000, 
                         -2px -2px 0px #000, 
                         2px -2px 0px #000, 
                         -2px 2px 0px #000;
        }

        .fc-button {
            transition: transform 0.1s, box-shadow 0.1s;
            border: 4px solid #000;
            border-radius: 4px;
            box-shadow: 6px 6px 0 0 #00003c; /* 深度阴影 */
            background-color: #fce8a8; /* 浅色背景 */
            color: #21216e; /* 深色文字 */
        }

        .fc-button:active {
            transform: translate(2px, 2px);
            box-shadow: 4px 4px 0 0 #00003c; /* 按下效果 */
        }

        .fc-instrument {
            @apply flex flex-col items-center justify-center p-4 m-2 text-3xl font-bold rounded-lg cursor-pointer transition duration-100 ease-in-out;
            min-height: 120px;
            min-width: 120px;
            border: 6px solid #000;
            box-shadow: 8px 8px 0 0 #5858cc; /* 乐器块的阴影 */
        }

        /* 乐器专属颜色 */
        #guitar { background-color: #58cc58; } /* 绿色 */
        #drum { background-color: #cc5858; } /* 红色 */
        #trumpet { background-color: #ccac58; } /* 黄色/金色 */
        #triangle { background-color: #58accc; } /* 蓝色 */

        /* 乐器按下效果 */
        .fc-instrument:active {
            transform: scale(0.98);
            box-shadow: 4px 4px 0 0 #5858cc;
        }

        /* 飘动的音符样式 */
        .floating-note {
            position: absolute;
            font-size: 2rem;
            opacity: 1;
            animation: float-up 2s forwards;
            pointer-events: none; /* 确保不影响点击事件 */
        }

        @keyframes float-up {
            0% {
                transform: translateY(0) scale(1);
                opacity: 1;
            }
            100% {
                transform: translateY(-100px) scale(0.5);
                opacity: 0;
            }
        }
    </style>
</head>
<body class="p-4 sm:p-8 flex flex-col items-center min-h-screen">

    <div class="max-w-4xl w-full">
        <!-- 标题区域 -->
        <h1 class="text-4xl sm:text-5xl text-center mb-6 pixel-font text-yellow-300">
            FC 8-Bit 音乐工作室
        </h1>

        <!-- 乐器区域 -->
        <div id="instruments-container" class="grid grid-cols-2 md:grid-cols-4 gap-4 p-4 border-8 border-black rounded-xl bg-gray-700 shadow-2xl mb-8">
            
            <button id="guitar" data-note="C4" class="fc-instrument pixel-font">
                🎸<br/>吉他
            </button>
            <button id="drum" data-note="A1" class="fc-instrument pixel-font">
                🥁<br/>鼓
            </button>
            <button id="trumpet" data-note="G4" class="fc-instrument pixel-font">
                🎺<br/>小号
            </button>
            <button id="triangle" data-note="E5" class="fc-instrument pixel-font">
                 المثلث<br/>三角铁
            </button>
        </div>

        <!-- 控制区域 -->
        <div class="flex flex-wrap justify-center gap-4 p-4 border-4 border-black rounded-lg bg-gray-800 shadow-xl">
            <button id="record-btn" class="fc-button py-2 px-6 text-lg font-bold w-full sm:w-auto bg-red-500 hover:bg-red-400">
                ⏺️ 录制 (Record)
            </button>
            <button id="play-btn" disabled class="fc-button py-2 px-6 text-lg font-bold w-full sm:w-auto bg-green-500 hover:bg-green-400 opacity-50">
                ▶️ 播放 (Play)
            </button>
            <button id="random-btn" class="fc-button py-2 px-6 text-lg font-bold w-full sm:w-auto bg-yellow-500 hover:bg-yellow-400">
                🔀 随机播放 (Random)
            </button>
            <button id="save-btn" disabled class="fc-button py-2 px-6 text-lg font-bold w-full sm:w-auto bg-blue-500 hover:bg-blue-400 opacity-50">
                💾 保存到云端
            </button>
        </div>

        <!-- 状态和消息区域 -->
        <div id="status-message" class="mt-6 p-3 text-center text-lg pixel-font border-4 border-yellow-300 bg-black text-yellow-300">
            点击任何乐器开始！请确保您的浏览器允许自动播放声音。
        </div>
        
        <!-- 用户信息区域 (用于多人协作时的身份标识) -->
        <div class="mt-4 text-center text-sm text-gray-400">
            用户 ID: <span id="user-id-display" class="font-mono">Loading...</span>
        </div>
    </div>

    <!-- Firebase 导入 -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc, onSnapshot } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";

        // ------------------
        // 1. Firebase/Firestore Setup
        // ------------------

        setLogLevel('debug'); // Enable debug logging for Firebase

        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');

        let app, db, auth, userId;
        let isAuthReady = false;
        
        // Firestore paths
        const MUSIC_COLLECTION = "recorded_music";
        
        // Utility function for Firestore document reference
        const getUserDocRef = (uid) => {
            const docPath = `/artifacts/${appId}/users/${uid}/${MUSIC_COLLECTION}/sequence`;
            return doc(db, docPath);
        };

        const initFirebase = async () => {
            try {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);

                onAuthStateChanged(auth, async (user) => {
                    if (user) {
                        userId = user.uid;
                        document.getElementById('user-id-display').textContent = userId;
                        isAuthReady = true;
                        showMessage(`用户已登录 (${userId.substring(0, 8)}...)`, 'yellow');
                        loadSavedMusic();
                    } else {
                        // Attempt to sign in
                        if (typeof __initial_auth_token !== 'undefined') {
                            await signInWithCustomToken(auth, __initial_auth_token);
                        } else {
                            await signInAnonymously(auth);
                        }
                    }
                });

            } catch (error) {
                console.error("Firebase Initialization Error:", error);
                showMessage("Firebase初始化失败，无法保存/加载音乐。", 'red');
            }
        };

        const saveMusic = async (sequence) => {
            if (!isAuthReady) {
                showMessage("身份验证尚未完成，请稍候...", 'red');
                return;
            }
            try {
                const docRef = getUserDocRef(userId);
                await setDoc(docRef, { 
                    sequence: JSON.stringify(sequence), // Stringify complex array structure
                    timestamp: new Date().toISOString()
                });
                showMessage("✅ 音乐序列已保存到云端!", 'green');
            } catch (e) {
                console.error("Error saving document: ", e);
                showMessage("❌ 保存失败，请检查控制台错误。", 'red');
            }
        };

        const loadSavedMusic = () => {
            if (!isAuthReady) return;

            const docRef = getUserDocRef(userId);
            
            // Use onSnapshot for real-time loading (optional, but good practice)
            onSnapshot(docRef, (docSnap) => {
                if (docSnap.exists()) {
                    try {
                        const data = docSnap.data();
                        const sequence = JSON.parse(data.sequence);
                        recordedSequence = sequence;
                        document.getElementById('play-btn').disabled = false;
                        document.getElementById('play-btn').classList.remove('opacity-50');
                        showMessage(`🎵 加载到 ${recordedSequence.length} 个事件`, 'green');
                    } catch (e) {
                        console.error("Error parsing recorded sequence:", e);
                        showMessage("加载的音乐数据格式错误。", 'red');
                    }
                } else {
                    showMessage("云端没有找到保存的音乐。", 'yellow');
                }
            }, (error) => {
                console.error("Error listening to document:", error);
                showMessage("❌ 加载数据失败。", 'red');
            });
        };


        // ------------------
        // 2. Game/Music Logic
        // ------------------

        let isRecording = false;
        let recordingStartTime = 0;
        let recordedSequence = []; // Stores { instrumentId, time }

        // Initialize Tone.js components
        const context = new Tone.Context();
        let isToneInitialized = false;

        const synth = new Tone.PolySynth(Tone.AMSynth, {
            envelope: {
                attack: 0.005,
                decay: 0.1,
                sustain: 0.3,
                release: 1
            },
            oscillator: { type: 'square' }
        }).toDestination();

        const drum = new Tone.MembraneSynth({
            envelope: {
                attack: 0.001,
                decay: 0.4,
                sustain: 0.01
            }
        }).toDestination();

        const triangle = new Tone.MetalSynth({
            frequency: 400,
            envelope: {
                attack: 0.001,
                decay: 0.2,
                release: 0.05
            },
            harmonicity: 3.1,
            modulationIndex: 10,
            octaves: 1.5
        }).toDestination();

        const instruments = {
            'guitar': { synth: synth, note: 'C4', emoji: '🎵' },
            'drum': { synth: drum, note: 'A1', emoji: '💥' },
            'trumpet': { synth: synth, note: 'G4', emoji: '🎶' }, // Using same synth for trumpet/guitar, just different notes
            'triangle': { synth: triangle, note: 'E5', emoji: '✨' },
        };
        
        // FC Note Pool for Random Playback
        const randomNotes = [
            { id: 'guitar', note: 'C4' },
            { id: 'trumpet', note: 'D4' },
            { id: 'guitar', note: 'E4' },
            { id: 'trumpet', note: 'G4' },
            { id: 'guitar', note: 'C5' },
            { id: 'drum', note: 'A1' },
            { id: 'drum', note: 'A1' },
            { id: 'triangle', note: 'E5' },
        ];


        // Utility Functions
        function showMessage(msg, color) {
            const statusEl = document.getElementById('status-message');
            statusEl.textContent = msg;
            statusEl.className = `mt-6 p-3 text-center text-lg pixel-font border-4 border-black bg-gray-900`;
            if (color === 'red') statusEl.classList.add('text-red-400', 'border-red-400');
            else if (color === 'green') statusEl.classList.add('text-green-400', 'border-green-400');
            else if (color === 'yellow') statusEl.classList.add('text-yellow-300', 'border-yellow-300');
        }

        function triggerFloatingNote(instrumentEl, emoji) {
            const rect = instrumentEl.getBoundingClientRect();
            const noteEl = document.createElement('div');
            noteEl.className = 'floating-note';
            noteEl.textContent = emoji;
            
            // Position the note above the element
            noteEl.style.left = `${rect.left + rect.width / 2}px`;
            noteEl.style.top = `${rect.top + rect.height / 2}px`;
            
            document.body.appendChild(noteEl);

            // Clean up the note after animation
            setTimeout(() => {
                noteEl.remove();
            }, 2000);
        }

        async function playInstrument(instrumentId) {
            if (!isToneInitialized) {
                await Tone.start();
                isToneInitialized = true;
                showMessage("🎶 音频已启动！", 'green');
            }

            const inst = instruments[instrumentId];
            const el = document.getElementById(instrumentId);
            
            // 播放声音
            if (instrumentId === 'drum') {
                 // Drum needs triggerAttackRelease(note, duration)
                inst.synth.triggerAttackRelease(inst.note, '8n');
            } else if (instrumentId === 'triangle') {
                 // Triangle is percussive, use a short duration
                 inst.synth.triggerAttackRelease(inst.note, '16n');
            } else {
                // Guitar and Trumpet (Synths)
                inst.synth.triggerAttackRelease(inst.note, '4n');
            }
            
            // 触发音符飘出
            triggerFloatingNote(el, inst.emoji);

            // 录制逻辑
            if (isRecording) {
                const time = context.currentTime - recordingStartTime;
                recordedSequence.push({ 
                    id: instrumentId, 
                    time: time 
                });
            }
        }
        
        function handleRecord() {
            isRecording = !isRecording;
            const recordBtn = document.getElementById('record-btn');
            const playBtn = document.getElementById('play-btn');
            const saveBtn = document.getElementById('save-btn');

            if (isRecording) {
                // 开始录制
                recordedSequence = [];
                recordingStartTime = context.currentTime;
                recordBtn.textContent = '🛑 停止录制 (Stop)';
                recordBtn.classList.remove('bg-red-500', 'hover:bg-red-400');
                recordBtn.classList.add('bg-gray-500', 'hover:bg-gray-400');
                
                playBtn.disabled = true;
                saveBtn.disabled = true;
                playBtn.classList.add('opacity-50');
                saveBtn.classList.add('opacity-50');
                showMessage("🔴 正在录制...", 'red');
            } else {
                // 停止录制
                recordBtn.textContent = '⏺️ 录制 (Record)';
                recordBtn.classList.remove('bg-gray-500', 'hover:bg-gray-400');
                recordBtn.classList.add('bg-red-500', 'hover:bg-red-400');

                if (recordedSequence.length > 0) {
                    playBtn.disabled = false;
                    saveBtn.disabled = false;
                    playBtn.classList.remove('opacity-50');
                    saveBtn.classList.remove('opacity-50');
                    showMessage(`✅ 录制完成，共 ${recordedSequence.length} 个事件。`, 'green');
                } else {
                    showMessage("录制完成，但没有事件。", 'yellow');
                }
            }
        }

        function handlePlayback() {
            if (!recordedSequence || recordedSequence.length === 0) {
                showMessage("没有可播放的录制序列。", 'yellow');
                return;
            }

            const playbackTime = context.currentTime;
            let maxTime = 0;

            showMessage("▶️ 正在播放录制...", 'yellow');

            // 1. 安排所有事件
            recordedSequence.forEach(event => {
                const time = event.time;
                const instId = event.id;
                
                // 安排音符的播放
                Tone.Transport.scheduleOnce((t) => {
                    playInstrument(instId);
                }, `+${time}`);

                if (time > maxTime) {
                    maxTime = time;
                }
            });

            // 2. 播放结束后的回调
            Tone.Transport.scheduleOnce(() => {
                Tone.Transport.stop();
                showMessage("✅ 播放完成。", 'green');
            }, `+${maxTime + 0.5}`); // +0.5s for release time

            // 3. 启动播放
            Tone.Transport.start(playbackTime);
        }

        function handleRandom() {
             const sequence = [];
             let currentTime = 0;
             const tempo = 0.25; // 1/4 second per step (FC tempo)

             // Create a 16-step random sequence
             for (let i = 0; i < 16; i++) {
                 const randomInst = randomNotes[Math.floor(Math.random() * randomNotes.length)];
                 sequence.push({
                     id: randomInst.id,
                     time: currentTime
                 });
                 currentTime += tempo;
             }

             let maxTime = currentTime;
             showMessage("🔀 正在播放随机音乐...", 'yellow');
             
             Tone.Transport.stop(); // Stop any current playback
             Tone.Transport.cancel(); // Clear previous events
             
             // Schedule all events
             sequence.forEach(event => {
                 Tone.Transport.scheduleOnce((t) => {
                     playInstrument(event.id);
                 }, `+${event.time}`);
             });

             // End callback
             Tone.Transport.scheduleOnce(() => {
                 Tone.Transport.stop();
                 showMessage("✅ 随机播放完成。", 'green');
             }, `+${maxTime + 0.5}`);

             Tone.Transport.start();
        }

        // ------------------
        // 3. Event Listeners
        // ------------------
        
        window.onload = () => {
            initFirebase();
            
            // Add listeners for instruments
            document.querySelectorAll('.fc-instrument').forEach(button => {
                button.addEventListener('click', (event) => {
                    // Prevent context menu on touch devices
                    event.preventDefault(); 
                    // Stop any ongoing playback for immediate instrument sound
                    if (Tone.Transport.state === 'started') {
                         Tone.Transport.stop();
                    }
                    playInstrument(button.id);
                });
            });

            // Add listeners for controls
            document.getElementById('record-btn').addEventListener('click', handleRecord);
            document.getElementById('play-btn').addEventListener('click', handlePlayback);
            document.getElementById('random-btn').addEventListener('click', handleRandom);
            document.getElementById('save-btn').addEventListener('click', () => saveMusic(recordedSequence));
        };
    </script>
</body>
</html>
