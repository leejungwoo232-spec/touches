<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<meta name="color-scheme" content="light">
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+KR:wght@400;500&display=swap" rel="stylesheet">
<title>탄성 천</title>
<style>
  :root {
    box-sizing: border-box;
    padding-top: env(safe-area-inset-top, 0px);
    padding-bottom: env(safe-area-inset-bottom, 0px);
    background: #ffffff;
  }
  *, *::before, *::after { box-sizing: inherit; margin: 0; padding: 0; }
  html { height: 100%; }
  body { height: 100%; background: #ffffff; overflow: hidden; -webkit-tap-highlight-color: transparent; touch-action: none; }
  #stage { position: fixed; inset: 0; width: 100%; height: 100%; display: block; outline: none; }
  #text {
    position: fixed; left: 50%; transform: translateX(-50%); width: min(680px, 86vw);
    top: 20vh; bottom: 43vh;
    display: flex; flex-direction: column; gap: 18px;
    pointer-events: none;
  }
  #log {
    flex: 1; min-height: 0; overflow: hidden;
    display: flex; flex-direction: column; justify-content: flex-end; align-items: stretch; gap: 8px;
    -webkit-mask-image: linear-gradient(to bottom, transparent 0, #000 35%);
            mask-image: linear-gradient(to bottom, transparent 0, #000 35%);
  }
  .msg {
    max-width: min(520px, 70%);
    padding: 8px 13px 9px;
    border-radius: 15px 15px 15px 4px;
    background: #f2f3f5; color: #3b3d43;
    font: 500 14px/1.6 "Noto Sans KR", -apple-system, BlinkMacSystemFont, "Apple SD Gothic Neo", sans-serif;
    white-space: pre-wrap; word-break: break-word;
    animation: arrive .28s cubic-bezier(.2,.8,.2,1);
  }
  .msg { align-self: flex-end; border-radius: 15px 15px 4px 15px; }
  .msg.them { align-self: flex-start; border-radius: 15px 15px 15px 4px; background: #ffffff; box-shadow: inset 0 0 0 1px #e2e3e7; color: #2f3136; }
  .note { align-self: center; }
  .note { font: 400 12px/1.5 "Noto Sans KR", -apple-system, BlinkMacSystemFont, "Apple SD Gothic Neo", sans-serif; color: #a9abb1; padding: 2px 2px; }
  #status {
    position: fixed; left: 7vw; top: calc(18px + env(safe-area-inset-top, 0px));
    font: 400 12px/1.5 "Noto Sans KR", -apple-system, BlinkMacSystemFont, "Apple SD Gothic Neo", sans-serif;
    color: #a9abb1; pointer-events: none;
  }
  #status[hidden] { display: none; }
  @keyframes arrive { from { opacity: 0; transform: translateY(8px); } }
  #draft { text-align: center; flex: none; max-height: 30%; overflow: hidden; display: flex; flex-direction: column; justify-content: flex-end; }
  #line {
    font: 500 14px/1.75 "Noto Sans KR", -apple-system, BlinkMacSystemFont, "Apple SD Gothic Neo", sans-serif;
    letter-spacing: 0.01em; color: #4a4c52;
    white-space: pre-wrap; word-break: break-word;
  }
  #line:empty::before { content: attr(data-placeholder); color: #c3c5ca; font-weight: 400; }
  #caret { display: inline-block; width: 1px; height: 1.05em; margin-left: 1px; vertical-align: -0.15em; background: #4a4c52; animation: blink 1.1s steps(1) infinite; }
  @keyframes blink { 50% { opacity: 0; } }
  @media (prefers-reduced-motion: reduce) { #caret, .msg { animation: none; } }
  #input {
    position: fixed; left: 50%; bottom: 43vh; width: 1px; height: 1px;
    opacity: 0; border: 0; padding: 0; resize: none; outline: none; background: transparent; color: transparent;
    font-size: 16px; /* iOS 자동 확대 방지 */
  }
</style>
<script src="https://cdn.jsdelivr.net/npm/mqtt@5/dist/mqtt.min.js"></script>
</head>
<body>
<canvas id="stage" tabindex="0" aria-label="키를 누르면 해당 위치의 천이 눌립니다"></canvas>
<p id="status" hidden></p>
<div id="text"><div id="log" aria-live="polite"></div><p id="draft"><span><span id="line" data-placeholder="화면을 클릭하고 입력해보세요"></span><span id="caret"></span></span></p></div>
<textarea id="input" autocomplete="off" autocorrect="off" autocapitalize="off" spellcheck="false" aria-label="입력"></textarea>
<script>
(() => {
  const cv = document.getElementById('stage');
  const ctx = cv.getContext('2d');
  const reduce = matchMedia('(prefers-reduced-motion: reduce)').matches;
  const L = (() => { const x = -0.45, y = -0.6, z = 0.66, n = Math.hypot(x, y, z); return [x/n, y/n, z/n]; })();

  // ── QWERTY 배열: [키 중심 x, y, 키 폭] (1 = 일반 키 한 칸) ──
  const KEYS = {};
  const put = (code, x, y, w = 1) => KEYS[code] = [x, y, w];
  put('Escape', 0.5, 0);
  ['F1','F2','F3','F4'].forEach((c,i) => put(c, 2.5+i, 0));
  ['F5','F6','F7','F8'].forEach((c,i) => put(c, 7+i, 0));
  ['F9','F10','F11','F12'].forEach((c,i) => put(c, 11.5+i, 0));
  ['Backquote','Digit1','Digit2','Digit3','Digit4','Digit5','Digit6','Digit7','Digit8','Digit9','Digit0','Minus','Equal']
    .forEach((c,i) => put(c, 0.5+i, 1.25));
  put('Backspace', 14, 1.25, 2);
  put('Tab', 0.75, 2.25, 1.5);
  ['KeyQ','KeyW','KeyE','KeyR','KeyT','KeyY','KeyU','KeyI','KeyO','KeyP','BracketLeft','BracketRight']
    .forEach((c,i) => put(c, 2+i, 2.25));
  put('Backslash', 14.25, 2.25, 1.5);
  put('CapsLock', 0.875, 3.25, 1.75);
  ['KeyA','KeyS','KeyD','KeyF','KeyG','KeyH','KeyJ','KeyK','KeyL','Semicolon','Quote']
    .forEach((c,i) => put(c, 2.25+i, 3.25));
  put('Enter', 13.875, 3.25, 2.25);
  put('ShiftLeft', 1.125, 4.25, 2.25);
  ['KeyZ','KeyX','KeyC','KeyV','KeyB','KeyN','KeyM','Comma','Period','Slash']
    .forEach((c,i) => put(c, 2.75+i, 4.25));
  put('ShiftRight', 13.625, 4.25, 2.75);
  put('ControlLeft', 0.625, 5.25, 1.25); put('MetaLeft', 1.875, 5.25, 1.25); put('AltLeft', 3.125, 5.25, 1.25);
  put('Space', 6.875, 5.25, 6.25);
  put('AltRight', 10.625, 5.25, 1.25); put('MetaRight', 11.875, 5.25, 1.25);
  put('ContextMenu', 13.125, 5.25, 1.25); put('ControlRight', 14.375, 5.25, 1.25);
  // e.code가 비어 오는 환경(한글 입력기 등) 대비: keyCode → code
  const KC = { 8:'Backspace', 9:'Tab', 13:'Enter', 16:'ShiftLeft', 17:'ControlLeft', 18:'AltLeft', 20:'CapsLock', 27:'Escape', 32:'Space',
    91:'MetaLeft', 93:'MetaRight', 186:'Semicolon', 187:'Equal', 188:'Comma', 189:'Minus', 190:'Period', 191:'Slash', 192:'Backquote',
    219:'BracketLeft', 220:'Backslash', 221:'BracketRight', 222:'Quote' };
  for (let i = 0; i < 26; i++) KC[65+i] = 'Key' + String.fromCharCode(65+i);
  for (let i = 0; i < 10; i++) KC[48+i] = 'Digit' + i;
  for (let i = 1; i <= 12; i++) KC[111+i] = 'F' + i;
  const HANGUL = 'ㅂㅈㄷㄱㅅㅛㅕㅑㅐㅔㅁㄴㅇㄹㅎㅗㅓㅏㅣㅋㅌㅊㅍㅠㅜㅡ', HQ = 'QWERTYUIOPASDFGHJKLZXCVBNM';
  function codeOf(e) {
    if (e.code && e.code !== 'Unidentified') return e.code;
    if (KC[e.keyCode]) return KC[e.keyCode];
    const k = (e.key || '');
    if (k.length === 1) {
      const u = k.toUpperCase();
      if (u >= 'A' && u <= 'Z') return 'Key' + u;
      if (k >= '0' && k <= '9') return 'Digit' + k;
      const h = HANGUL.indexOf(k); if (h >= 0) return 'Key' + HQ[h];
      if (k === ' ') return 'Space';
    }
    return k || 'unknown';
  }
  const KB_W = 15, KB_Y0 = -0.5, KB_H = 6.25;

  let vw, vh, s, W, H, img, buf;
  let ux, uy, ox, oy, oyT, u, Rc, tip, Dmax;
  let cell, gw, gh, hg, gxg, gyg, colI, colF, rowJ, rowF;

  function resize() {
    vw = innerWidth; vh = innerHeight;
    s = Math.max(1, Math.ceil(vw / 720));
    W = Math.ceil(vw / s); H = Math.ceil(vh / s);
    cv.width = W; cv.height = H;
    img = ctx.createImageData(W, H); buf = img.data; buf.fill(255);
    // 아래쪽: 내 키보드 / 위쪽: 상대 키보드 (마주 앉은 것처럼 180° 돌아가 있음)
    const mx = vw * 0.07, zh = vh * 0.38;
    ux = (vw - 2*mx) / KB_W; uy = zh / KB_H;
    ox = mx; oy = (vh * 0.97 - zh) - KB_Y0 * uy;          // 내 키보드
    oyT = vh * 0.03 - KB_Y0 * uy;                        // 상대 키보드
    u = Math.min(ux, uy);
    Rc = u * 0.95; tip = u * 0.08; Dmax = Rc * 0.36;   // 이전의 절반 크기
    cell = Math.max(3, u * 0.055);
    gw = Math.ceil(vw / cell) + 2; gh = Math.ceil(vh / cell) + 2;
    hg = new Float32Array(gw*gh);
    gxg = new Float32Array(gw*gh); gyg = new Float32Array(gw*gh);
    colI = new Int32Array(W); colF = new Float32Array(W);
    rowJ = new Int32Array(H); rowF = new Float32Array(H);
    for (let i = 0; i < W; i++) { const g = ((i + 0.5) * s) / cell - 0.5; colI[i] = Math.max(0, Math.min(gw-2, Math.floor(g))); colF[i] = Math.min(1, Math.max(0, g - colI[i])); }
    for (let j = 0; j < H; j++) { const g = ((j + 0.5) * s) / cell - 0.5; rowJ[j] = Math.max(0, Math.min(gh-2, Math.floor(g))); rowF[j] = Math.min(1, Math.max(0, g - rowJ[j])); }
    blank = false;
  }

  const dents = new Map();
  function press(id, pos) {
    let dn = dents.get(id);
    if (!dn) { dn = { d: 0, v: 0 }; dents.set(id, dn); }
    Object.assign(dn, pos, { held: true, t0: performance.now() });
    dn.v += Dmax * 16;            // 치는 순간 바로 들어가도록 첫 힘
    dn.minUntil = dn.t0 + 130;    // 아주 짧게 쳐도 충분히 눌리도록
  }
  function release(id) { const dn = dents.get(id); if (dn) dn.held = false; }

  // 입력: 숨은 입력창이 글자(한글 조합 포함)를 받고, 위쪽 텍스트로 보여줌
  const input = document.getElementById('input');
  const line = document.getElementById('line');
  const sync = () => {
    line.textContent = input.value;
    const n = input.value.length;
    if (input.selectionStart !== n) input.setSelectionRange(n, n);   // 커서는 항상 끝
  };
  ['input', 'compositionstart', 'compositionupdate', 'compositionend'].forEach(ev => input.addEventListener(ev, () => requestAnimationFrame(sync)));
  const log = document.getElementById('log');
  function send() {
    const txt = input.value.replace(/^\n+|\s+$/g, '');
    if (!txt.trim()) return;
    addMsg(txt.slice(0, 1000), false);
    send_chat(txt.slice(0, 1000));
    input.value = ''; sync();
  }
  function addMsg(txt, them) {
    const m = document.createElement('div');
    m.className = them ? 'msg them' : 'msg'; m.textContent = txt;
    log.appendChild(m);
    while (log.children.length > 60) log.firstChild.remove();
  }
  function addNote(txt) {
    const m = document.createElement('div');
    m.className = 'note'; m.textContent = txt;
    log.appendChild(m);
  }
  // Enter = 보내기, Shift+Enter = 아래 줄에 이어 쓰기
  input.addEventListener('keydown', e => {
    if (e.key !== 'Enter' || e.shiftKey) return;
    if (e.isComposing || e.keyCode === 229) return;   // 한글 조합 중 Enter는 글자 확정만
    e.preventDefault();
    send();
  });
  const grabFocus = () => { try { window.focus(); input.focus({ preventScroll: true }); } catch (_) {} };
  grabFocus();
  addEventListener('load', grabFocus);
  const hideHint = () => {};
  function onDown(e) {
    hideHint();
    const code = codeOf(e);
    if ((e.metaKey || e.ctrlKey) && !/^(Meta|Control)/.test(code)) return;
    // 글자 입력은 그대로 두고, 포커스 이동·커서 이동만 막음
    if (code === 'Tab' || /^Arrow|^Home$|^End$|^Page/.test(code)) e.preventDefault();
    if (dents.get(code)?.held) return;              // 자동 반복 무시
    pressKey(code, code);
    netKey(code, true);
  }
  function pressKey(id, code, them) {
    const k = KEYS[code] || [KB_W/2, KB_Y0 + KB_H/2, 1];            // 배열 밖 키는 가운데
    if (!them) press(id, { kx: k[0], ky: k[1], kw: k[2], zone: 0 });
    else       press(id, { kx: KB_W - k[0], ky: 2*KB_Y0 + KB_H - k[1], kw: k[2], zone: 1 });
  }
  function onUp(e) {
    const code = codeOf(e);
    release(code);
    netKey(code, false);
    if (/^Meta/.test(code)) dents.forEach(d => d.held = false);  // Mac: Cmd를 누른 동안 다른 키의 keyup이 안 오는 경우
  }
  addEventListener('keydown', onDown, true);
  addEventListener('keyup', onUp, true);
  addEventListener('blur', () => dents.forEach(d => d.held = false));
  // ── 두 사람만 들어올 수 있는 방 ──
  // 공개 MQTT 중계 서버를 통해 메시지를 주고받음 (계정·설정 없음, 일반 웹 연결이라 대부분의 네트워크에서 통과)
  // 주소 뒤에 ?room=이름 을 붙이면 다른 방. 방마다 먼저 들어온 두 사람만 대화에 참여.
  const statusEl = document.getElementById('status');
  const setStatus = txt => { statusEl.hidden = !txt; statusEl.textContent = txt || ''; };
  const roomName = (new URLSearchParams(location.search).get('room') || 'main').replace(/[^a-zA-Z0-9_-]/g, '').slice(0, 32) || 'main';
  const BASE = 'elastic-cloth-7q2x/v1/' + roomName + '/';
  const BROKERS = ['wss://broker.emqx.io:8084/mqtt', 'wss://broker.hivemq.com:8884/mqtt'];
  const CODE_RE = /^[A-Za-z0-9]{1,24}$/;
  const myId = Math.random().toString(36).slice(2, 12);
  const since = Date.now() + Math.random();
  const seen = new Map();              // id -> { since, last }
  const net = { client: null, inRoom: false, partner: null, ready: false };
  const dec = new TextDecoder();

  function netSend(obj) {
    if (!net.client || !net.inRoom || !net.partner) return;
    try { net.client.publish(BASE + 'm', JSON.stringify(Object.assign({ from: myId, to: net.partner }, obj)), { qos: 0 }); } catch (_) {}
  }
  function send_chat(txt) { netSend({ t: 'chat', text: txt }); }
  function netKey(code, down) { if (CODE_RE.test(code)) netSend({ t: 'key', code, down: !!down }); }
  function releasePartner() { for (const [id, dn] of dents) if (id.startsWith('r:')) dn.held = false; }

  function announce() {
    if (!net.client) return;
    try { net.client.publish(BASE + 'p/' + myId, JSON.stringify({ since, t: Date.now() }), { qos: 1, retain: true }); } catch (_) {}
  }
  function evaluate() {
    if (!net.ready) return;
    const now = Date.now();
    for (const [id, p] of seen) if (id !== myId && now - p.last > 25000) seen.delete(id);   // 오래 소식 없는 흔적은 버림
    seen.set(myId, { since, last: now });
    const two = [...seen.entries()].sort((x, y) => (x[1].since - y[1].since) || (x[0] < y[0] ? -1 : 1)).slice(0, 2).map(e => e[0]);
    const was = net.partner;
    net.inRoom = two.includes(myId);
    net.partner = net.inRoom ? (two.find(id => id !== myId) || null) : null;
    if (!net.inRoom) setStatus('지금은 두 사람이 대화 중이에요. 자리가 나면 들어올 수 있어요');
    else if (net.partner) setStatus('상대와 연결됨');
    else setStatus('상대를 기다리는 중 · 방 ' + roomName);
    if (net.partner && net.partner !== was) addNote('상대가 들어왔어요');
    if (was && net.partner !== was) { addNote('상대가 나갔어요'); releasePartner(); }
  }
  function onMessage(topic, payload, packet) {
    let s = ''; try { s = typeof payload === 'string' ? payload : dec.decode(payload); } catch (_) { return; }
    if (topic.startsWith(BASE + 'p/')) {
      const id = topic.slice((BASE + 'p/').length);
      if (!/^[a-z0-9]{1,16}$/.test(id) || id === myId) return;
      if (!s) { seen.delete(id); evaluate(); return; }
      let d; try { d = JSON.parse(s); } catch (_) { return; }
      if (!d || typeof d.since !== 'number' || typeof d.t !== 'number') return;
      const last = packet && packet.retain ? Math.min(Date.now(), d.t) : Date.now();
      seen.set(id, { since: d.since, last });
      evaluate();
      return;
    }
    if (topic === BASE + 'm') {
      let d; try { d = JSON.parse(s); } catch (_) { return; }
      if (!d || d.to !== myId || !net.partner || d.from !== net.partner) return;
      if (d.t === 'chat' && typeof d.text === 'string' && d.text.trim()) addMsg(d.text.slice(0, 1000), true);
      else if (d.t === 'key' && typeof d.code === 'string' && CODE_RE.test(d.code)) {
        const id = 'r:' + d.code;
        if (d.down) { if (!dents.get(id)?.held) pressKey(id, d.code, true); }
        else release(id);
      }
    }
  }
  function loadLib() {
    if (window.mqtt) return Promise.resolve(true);
    return new Promise(res => {
      const s = document.createElement('script');
      s.src = 'https://unpkg.com/mqtt@5/dist/mqtt.min.js';
      s.onload = () => res(!!window.mqtt); s.onerror = () => res(false);
      document.head.appendChild(s);
    });
  }
  async function connect(bi = 0) {
    if (!(await loadLib())) { setStatus('연결 기능을 불러오지 못했어요 (인터넷 연결이나 광고 차단 확장 프로그램을 확인해주세요)'); return; }
    setStatus('연결하는 중');
    let ever = false;
    const c = window.mqtt.connect(BROKERS[bi], {
      clientId: 'ec_' + myId, clean: true, keepalive: 10, reconnectPeriod: 2000, connectTimeout: 8000,
      will: { topic: BASE + 'p/' + myId, payload: '', qos: 1, retain: true },
    });
    net.client = c;
    c.on('connect', () => {
      ever = true;
      c.subscribe([BASE + 'p/+', BASE + 'm'], { qos: 0 }, () => {
        announce();
        setTimeout(() => { net.ready = true; evaluate(); }, 1200);   // 방에 있던 사람들 정보를 먼저 받고 판단
      });
    });
    c.on('message', onMessage);
    c.on('offline', () => { if (ever) setStatus('연결이 잠시 끊겼어요. 다시 잇는 중'); });
    c.on('error', e => console.warn('[mqtt]', e && e.message));
    // 첫 서버에 못 붙으면 두 번째 서버로
    setTimeout(() => { if (!ever && bi + 1 < BROKERS.length) { try { c.end(true); } catch (_) {} connect(bi + 1); } }, 9000);
  }
  setInterval(() => { announce(); evaluate(); }, 8000);
  addEventListener('load', () => connect());
  addEventListener('pagehide', () => {
    if (!net.client) return;
    try { net.client.publish(BASE + 'p/' + myId, '', { qos: 0, retain: true }); net.client.end(); } catch (_) {}
  });

  // 터치/마우스: 누른 자리
  cv.addEventListener('pointerdown', e => { e.preventDefault(); grabFocus(); }, true);
  cv.addEventListener('pointerdown', e => press('p'+e.pointerId, { px: e.clientX / vw, py: e.clientY / vh, kw: 1 }));
  addEventListener('pointerup', e => release('p'+e.pointerId));
  addEventListener('pointercancel', e => release('p'+e.pointerId));

  let last = performance.now(), blank = false, seed = 1;
  const rnd = () => ((seed = (seed * 16807) % 2147483647) / 2147483647) - 0.5;

  function frame() {
    const now = performance.now();
    const dt = Math.max(0, Math.min(0.033, (now - last) / 1000)); last = now;
    for (const [id, dn] of dents) {
      let target = 0, k, c;
      const on = dn.held || now < dn.minUntil;
      if (on) {
        const t = Math.min(1, (now - dn.t0) / 1400);
        target = Dmax * (0.8 + 0.2 * (1 - Math.pow(1 - t, 3)));
        k = 420; c = 36;
      } else { k = 190; c = reduce ? 28 : 6; }
      dn.v += (k * (target - dn.d) - c * dn.v) * dt;
      dn.d += dn.v * dt;
      if (!isFinite(dn.d) || !isFinite(dn.v)) { dn.d = 0; dn.v = 0; }
      if (!on && Math.abs(dn.d) < 0.02 && Math.abs(dn.v) < 0.02) dents.delete(id);
    }
    if (dents.size) { build(); render(); blank = false; }
    else if (!blank) { buf.fill(255); ctx.putImageData(img, 0, 0); blank = true; }
    requestAnimationFrame(frame);
  }

  // 여러 눌림을 한 장의 천으로: 두 원뿔이 겹치면 물방울처럼 하나로 이어짐
  function build() {
    hg.fill(1);
    const Dref = Dmax * 1.08, iR = 1 / Dref;
    const w = Rc * 0.4, cut = Rc + w;
    for (const dn of dents.values()) {
      let cx, cy;
      if (dn.px !== undefined) { cx = dn.px * vw; cy = dn.py * vh; }
      else { cx = ox + dn.kx * ux; cy = (dn.zone ? oyT : oy) + dn.ky * uy; }
      const half = Math.max(0, (dn.kw - 1) / 2) * ux;
      const d = dn.d;
      const i0 = Math.max(0, Math.floor((cx - half - cut) / cell)), i1 = Math.min(gw-1, Math.ceil((cx + half + cut) / cell));
      const j0 = Math.max(0, Math.floor((cy - cut) / cell)),        j1 = Math.min(gh-1, Math.ceil((cy + cut) / cell));
      for (let j = j0; j <= j1; j++) {
        const dy = (j + 0.5) * cell - cy;
        let k = j * gw + i0;
        for (let i = i0; i <= i1; i++, k++) {
          const ax = Math.abs((i + 0.5) * cell - cx) - half;
          const dx = ax > 0 ? ax : 0;
          const r = Math.sqrt(dx*dx + dy*dy);
          if (r >= cut) continue;
          const sq = Math.sqrt(r*r + tip*tip) - tip;
          // 원뿔 + 부드러운 가장자리
          let f;
          if (sq < Rc - w) f = 1 - sq / Rc;
          else { const z = Rc + w - sq; f = z > 0 ? (z * z) / (4 * w * Rc) : 0; }
          // 천의 남은 높이를 곱해감: 가까운 두 눌림은 서로를 끌어내려 하나의 골이 됨
          hg[k] *= 1 - (d * f) * iR;
        }
      }
    }
    for (let k = 0, n = hg.length; k < n; k++) hg[k] = Dref * (1 - hg[k]);   // 깊이로 변환
    // 천의 굽힘 강성: 높이를 살짝 번지게 해서 이음새를 없앰
    for (let pass = 0; pass < 3; pass++) {
      for (let j = 0; j < gh; j++) {        // 가로
        let k = j * gw, prev = hg[k];
        for (let i = 1; i < gw-1; i++) { const cur = hg[k+i]; hg[k+i] = (prev + 2*cur + hg[k+i+1]) * 0.25; prev = cur; }
      }
      for (let i = 0; i < gw; i++) {        // 세로
        let prev = hg[i];
        for (let j = 1; j < gh-1; j++) { const k = j*gw + i, cur = hg[k]; hg[k] = (prev + 2*cur + hg[k+gw]) * 0.25; prev = cur; }
      }
    }
    // 격자 기울기 (천의 높이 = -hg)
    const inv = 1 / (2 * cell);
    for (let j = 1; j < gh-1; j++) {
      let k = j * gw + 1;
      for (let i = 1; i < gw-1; i++, k++) { gxg[k] = -(hg[k+1] - hg[k-1]) * inv; gyg[k] = -(hg[k+gw] - hg[k-gw]) * inv; }
    }
  }

  function render() {
    const Lx = L[0], Ly = L[1], Lz = L[2], iD = 1 / Dmax;
    for (let j = 0, p = 0; j < H; j++) {
      const r0 = rowJ[j] * gw, fy = rowF[j], fy1 = 1 - fy;
      for (let i = 0; i < W; i++, p += 4) {
        const a = r0 + colI[i], fx = colF[i], fx1 = 1 - fx;
        const w00 = fx1*fy1, w10 = fx*fy1, w01 = fx1*fy, w11 = fx*fy;
        const hx = gxg[a]*w00 + gxg[a+1]*w10 + gxg[a+gw]*w01 + gxg[a+gw+1]*w11;
        const hy = gyg[a]*w00 + gyg[a+1]*w10 + gyg[a+gw]*w01 + gyg[a+gw+1]*w11;
        if (hx*hx + hy*hy < 1e-7) { buf[p] = buf[p+1] = buf[p+2] = 255; continue; }
        const hh = -(hg[a]*w00 + hg[a+1]*w10 + hg[a+gw]*w01 + hg[a+gw+1]*w11);
        const len = Math.sqrt(hx*hx + hy*hy + 1);
        let b = ((-hx*Lx - hy*Ly + Lz) / len) / Lz;
        if (b > 1) b = 1;
        if (hh < 0) b *= 1 - 0.05 * Math.min(1.5, -hh * iD);
        const sh = (1 - b) * 255 + rnd();
        const rr = 255 - sh * 1.04, gg = 255 - sh, bb = 255 - sh * 0.9;
        buf[p]   = rr > 255 ? 255 : rr;
        buf[p+1] = gg > 255 ? 255 : gg;
        buf[p+2] = bb > 255 ? 255 : bb;
      }
    }
    ctx.putImageData(img, 0, 0);
  }

  addEventListener('resize', resize);
  resize();
  requestAnimationFrame(frame);
})();
</script>
</body>
</html>
