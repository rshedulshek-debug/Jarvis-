<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>JARVIS X | ALWAYS LISTENING</title>
    <style>
        :root { --neon: #00ff41; --bg: #050505; }
        body { background: var(--bg); color: var(--neon); font-family: 'Courier New', monospace; margin: 0; display: flex; flex-direction: column; height: 100vh; overflow: hidden; }
        
        /* Visualizer Section */
        .vis-container { height: 160px; display: flex; justify-content: center; align-items: center; background: radial-gradient(circle, #001a00 0%, #050505 70%); border-bottom: 1px solid var(--neon); }
        .circle { width: 90px; height: 90px; border: 3px solid var(--neon); border-radius: 50%; display: flex; justify-content: center; align-items: center; box-shadow: 0 0 20px var(--neon); transition: 0.3s; position: relative; }
        
        /* Animation when Jarvis is listening or speaking */
        .active-mode { animation: ripple 0.8s infinite alternate; }
        
        @keyframes ripple {
            from { transform: scale(1); box-shadow: 0 0 10px var(--neon); }
            to { transform: scale(1.2); box-shadow: 0 0 35px var(--neon); }
        }

        .terminal { flex: 1; padding: 15px; overflow-y: auto; background: #000; font-size: 13px; scroll-behavior: smooth; }
        .input-group { padding: 15px; background: #111; border-top: 1px solid var(--neon); }
        .row { display: flex; gap: 8px; margin-top: 10px; }
        
        input { background: #000; border: 1px solid var(--neon); color: var(--neon); padding: 12px; flex: 1; outline: none; }
        button { background: var(--neon); color: #000; border: none; padding: 10px 20px; font-weight: bold; cursor: pointer; text-transform: uppercase; }
        
        .user-log { color: #fff; margin: 8px 0; border-left: 3px solid #fff; padding-left: 10px; }
        .jarvis-log { color: var(--neon); margin: 8px 0; border-left: 3px solid var(--neon); padding-left: 10px; }
        .system-msg { color: #555; font-size: 10px; text-align: center; }
    </style>
</head>
<body>

<div class="vis-container">
    <div class="circle" id="visualizer">
        <div id="status-text" style="font-size: 10px; font-weight: bold;">OFFLINE</div>
    </div>
</div>

<div class="terminal" id="logs">
    <div class="system-msg">CORE UPDATED: CONTINUOUS LISTENING ENABLED</div>
    <div class="jarvis-log"><strong>JARVIS:</strong> Systems online. Waiting for your command, Sir.</div>
</div>

<div class="input-group">
    <input type="password" id="api_key" placeholder="PASTE OPENROUTER API KEY HERE">
    <div class="row">
        <input type="text" id="manual_input" placeholder="Type or just speak...">
        <button onclick="startJarvis()" id="micBtn">START SYSTEM</button>
        <button onclick="sendManual()">SEND</button>
    </div>
</div>

<script>
    const logs = document.getElementById('logs');
    const apiInput = document.getElementById('api_key');
    const manualInput = document.getElementById('manual_input');
    const visualizer = document.getElementById('visualizer');
    const micBtn = document.getElementById('micBtn');
    const statusText = document.getElementById('status-text');

    let autoListen = false;
    const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition;
    let recognition;

    if (SpeechRecognition) {
        recognition = new SpeechRecognition();
        recognition.continuous = false; // Mobile browsers handle false better for auto-restart
        recognition.lang = 'en-US';

        recognition.onstart = () => {
            statusText.innerText = "LISTENING";
            visualizer.classList.add('active-mode');
            micBtn.innerText = "SYSTEM ACTIVE";
            micBtn.style.background = "#fff";
        };

        recognition.onend = () => {
            visualizer.classList.remove('active-mode');
            // Auto-restart logic
            if (autoListen) {
                setTimeout(() => {
                    try { recognition.start(); } catch(e) {}
                }, 400);
            } else {
                statusText.innerText = "IDLE";
                micBtn.innerText = "START SYSTEM";
                micBtn.style.background = "var(--neon)";
            }
        };

        recognition.onresult = (e) => {
            const text = e.results[0][0].transcript;
            processInput(text);
        };

        recognition.onerror = (e) => {
            console.log("Mic Error: " + e.error);
            if(e.error === 'not-allowed') alert("Please allow Microphone access on HTTPS!");
        };
    }

    function addLog(text, sender) {
        const div = document.createElement('div');
        div.className = sender === 'user' ? 'user-log' : 'jarvis-log';
        div.innerHTML = `<strong>${sender.toUpperCase()}:</strong> ${text}`;
        logs.appendChild(div);
        logs.scrollTop = logs.scrollHeight;
    }

    function speak(text) {
        window.speechSynthesis.cancel();
        const ut = new SpeechSynthesisUtterance(text);
        ut.pitch = 0.85; 
        ut.rate = 1.0;
        
        ut.onstart = () => {
            autoListen = false; // Stop listening while Jarvis is talking
            recognition.stop();
            visualizer.classList.add('active-mode');
            statusText.innerText = "SPEAKING";
        };
        
        ut.onend = () => {
            visualizer.classList.remove('active-mode');
            autoListen = true; // Resume listening after Jarvis finishes
            recognition.start();
        };
        
        window.speechSynthesis.speak(ut);
    }

    async function processInput(query) {
        if(!query) return;
        addLog(query, 'user');
        
        const key = apiInput.value.trim();
        if(!key) {
            addLog("ERROR: API KEY MISSING.", "jarvis");
            speak("Sir, please provide the API key to proceed.");
            return;
        }

        try {
            const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
                method: "POST",
                headers: { 
                    "Authorization": "Bearer " + key, 
                    "Content-Type": "application/json" 
                },
                body: JSON.stringify({
                    model: "google/gemini-2.0-flash-001",
                    messages: [
                        {role: "system", content: "You are Jarvis X. Calm, smart, and efficient. Use 'Sir' occasionally. Brief answers only."},
                        {role: "user", content: query}
                    ]
                })
            });
            const data = await response.json();
            const reply = data.choices[0].message.content;
            addLog(reply, 'jarvis');
            speak(reply);
        } catch (err) {
            addLog("CONNECTION FAILED. RECONNECTING...", "jarvis");
            autoListen = true;
            recognition.start();
        }
    }

    function startJarvis() {
        if (!SpeechRecognition) return alert("Browser not supported");
        autoListen = true;
        recognition.start();
    }

    function sendManual() {
        const val = manualInput.value;
        if(val) { processInput(val); manualInput.value = ''; }
    }
</script>
</body>
</html>
