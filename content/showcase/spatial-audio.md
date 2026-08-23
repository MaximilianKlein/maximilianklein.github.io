---
date: '2026-01-02T18:33:12+02:00'
title: 'Cross-Talking Stereo ITD/ILD Tester'
type: 'showcase'
ai-support: 'Generated'
---

Sitting in remote meetings all day long I'm regularly wondering: Why is it so much harder separating voices from different participants if they cross talk than in a meeting room. I knew from university that we can identify where a sound comes from simply by utilizing delay and volume differences between the ears. Then I thought: That might make the difference and it sounds like something you can solve with software. I drafted a prototype with some AI tooling and was surprised how well it worked. I found out that Zoom has some kind of feature like this that they patented. It is in combination with the position of the video AFAIK. That's where I stopped working on this. I'm not a patent expert, but I still think it is a fun demo and maybe something that is integrated into video call software everywhere.

---

### What is this tool?

When multiple people talk at the same time, it can be hard to follow any single conversation. This interactive tool uses interaural time differences (ITD) and interaural level differences (ILD) to spatially separate multiple voice recordings, making it easier to distinguish between different speakers even when they overlap.

The tool automatically loads three voice recordings and lets you position each one in stereo space by adjusting:

- **ITD (Interaural Time Difference)**: A tiny delay between left and right ears (in microseconds), simulating sounds coming from different horizontal positions
- **ILD (Interaural Level Difference)**: A volume difference between left and right channels (in decibels)

By giving each speaker a different spatial position, your brain can more easily separate and follow individual conversations, even when they're playing simultaneously.

<div style="margin: 2rem 0;">

<style>
  .spatial-audio-container { font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; margin: 20px 0; line-height: 1.25; }
  .spatial-audio-container h1 { margin: 0 0 10px; }
  .spatial-audio-container .controls { display: flex; gap: 10px; flex-wrap: wrap; align-items: center; margin: 12px 0 18px; }
  .spatial-audio-container button {
    padding: 10px 16px;
    cursor: pointer;
    border: 1px solid #ccc;
    border-radius: 6px;
    background-color: #f8f9fa;
    color: #333;
    font-size: 14px;
    font-weight: 500;
    transition: all 0.2s ease;
  }
  .spatial-audio-container button:hover {
    background-color: #e9ecef;
    border-color: #999;
  }
  .spatial-audio-container button:active {
    background-color: #dee2e6;
    transform: translateY(1px);
  }
  .spatial-audio-container input[type="range"] { width: 220px; }
  .spatial-audio-container .hint { color: #444; font-size: 14px; max-width: 900px; }
  .spatial-audio-container .track { border: 1px solid #ddd; border-radius: 10px; padding: 12px; margin: 10px 0; }
  .spatial-audio-container .row { display: grid; grid-template-columns: 170px 1fr 110px; gap: 10px; align-items: center; margin: 8px 0; }
  .spatial-audio-container .row label { font-weight: 600; }
  .spatial-audio-container .val { text-align: right; font-variant-numeric: tabular-nums; color: #333; }
  .spatial-audio-container .titleRow { display:flex; justify-content: space-between; gap: 10px; align-items: baseline; }
  .spatial-audio-container .fileName { font-weight: 700; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; max-width: 70vw; }
  .spatial-audio-container .small { font-size: 13px; color: #555; }
  .spatial-audio-container .warn { color: #7a3; }
</style>

<div class="spatial-audio-container">
  <h1>Multi-talkers Stereo ITD/ILD Tester</h1>
  <div class="hint">
    Three voice recordings are automatically loaded. Give each one a different <b>ITD</b> (tiny L/R delay) and/or <b>ILD</b> (L/R level difference).
    This simulates "coming from different directions", which can make overlapping speech easier to follow.
    <div class="small warn">Tip: Try ITD values like -400µs, 0µs, +400µs, and modest ILD like ±2–6 dB.</div>
  </div>

  <div class="controls">
    <button id="play">Play</button>
    <button id="stop">Stop</button>
    <button id="arrangeSpatial">Arrange Spatial</button>
    <label style="display:flex; align-items:center; gap:8px;">
      Master
      <input id="master" type="range" min="0" max="1" step="0.01" value="0.9" />
      <span id="masterVal" class="val">0.90</span>
    </label>
    <label style="display:flex; align-items:center; gap:8px;">
      Loop
      <input id="loop" type="checkbox" checked />
    </label>
  </div>

  <div id="tracks"></div>

  <script>
    // ---- Web Audio helpers ----
    const clamp = (x, a, b) => Math.min(b, Math.max(a, x));
    const dbToGain = (db) => Math.pow(10, db / 20);

    let audioCtx = null;
    let masterGain = null;

    // Track state
    const tracks = []; // {fileName, buffer, settings, nodes, uiRefs}
    const trackUIElements = []; // Store UI refs for each track

    function ensureAudio() {
      if (!audioCtx) {
        audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        masterGain = audioCtx.createGain();
        masterGain.gain.value = parseFloat(document.getElementById("master").value);
        masterGain.connect(audioCtx.destination);
      }
      return audioCtx;
    }

    async function decodeFile(file) {
      const ctx = ensureAudio();
      const arr = await file.arrayBuffer();
      return await ctx.decodeAudioData(arr);
    }

    function createTrackNodes(track) {
      const ctx = ensureAudio();

      // Source
      const src = ctx.createBufferSource();
      src.buffer = track.buffer;
      src.loop = document.getElementById("loop").checked;

      // Per-track gain
      const trackGain = ctx.createGain();
      trackGain.gain.value = track.settings.volume;

      // Optional stereo panner (extra - doesn't replace ITD/ILD)
      const panner = ctx.createStereoPanner();
      panner.pan.value = track.settings.pan;

      // Build stereo from (possibly mono) source:
      // src -> trackGain -> (split to L/R paths with delay + gain) -> merge -> panner -> master
      // We'll treat the source as mono-ish by duplicating its output to both ears.
      // If the buffer is stereo already, this still works, but you're effectively mixing channels together.
      // For best testing, use mono voice recordings.

      const input = trackGain;

      // Two parallel paths fed from the same input
      const leftDelay = ctx.createDelay(0.02);  // max 20ms
      const rightDelay = ctx.createDelay(0.02);
      const leftGain = ctx.createGain();
      const rightGain = ctx.createGain();

      // ITD: positive means sound arrives later to LEFT (i.e., comes from right side)
      // We'll implement by delaying one ear only (the farther ear).
      const itdSec = track.settings.itdMicros / 1e6;
      const dL = itdSec > 0 ? clamp(itdSec, 0, 0.02) : 0;
      const dR = itdSec < 0 ? clamp(-itdSec, 0, 0.02) : 0;
      leftDelay.delayTime.value = dL;
      rightDelay.delayTime.value = dR;

      // ILD in dB: positive means LEFT louder
      // We'll split ILD equally: +X dB left, -X dB right (relative)
      const ild = track.settings.ildDb;
      leftGain.gain.value = dbToGain(+ild / 2);
      rightGain.gain.value = dbToGain(-ild / 2);

      const merger = ctx.createChannelMerger(2);

      // Wiring
      src.connect(trackGain);

      input.connect(leftDelay);
      input.connect(rightDelay);

      leftDelay.connect(leftGain);
      rightDelay.connect(rightGain);

      leftGain.connect(merger, 0, 0);  // to Left channel
      rightGain.connect(merger, 0, 1); // to Right channel

      merger.connect(panner);
      panner.connect(masterGain);

      track.nodes = { src, trackGain, leftDelay, rightDelay, leftGain, rightGain, merger, panner };
    }

    function stopAll() {
      for (const t of tracks) {
        if (t.nodes?.src) {
          try { t.nodes.src.stop(); } catch {}
        }
        t.nodes = null;
      }
    }

    function playAll() {
      ensureAudio();
      stopAll();

      const when = audioCtx.currentTime + 0.02; // small scheduling pad
      for (const t of tracks) {
        createTrackNodes(t);
        t.nodes.src.start(when);
      }
    }

    function updateTrackAudioParams(track) {
      // If not playing, just update settings; params will apply on next Play.
      if (!track.nodes) return;

      const ctx = ensureAudio();

      // Track gain
      track.nodes.trackGain.gain.setValueAtTime(track.settings.volume, ctx.currentTime);

      // Pan
      track.nodes.panner.pan.setValueAtTime(track.settings.pan, ctx.currentTime);

      // ITD: delay one side only
      const itdSec = track.settings.itdMicros / 1e6;
      const dL = itdSec > 0 ? clamp(itdSec, 0, 0.02) : 0;
      const dR = itdSec < 0 ? clamp(-itdSec, 0, 0.02) : 0;
      track.nodes.leftDelay.delayTime.setValueAtTime(dL, ctx.currentTime);
      track.nodes.rightDelay.delayTime.setValueAtTime(dR, ctx.currentTime);

      // ILD
      const ild = track.settings.ildDb;
      track.nodes.leftGain.gain.setValueAtTime(dbToGain(+ild / 2), ctx.currentTime);
      track.nodes.rightGain.gain.setValueAtTime(dbToGain(-ild / 2), ctx.currentTime);
    }

    // ---- UI building ----
    function mkSlider({min, max, step, value, onInput}) {
      const s = document.createElement("input");
      s.type = "range";
      s.min = min; s.max = max; s.step = step; s.value = value;
      s.addEventListener("input", () => onInput(parseFloat(s.value)));
      return s;
    }

    function addTrackUI(track) {
      const container = document.getElementById("tracks");
      const el = document.createElement("div");
      el.className = "track";

      const title = document.createElement("div");
      title.className = "titleRow";

      const name = document.createElement("div");
      name.className = "fileName";
      name.textContent = track.fileName;

      const meta = document.createElement("div");
      meta.className = "small";
      meta.textContent = `${Math.round(track.buffer.duration * 100) / 100}s • ${track.buffer.numberOfChannels}ch • ${track.buffer.sampleRate}Hz`;

      title.appendChild(name);
      title.appendChild(meta);
      el.appendChild(title);

      // Volume
      const volVal = document.createElement("div"); volVal.className = "val";
      const volSlider = mkSlider({
        min: 0, max: 1, step: 0.01, value: track.settings.volume,
        onInput: (v) => {
          track.settings.volume = v;
          volVal.textContent = v.toFixed(2);
          updateTrackAudioParams(track);
        }
      });
      volVal.textContent = track.settings.volume.toFixed(2);
      el.appendChild(row("Volume", volSlider, volVal));

      // ITD (µs)
      const itdVal = document.createElement("div"); itdVal.className = "val";
      const itdSlider = mkSlider({
        min: -800, max: 800, step: 10, value: track.settings.itdMicros,
        onInput: (v) => {
          track.settings.itdMicros = v;
          itdVal.textContent = `${Math.round(v)} µs`;
          updateTrackAudioParams(track);
        }
      });
      itdVal.textContent = `${Math.round(track.settings.itdMicros)} µs`;
      el.appendChild(row("ITD (L/R delay)", itdSlider, itdVal));

      // ILD (dB)
      const ildVal = document.createElement("div"); ildVal.className = "val";
      const ildSlider = mkSlider({
        min: -18, max: 18, step: 0.5, value: track.settings.ildDb,
        onInput: (v) => {
          track.settings.ildDb = v;
          ildVal.textContent = `${v.toFixed(1)} dB`;
          updateTrackAudioParams(track);
        }
      });
      ildVal.textContent = `${track.settings.ildDb.toFixed(1)} dB`;
      el.appendChild(row("ILD (L louder +)", ildSlider, ildVal));

      // Pan (-1..+1)
      const panVal = document.createElement("div"); panVal.className = "val";
      const panSlider = mkSlider({
        min: -1, max: 1, step: 0.01, value: track.settings.pan,
        onInput: (v) => {
          track.settings.pan = v;
          panVal.textContent = v.toFixed(2);
          updateTrackAudioParams(track);
        }
      });
      panVal.textContent = track.settings.pan.toFixed(2);
      el.appendChild(row("Pan (extra)", panSlider, panVal));

      // Quick presets
      const presetRow = document.createElement("div");
      presetRow.className = "controls";
      presetRow.style.margin = "10px 0 0";

      const btn = (label, fn) => {
        const b = document.createElement("button");
        b.textContent = label;
        b.addEventListener("click", fn);
        return b;
      };

      presetRow.appendChild(btn("Center", () => {
        track.settings.itdMicros = 0;
        track.settings.ildDb = 0;
        track.settings.pan = 0;
        // Update sliders + labels
        volSlider.value = track.settings.volume;
        itdSlider.value = 0; ildSlider.value = 0; panSlider.value = 0;
        itdVal.textContent = "0 µs"; ildVal.textContent = "0.0 dB"; panVal.textContent = "0.00";
        updateTrackAudioParams(track);
      }));

      presetRow.appendChild(btn("Hard Left-ish", () => {
        track.settings.itdMicros = -500;
        track.settings.ildDb = +6;
        track.settings.pan = -0.4;
        itdSlider.value = -500; ildSlider.value = 6; panSlider.value = -0.4;
        itdVal.textContent = "-500 µs"; ildVal.textContent = "6.0 dB"; panVal.textContent = "-0.40";
        updateTrackAudioParams(track);
      }));

      presetRow.appendChild(btn("Hard Right-ish", () => {
        track.settings.itdMicros = +500;
        track.settings.ildDb = -6;
        track.settings.pan = +0.4;
        itdSlider.value = 500; ildSlider.value = -6; panSlider.value = 0.4;
        itdVal.textContent = "500 µs"; ildVal.textContent = "-6.0 dB"; panVal.textContent = "0.40";
        updateTrackAudioParams(track);
      }));

      el.appendChild(presetRow);

      container.appendChild(el);

      // Store UI references for later updates
      trackUIElements.push({
        track,
        volSlider,
        itdSlider,
        ildSlider,
        panSlider,
        volVal,
        itdVal,
        ildVal,
        panVal
      });
    }

    function row(labelText, controlEl, valEl) {
      const r = document.createElement("div");
      r.className = "row";
      const l = document.createElement("label");
      l.textContent = labelText;
      r.appendChild(l);
      r.appendChild(controlEl);
      r.appendChild(valEl);
      return r;
    }

    // ---- Wire up top-level controls ----
    document.getElementById("master").addEventListener("input", (e) => {
      const v = parseFloat(e.target.value);
      document.getElementById("masterVal").textContent = v.toFixed(2);
      if (masterGain) masterGain.gain.value = v;
    });

    document.getElementById("play").addEventListener("click", async () => {
      ensureAudio();
      if (audioCtx.state === "suspended") {
        await audioCtx.resume();
      }
      if (tracks.length === 0) {
        // If tracks haven't loaded yet, load them now
        await loadAudioFiles();
      }
      playAll();
    });

    document.getElementById("stop").addEventListener("click", () => {
      stopAll();
    });

    document.getElementById("loop").addEventListener("change", () => {
      // takes effect next time you press Play (or you can re-play)
    });

    // Arrange spatial button - toggles between spatial arrangement and center
    const arrangeBtn = document.getElementById("arrangeSpatial");
    arrangeBtn.addEventListener("click", () => {
      if (tracks.length < 3) return;

      const isCurrentlySpatial = arrangeBtn.textContent === "Arrange Spatial";

      if (isCurrentlySpatial) {
        // Apply spatial arrangement: Center, Hard Left, Hard Right
        const presets = [
          { itdMicros: 0, ildDb: 0, pan: 0 },      // Center
          { itdMicros: -500, ildDb: +6, pan: -0.4 }, // Hard Left
          { itdMicros: +500, ildDb: -6, pan: +0.4 }  // Hard Right
        ];

        for (let i = 0; i < Math.min(tracks.length, 3); i++) {
          const preset = presets[i];
          const track = tracks[i];
          const ui = trackUIElements[i];

          if (!track || !ui) continue;

          track.settings.itdMicros = preset.itdMicros;
          track.settings.ildDb = preset.ildDb;
          track.settings.pan = preset.pan;

          // Update sliders
          ui.itdSlider.value = preset.itdMicros;
          ui.ildSlider.value = preset.ildDb;
          ui.panSlider.value = preset.pan;

          // Update labels
          ui.itdVal.textContent = `${Math.round(preset.itdMicros)} µs`;
          ui.ildVal.textContent = `${preset.ildDb.toFixed(1)} dB`;
          ui.panVal.textContent = preset.pan.toFixed(2);

          // Update audio if playing
          updateTrackAudioParams(track);
        }

        arrangeBtn.textContent = "Arrange Center";
      } else {
        // Reset all to center
        const centerPreset = { itdMicros: 0, ildDb: 0, pan: 0 };

        for (let i = 0; i < tracks.length; i++) {
          const track = tracks[i];
          const ui = trackUIElements[i];

          if (!track || !ui) continue;

          track.settings.itdMicros = centerPreset.itdMicros;
          track.settings.ildDb = centerPreset.ildDb;
          track.settings.pan = centerPreset.pan;

          // Update sliders
          ui.itdSlider.value = centerPreset.itdMicros;
          ui.ildSlider.value = centerPreset.ildDb;
          ui.panSlider.value = centerPreset.pan;

          // Update labels
          ui.itdVal.textContent = `${Math.round(centerPreset.itdMicros)} µs`;
          ui.ildVal.textContent = `${centerPreset.ildDb.toFixed(1)} dB`;
          ui.panVal.textContent = centerPreset.pan.toFixed(2);

          // Update audio if playing
          updateTrackAudioParams(track);
        }

        arrangeBtn.textContent = "Arrange Spatial";
      }
    });

    // Auto-load the three audio files
    async function loadAudioFiles() {
      // Clear existing
      stopAll();
      tracks.length = 0;
      trackUIElements.length = 0;
      document.getElementById("tracks").innerHTML = "";

      // Load the three fixed audio files
      const audioFiles = [
        '/audio/000000.wav',
        '/audio/000001.wav',
        '/audio/000002.wav'
      ];

      // Fetch all files first (doesn't require audio context)
      const fetchPromises = audioFiles.map(async (url) => {
        try {
          const response = await fetch(url);
          if (!response.ok) {
            console.warn(`Failed to load ${url}: ${response.statusText}`);
            return null;
          }
          return { url, arrayBuffer: await response.arrayBuffer() };
        } catch (error) {
          console.error(`Error fetching ${url}:`, error);
          return null;
        }
      });

      const fetchedFiles = await Promise.all(fetchPromises);

      // Now decode them (this requires audio context, but we'll create it if needed)
      for (const fileData of fetchedFiles) {
        if (!fileData) continue;

        try {
          // Create audio context if needed (but don't require user interaction)
          const ctx = ensureAudio();

          // Try to resume if suspended, but don't wait for it
          if (ctx.state === "suspended") {
            ctx.resume().catch(() => {
              // Will resume when user interacts
            });
          }

          const buffer = await ctx.decodeAudioData(fileData.arrayBuffer);
          const fileName = fileData.url.split('/').pop();
          const track = {
            fileName: fileName,
            buffer,
            settings: {
              volume: 0.9,
              itdMicros: 0, // -800..800 typical human range ~ +/- 700us
              ildDb: 0,     // -18..18
              pan: 0
            },
            nodes: null
          };
          tracks.push(track);
          addTrackUI(track);
        } catch (error) {
          console.error(`Error decoding ${fileData.url}:`, error);
        }
      }
    }

    // Load files immediately when page loads
    loadAudioFiles();
  </script>
</div>

</div>

Try adjusting the ITD and ILD values for each track to position them in different parts of the stereo field. The preset buttons provide quick starting points for left, center, and right positioning.

**Audio Credit:** Includes excerpts from [*Passive_Houses_Presentation*](https://archive.org/details/Passive_Houses_Presentation) by MMCTV, licensed under [CC BY 3.0](https://creativecommons.org/licenses/by/3.0/).
