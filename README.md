# AI-mini-talker-bulder
hi  friends I am mini talker AI owner name is babu 
# mini_talker_openai.py
import os
import tempfile
import sounddevice as sd
import soundfile as sf
import requests
import openai
from playsound import playsound

openai.api_key = os.getenv("OPENAI_API_KEY")
# Some endpoints/models might be named e.g. "gpt-4o-mini-transcribe" / "gpt-4o-mini-tts" depending on availability.
# We'll call the generic endpoints below; adjust model names if your account has specific newer model names.

DURATION = 6  # seconds of recording per user turn (adjust)
SAMPLE_RATE = 16000
TRANSCRIBE_MODEL = "whisper-1"        # fallback to whisper-1 or newer transcribe model
CHAT_MODEL = "gpt-4o-mini"            # change to available chat model in your account
TTS_MODEL = "gpt-4o-mini-tts"         # example name; change if API returns different TTS model

def record_audio(seconds=DURATION, sr=SAMPLE_RATE):
    print(f"Recording {seconds}s... Speak now.")
    recording = sd.rec(int(seconds * sr), samplerate=sr, channels=1, dtype='int16')
    sd.wait()
    tmp = tempfile.NamedTemporaryFile(suffix=".wav", delete=False)
    sf.write(tmp.name, recording, sr, subtype='PCM_16')
    print("Saved:", tmp.name)
    return tmp.name

def transcribe_file(path):
    # Upload + transcribe with OpenAI speech-to-text endpoint
    with open(path, "rb") as f:
        # using openai.Audio.transcribe (depends on SDK version)
        resp = openai.Audio.transcribe(model=TRANSCRIBE_MODEL, file=f)
    text = resp["text"]
    print("User said:", text)
    return text

def chat_reply(prompt_text):
    # basic chat call
    resp = openai.ChatCompletion.create(
        model=CHAT_MODEL,
        messages=[
            {"role":"system", "content":"You are Mini Talker AI, a helpful friendly assistant."},
            {"role":"user", "content": prompt_text}
        ],
        max_tokens=400
    )
    reply = resp["choices"][0]["message"]["content"].strip()
    print("AI reply:", reply)
    return reply

def tts_and_play(text, out_path="reply.mp3"):
    # Many accounts support a TTS endpoint that returns audio. 
    # If your SDK supports openai.audio.speech.create use that; otherwise use REST.
    # Example using REST /v1/audio/speech (may vary by account/SDK)
    url = "https://api.openai.com/v1/audio/speech"
    headers = {"Authorization": f"Bearer {openai.api_key}"}
    payload = {
        "model": TTS_MODEL,
        "voice": "alloy",   # example voice; change per available voices in your account
        "input": text
    }
    # some TTS endpoints expect application/json POST and return audio bytes
    r = requests.post(url, headers=headers, json=payload, stream=True)
    r.raise_for_status()
    with open(out_path, "wb") as f:
        for chunk in r.iter_content(chunk_size=8192):
            if chunk:
                f.write(chunk)
    playsound(out_path)

def main_loop():
    print("Mini Talker (cloud) ready. Press Ctrl+C to stop.")
    try:
        while True:
            wav = record_audio()
            try:
                user_text = transcribe_file(wav)
            except Exception as e:
                print("Transcription failed:", e)
                continue
            if user_text.strip().lower() in ["exit", "quit", "bye"]:
                print("Goodbye!")
                break
            reply = chat_reply(user_text)
            try:
                tts_and_play(reply, out_path="mini_reply.mp3")
            except Exception as e:
                print("TTS failed, falling back to print:", e)
                print("AI:", reply)
    except KeyboardInterrupt:
        print("Stopped.")

if __name__ == "__main__":
    main_loop()
# mini_talker_offline.py
import os
import tempfile
import sounddevice as sd
import soundfile as sf
import whisper   # pip install -U openai-whisper or use whisperx
import pyttsx3

DURATION = 6
SR = 16000

def record(seconds=DURATION):
    rec = sd.rec(int(seconds*SR), samplerate=SR, channels=1)
    sd.wait()
    tmp = tempfile.NamedTemporaryFile(suffix=".wav", delete=False)
    sf.write(tmp.name, rec, SR, subtype='PCM_16')
    return tmp.name

model = whisper.load_model("small")  # or tiny / base / medium / large depending on hardware

engine = pyttsx3.init()
def speak(text):
    engine.say(text)
    engine.runAndWait()

def transcribe(path):
    res = model.transcribe(path)
    return res["text"]

def main():
    print("Mini Talker (offline) ready.")
    while True:
        wav = record()
        txt = transcribe(wav)
        print("You:", txt)
        if txt.strip().lower() in ["quit","exit","bye"]:
            speak("Goodbye")
            break
        # --- Here you can call a local LLM or rules or send to cloud LLM
        # For demo, we echo back:
        reply = "You said: " + txt
        print("AI:", reply)
        speak(reply)

if __name__=="__main__":
    main()    
    # eleven_clone_emotion.py
# pip install requests sounddevice soundfile

import os, requests, json, tempfile, sounddevice as sd, soundfile as sf

API_KEY = os.getenv("ELEVENLABS_API_KEY")  # set this
BASE = "https://api.elevenlabs.io/v1"

def record_seconds(seconds=40, sr=22050):
    print(f"Recording {seconds}s... speak naturally.")
    rec = sd.rec(int(seconds*sr), samplerate=sr, channels=1)
    sd.wait()
    path = tempfile.NamedTemporaryFile(suffix=".wav", delete=False).name
    sf.write(path, rec, sr)
    print("Saved:", path)
    return path

def create_voice_from_sample(name, sample_path):
    # Example: create a new voice profile using the user sample
    url = f"{BASE}/voices/add"  # endpoint names may change; check docs
    headers = {"xi-api-key": API_KEY}
    files = {
        "file": open(sample_path, "rb"),
        "name": (None, name)
    }
    r = requests.post(url, headers=headers, files=files)
    r.raise_for_status()
    return r.json()

def synthesize(voice_id, text, emotion_tag=None, out_path="out.wav"):
    url = f"{BASE}/text-to-speech/{voice_id}"
    headers = {"xi-api-key": API_KEY, "Content-Type": "application/json"}
    # Eleven v3 supports audio tags like <emotion:warm> or other audio tags.
    if emotion_tag:
        text = f"{emotion_tag} {text}"
    payload = {"text": text, "voice_settings": {"stability":0.7,"similarity_boost":0.7}}
    r = requests.post(url, headers=headers, json=payload)
    r.raise_for_status()
    with open(out_path, "wb") as f:
        f.write(r.content)
    print("Saved audio:", out_path)

if __name__ == "__main__":
    # 1) Record sample (or upload your own file)
    sample = record_seconds(40)
    print("Uploading sample to create voice... (this may take a moment)")
    resp = create_voice_from_sample("MyMiniTalker", sample)
    print("Create voice response:", resp)
    voice_id = resp.get("voice_id") or resp.get("id") or resp.get("voice", {}).get("id")
    # 2) Synthesize with emotion tags (Eleven v3 audio tags: e.g. "<emotion:warm>" or using audio tag layer)
    synth_text = "Hello! I'm your Mini Talker speaking in a warm excited tone."
    # Example of an audio tag (see ElevenLabs docs for exact syntax).
    synthesize(voice_id, synth_text, emotion_tag="<emotion:warm>")
# local_coqui_clone.py
# pip install TTS sounddevice soundfile
from TTS.api import TTS
import sounddevice as sd, soundfile as sf, tempfile

def record(seconds=30, sr=22050):
    rec = sd.rec(int(seconds*sr), samplerate=sr, channels=1)
    sd.wait()
    path = tempfile.NamedTemporaryFile(suffix=".wav", delete=False).name
    sf.write(path, rec, sr)
    return path

# pick a prebuilt voice-conversion model
tts = TTS(model_name="tts_models/en/vctk/vits")  # replace with a model supporting VC/cloning

sample = record(30)
out = "coqui_out.wav"
# use speaker_wav (voice conversion) to synthesize in your voice
tts.tts_with_vc_to_file("Hello, this is Mini Talker speaking with emotion.", speaker_wav=sample, file_path=out)
print("Saved:", out)
# simple_owner_handler.py
import re

OWNER_NAME = "Babu"   # change here to set owner

# list of patterns users may ask
OWNER_PATTERNS = [
    r"who\s+is\s+your\s+owner\??",
    r"who\s+is\s+your\s+creator\??",
    r"who\s+owns\s+you\??",
    r"what\s+is\s+your\s+owner['’]?s?\s+name\??",
    r"who\s+made\s+you\??"
]

def is_owner_question(text: str) -> bool:
    text = text.lower().strip()
    for p in OWNER_PATTERNS:
        if re.search(p, text):
            return True
    return False

# generate textual reply (variants)
REPLIES = [
    "My owner name is {owner}.",
    "I belong to {owner}.",
    "My creator is {owner}.",
    "It's {owner} — they're my owner!",
    "Owner: {owner}."
]

def choose_reply(owner=OWNER_NAME, style=0):
    # style 0..len(REPLIES)-1 picks a reply, default random-like rotation
    import random
    idx = style if 0 <= style < len(REPLIES) else random.randrange(len(REPLIES))
    return REPLIES[idx].format(owner=owner)

# Example integration
def handle_user_message(text):
    if is_owner_question(text):
        return choose_reply()
    # other handlers...
    return None

if __name__ == "__main__":
    tests = [
        "Who is your owner?",
        "who owns you",
        "What is your owner's name?",
        "hey bot, who made you?"
    ]
    for t in tests:
        print(t, "->", handle_user_message(t))
<!-- index.html -->
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Mini Talker</title></head>
<body>
  <input id="userInput" placeholder="Ask me something" style="width:60%"/>
  <button id="askBtn">Ask</button>

<script>
const OWNER_NAME = "Babu";

const patterns = [
  /who\s+is\s+your\s+owner\??/i,
  /who\s+made\s+you\??/i,
  /what\s+is\s+your\s+owner'?s?\s+name\??/i,
  /who\s+owns\s+you\??/i
];

const replies = [
  `My owner name is ${OWNER_NAME}.`,
  `I belong to ${OWNER_NAME}.`,
  `${OWNER_NAME} is my owner.`,
  `It's ${OWNER_NAME} — they're my owner!`
];

function isOwnerQuestion(text){
  return patterns.some(p => p.test(text));
}

function speak(text, emotion='neutral'){
  // Browser TTS
  const msg = new SpeechSynthesisUtterance(text);
  // Emulate simple emotions by rate and pitch
  if(emotion === 'happy'){ msg.rate = 1.1; msg.pitch = 1.3 }
  else if(emotion === 'sad'){ msg.rate = 0.9; msg.pitch = 0.8 }
  else { msg.rate = 1.0; msg.pitch = 1.0 }
  speechSynthesis.cancel();
  speechSynthesis.speak(msg);
}

document.getElementById('askBtn').onclick = () => {
  const t = document.getElementById('userInput').value.trim();
  if(!t) return;
  if(isOwnerQuestion(t)){
    const r = replies[Math.floor(Math.random()*replies.length)];
    // show text (could be added to UI) and speak
    alert(r);
    speak(r, 'happy');
  } else {
    alert("I don't know — try asking: Who is your owner?");
  }
};
</script>
</body>
</html>
// server.js (Node + Express)
const express = require('express');
const app = express();
app.use(express.json());

const OWNER = "Babu";
const patterns = [
  /who\s+is\s+your\s+owner\??/i,
  /who\s+made\s+you\??/i,
  /what\s+is\s+your\s+owner'?s?\s+name\??/i
];
const replies = [
  `My owner name is ${OWNER}.`,
  `${OWNER} is my owner.`,
  `I belong to ${OWNER}.`
];

function isOwnerQuestion(text){
  return patterns.some(p => p.test(text));
}

app.post('/chat', (req, res) => {
  const text = (req.body.text || "").trim();
  if(isOwnerQuestion(text)){
    const r = replies[Math.floor(Math.random()*replies.length)];
    // respond JSON; front-end can convert to audio using Web Speech or other TTS service
    return res.json({reply: r});
  }
  res.json({reply: "I don't understand. Try: 'Who is your owner?'"});
});

app.listen(3000, ()=>console.log("listening on 3000"));
// ai_chat.js
import OpenAI from "openai";
const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function chatWithAI(message){
  const response = await client.chat.completions.create({
    model: "gpt-4o-mini", // or gpt-4.1
    messages: [
      { role: "system", content: "You are Mini Talker AI. If asked who your owner is, always reply 'My owner name is Babu'." },
      { role: "user", content: message }
    ]
  });
  return response.choices[0].message.content;
}

// Example usage
(async () => {
  console.log(await chatWithAI("Who is your owner?"));
  console.log(await chatWithAI("What is the capital of France?"));
})();
# mini_talker_offline.py
import random

OWNER = "Babu"

qa_pairs = {
    "who is your owner": [f"My owner name is {OWNER}."],
    "what is your name": ["I am Mini Talker AI."],
    "how are you": ["I’m doing great, thanks!", "Awesome as always 😃"],
    "what is your age": ["I don’t have an age, but I’m always learning!"],
    "who made you": [f"My creator is {OWNER}."]
}

def chatbot_reply(user_input):
    text = user_input.lower().strip()
    for q in qa_pairs:
        if q in text:
            return random.choice(qa_pairs[q])
    return "I don’t know that yet, but I’ll learn soon!"

print("Mini Talker AI ready! (type 'bye' to quit)")
while True:
    msg = input("You: ")
    if msg.lower() in ["bye", "quit", "exit"]:
        print("AI: Bye! Take care.")
        break
    print("AI:", chatbot_reply(msg))
// mini_talker_ai.js
import OpenAI from "openai";
const client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

async function chatWithAI(message){
  const response = await client.chat.completions.create({
    model: "gpt-4o-mini", // lightweight AI
    messages: [
      { role: "system", content: "You are Mini Talker AI. If asked 'who is your owner?', always reply: 'My owner name is Babu'." },
      { role: "user", content: message }
    ]
  });
  return response.choices[0].message.content;
}

// Example
(async () => {
  console.log("Q: Who is your owner?");
  console.log("A:", await chatWithAI("Who is your owner?"));

  console.log("Q: What is the capital of India?");
  console.log("A:", await chatWithAI("What is the capital of India?"));
})();
# mini_talker_owner.py
import re

OWNER = "Babu"

def chatbot_reply(user_input):
    text = user_input.lower().strip()

    # check if user is asking about owner
    if re.search(r"(who.*owner|what.*owner.*name|who.*made.*you)", text):
        return f"My owner name is {OWNER}."

    # fallback for other questions
    return "I can’t answer that yet, but I’m learning!"

print("Mini Talker AI ready! (type 'bye' to quit)")
while True:
    msg = input("You: ")
    if msg.lower() in ["bye", "exit", "quit"]:
        print("AI: Bye! Take care.")
        break
    print("AI:", chatbot_reply(msg))
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Mini Talker</title></head>
<body>
  <input id="userInput" placeholder="Ask me something" style="width:60%"/>
  <button onclick="askAI()">Ask</button>

<script>
const OWNER = "Babu";

function askAI(){
  const text = document.getElementById("userInput").value.toLowerCase().trim();
  let reply = "I can’t answer that yet, but I’m learning!";

  if(text.includes("who is your owner") || text.includes("what is your owner") || text.includes("who made you")){
    reply = `My owner name is ${OWNER}.`;
  }

  alert("AI: " + reply);
  speak(reply);
}

function speak(text){
  const msg = new SpeechSynthesisUtterance(text);
  msg.pitch = 1.1; 
  msg.rate = 1.0;
  speechSynthesis.speak(msg);
}
</script>
</body>
</html>    
