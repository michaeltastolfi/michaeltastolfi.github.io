<!DOCTYPE html>
<!--
  MTG Card Price Scanner — Meta Ray-Ban Display Web App + Phone Companion
  ────────────────────────────────────────────────────────────────────────
  Two surfaces, one file, hosted at one URL:
    https://yourhost/mtg-scanner.html#phone     ← run on the paired phone
    https://yourhost/mtg-scanner.html#glasses   ← installed as the MRBD Web App

  Why two surfaces?
    Meta's Web Apps API for MRBD does NOT currently expose Camera, Microphone,
    or text input (see https://wearables.developer.meta.com/docs/develop/webapps/build/).
    So the camera + OCR + Scryfall lookup happens on the phone, and the glasses
    show a 600×600 dark/additive HUD with the latest recognized card + its
    TCGplayer Market price.

  Pricing source:
    Scryfall ships TCGplayer Market USD in its `prices.usd` field (and foil in
    `prices.usd_foil`). One API call gets you both the card metadata AND the
    TCGplayer Market price — no TCGplayer partner key required.

  Sync between the two surfaces:
    localStorage + the cross-tab `storage` event. Works perfectly if both
    surfaces are tabs in the same browser. For real cross-device sync (phone
    in one browser, glasses in another), swap pushScan() and the poll loop
    for a tiny WebSocket/SSE/Firebase relay — the shape of the data stays
    identical.
-->
<html lang="en">
<head>
<meta charset="UTF-8">
<title>MTG Scanner · Meta Ray-Ban Display</title>
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<link rel="icon" href="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'%3E%3Crect width='100' height='100' rx='14' fill='%23000'/%3E%3Ctext x='50' y='68' font-family='serif' font-size='62' font-weight='900' text-anchor='middle' fill='%23d4a83a'%3E$%3C/text%3E%3C/svg%3E">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Cinzel:wght@500;700;900&family=IBM+Plex+Sans:wght@400;500;600;700&family=IBM+Plex+Mono:wght@400;500&display=swap" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/tesseract.js/5.1.0/tesseract.min.js"></script>
<style>
  :root {
    /* Phone palette — parchment + ink, gold accent (MTG card visual lineage) */
    --paper:   #f5ecd7;
    --paper-d: #e6dcc4;
    --ink:     #14110d;
    --ink-2:   #2b251c;
    --gold:    #d4a83a;
    --gold-d:  #b08a23;
    --rust:    #8b3a2b;
    --moss:    #4a6b3a;

    /* MTG color identity pips */
    --w: #f8e7b9; --u: #6ec1ed; --b: #b1a8a4; --r: #ec6e57; --g: #6dbf7c;

    /* Glasses palette — additive waveguide HUD. Pure black = transparent IRL.
       Bright high-saturation colors are what survives the see-through display. */
    --hud-bg:     #000;
    --hud-cyan:   #00d4ff;
    --hud-text:   #e8f4ff;
    --hud-cream:  #f3e3b3;
    --hud-good:   #7cfc00;
    --hud-warn:   #ffaf3d;
    --hud-bad:    #ff5470;
    --hud-dim:    #4a6070;
    --hud-rule:   rgba(0,212,255,0.25);
  }
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
  html, body { width: 100%; min-height: 100vh; }
  body {
    font-family: 'IBM Plex Sans', system-ui, sans-serif;
    background: #0e0a07;
    color: var(--paper);
    -webkit-font-smoothing: antialiased;
    overflow-x: hidden;
  }
  button { font-family: inherit; cursor: pointer; }

  /* ═══════════════════════════════════════════════════════════════════
     MODE PICKER
     ═══════════════════════════════════════════════════════════════════ */
  #picker {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 40px 24px;
    background:
      radial-gradient(ellipse at 20% 10%, rgba(212,168,58,0.10), transparent 55%),
      radial-gradient(ellipse at 85% 90%, rgba(110,193,237,0.06), transparent 55%),
      linear-gradient(180deg, #120c08 0%, #0a0705 100%);
    position: relative;
  }
  #picker::before {
    content: '';
    position: absolute; inset: 0;
    background-image:
      repeating-linear-gradient(45deg, rgba(212,168,58,0.025) 0 1px, transparent 1px 8px);
    pointer-events: none;
  }
  .picker-mark {
    font-family: 'Cinzel', serif;
    font-weight: 900;
    font-size: clamp(36px, 8vw, 72px);
    letter-spacing: 0.08em;
    color: var(--paper);
    line-height: 0.95;
    text-align: center;
    margin-bottom: 8px;
    text-shadow: 0 2px 0 rgba(0,0,0,0.4);
  }
  .picker-mark .amp { color: var(--gold); font-style: italic; }
  .picker-sub {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 11px;
    letter-spacing: 0.3em;
    text-transform: uppercase;
    color: var(--gold-d);
    margin-bottom: 48px;
    text-align: center;
  }
  .picker-cards {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
    gap: 22px;
    max-width: 760px;
    width: 100%;
    margin-bottom: 32px;
  }
  .pcard {
    background: linear-gradient(180deg, #1d160e 0%, #15100a 100%);
    border: 1px solid rgba(212,168,58,0.35);
    padding: 24px;
    text-align: left;
    color: var(--paper);
    transition: transform 180ms ease, border-color 180ms ease, box-shadow 220ms ease;
    position: relative;
    overflow: hidden;
  }
  .pcard:hover {
    transform: translateY(-3px);
    border-color: var(--gold);
    box-shadow: 0 12px 32px rgba(0,0,0,0.5), 0 0 0 1px rgba(212,168,58,0.3);
  }
  .pcard-icon {
    font-family: 'Cinzel', serif;
    font-size: 44px;
    line-height: 1;
    color: var(--gold);
    margin-bottom: 14px;
  }
  .pcard h3 {
    font-family: 'Cinzel', serif;
    font-weight: 700;
    font-size: 22px;
    letter-spacing: 0.04em;
    margin-bottom: 8px;
    color: var(--paper);
  }
  .pcard p {
    font-size: 13px;
    line-height: 1.55;
    color: var(--paper-d);
    opacity: 0.78;
    margin-bottom: 16px;
  }
  .pcard .tag {
    display: inline-block;
    font-family: 'IBM Plex Mono', monospace;
    font-size: 10px;
    letter-spacing: 0.2em;
    text-transform: uppercase;
    padding: 4px 10px;
    border: 1px solid rgba(212,168,58,0.4);
    color: var(--gold);
  }
  .picker-note {
    max-width: 640px;
    text-align: center;
    color: rgba(245,236,215,0.55);
    font-size: 12px;
    line-height: 1.7;
    margin-top: 12px;
  }
  .picker-note code {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 11px;
    background: rgba(212,168,58,0.10);
    padding: 2px 6px;
    color: var(--gold);
  }

  /* ═══════════════════════════════════════════════════════════════════
     PHONE SURFACE
     ═══════════════════════════════════════════════════════════════════ */
  #phone {
    min-height: 100vh;
    max-width: 480px;
    margin: 0 auto;
    display: flex;
    flex-direction: column;
    background: linear-gradient(180deg, #14100b 0%, #0a0805 100%);
  }
  .ph-head {
    padding: 18px 20px 14px;
    border-bottom: 1px solid rgba(212,168,58,0.2);
    display: flex;
    align-items: baseline;
    justify-content: space-between;
  }
  .ph-head h1 {
    font-family: 'Cinzel', serif;
    font-weight: 700;
    font-size: 20px;
    letter-spacing: 0.08em;
    color: var(--paper);
  }
  .ph-head .role {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 10px;
    letter-spacing: 0.25em;
    color: var(--gold);
    text-transform: uppercase;
  }

  .scanner {
    position: relative;
    aspect-ratio: 3/4;
    background: #000;
    overflow: hidden;
    margin: 16px;
    border: 1px solid rgba(212,168,58,0.25);
  }
  #video {
    width: 100%; height: 100%;
    object-fit: cover;
    display: block;
    background: #000;
  }
  .scan-frame {
    position: absolute; inset: 8%;
    border: 2px solid rgba(212,168,58,0.7);
    pointer-events: none;
    box-shadow: 0 0 0 9999px rgba(0,0,0,0.35);
  }
  .scan-frame::before, .scan-frame::after,
  .scan-frame > span:nth-child(1), .scan-frame > span:nth-child(2) {
    content: '';
    position: absolute;
    width: 22px; height: 22px;
    border: 3px solid var(--gold);
  }
  .scan-frame::before { top: -3px; left: -3px; border-right: none; border-bottom: none; }
  .scan-frame::after  { top: -3px; right: -3px; border-left: none; border-bottom: none; }
  .scan-frame > span:nth-child(1) { bottom: -3px; left: -3px; border-right: none; border-top: none; }
  .scan-frame > span:nth-child(2) { bottom: -3px; right: -3px; border-left: none; border-top: none; }

  .title-band {
    position: absolute;
    left: 8%; right: 8%; top: 8%;
    height: 13%;
    background: rgba(212,168,58,0.12);
    border-bottom: 1px dashed rgba(212,168,58,0.5);
    pointer-events: none;
  }
  .title-band-label {
    position: absolute;
    left: 8%; top: calc(8% + 13%);
    font-family: 'IBM Plex Mono', monospace;
    font-size: 9px;
    letter-spacing: 0.2em;
    color: var(--gold);
    background: rgba(0,0,0,0.65);
    padding: 2px 6px;
    text-transform: uppercase;
    pointer-events: none;
  }

  .controls {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 10px;
    padding: 0 16px 8px;
  }
  .btn {
    border: 1px solid var(--gold);
    background: var(--gold);
    color: var(--ink);
    font-family: 'IBM Plex Sans', sans-serif;
    font-weight: 700;
    font-size: 14px;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    padding: 14px 16px;
    transition: all 160ms ease;
  }
  .btn:hover { background: #e8c155; }
  .btn:disabled { opacity: 0.4; cursor: not-allowed; }
  .btn.ghost {
    background: transparent;
    color: var(--gold);
  }
  .btn.ghost:hover { background: rgba(212,168,58,0.12); }

  .status {
    margin: 4px 16px 0;
    padding: 12px 14px;
    border-left: 3px solid var(--gold);
    background: rgba(212,168,58,0.06);
    font-family: 'IBM Plex Mono', monospace;
    font-size: 12px;
    color: var(--paper-d);
    min-height: 44px;
    line-height: 1.6;
  }
  .status .label {
    color: var(--gold);
    letter-spacing: 0.15em;
    text-transform: uppercase;
    font-size: 10px;
    display: block;
    margin-bottom: 2px;
  }
  .status.error { border-color: var(--rust); background: rgba(139,58,43,0.12); }
  .status.error .label { color: var(--rust); }
  .status.ok { border-color: var(--moss); background: rgba(74,107,58,0.10); }
  .status.ok .label { color: var(--moss); }

  .manual {
    margin: 14px 16px 4px;
    padding: 14px;
    border: 1px dashed rgba(212,168,58,0.35);
  }
  .manual h4 {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 10px;
    letter-spacing: 0.22em;
    color: var(--gold-d);
    text-transform: uppercase;
    margin-bottom: 10px;
  }
  .manual-row { display: flex; gap: 8px; }
  .manual input {
    flex: 1;
    background: rgba(0,0,0,0.4);
    border: 1px solid rgba(212,168,58,0.35);
    color: var(--paper);
    font-family: 'IBM Plex Sans', sans-serif;
    font-size: 14px;
    padding: 10px 12px;
    outline: none;
  }
  .manual input:focus { border-color: var(--gold); }
  .manual .btn { padding: 10px 14px; font-size: 12px; }

  .result {
    margin: 14px 16px;
    padding: 14px;
    background: rgba(245,236,215,0.04);
    border: 1px solid rgba(212,168,58,0.25);
    display: none;
    gap: 14px;
  }
  .result.show { display: flex; }
  .result img {
    width: 88px; height: auto;
    border-radius: 4px;
    flex-shrink: 0;
  }
  .result .meta { flex: 1; min-width: 0; }
  .result .rname {
    font-family: 'Cinzel', serif;
    font-weight: 700;
    font-size: 16px;
    line-height: 1.2;
    color: var(--paper);
    margin-bottom: 4px;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .result .rset {
    font-size: 12px;
    color: var(--paper-d);
    opacity: 0.7;
    margin-bottom: 8px;
  }
  .result .rprice {
    display: flex;
    gap: 12px;
    align-items: baseline;
    flex-wrap: wrap;
  }
  .rprice .pmain {
    font-family: 'IBM Plex Mono', monospace;
    font-weight: 500;
    font-size: 22px;
    color: var(--moss);
  }
  .rprice .pmain.none { color: rgba(245,236,215,0.4); }
  .rprice .plbl {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 9px;
    letter-spacing: 0.2em;
    color: var(--gold);
    text-transform: uppercase;
  }
  .rprice .pfoil {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 12px;
    color: var(--paper-d);
  }

  .ph-foot {
    padding: 20px 16px 28px;
    text-align: center;
    font-family: 'IBM Plex Mono', monospace;
    font-size: 10px;
    letter-spacing: 0.18em;
    color: rgba(245,236,215,0.4);
    text-transform: uppercase;
  }
  .ph-foot a { color: var(--gold); text-decoration: none; border-bottom: 1px solid currentColor; }

  /* ═══════════════════════════════════════════════════════════════════
     GLASSES SURFACE — 600×600 ADDITIVE WAVEGUIDE HUD
     Spec from https://wearables.developer.meta.com/docs/develop/webapps/build/
     ═══════════════════════════════════════════════════════════════════ */
  #glasses {
    min-height: 100vh;
    background: #0a0a0a;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 24px;
    gap: 14px;
  }
  .preview-tag {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 10px;
    letter-spacing: 0.3em;
    color: var(--hud-dim);
    text-transform: uppercase;
  }
  .hud {
    width: 600px; height: 600px;
    max-width: 96vw;
    max-height: 96vw;
    background: var(--hud-bg);
    color: var(--hud-text);
    position: relative;
    overflow: hidden;
    font-family: 'IBM Plex Sans', sans-serif;
  }
  /* When loaded directly on the glasses, the body IS the viewport */
  body.glasses-only { background: #000; }
  body.glasses-only #glasses { padding: 0; gap: 0; min-height: 100vh; justify-content: flex-start; }
  body.glasses-only .preview-tag, body.glasses-only .hud-foot { display: none; }
  body.glasses-only .hud { width: 600px; height: 600px; }

  .hud-top {
    padding: 18px 24px 12px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    border-bottom: 1px solid var(--hud-rule);
  }
  .hud-title {
    font-family: 'Cinzel', serif;
    font-weight: 700;
    font-size: 18px;
    letter-spacing: 0.16em;
    color: var(--hud-cream);
  }
  .hud-status {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 11px;
    letter-spacing: 0.18em;
    color: var(--hud-cyan);
    text-transform: uppercase;
    display: flex;
    align-items: center;
    gap: 8px;
  }
  .hud-dot {
    width: 8px; height: 8px;
    background: var(--hud-good);
    border-radius: 50%;
    box-shadow: 0 0 8px var(--hud-good);
    animation: pulse 1.6s ease-in-out infinite;
  }
  .hud-dot.idle { background: var(--hud-dim); box-shadow: none; animation: none; }
  @keyframes pulse {
    0%, 100% { opacity: 1; }
    50% { opacity: 0.4; }
  }

  .hud-feature {
    padding: 18px 24px;
    display: grid;
    grid-template-columns: 168px 1fr;
    gap: 18px;
    align-items: start;
    border-bottom: 1px solid var(--hud-rule);
    min-height: 260px;
  }
  .feat-img {
    width: 168px; height: 234px;
    background: #0a0a0a;
    border: 1px solid var(--hud-rule);
    position: relative;
    overflow: hidden;
  }
  .feat-img img {
    width: 100%; height: 100%;
    object-fit: cover;
    display: block;
    /* Reduce contribution to additive display by lowering brightness slightly
       — pure full-brightness card art would render very intensely on glass */
    filter: brightness(0.92) saturate(1.05);
  }
  .feat-img.empty::after {
    content: '◇';
    position: absolute; inset: 0;
    display: flex; align-items: center; justify-content: center;
    color: var(--hud-dim);
    font-size: 48px;
  }
  /* The price tag overlaid on the card image */
  .feat-img .price-overlay {
    position: absolute;
    left: 50%; bottom: 10px;
    transform: translateX(-50%);
    background: rgba(0,0,0,0.78);
    border: 1.5px solid var(--hud-good);
    color: var(--hud-good);
    font-family: 'IBM Plex Mono', monospace;
    font-weight: 500;
    font-size: 20px;
    padding: 5px 12px;
    letter-spacing: 0.04em;
    white-space: nowrap;
    box-shadow: 0 0 12px rgba(124,252,0,0.35);
  }
  .feat-img .price-overlay.none {
    border-color: var(--hud-dim);
    color: var(--hud-dim);
    box-shadow: none;
  }
  .feat-meta {
    min-width: 0;
    padding-top: 4px;
  }
  .feat-name {
    font-family: 'Cinzel', serif;
    font-weight: 700;
    font-size: 26px;
    line-height: 1.1;
    color: var(--hud-cream);
    margin-bottom: 4px;
    overflow-wrap: break-word;
  }
  .feat-type {
    font-size: 13px;
    color: var(--hud-text);
    opacity: 0.7;
    margin-bottom: 10px;
    line-height: 1.3;
  }
  .feat-set {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 11px;
    letter-spacing: 0.16em;
    text-transform: uppercase;
    color: var(--hud-cyan);
    margin-bottom: 14px;
  }
  .feat-price-block {
    display: flex;
    flex-direction: column;
    gap: 6px;
    margin-top: 4px;
  }
  .feat-price-row {
    display: flex;
    align-items: baseline;
    gap: 10px;
  }
  .feat-price-lbl {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 9px;
    letter-spacing: 0.22em;
    text-transform: uppercase;
    color: var(--hud-dim);
    width: 92px;
    flex-shrink: 0;
  }
  .feat-price-val {
    font-family: 'IBM Plex Mono', monospace;
    font-weight: 500;
    font-size: 18px;
    color: var(--hud-good);
  }
  .feat-price-val.foil {
    font-size: 14px;
    color: var(--hud-warn);
  }
  .feat-price-val.none {
    color: var(--hud-dim);
    font-size: 14px;
  }

  .hud-list {
    padding: 12px 16px;
    display: flex;
    flex-direction: column;
    gap: 4px;
  }
  .hud-list-lbl {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 9px;
    letter-spacing: 0.22em;
    color: var(--hud-dim);
    text-transform: uppercase;
    padding: 0 8px 4px;
  }
  .hud-item {
    display: grid;
    grid-template-columns: 1fr auto;
    gap: 12px;
    align-items: center;
    padding: 7px 10px;
    border: 1.5px solid transparent;
    background: transparent;
    color: var(--hud-text);
    font-size: 13px;
    text-align: left;
    width: 100%;
  }
  .hud-item .hi-name {
    font-family: 'IBM Plex Sans', sans-serif;
    font-weight: 500;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .hud-item .hi-price {
    font-family: 'IBM Plex Mono', monospace;
    color: var(--hud-good);
    font-size: 13px;
  }
  .hud-item .hi-price.none { color: var(--hud-dim); }
  .hud-item.focusable:focus {
    outline: none;
    border-color: var(--hud-cyan);
    background: rgba(0,212,255,0.08);
    box-shadow: 0 0 12px rgba(0,212,255,0.25);
  }
  .hud-empty {
    padding: 32px 24px;
    text-align: center;
    color: var(--hud-dim);
    font-size: 14px;
    line-height: 1.6;
  }
  .hud-empty .hud-empty-mono {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 11px;
    letter-spacing: 0.18em;
    margin-top: 8px;
    color: var(--hud-cyan);
  }

  .hud-foot {
    font-family: 'IBM Plex Mono', monospace;
    font-size: 10px;
    letter-spacing: 0.22em;
    color: var(--hud-dim);
    text-transform: uppercase;
    margin-top: 6px;
  }
  .hud-foot kbd {
    background: #1a1a1a;
    border: 1px solid #333;
    padding: 2px 6px;
    color: var(--hud-cyan);
    font-family: inherit;
    font-size: 9px;
  }

  /* small screen tweaks */
  @media (max-width: 520px) {
    .hud-feature { grid-template-columns: 130px 1fr; min-height: 200px; }
    .feat-img { width: 130px; height: 182px; }
    .feat-name { font-size: 22px; }
  }
</style>
</head>
<body>

<!-- ═════════════════ MODE PICKER ═════════════════ -->
<div id="picker">
  <div class="picker-mark">SCRY <span class="amp">&amp;</span> PRICE</div>
  <div class="picker-sub">MTG Card Scanner · Meta Ray-Ban Display</div>

  <div class="picker-cards">
    <button class="pcard" onclick="goto('phone')">
      <div class="pcard-icon">◉</div>
      <h3>Phone Scanner</h3>
      <p>Live camera with OCR recognition. Reads the card title, looks it up on Scryfall, fetches TCGplayer Market price.</p>
      <span class="tag">Open on phone</span>
    </button>
    <button class="pcard" onclick="goto('glasses')">
      <div class="pcard-icon">◇</div>
      <h3>Glasses HUD</h3>
      <p>600 × 600 additive display per MRBD spec. Arrow-key navigation, dark background, bright high-contrast UI.</p>
      <span class="tag">Open on glasses</span>
    </button>
  </div>

  <p class="picker-note">
    Open both views — the phone scans, the glasses display. They sync via shared storage.
    For full cross-device testing, host this file and follow Meta's MRBD test docs to install the glasses URL as a Web App.
    Add <code>?demo</code> to either URL to preload sample scans.
  </p>
</div>

<!-- ═════════════════ PHONE SURFACE ═════════════════ -->
<div id="phone" hidden>
  <div class="ph-head">
    <h1>SCRY &amp; PRICE</h1>
    <span class="role">Scanner · Phone</span>
  </div>

  <div class="scanner">
    <video id="video" autoplay playsinline muted></video>
    <div class="title-band"></div>
    <div class="title-band-label">Title region — OCR target</div>
    <div class="scan-frame"><span></span><span></span></div>
  </div>

  <div class="controls">
    <button class="btn" id="startBtn">Start camera</button>
    <button class="btn ghost" id="captureBtn" disabled>Capture &amp; identify</button>
  </div>

  <div class="status" id="status">
    <span class="label">Ready</span>
    Tap <em>Start camera</em>, line up the card so its title fills the dashed band, then tap <em>Capture</em>.
  </div>

  <div class="manual">
    <h4>Fallback · type the name</h4>
    <div class="manual-row">
      <input id="manualInput" type="text" placeholder="e.g. Black Lotus, Sol Ring, Ragavan…" autocomplete="off" autocorrect="off" spellcheck="false">
      <button class="btn" id="manualBtn">Look up</button>
    </div>
  </div>

  <div class="result" id="result">
    <img id="resImg" alt="">
    <div class="meta">
      <div class="rname" id="resName">—</div>
      <div class="rset" id="resSet">—</div>
      <div class="rprice">
        <span class="plbl">TCG Market</span>
        <span class="pmain" id="resPrice">$—</span>
        <span class="pfoil" id="resFoil"></span>
      </div>
    </div>
  </div>

  <div class="ph-foot">
    <a href="#glasses" onclick="goto('glasses'); return false;">Open glasses view →</a>
  </div>
</div>

<!-- ═════════════════ GLASSES SURFACE (MRBD spec) ═════════════════ -->
<div id="glasses" hidden>
  <div class="preview-tag">Glasses preview · 600 × 600 additive overlay</div>

  <div class="hud" id="hudRoot">
    <div class="hud-top">
      <div class="hud-title">SCRY · PRICE</div>
      <div class="hud-status">
        <span class="hud-dot idle" id="hudDot"></span>
        <span id="hudStat">WAITING</span>
      </div>
    </div>

    <div class="hud-feature" id="featBlock">
      <div class="feat-img empty" id="featImgWrap">
        <!-- img + price-overlay injected dynamically -->
      </div>
      <div class="feat-meta">
        <div class="feat-name" id="featName">No card yet</div>
        <div class="feat-type" id="featType"></div>
        <div class="feat-set" id="featSet"></div>
        <div class="feat-price-block" id="featPriceBlock"></div>
      </div>
    </div>

    <div class="hud-list" id="hudList">
      <div class="hud-list-lbl">Recent scans</div>
      <div class="hud-empty" id="hudEmpty">
        Scan a card on the phone to see it appear here.
        <div class="hud-empty-mono">Open <span style="color: var(--hud-cream);">?demo</span> to preview</div>
      </div>
    </div>
  </div>

  <div class="hud-foot">
    <kbd>↑</kbd> <kbd>↓</kbd> Navigate &nbsp;·&nbsp; <kbd>Enter</kbd> Feature &nbsp;·&nbsp; <kbd>Esc</kbd> Back to picker
  </div>
</div>

<script>
'use strict';

/* ═════════════════════════════════════════════════════════════════════
   STORAGE  —  unified across artifact env (window.storage), self-hosted
   (localStorage), and cross-tab updates via the 'storage' event.
   ═════════════════════════════════════════════════════════════════════ */
const STORAGE_KEY = 'mtg-scans-v1';
const MAX_RECENT = 8;
const hasArtifactStorage = typeof window.storage !== 'undefined'
  && window.storage && typeof window.storage.get === 'function';

async function loadScans() {
  try {
    if (hasArtifactStorage) {
      const r = await window.storage.get(STORAGE_KEY);
      return r ? JSON.parse(r.value) : [];
    }
    const raw = localStorage.getItem(STORAGE_KEY);
    return raw ? JSON.parse(raw) : [];
  } catch (e) {
    console.warn('loadScans failed', e);
    return [];
  }
}

async function saveScans(arr) {
  const payload = JSON.stringify(arr);
  try {
    if (hasArtifactStorage) {
      await window.storage.set(STORAGE_KEY, payload);
    }
    // Always also write to localStorage so the cross-tab 'storage' event fires
    localStorage.setItem(STORAGE_KEY, payload);
  } catch (e) {
    console.warn('saveScans failed', e);
  }
}

async function pushScan(card) {
  const existing = await loadScans();
  // Dedupe by oracle_id (or scryfall id), most-recent-first
  const id = card.oracle_id || card.id;
  const updated = [card, ...existing.filter(c => (c.oracle_id || c.id) !== id)].slice(0, MAX_RECENT);
  await saveScans(updated);
  return updated;
}

/* ═════════════════════════════════════════════════════════════════════
   MODE ROUTING
   ═════════════════════════════════════════════════════════════════════ */
function currentMode() {
  const h = (location.hash || '').replace('#', '').split('?')[0].toLowerCase();
  if (h === 'phone' || h === 'glasses') return h;
  return null;
}
function isDemo() {
  return location.hash.includes('demo') || location.search.includes('demo');
}

function goto(mode) {
  location.hash = mode + (isDemo() ? '?demo' : '');
  render();
}

function render() {
  const mode = currentMode();
  document.getElementById('picker').hidden = !!mode;
  document.getElementById('phone').hidden = mode !== 'phone';
  document.getElementById('glasses').hidden = mode !== 'glasses';

  if (mode === 'glasses') {
    if (isDemo()) seedDemoScans();
    initGlasses();
  } else if (mode === 'phone') {
    if (isDemo()) seedDemoScans();
    initPhone();
  }
}
window.addEventListener('hashchange', render);

/* ═════════════════════════════════════════════════════════════════════
   SCRYFALL  —  fuzzy name lookup. Scryfall ships TCGplayer Market USD
   in `prices.usd` and foil in `prices.usd_foil`.
   ═════════════════════════════════════════════════════════════════════ */
async function scryfallLookup(query) {
  const url = `https://api.scryfall.com/cards/named?fuzzy=${encodeURIComponent(query)}`;
  const res = await fetch(url, { headers: { 'Accept': 'application/json' } });
  if (!res.ok) {
    if (res.status === 404) throw new Error(`No card matches "${query}"`);
    throw new Error(`Scryfall returned ${res.status}`);
  }
  const c = await res.json();
  return normaliseCard(c);
}

function normaliseCard(c) {
  // Handle double-faced cards: image_uris lives under card_faces[0] in that case
  const imageRoot = c.image_uris || (c.card_faces && c.card_faces[0] && c.card_faces[0].image_uris) || {};
  return {
    id: c.id,
    oracle_id: c.oracle_id,
    name: c.name,
    type_line: c.type_line || '',
    set_name: c.set_name || '',
    set: (c.set || '').toUpperCase(),
    collector_number: c.collector_number || '',
    rarity: c.rarity || '',
    image_normal: imageRoot.normal || imageRoot.large || imageRoot.small || '',
    image_small: imageRoot.small || imageRoot.normal || '',
    price_usd: c.prices && c.prices.usd ? c.prices.usd : null,
    price_usd_foil: c.prices && c.prices.usd_foil ? c.prices.usd_foil : null,
    price_eur: c.prices && c.prices.eur ? c.prices.eur : null,
    tcgplayer_url: (c.purchase_uris && c.purchase_uris.tcgplayer) || '',
    scanned_at: Date.now()
  };
}

/* ═════════════════════════════════════════════════════════════════════
   PHONE  —  Camera, capture, OCR, lookup
   ═════════════════════════════════════════════════════════════════════ */
let videoStream = null;
let ocrWorker = null;

function setStatus(text, klass) {
  const el = document.getElementById('status');
  el.className = 'status' + (klass ? ' ' + klass : '');
  const labelMap = { error: 'Error', ok: 'Identified', '': 'Status', undefined: 'Status' };
  const lbl = labelMap[klass || ''] || 'Status';
  el.innerHTML = `<span class="label">${lbl}</span>${text}`;
}

async function initPhone() {
  // Bind once
  if (initPhone._bound) return;
  initPhone._bound = true;

  document.getElementById('startBtn').addEventListener('click', startCamera);
  document.getElementById('captureBtn').addEventListener('click', captureAndIdentify);
  document.getElementById('manualBtn').addEventListener('click', doManualLookup);
  document.getElementById('manualInput').addEventListener('keydown', (e) => {
    if (e.key === 'Enter') doManualLookup();
  });
}

async function startCamera() {
  const startBtn = document.getElementById('startBtn');
  const captureBtn = document.getElementById('captureBtn');
  try {
    setStatus('Requesting camera permission…');
    videoStream = await navigator.mediaDevices.getUserMedia({
      video: { facingMode: { ideal: 'environment' }, width: { ideal: 1280 }, height: { ideal: 1280 } },
      audio: false
    });
    const video = document.getElementById('video');
    video.srcObject = videoStream;
    await video.play();
    startBtn.textContent = 'Camera on';
    startBtn.disabled = true;
    captureBtn.disabled = false;
    setStatus('Line up the card. The dashed band marks where the card title should sit.', 'ok');
  } catch (e) {
    setStatus(`Camera unavailable: ${e.message}. Use the manual lookup below.`, 'error');
  }
}

function cropTitleRegion(video) {
  // Mirror the on-screen title-band: inset 8% on all sides, top 13% of THAT box
  const vw = video.videoWidth, vh = video.videoHeight;
  if (!vw || !vh) return null;

  const inset = 0.08;
  const bandH = 0.13;
  const innerW = vw * (1 - 2 * inset);
  const innerH = vh * (1 - 2 * inset);
  const sx = vw * inset;
  const sy = vh * inset;
  const sw = innerW;
  const sh = innerH * bandH;

  const c = document.createElement('canvas');
  // Upscale a touch for OCR
  const scale = 2;
  c.width = Math.floor(sw * scale);
  c.height = Math.floor(sh * scale);
  const ctx = c.getContext('2d');
  ctx.drawImage(video, sx, sy, sw, sh, 0, 0, c.width, c.height);

  // Light pre-processing: grayscale + contrast bump
  const img = ctx.getImageData(0, 0, c.width, c.height);
  const d = img.data;
  for (let i = 0; i < d.length; i += 4) {
    const v = 0.299 * d[i] + 0.587 * d[i+1] + 0.114 * d[i+2];
    // Stretch contrast around 128
    const k = Math.max(0, Math.min(255, (v - 128) * 1.4 + 128));
    d[i] = d[i+1] = d[i+2] = k;
  }
  ctx.putImageData(img, 0, 0);
  return c;
}

async function ensureOCR() {
  if (ocrWorker) return ocrWorker;
  if (typeof Tesseract === 'undefined') {
    throw new Error('OCR engine failed to load (CDN unreachable). Use the manual lookup below.');
  }
  setStatus('Loading OCR engine (one-time, ~3 MB)…');
  ocrWorker = await Tesseract.createWorker('eng');
  // Card titles are a single line with mixed letters, apostrophes, commas, hyphens
  await ocrWorker.setParameters({
    tessedit_char_whitelist:
      "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz ,'-",
    tessedit_pageseg_mode: '7' // single text line
  });
  return ocrWorker;
}

function cleanTitleText(raw) {
  return (raw || '')
    .replace(/[\r\n]+/g, ' ')
    .replace(/[^A-Za-z ,'\-]/g, ' ')
    .replace(/\s+/g, ' ')
    .trim();
}

async function captureAndIdentify() {
  const captureBtn = document.getElementById('captureBtn');
  captureBtn.disabled = true;
  try {
    const video = document.getElementById('video');
    if (!video.videoWidth) throw new Error('Camera not running yet');

    const cropped = cropTitleRegion(video);
    if (!cropped) throw new Error('Could not capture frame');

    const worker = await ensureOCR();
    setStatus('Reading card title (OCR)…');
    const { data: { text } } = await worker.recognize(cropped);
    const guess = cleanTitleText(text);

    if (!guess || guess.length < 3) {
      throw new Error(`OCR couldn't read a name. Try better lighting, or type the name below.`);
    }

    setStatus(`OCR read “${guess}” — searching Scryfall…`);
    const card = await scryfallLookup(guess);
    await pushScan(card);
    showPhoneResult(card);
    setStatus(`Matched ${card.name} (${card.set_name}).`, 'ok');
  } catch (e) {
    setStatus(e.message || String(e), 'error');
  } finally {
    captureBtn.disabled = false;
  }
}

async function doManualLookup() {
  const input = document.getElementById('manualInput');
  const q = input.value.trim();
  if (!q) {
    setStatus('Type a card name first.', 'error');
    return;
  }
  try {
    setStatus(`Searching Scryfall for “${q}”…`);
    const card = await scryfallLookup(q);
    await pushScan(card);
    showPhoneResult(card);
    setStatus(`Matched ${card.name} (${card.set_name}).`, 'ok');
    input.value = '';
  } catch (e) {
    setStatus(e.message || String(e), 'error');
  }
}

function formatPrice(p) {
  if (p === null || p === undefined || p === '') return null;
  const n = parseFloat(p);
  if (isNaN(n)) return null;
  return `$${n.toFixed(2)}`;
}

function showPhoneResult(card) {
  document.getElementById('result').classList.add('show');
  document.getElementById('resImg').src = card.image_small || card.image_normal || '';
  document.getElementById('resName').textContent = card.name;
  document.getElementById('resSet').textContent =
    `${card.set_name}${card.collector_number ? ' · #' + card.collector_number : ''}${card.rarity ? ' · ' + card.rarity : ''}`;

  const p = formatPrice(card.price_usd);
  const priceEl = document.getElementById('resPrice');
  if (p) { priceEl.textContent = p; priceEl.classList.remove('none'); }
  else   { priceEl.textContent = 'no listing'; priceEl.classList.add('none'); }

  const foil = formatPrice(card.price_usd_foil);
  document.getElementById('resFoil').textContent = foil ? `· foil ${foil}` : '';
}

/* ═════════════════════════════════════════════════════════════════════
   GLASSES  —  600×600 HUD, arrow-key + Enter focus management
   ═════════════════════════════════════════════════════════════════════ */
let glassesState = { scans: [], featuredIdx: 0 };
let glassesPollTimer = null;

async function initGlasses() {
  if (initGlasses._bound) {
    await refreshGlasses();
    return;
  }
  initGlasses._bound = true;

  // Detect "real glasses" mode (no hash routing, served at root) — for now,
  // we use a body class to drop the preview chrome. Toggle with ?bare in URL.
  if (location.search.includes('bare') || location.hash.includes('bare')) {
    document.body.classList.add('glasses-only');
  }

  await refreshGlasses();

  // Cross-tab live updates
  window.addEventListener('storage', (e) => {
    if (e.key === STORAGE_KEY) refreshGlasses();
  });
  // Belt-and-suspenders polling for environments without 'storage' events
  if (glassesPollTimer) clearInterval(glassesPollTimer);
  glassesPollTimer = setInterval(refreshGlasses, 2500);

  // Arrow / Enter / Esc focus management per MRBD docs
  document.addEventListener('keydown', handleGlassesKey);
}

function handleGlassesKey(e) {
  if (document.getElementById('glasses').hidden) return;
  const focusables = Array.from(document.querySelectorAll('.hud-item.focusable:not([disabled])'));
  switch (e.key) {
    case 'ArrowUp':
    case 'ArrowLeft': {
      if (!focusables.length) return;
      const i = focusables.indexOf(document.activeElement);
      const next = i <= 0 ? focusables.length - 1 : i - 1;
      focusables[next].focus();
      e.preventDefault();
      break;
    }
    case 'ArrowDown':
    case 'ArrowRight': {
      if (!focusables.length) return;
      const i = focusables.indexOf(document.activeElement);
      const next = i === -1 ? 0 : (i + 1) % focusables.length;
      focusables[next].focus();
      e.preventDefault();
      break;
    }
    case 'Enter': {
      if (document.activeElement && document.activeElement.classList.contains('focusable')) {
        document.activeElement.click();
        e.preventDefault();
      }
      break;
    }
    case 'Escape': {
      location.hash = '';
      render();
      e.preventDefault();
      break;
    }
  }
}

async function refreshGlasses() {
  const scans = await loadScans();
  glassesState.scans = scans;
  if (glassesState.featuredIdx >= scans.length) glassesState.featuredIdx = 0;

  const dot = document.getElementById('hudDot');
  const stat = document.getElementById('hudStat');
  if (scans.length) {
    dot.classList.remove('idle');
    stat.textContent = `LIVE · ${scans.length} CARD${scans.length === 1 ? '' : 'S'}`;
  } else {
    dot.classList.add('idle');
    stat.textContent = 'WAITING';
  }

  renderFeatured();
  renderList();
}

function renderFeatured() {
  const card = glassesState.scans[glassesState.featuredIdx];
  const imgWrap = document.getElementById('featImgWrap');
  const name = document.getElementById('featName');
  const type = document.getElementById('featType');
  const setEl = document.getElementById('featSet');
  const priceBlock = document.getElementById('featPriceBlock');

  imgWrap.innerHTML = '';
  if (!card) {
    imgWrap.classList.add('empty');
    name.textContent = 'No card yet';
    type.textContent = '';
    setEl.textContent = '';
    priceBlock.innerHTML = '';
    return;
  }

  imgWrap.classList.remove('empty');
  if (card.image_normal || card.image_small) {
    const img = document.createElement('img');
    img.alt = card.name;
    img.src = card.image_normal || card.image_small;
    img.referrerPolicy = 'no-referrer';
    imgWrap.appendChild(img);
  }
  // Price overlay on the card image
  const priceStr = formatPrice(card.price_usd);
  const overlay = document.createElement('div');
  overlay.className = 'price-overlay' + (priceStr ? '' : ' none');
  overlay.textContent = priceStr || '—';
  imgWrap.appendChild(overlay);

  name.textContent = card.name;
  type.textContent = card.type_line || '';
  setEl.textContent = `${card.set} · ${card.set_name}${card.collector_number ? ' · #' + card.collector_number : ''}`;

  priceBlock.innerHTML = '';
  priceBlock.appendChild(priceRow('TCG Market', formatPrice(card.price_usd), 'main'));
  if (card.price_usd_foil) {
    priceBlock.appendChild(priceRow('TCG Foil', formatPrice(card.price_usd_foil), 'foil'));
  }
  if (card.price_eur) {
    priceBlock.appendChild(priceRow('Cardmarket', '€' + parseFloat(card.price_eur).toFixed(2), 'foil'));
  }
}

function priceRow(label, value, kind) {
  const row = document.createElement('div');
  row.className = 'feat-price-row';
  const lbl = document.createElement('div');
  lbl.className = 'feat-price-lbl';
  lbl.textContent = label;
  const val = document.createElement('div');
  val.className = 'feat-price-val' + (kind === 'foil' ? ' foil' : '') + (value ? '' : ' none');
  val.textContent = value || 'no listing';
  row.appendChild(lbl);
  row.appendChild(val);
  return row;
}

function renderList() {
  const list = document.getElementById('hudList');
  // Wipe everything after the label
  const lbl = list.querySelector('.hud-list-lbl');
  list.innerHTML = '';
  list.appendChild(lbl);

  if (!glassesState.scans.length) {
    const empty = document.createElement('div');
    empty.className = 'hud-empty';
    empty.innerHTML = `Scan a card on the phone to see it appear here.
      <div class="hud-empty-mono">Open <span style="color: var(--hud-cream);">?demo</span> to preview</div>`;
    list.appendChild(empty);
    return;
  }

  // Skip the currently featured card in the list (it's already big at the top)
  glassesState.scans.forEach((c, i) => {
    if (i === glassesState.featuredIdx) return;
    const btn = document.createElement('button');
    btn.className = 'hud-item focusable';
    btn.setAttribute('data-idx', i);
    const nameSpan = document.createElement('span');
    nameSpan.className = 'hi-name';
    nameSpan.textContent = c.name;
    const priceSpan = document.createElement('span');
    const p = formatPrice(c.price_usd);
    priceSpan.className = 'hi-price' + (p ? '' : ' none');
    priceSpan.textContent = p || '—';
    btn.appendChild(nameSpan);
    btn.appendChild(priceSpan);
    btn.addEventListener('click', () => {
      glassesState.featuredIdx = parseInt(btn.getAttribute('data-idx'), 10);
      renderFeatured();
      renderList();
    });
    list.appendChild(btn);
  });

  // Auto-focus first focusable so arrow keys do something immediately
  const firstFocus = list.querySelector('.focusable');
  if (firstFocus && !document.activeElement.classList.contains('focusable')) {
    firstFocus.focus({ preventScroll: true });
  }
}

/* ═════════════════════════════════════════════════════════════════════
   DEMO SEED  —  for previewing the glasses HUD without running a scan
   ═════════════════════════════════════════════════════════════════════ */
async function seedDemoScans() {
  if (seedDemoScans._done) return;
  seedDemoScans._done = true;
  const existing = await loadScans();
  if (existing.length) return; // don't clobber real data
  const samples = [
    'Sol Ring',
    'Lightning Bolt',
    'Counterspell',
    'Birds of Paradise',
    'Ragavan, Nimble Pilferer'
  ];
  for (const name of samples) {
    try {
      const c = await scryfallLookup(name);
      await pushScan(c);
      // Tiny stagger so the IDs don't all collide on scanned_at
      await new Promise(r => setTimeout(r, 90));
    } catch (e) {
      console.warn('demo seed failed for', name, e);
    }
  }
  // Nudge glasses view if it's the active one
  if (currentMode() === 'glasses') refreshGlasses();
}

/* ═════════════════════════════════════════════════════════════════════
   BOOT
   ═════════════════════════════════════════════════════════════════════ */
render();

// Cleanup the OCR worker on page unload so we don't leak memory across HMR/refresh
window.addEventListener('beforeunload', () => {
  if (ocrWorker) ocrWorker.terminate();
  if (videoStream) videoStream.getTracks().forEach(t => t.stop());
});
</script>
</body>
</html>
