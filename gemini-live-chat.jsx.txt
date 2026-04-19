import { useState, useRef, useCallback, useEffect } from "react";

// ─── Audio helpers ────────────────────────────────────────────────────────────
const f32ToI16 = (f32) => {
  const i16 = new Int16Array(f32.length);
  for (let i = 0; i < f32.length; i++) {
    const s = Math.max(-1, Math.min(1, f32[i]));
    i16[i] = s < 0 ? s * 0x8000 : s * 0x7fff;
  }
  return i16;
};
const b64ToF32 = (b64) => {
  const bin = atob(b64);
  const u8 = new Uint8Array(bin.length);
  for (let i = 0; i < bin.length; i++) u8[i] = bin.charCodeAt(i);
  const i16 = new Int16Array(u8.buffer);
  const f32 = new Float32Array(i16.length);
  for (let i = 0; i < i16.length; i++) f32[i] = i16[i] / 0x8000;
  return f32;
};
const abToB64 = (ab) => {
  const u8 = new Uint8Array(ab);
  let bin = "";
  for (let i = 0; i < u8.length; i++) bin += String.fromCharCode(u8[i]);
  return btoa(bin);
};

const WS = (k) =>
  `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1alpha.GenerativeService.BidiGenerateContent?key=${k}`;
const MODEL = "models/gemini-2.0-flash-live-001";

// ─── Waveform bars component ──────────────────────────────────────────────────
function WaveBar({ active, color, delay }) {
  return (
    <div
      style={{
        width: 3,
        borderRadius: 2,
        background: color,
        animationName: active ? "wavebar" : "none",
        animationDuration: "0.8s",
        animationTimingFunction: "ease-in-out",
        animationIterationCount: "infinite",
        animationDelay: delay,
        height: active ? undefined : 4,
        minHeight: 4,
        maxHeight: 28,
      }}
    />
  );
}

// ─── Status indicator config ──────────────────────────────────────────────────
const STATUS = {
  idle:       { color: "#3a3a5c", label: "Offline",      emoji: "○" },
  connecting: { color: "#f59e0b", label: "Connecting…",  emoji: "◌" },
  ready:      { color: "#22d3a5", label: "Ready",        emoji: "●" },
  listening:  { color: "#38bdf8", label: "Listening",    emoji: "◉" },
  thinking:   { color: "#a78bfa", label: "Thinking",     emoji: "◌" },
  speaking:   { color: "#f472b6", label: "Speaking",     emoji: "◉" },
};

export default function GeminiLive() {
  const [apiKey, setApiKey]         = useState("");
  const [showKey, setShowKey]       = useState(true);
  const [showKeyText, setShowKeyText] = useState(false);
  const [connected, setConnected]   = useState(false);
  const [status, setStatus]         = useState("idle");
  const [messages, setMessages]     = useState([]);
  const [textInput, setTextInput]   = useState("");
  const [micOn, setMicOn]           = useState(false);
  const [err, setErr]               = useState("");

  // refs — avoid stale closures
  const wsRef         = useRef(null);
  const captureCtx    = useRef(null);
  const playCtx       = useRef(null);
  const procRef       = useRef(null);
  const streamRef     = useRef(null);
  const setupDone     = useRef(false);
  const nextPlayTime  = useRef(0);
  const micOnRef      = useRef(false);
  const connectedRef  = useRef(false);
  const modelMsgId    = useRef(null);
  const userMsgId     = useRef(null);
  const endRef        = useRef(null);

  useEffect(() => { micOnRef.current   = micOn;    }, [micOn]);
  useEffect(() => { connectedRef.current = connected; }, [connected]);
  useEffect(() => { endRef.current?.scrollIntoView({ behavior: "smooth" }); }, [messages]);

  // ── Schedule PCM audio on WebAudio timeline (gap-free) ────────────────────
  const scheduleAudio = useCallback((f32) => {
    const ctx = playCtx.current;
    if (!ctx) return;
    const buf = ctx.createBuffer(1, f32.length, 24000);
    buf.getChannelData(0).set(f32);
    const src = ctx.createBufferSource();
    src.buffer = buf;
    src.connect(ctx.destination);
    const now = ctx.currentTime;
    const at  = Math.max(now, nextPlayTime.current);
    src.start(at);
    nextPlayTime.current = at + buf.duration;
    setStatus("speaking");
    src.onended = () => {
      if (nextPlayTime.current <= ctx.currentTime + 0.05) {
        setStatus(micOnRef.current ? "listening" : "ready");
      }
    };
  }, []);

  // ── WS message handler ─────────────────────────────────────────────────────
  const onMessage = useCallback((evt) => {
    try {
      const msg = JSON.parse(evt.data);

      if (msg.setupComplete) {
        setupDone.current = true;
        setConnected(true);
        setStatus("ready");
        setErr("");
        return;
      }

      const sc = msg.serverContent;
      if (!sc) return;

      if (sc.modelTurn?.parts) {
        for (const part of sc.modelTurn.parts) {
          if (part.inlineData?.data) {
            scheduleAudio(b64ToF32(part.inlineData.data));
          }
          if (part.text) {
            const id = modelMsgId.current || (modelMsgId.current = Date.now());
            setMessages((prev) => {
              const idx = prev.findIndex((m) => m.id === id);
              if (idx >= 0) {
                const u = [...prev];
                u[idx] = { ...u[idx], text: u[idx].text + part.text };
                return u;
              }
              return [...prev, { id, role: "model", text: part.text, streaming: true }];
            });
          }
        }
      }

      // Audio output transcription
      if (sc.outputTranscription?.text) {
        const id = modelMsgId.current || (modelMsgId.current = Date.now() + 1);
        const t  = sc.outputTranscription.text;
        setMessages((prev) => {
          const idx = prev.findIndex((m) => m.id === id);
          if (idx >= 0) {
            const u = [...prev];
            u[idx] = { ...u[idx], text: u[idx].text + t };
            return u;
          }
          return [...prev, { id, role: "model", text: t, streaming: true }];
        });
      }

      // Voice input transcription
      if (sc.inputTranscription?.text) {
        const id = userMsgId.current || (userMsgId.current = Date.now() + 2);
        const t  = sc.inputTranscription.text;
        setMessages((prev) => {
          const idx = prev.findIndex((m) => m.id === id);
          if (idx >= 0) {
            const u = [...prev];
            u[idx] = { ...u[idx], text: u[idx].text + t };
            return u;
          }
          return [...prev, { id, role: "user", text: t, streaming: true }];
        });
      }

      if (sc.turnComplete) {
        modelMsgId.current = null;
        userMsgId.current  = null;
        setMessages((prev) => prev.map((m) => ({ ...m, streaming: false })));
        setStatus(micOnRef.current ? "listening" : "ready");
      }
    } catch (e) {
      console.error("WS parse error:", e);
    }
  }, [scheduleAudio]);

  // ── Mic helpers (defined before connect/disconnect) ─────────────────────────
  const stopMicInternal = () => {
    procRef.current?.disconnect();
    procRef.current = null;
    captureCtx.current?.close().catch(() => {});
    captureCtx.current = null;
    streamRef.current?.getTracks().forEach((t) => t.stop());
    streamRef.current = null;
    setMicOn(false);
  };

  const startMic = useCallback(async () => {
    if (!setupDone.current) return;
    try {
      const stream = await navigator.mediaDevices.getUserMedia({
        audio: { channelCount: 1, sampleRate: 16000, echoCancellation: true, noiseSuppression: true },
      });
      streamRef.current = stream;
      const ctx = new AudioContext({ sampleRate: 16000 });
      captureCtx.current = ctx;
      const src  = ctx.createMediaStreamSource(stream);
      const proc = ctx.createScriptProcessor(4096, 1, 1);
      procRef.current = proc;
      proc.onaudioprocess = (e) => {
        if (!wsRef.current || wsRef.current.readyState !== WebSocket.OPEN) return;
        const i16    = f32ToI16(e.inputBuffer.getChannelData(0));
        const b64    = abToB64(i16.buffer);
        wsRef.current.send(JSON.stringify({
          realtime_input: { media_chunks: [{ mime_type: "audio/pcm;rate=16000", data: b64 }] },
        }));
      };
      src.connect(proc);
      proc.connect(ctx.destination);
      setMicOn(true);
      setStatus("listening");
    } catch (e) {
      setErr("Microphone access denied. Allow mic permission and try again.");
    }
  }, []);

  const stopMic = useCallback(() => {
    stopMicInternal();
    if (connectedRef.current) setStatus("ready");
  }, []);

  // ── Connect ────────────────────────────────────────────────────────────────
  const connect = useCallback(() => {
    if (!apiKey.trim()) { setErr("Please enter your Gemini API key first."); return; }
    setStatus("connecting");
    setErr("");
    try {
      playCtx.current     = new AudioContext({ sampleRate: 24000 });
      nextPlayTime.current = 0;
      const socket = new WebSocket(WS(apiKey.trim()));
      wsRef.current = socket;
      socket.onopen = () => {
        socket.send(JSON.stringify({
          setup: {
            model: MODEL,
            generation_config: {
              response_modalities: ["AUDIO"],
              speech_config: { voice_config: { prebuilt_voice_config: { voice_name: "Puck" } } },
            },
            input_audio_transcription: {},
            output_audio_transcription: {},
            system_instruction: {
              parts: [{ text: "You are a helpful, friendly AI assistant. Be concise and conversational. Keep responses natural and short unless asked for more detail." }],
            },
          },
        }));
      };
      socket.onmessage = onMessage;
      socket.onerror   = () => { setErr("Connection error. Check your API key."); setStatus("idle"); setConnected(false); };
      socket.onclose   = () => { setConnected(false); setStatus("idle"); setupDone.current = false; stopMicInternal(); };
      setShowKey(false);
    } catch (e) {
      setErr("Failed to connect: " + e.message);
      setStatus("idle");
    }
  }, [apiKey, onMessage]);

  // ── Disconnect ─────────────────────────────────────────────────────────────
  const disconnect = useCallback(() => {
    stopMicInternal();
    wsRef.current?.close();
    wsRef.current = null;
    playCtx.current?.close().catch(() => {});
    playCtx.current = null;
    setConnected(false);
    setStatus("idle");
    setMessages([]);
    setMicOn(false);
  }, []);

  // ── Send text ──────────────────────────────────────────────────────────────
  const sendText = useCallback(() => {
    const txt = textInput.trim();
    if (!txt || !wsRef.current || wsRef.current.readyState !== WebSocket.OPEN) return;
    const id = Date.now();
    setMessages((prev) => [...prev, { id, role: "user", text: txt }]);
    wsRef.current.send(JSON.stringify({
      client_content: { turns: [{ role: "user", parts: [{ text: txt }] }], turn_complete: true },
    }));
    setTextInput("");
    setStatus("thinking");
    modelMsgId.current = Date.now() + 100;
  }, [textInput]);

  const sc = STATUS[status] || STATUS.idle;
  const waveActive  = micOn || status === "speaking";
  const waveColor   = status === "speaking" ? "#f472b6" : "#38bdf8";

  return (
    <>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Space+Mono:wght@400;700&family=DM+Sans:wght@300;400;500;600&display=swap');
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { background: #07070f; }
        ::-webkit-scrollbar { width: 3px; }
        ::-webkit-scrollbar-track { background: transparent; }
        ::-webkit-scrollbar-thumb { background: #1e1e36; border-radius: 2px; }

        @keyframes wavebar {
          0%,100% { height: 4px; }
          50%      { height: 24px; }
        }
        @keyframes ripple {
          0%   { transform: scale(1); opacity: 0.7; }
          100% { transform: scale(2.1); opacity: 0; }
        }
        @keyframes spin {
          to { transform: rotate(360deg); }
        }
        @keyframes blink {
          0%,100% { opacity: 1; }
          50%      { opacity: 0; }
        }
        @keyframes fadeIn {
          from { opacity:0; transform: translateY(6px); }
          to   { opacity:1; transform: translateY(0); }
        }
        @keyframes glow {
          0%,100% { box-shadow: 0 0 12px rgba(56,189,248,0.3); }
          50%      { box-shadow: 0 0 28px rgba(56,189,248,0.6); }
        }

        .msg { animation: fadeIn 0.25s ease forwards; }

        .key-input {
          background: #0f0f1e;
          border: 1px solid #1e1e38;
          border-radius: 10px;
          color: #c8d0e8;
          font-family: 'Space Mono', monospace;
          font-size: 12px;
          padding: 10px 14px;
          outline: none;
          width: 100%;
          transition: border-color 0.2s;
        }
        .key-input:focus { border-color: #38bdf8; }
        .key-input::placeholder { color: #2a2a48; }

        .text-input {
          flex: 1;
          background: #0f0f1e;
          border: 1px solid #1e1e38;
          border-radius: 22px;
          color: #c8d0e8;
          font-family: 'DM Sans', sans-serif;
          font-size: 14px;
          padding: 10px 16px;
          outline: none;
          transition: border-color 0.2s;
        }
        .text-input:focus { border-color: #38bdf8; }
        .text-input::placeholder { color: #2a2a48; }
        .text-input:disabled { opacity: 0.4; }

        .btn-send {
          width: 40px; height: 40px;
          border-radius: 50%;
          border: none;
          cursor: pointer;
          font-size: 15px;
          display: flex; align-items: center; justify-content: center;
          transition: all 0.2s;
          flex-shrink: 0;
        }
        .btn-send:disabled { cursor: not-allowed; opacity: 0.4; }
      `}</style>

      <div style={{
        display: "flex", flexDirection: "column",
        height: "100vh", maxWidth: 480, margin: "0 auto",
        background: "#07070f", fontFamily: "'DM Sans', sans-serif",
        color: "#c8d0e8", position: "relative", overflow: "hidden",
      }}>

        {/* ── Ambient glow bg ── */}
        <div style={{
          position: "absolute", top: -120, left: "50%",
          transform: "translateX(-50%)",
          width: 360, height: 360, borderRadius: "50%",
          background: `radial-gradient(ellipse, ${sc.color}18 0%, transparent 70%)`,
          pointerEvents: "none", transition: "background 0.6s ease",
          zIndex: 0,
        }} />

        {/* ── Header ── */}
        <div style={{
          padding: "14px 18px", display: "flex",
          alignItems: "center", justifyContent: "space-between",
          borderBottom: "1px solid #0e0e1e", flexShrink: 0,
          position: "relative", zIndex: 2, background: "#07070f",
        }}>
          <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
            {/* Logo */}
            <div style={{
              width: 36, height: 36, borderRadius: 10,
              background: "linear-gradient(135deg, #4285f4 0%, #ea4335 40%, #fbbc04 70%, #34a853 100%)",
              display: "flex", alignItems: "center", justifyContent: "center",
              fontFamily: "'Space Mono', monospace", fontWeight: 700, fontSize: 16, color: "white",
              flexShrink: 0,
            }}>G</div>

            <div>
              <div style={{ fontSize: 15, fontWeight: 600, color: "#e0e8ff", letterSpacing: -0.3 }}>
                Gemini Live
              </div>
              <div style={{ display: "flex", alignItems: "center", gap: 6, marginTop: 2 }}>
                {/* Animated dot */}
                {(status === "connecting") ? (
                  <div style={{
                    width: 7, height: 7, border: `2px solid ${sc.color}`,
                    borderTopColor: "transparent", borderRadius: "50%",
                    animation: "spin 0.8s linear infinite",
                  }} />
                ) : (
                  <div style={{
                    width: 6, height: 6, borderRadius: "50%",
                    background: sc.color,
                    boxShadow: status !== "idle" ? `0 0 8px ${sc.color}` : "none",
                    animation: ["listening","speaking","thinking"].includes(status)
                      ? "glow 1.4s ease-in-out infinite" : "none",
                  }} />
                )}
                <span style={{
                  fontFamily: "'Space Mono', monospace",
                  fontSize: 10, color: sc.color, letterSpacing: 1,
                  textTransform: "uppercase",
                }}>{sc.label}</span>
              </div>
            </div>
          </div>

          {/* Settings toggle */}
          <button
            onClick={() => setShowKey((s) => !s)}
            style={{
              background: showKey ? "#1a1a30" : "transparent",
              border: "1px solid #1e1e36",
              borderRadius: 8, padding: "7px 10px",
              color: "#5050a0", cursor: "pointer", fontSize: 14,
              transition: "all 0.2s",
            }}
          >
            {showKey ? "✕" : "⚙"}
          </button>
        </div>

        {/* ── API Key Panel ── */}
        {showKey && (
          <div style={{
            background: "#09091a", borderBottom: "1px solid #0e0e1e",
            padding: "14px 18px", flexShrink: 0, position: "relative", zIndex: 2,
          }}>
            <label style={{ fontFamily: "'Space Mono', monospace", fontSize: 10, color: "#3a3a6a", letterSpacing: 1.5, display: "block", marginBottom: 8 }}>
              API KEY
            </label>
            <div style={{ position: "relative" }}>
              <input
                className="key-input"
                type={showKeyText ? "text" : "password"}
                value={apiKey}
                onChange={(e) => setApiKey(e.target.value)}
                onKeyDown={(e) => e.key === "Enter" && connect()}
                placeholder="AIzaSy..."
                style={{ paddingRight: 80 }}
              />
              <div style={{ position: "absolute", right: 8, top: "50%", transform: "translateY(-50%)", display: "flex", gap: 6 }}>
                <button
                  onClick={() => setShowKeyText((s) => !s)}
                  style={{ background: "transparent", border: "none", color: "#3a3a60", cursor: "pointer", fontSize: 13, padding: "2px 4px" }}
                >
                  {showKeyText ? "🙈" : "👁"}
                </button>
              </div>
            </div>

            <div style={{ display: "flex", gap: 8, marginTop: 10 }}>
              <button
                onClick={connected ? disconnect : connect}
                style={{
                  flex: 1, padding: "9px 0",
                  background: connected
                    ? "linear-gradient(135deg, #7f1d1d, #991b1b)"
                    : "linear-gradient(135deg, #1565c0, #4285f4)",
                  border: "none", borderRadius: 9, color: "white",
                  fontFamily: "'Space Mono', monospace", fontSize: 12,
                  fontWeight: 700, cursor: "pointer", letterSpacing: 0.5,
                  transition: "opacity 0.2s",
                }}
              >
                {connected ? "DISCONNECT" : "CONNECT"}
              </button>
            </div>

            {err && (
              <div style={{ fontSize: 12, color: "#f87171", marginTop: 8, fontFamily: "'Space Mono', monospace", lineHeight: 1.4 }}>
                ⚠ {err}
              </div>
            )}
            <div style={{ fontSize: 11, color: "#252545", marginTop: 8, fontFamily: "'Space Mono', monospace" }}>
              aistudio.google.com/app/apikey
            </div>
          </div>
        )}

        {/* ── Chat messages ── */}
        <div style={{
          flex: 1, overflowY: "auto", padding: "16px 18px",
          display: "flex", flexDirection: "column", gap: 14,
          position: "relative", zIndex: 1,
        }}>
          {messages.length === 0 && (
            <div style={{ display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", flex: 1, marginTop: 40 }}>
              {/* Big mic visual */}
              <div style={{
                width: 80, height: 80, borderRadius: "50%",
                border: "2px solid #1a1a30",
                display: "flex", alignItems: "center", justifyContent: "center",
                fontSize: 34, marginBottom: 20, color: "#2a2a50",
              }}>
                🎙
              </div>
              <div style={{ fontFamily: "'Space Mono', monospace", fontSize: 12, color: "#252545", letterSpacing: 1, marginBottom: 6 }}>
                GEMINI LIVE VOICE
              </div>
              <div style={{ fontSize: 13, color: "#2a2a4a", textAlign: "center", maxWidth: 220, lineHeight: 1.6 }}>
                {connected ? "Tap the mic button to start talking" : "Connect your API key to begin"}
              </div>
            </div>
          )}

          {messages.map((msg) => (
            <div
              key={msg.id}
              className="msg"
              style={{ display: "flex", justifyContent: msg.role === "user" ? "flex-end" : "flex-start", alignItems: "flex-end", gap: 8 }}
            >
              {msg.role === "model" && (
                <div style={{
                  width: 26, height: 26, borderRadius: 8, flexShrink: 0,
                  background: "linear-gradient(135deg, #4285f4, #1a56c4)",
                  display: "flex", alignItems: "center", justifyContent: "center",
                  fontFamily: "'Space Mono', monospace", fontSize: 12, fontWeight: 700, color: "white",
                }}>G</div>
              )}
              <div style={{
                maxWidth: "73%",
                padding: "9px 14px",
                borderRadius: msg.role === "user" ? "16px 16px 4px 16px" : "16px 16px 16px 4px",
                background: msg.role === "user"
                  ? "linear-gradient(135deg, #1d4ed8, #2563eb)"
                  : "#0e0e1e",
                border: msg.role === "model" ? "1px solid #191930" : "none",
                fontSize: 14, lineHeight: 1.55,
                color: msg.role === "user" ? "#ddeeff" : "#b0b8d8",
                fontWeight: msg.role === "user" ? 400 : 300,
              }}>
                {msg.text || (
                  <span style={{ color: "#30305a", fontStyle: "italic", fontSize: 13 }}>
                    🔊 Speaking…
                  </span>
                )}
                {msg.streaming && (
                  <span style={{
                    display: "inline-block", width: 2, height: 14,
                    background: "#4285f4", marginLeft: 3, borderRadius: 1,
                    verticalAlign: "middle",
                    animation: "blink 0.9s step-end infinite",
                  }} />
                )}
              </div>
            </div>
          ))}
          <div ref={endRef} />
        </div>

        {/* ── Controls ── */}
        <div style={{
          padding: "16px 18px 28px", background: "#07070f",
          borderTop: "1px solid #0e0e1e", flexShrink: 0,
          position: "relative", zIndex: 2,
        }}>
          {/* Waveform + mic button row */}
          <div style={{ display: "flex", alignItems: "center", justifyContent: "center", gap: 6, marginBottom: 16 }}>

            {/* Left wave bars */}
            {[0.3, 0.1, 0.5, 0.2].map((d, i) => (
              <WaveBar key={i} active={waveActive} color={waveColor} delay={`${d}s`} />
            ))}

            {/* Mic button */}
            <div style={{ position: "relative", margin: "0 12px" }}>
              {micOn && <>
                <div style={{
                  position: "absolute", inset: -8, borderRadius: "50%",
                  border: `1px solid #ef444466`,
                  animation: "ripple 1.6s ease-out infinite",
                }} />
                <div style={{
                  position: "absolute", inset: -8, borderRadius: "50%",
                  border: `1px solid #ef444444`,
                  animation: "ripple 1.6s ease-out 0.55s infinite",
                }} />
              </>}
              <button
                onClick={micOn ? stopMic : startMic}
                disabled={!connected}
                style={{
                  width: 66, height: 66, borderRadius: "50%",
                  border: "none", cursor: connected ? "pointer" : "not-allowed",
                  background: !connected
                    ? "#0e0e1e"
                    : micOn
                      ? "linear-gradient(135deg, #ef4444, #b91c1c)"
                      : "linear-gradient(135deg, #1d4ed8, #4285f4)",
                  fontSize: 26,
                  display: "flex", alignItems: "center", justifyContent: "center",
                  position: "relative", zIndex: 1,
                  boxShadow: micOn
                    ? "0 0 0 3px #ef444440, 0 8px 30px rgba(239,68,68,.4)"
                    : connected
                      ? "0 0 0 3px #4285f420, 0 8px 30px rgba(66,133,244,.3)"
                      : "none",
                  transition: "all 0.25s ease",
                  transform: micOn ? "scale(1.05)" : "scale(1)",
                }}
              >
                {micOn ? "⏹" : "🎙"}
              </button>
            </div>

            {/* Right wave bars */}
            {[0.4, 0.15, 0.35, 0.05].map((d, i) => (
              <WaveBar key={i} active={waveActive} color={waveColor} delay={`${d}s`} />
            ))}
          </div>

          {/* Mic status label */}
          <div style={{
            textAlign: "center", marginBottom: 14,
            fontFamily: "'Space Mono', monospace", fontSize: 10,
            letterSpacing: 1.5, textTransform: "uppercase",
            color: micOn ? "#ef4444" : status === "speaking" ? "#f472b6" : "#252545",
            minHeight: 14,
          }}>
            {micOn ? "● MIC ACTIVE" : status === "speaking" ? "◉ AI SPEAKING" : ""}
          </div>

          {/* Text input row */}
          <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
            <input
              className="text-input"
              value={textInput}
              onChange={(e) => setTextInput(e.target.value)}
              onKeyDown={(e) => e.key === "Enter" && sendText()}
              placeholder={connected ? "Type a message…" : "Connect to start…"}
              disabled={!connected}
            />
            <button
              className="btn-send"
              onClick={sendText}
              disabled={!connected || !textInput.trim()}
              style={{
                background: connected && textInput.trim()
                  ? "linear-gradient(135deg, #1d4ed8, #4285f4)"
                  : "#0e0e1e",
                color: "white",
                boxShadow: connected && textInput.trim()
                  ? "0 4px 14px rgba(66,133,244,.4)" : "none",
              }}
            >
              ➤
            </button>
          </div>
        </div>
      </div>
    </>
  );
}
