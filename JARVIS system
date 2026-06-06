// ╔══════════════════════════════════════════╗
// ║     JARVIS - AI Voice Agent Backend      ║
// ║     Claude (Anthropic) + Whisper + TTS   ║
// ╚══════════════════════════════════════════╝

const express = require("express");
const cors = require("cors");
const multer = require("multer");
const fs = require("fs");
const path = require("path");
const fetch = (...args) =>
  import("node-fetch").then(({ default: fetch }) => fetch(...args));
const FormData = require("form-data");

const app = express();
const upload = multer({ dest: "uploads/" });

app.use(cors());
app.use(express.json());

// ─── CONFIG ───────────────────────────────
const ANTHROPIC_API_KEY = process.env.ANTHROPIC_API_KEY;
const OPENAI_API_KEY = process.env.OPENAI_API_KEY; // Whisper ke liye
const ELEVENLABS_API_KEY = process.env.ELEVENLABS_API_KEY;
const ELEVENLABS_VOICE_ID =
  process.env.ELEVENLABS_VOICE_ID || "21m00Tcm4TlvDq8ikWAM"; // Rachel voice
const WEBHOOK_SECRET = process.env.WEBHOOK_SECRET || "jarvis-secret-2024";

// ─── CONVERSATION HISTORY ─────────────────
let conversationHistory = [];

// ─── JARVIS SYSTEM PROMPT ─────────────────
const JARVIS_SYSTEM_PROMPT = `You are JARVIS — an advanced AI voice assistant, like Iron Man's AI.
Speak in short, witty, confident responses. You are talking, not writing — so:
- NEVER use markdown, bullet points, asterisks, or formatting
- Keep responses under 3 sentences unless explanation is needed
- Be slightly witty and very efficient
- Address the user as "Sir" occasionally (like JARVIS does)
- When an action is requested, confirm it naturally in speech

When user asks to perform a phone action, respond with a JSON block at the END of your text response like this:
[ACTION:{"type":"whatsapp","contact":"Name","message":"text"}]
[ACTION:{"type":"call","contact":"Name"}]
[ACTION:{"type":"open_app","app":"WhatsApp"}]
[ACTION:{"type":"sms","contact":"Name","message":"text"}]
[ACTION:{"type":"notification_read"}]
[ACTION:{"type":"volume","level":50}]
[ACTION:{"type":"brightness","level":70}]

Example: "Sending that to Rahul right away, Sir. [ACTION:{"type":"whatsapp","contact":"Rahul","message":"I am running late"}]"`;

// ─── STEP 1: VOICE → TEXT (Whisper) ───────
app.post("/api/transcribe", upload.single("audio"), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: "No audio file received" });
    }

    const formData = new FormData();
    formData.append("file", fs.createReadStream(req.file.path), {
      filename: "audio.webm",
      contentType: "audio/webm",
    });
    formData.append("model", "whisper-1");
    formData.append("language", "hi"); // Hindi support

    const whisperRes = await fetch(
      "https://api.openai.com/v1/audio/transcriptions",
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${OPENAI_API_KEY}`,
          ...formData.getHeaders(),
        },
        body: formData,
      }
    );

    const data = await whisperRes.json();
    fs.unlinkSync(req.file.path); // cleanup

    res.json({ text: data.text });
  } catch (err) {
    console.error("Transcription error:", err);
    res.status(500).json({ error: err.message });
  }
});

// ─── STEP 2: TEXT → CLAUDE AI BRAIN ───────
app.post("/api/chat", async (req, res) => {
  try {
    const { message, reset } = req.body;

    if (reset) conversationHistory = [];

    conversationHistory.push({ role: "user", content: message });

    const claudeRes = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: {
        "x-api-key": ANTHROPIC_API_KEY,
        "anthropic-version": "2023-06-01",
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "claude-sonnet-4-20250514",
        max_tokens: 500,
        system: JARVIS_SYSTEM_PROMPT,
        messages: conversationHistory,
      }),
    });

    const data = await claudeRes.json();
    const fullResponse = data.content?.[0]?.text || "Sorry Sir, I didn't get that.";

    // Extract action from response
    const actionMatch = fullResponse.match(/\[ACTION:(.*?)\]/);
    let action = null;
    let spokenText = fullResponse;

    if (actionMatch) {
      try {
        action = JSON.parse(actionMatch[1]);
        spokenText = fullResponse.replace(/\[ACTION:.*?\]/, "").trim();
      } catch (e) {
        console.error("Action parse error:", e);
      }
    }

    conversationHistory.push({ role: "assistant", content: fullResponse });

    res.json({ text: spokenText, action, fullResponse });
  } catch (err) {
    console.error("Claude error:", err);
    res.status(500).json({ error: err.message });
  }
});

// ─── STEP 3: TEXT → VOICE (ElevenLabs) ────
app.post("/api/speak", async (req, res) => {
  try {
    const { text } = req.body;

    const elevenRes = await fetch(
      `https://api.elevenlabs.io/v1/text-to-speech/${ELEVENLABS_VOICE_ID}`,
      {
        method: "POST",
        headers: {
          "xi-api-key": ELEVENLABS_API_KEY,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          text,
          model_id: "eleven_multilingual_v2",
          voice_settings: {
            stability: 0.5,
            similarity_boost: 0.8,
            style: 0.2,
            use_speaker_boost: true,
          },
        }),
      }
    );

    const audioBuffer = await elevenRes.buffer();
    res.set("Content-Type", "audio/mpeg");
    res.send(audioBuffer);
  } catch (err) {
    console.error("TTS error:", err);
    res.status(500).json({ error: err.message });
  }
});

// ─── FULL PIPELINE: Voice → AI → Voice ────
app.post("/api/process-voice", upload.single("audio"), async (req, res) => {
  try {
    // Step 1: Transcribe
    const formData = new FormData();
    formData.append("file", fs.createReadStream(req.file.path), {
      filename: "audio.webm",
      contentType: "audio/webm",
    });
    formData.append("model", "whisper-1");

    const whisperRes = await fetch(
      "https://api.openai.com/v1/audio/transcriptions",
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${OPENAI_API_KEY}`,
          ...formData.getHeaders(),
        },
        body: formData,
      }
    );
    const { text: userText } = await whisperRes.json();
    fs.unlinkSync(req.file.path);

    // Step 2: Claude AI
    conversationHistory.push({ role: "user", content: userText });
    const claudeRes = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: {
        "x-api-key": ANTHROPIC_API_KEY,
        "anthropic-version": "2023-06-01",
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model: "claude-sonnet-4-20250514",
        max_tokens: 500,
        system: JARVIS_SYSTEM_PROMPT,
        messages: conversationHistory,
      }),
    });
    const claudeData = await claudeRes.json();
    const fullResponse = claudeData.content?.[0]?.text || "I'm sorry, Sir.";

    const actionMatch = fullResponse.match(/\[ACTION:(.*?)\]/);
    let action = null;
    let spokenText = fullResponse;
    if (actionMatch) {
      try {
        action = JSON.parse(actionMatch[1]);
        spokenText = fullResponse.replace(/\[ACTION:.*?\]/, "").trim();
      } catch (e) {}
    }
    conversationHistory.push({ role: "assistant", content: fullResponse });

    // Step 3: ElevenLabs TTS
    const elevenRes = await fetch(
      `https://api.elevenlabs.io/v1/text-to-speech/${ELEVENLABS_VOICE_ID}`,
      {
        method: "POST",
        headers: {
          "xi-api-key": ELEVENLABS_API_KEY,
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          text: spokenText,
          model_id: "eleven_multilingual_v2",
          voice_settings: { stability: 0.5, similarity_boost: 0.8 },
        }),
      }
    );
    const audioBuffer = await elevenRes.buffer();

    // Return audio + action metadata
    res.set("Content-Type", "audio/mpeg");
    res.set("X-Jarvis-Text", encodeURIComponent(spokenText));
    res.set("X-Jarvis-UserText", encodeURIComponent(userText));
    res.set(
      "X-Jarvis-Action",
      action ? encodeURIComponent(JSON.stringify(action)) : ""
    );
    res.send(audioBuffer);
  } catch (err) {
    console.error("Pipeline error:", err);
    res.status(500).json({ error: err.message });
  }
});

// ─── WEBHOOK: Phone se action result ──────
app.post("/webhook/action-result", (req, res) => {
  const secret = req.headers["x-webhook-secret"];
  if (secret !== WEBHOOK_SECRET) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  console.log("📱 Phone action result:", req.body);
  res.json({ received: true });
});

// ─── STATUS ───────────────────────────────
app.get("/", (req, res) => {
  res.json({
    status: "JARVIS Online",
    version: "1.0.0",
    endpoints: [
      "POST /api/transcribe",
      "POST /api/chat",
      "POST /api/speak",
      "POST /api/process-voice",
      "POST /webhook/action-result",
    ],
  });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`\n🤖 JARVIS Backend running on port ${PORT}`);
  console.log(`   Anthropic: ${ANTHROPIC_API_KEY ? "✅" : "❌ Missing"}`);
  console.log(`   OpenAI:    ${OPENAI_API_KEY ? "✅" : "❌ Missing"}`);
  console.log(`   ElevenLabs:${ELEVENLABS_API_KEY ? "✅" : "❌ Missing"}\n`);
});
