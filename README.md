# ai-chat-multilang-voice
cat << 'EOF' > app.py
from flask import Flask, render_template, request, jsonify, session
import openai

app = Flask(__name__)
app.secret_key = "supersecretkey"

openai.api_key = "YOUR_API_KEY"

@app.route("/")
def home():
    return render_template("index.html")

@app.route("/chat", methods=["POST"])
def chat():
    user = request.json.get("message")
    if "conversation" not in session:
        session["conversation"] = []
    session["conversation"].append({"role": "user", "content": user})

    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=session["conversation"]
    )
    reply = response.choices[0].message.content
    session["conversation"].append({"role": "assistant", "content": reply})
    return jsonify({"reply": reply})

@app.route("/reset", methods=["POST"])
def reset():
    session.pop("conversation", None)
    return jsonify({"status":"reset"})

if __name__ == "__main__":
    app.run(debug=True)
EOF

cat << 'EOF' > requirements.txt
Flask==2.3.2
openai==1.32.0
gunicorn==21.2.0
EOF

cat << 'EOF' > Procfile
web: gunicorn app:app
EOF

cat << 'EOF' > runtime.txt
python-3.11.4
EOF

mkdir templates
cat << 'EOF' > templates/index.html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>AI Chat Multi-Language Voice</title>
<link rel="stylesheet" href="/static/style.css">
</head>
<body class="light-mode">
<div id="chat-container">
    <div id="header">
        <h2>AI Chat 🌐</h2>
        <button id="toggle-mode">🌙</button>
        <select id="language-select">
            <option value="en-US" selected>English</option>
            <option value="es-ES">Spanish</option>
            <option value="fr-FR">French</option>
            <option value="de-DE">German</option>
            <option value="zh-CN">Chinese</option>
            <option value="ja-JP">Japanese</option>
        </select>
    </div>
    <div id="chat-box"></div>
    <div id="input-container">
        <input id="userInput" placeholder="Type a message..." autocomplete="off">
        <button onclick="sendMessage()">Send</button>
        <button id="voice-btn">🎤</button>
        <button onclick="resetChat()">Reset</button>
    </div>
</div>

<script>
const chatBox = document.getElementById("chat-box");
const body = document.body;
const toggleBtn = document.getElementById("toggle-mode");
const voiceBtn = document.getElementById("voice-btn");
const languageSelect = document.getElementById("language-select");

toggleBtn.addEventListener("click", () => {
    if(body.classList.contains("light-mode")){
        body.classList.remove("light-mode");
        body.classList.add("dark-mode");
        toggleBtn.textContent = "☀️";
    } else {
        body.classList.remove("dark-mode");
        body.classList.add("light-mode");
        toggleBtn.textContent = "🌙";
    }
});

let recognition;
function initRecognition(){
    const lang = languageSelect.value;
    if('webkitSpeechRecognition' in window){
        recognition = new webkitSpeechRecognition();
        recognition.continuous = false;
        recognition.interimResults = false;
        recognition.lang = lang;

        recognition.onresult = function(event){
            const transcript = event.results[0][0].transcript;
            document.getElementById("userInput").value = transcript;
            sendMessage();
        }
        recognition.onerror = function(e){
            console.error("Speech recognition error", e);
        }
    }
}

voiceBtn.addEventListener("click", () => {
    initRecognition();
    if(recognition) recognition.start();
});

async function sendMessage() {
    const inputField = document.getElementById("userInput");
    const val = inputField.value;
    if (!val) return;
    addMessage("user", val);
    inputField.value = "";

    const typing = addMessage("ai", "AI is typing...", true);

    const res = await fetch("/chat", {
        method:"POST",
        headers:{"Content-Type":"application/json"},
        body: JSON.stringify({message: val})
    });

    const data = await res.json();
    typeWriter(typing, data.reply, true);
}

function addMessage(s, text, typing=false){
    const d = document.createElement("div");
    d.className = `message ${s}`;
    d.innerHTML = `<div class="avatar">${s==="user"?"🧑":"🤖"}</div><div class="text">${text}</div>`;
    chatBox.appendChild(d);
    chatBox.scrollTop = chatBox.scrollHeight;
    if(typing) return d.querySelector(".text");
}

function typeWriter(el, txt, speak=false){
    el.innerHTML="";
    let i=0;
    function t(){
        if(i<txt.length){
            el.innerHTML+=txt.charAt(i++);
            chatBox.scrollTop = chatBox.scrollHeight;
            setTimeout(t,25);
        } else if(speak){
            speakText(txt);
        }
    }
    t();
}

function speakText(txt){
    if('speechSynthesis' in window){
        const u = new SpeechSynthesisUtterance(txt);
        u.lang = languageSelect.value;
        window.speechSynthesis.speak(u);
    }
}

async function resetChat(){
    await fetch("/reset",{method:"POST"});
    chatBox.innerHTML="";
}
</script>
</body>
</html>
EOF

mkdir static
cat << 'EOF' > static/style.css
body { font-family:"Segoe UI",Arial; margin:0; background:#f5f5f7; }
body.dark-mode { background:#343541; color:#e5e5e5; }
#chat-container{width:500px;max-width:90%; margin:50px auto; border-radius:10px; overflow:hidden; box-shadow:0 5px 20px rgba(0,0,0,0.2); background:#fff;}
body.dark-mode #chat-container{background:#444654;}
#header{display:flex;justify-content:space-between;align-items:center;background:#4f8ef7;color:#fff;padding:10px 15px;}
body.dark-mode #header{background:#1f1f1f;}
#chat-box{padding:15px;max-height:500px;overflow-y:auto;background:#eaeaea;transition:background 0.3s;}
body.dark-mode #chat-box{background:#343541;}
.message{display:flex;margin-bottom:15px;align-items:flex-start;animation:fadeIn 0.3s ease-in-out;}
.message.user{justify-content:flex-end;}
.message .avatar{width:35px;height:35px;margin-right:10px;font-size:25px;}
.message.user .avatar{margin-left:10px;margin-right:0;}
.message .text{max-width:70%;padding:10px 15px;border-radius:15px;background:#f1f0f0;word-wrap:break-word;transition:background 0.3s;}
body.dark-mode .message .text{background:#5a5c69;color:#e5e5e5;}
.message.user .text{background:#4f8ef7;color:white;}
body.dark-mode .message.user .text{background:#3b6cd4;}
#input-container{display:flex;padding:10px;border-top:1px solid #ddd;background:#fff;}
body.dark-mode #input-container{background:#3b3b3b;border-top:1px solid #555;}
input{flex:1;padding:10px;border-radius:20px;border:1px solid #ccc;outline:none;}
button{margin-left:5px;padding:10px 15px;border:none;border-radius:20px;cursor:pointer;background:#4f8ef7;color:white;font-weight:bold;}
button:hover{background:#3b6cd4;}
@keyframes fadeIn{from{opacity:0;transform:translateY(5px);}to{opacity:1;transform:translateY(0);}}
EOF

zip -r ai-chat-multilang-voice.zip .
