<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Password Generator</title>
<style>
  :root{
    --bg:#0f1724; --card:#0b1220; --muted:#9aa4b2; --accent:#7c5cff;
    --success:#22c55e; --danger:#ef4444;
  }
  body{font-family:Inter,system-ui,Segoe UI,Roboto,"Helvetica Neue",Arial;background:linear-gradient(135deg,#071020,#0f1724);color:#e6eef6;min-height:100vh;display:flex;align-items:center;justify-content:center;padding:24px}
  .card{background:linear-gradient(180deg,rgba(255,255,255,0.02), rgba(255,255,255,0.01));backdrop-filter: blur(6px);border-radius:14px;padding:20px;width:100%;max-width:720px;box-shadow:0 10px 30px rgba(2,6,23,0.6)}
  h1{margin:0 0 8px;font-size:20px}
  p.lead{margin:0 0 18px;color:var(--muted);font-size:13px}
  .row{display:flex;gap:12px;align-items:center;margin-bottom:12px;flex-wrap:wrap}
  label{font-size:13px;color:var(--muted);}
  input[type="number"]{width:80px;padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:inherit}
  .controls{display:flex;gap:10px;flex-wrap:wrap}
  .checkbox{display:flex;align-items:center;gap:8px;background:rgba(255,255,255,0.02);padding:8px;border-radius:8px}
  button{background:var(--accent);border:none;color:white;padding:10px 14px;border-radius:10px;font-weight:600;cursor:pointer}
  button:active{transform:translateY(1px)}
  .output{display:flex;gap:8px;align-items:center;margin-top:10px}
  .password{font-family:monospace;padding:12px;border-radius:10px;background:rgba(255,255,255,0.02);flex:1;word-break:break-all}
  .tiny{font-size:12px;color:var(--muted)}
  .meter{height:10px;border-radius:8px;background:rgba(255,255,255,0.03);overflow:hidden;margin-top:8px}
  .meter > i{height:100%;display:block;width:0%;background:linear-gradient(90deg,#ef4444,#f59e0b,#22c55e)}
  .muted-block{background:rgba(255,255,255,0.02);padding:12px;border-radius:10px;margin-top:14px;color:var(--muted)}
  .copy-btn{background:transparent;border:1px solid rgba(255,255,255,0.05);padding:8px;border-radius:8px;color:var(--muted);cursor:pointer}
  footer{margin-top:12px;font-size:13px;color:var(--muted)}
  @media (max-width:520px){.row{flex-direction:column;align-items:stretch} input[type="number"]{width:100%}}
</style>
</head>
<body>
  <div class="card" role="main" aria-labelledby="title">
    <h1 id="title">Password Generator</h1>
    <p class="lead">Generate secure passwords with options for length and character types. Click generate, then copy.</p>

    <div class="row">
      <label class="tiny" for="length">Length</label>
      <input id="length" type="number" min="4" max="128" value="16" aria-label="Password length">
      <div class="tiny" style="margin-left:auto">Entropy estimate shown below</div>
    </div>

    <div class="row controls" role="group" aria-label="character options">
      <label class="checkbox"><input id="lower" type="checkbox" checked> <span class="tiny">Lowercase</span></label>
      <label class="checkbox"><input id="upper" type="checkbox" checked> <span class="tiny">Uppercase</span></label>
      <label class="checkbox"><input id="numbers" type="checkbox" checked> <span class="tiny">Numbers</span></label>
      <label class="checkbox"><input id="symbols" type="checkbox" checked> <span class="tiny">Symbols</span></label>
      <label class="checkbox"><input id="exclude-ambiguous" type="checkbox"> <span class="tiny">Exclude ambiguous (0,O,l,1)</span></label>
    </div>

    <div class="row">
      <button id="generate">Generate</button>
      <button id="shuffle" class="copy-btn" title="Shuffle last password">Shuffle</button>
      <button id="copy" class="copy-btn">Copy</button>
      <button id="clipboard-rules" class="copy-btn" title="Show tips">Tips</button>
    </div>

    <div class="output">
      <div class="password" id="password" aria-live="polite">— your password will appear here —</div>
      <button id="regen" class="copy-btn" title="Generate another quickly">↻</button>
    </div>

    <div class="meter" aria-hidden="true">
      <i id="meter-bar" style="width:0%"></i>
    </div>
    <div style="display:flex;justify-content:space-between;margin-top:6px">
      <div class="tiny" id="strength-text">Strength: —</div>
      <div class="tiny" id="entropy-text">Entropy: — bits</div>
    </div>

    <div class="muted-block" id="rules" style="display:none">
      <strong>Tips:</strong>
      <ul style="margin:6px 0 0 18px">
        <li>Use length ≥ 12 for most accounts.</li>
        <li>Use a password manager to store long unique passwords.</li>
        <li>Exclude ambiguous characters if you need to read it aloud or type on a phone.</li>
      </ul>
    </div>

    <footer>Generated locally in your browser — nothing is sent anywhere.</footer>
  </div>

<script>
(() => {
  const LOWER = 'abcdefghijklmnopqrstuvwxyz';
  const UPPER = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
  const NUMS  = '0123456789';
  const SYMS  = '!@#$%^&*()-_=+[]{};:,.<>/?~';

  const el = id => document.getElementById(id);
  const lengthEl = el('length');
  const lowerEl = el('lower'), upperEl = el('upper'), numbersEl = el('numbers'), symbolsEl = el('symbols'), excludeAmb = el('exclude-ambiguous');
  const passwordEl = el('password'), generateBtn = el('generate'), copyBtn = el('copy'), meterBar = el('meter-bar'), strengthText = el('strength-text'), entropyText = el('entropy-text');
  const shuffleBtn = el('shuffle'), regenBtn = el('regen'), tipsBtn = el('clipboard-rules'), rulesBlock = el('rules');

  let lastPassword = '';

  function buildCharset(){
    let set = '';
    if(lowerEl.checked) set += LOWER;
    if(upperEl.checked) set += UPPER;
    if(numbersEl.checked) set += NUMS;
    if(symbolsEl.checked) set += SYMS;
    if(excludeAmb.checked){
      set = set.replace(/[0OIl1]/g,''); // remove ambiguous characters
    }
    return set;
  }

  function secureRandomInt(max){
    // Returns integer in [0, max)
    const array = new Uint32Array(1);
    window.crypto.getRandomValues(array);
    return array[0] % max;
  }

  function generatePassword(){
    const len = Math.max(4, Math.min(128, parseInt(lengthEl.value)||16));
    const charset = buildCharset();
    if(!charset.length){
      alert('Pick at least one character set (lower / upper / numbers / symbols).');
      return '';
    }

    // Ensure at least one from each selected set for better coverage
    const pools = [];
    if(lowerEl.checked) pools.push(LOWER);
    if(upperEl.checked) pools.push(UPPER);
    if(numbersEl.checked) pools.push(NUMS);
    if(symbolsEl.checked) pools.push(SYMS);

    let result = [];
    // Place one required char from each chosen pool first (if length allows)
    const requiredCount = Math.min(len, pools.length);
    for(let i=0;i<requiredCount;i++){
      let p = pools[i];
      let ch;
      do {
        ch = p[secureRandomInt(p.length)];
      } while(excludeAmb.checked && /[0OIl1]/.test(ch));
      result.push(ch);
    }

    // Fill remaining with random from full charset
    for(let i=result.length;i<len;i++){
      let ch;
      do {
        ch = charset[secureRandomInt(charset.length)];
      } while(!ch);
      result.push(ch);
    }

    // Shuffle result using Fisher-Yates with crypto
    for(let i=result.length-1;i>0;i--){
      const j = secureRandomInt(i+1);
      [result[i], result[j]] = [result[j], result[i]];
    }

    lastPassword = result.join('');
    return lastPassword;
  }

  function estimateEntropy(password, charsetSize){
    // Bits of entropy ≈ length * log2(charsetSize)
    // But if forced includes required categories, it's still approximate.
    if(!charsetSize || !password) return 0;
    const bits = password.length * Math.log2(charsetSize);
    return Math.round(bits*10)/10;
  }

  function updateUI(pw){
    if(!pw) {
      passwordEl.textContent = '— your password will appear here —';
      meterBar.style.width = '0%';
      strengthText.textContent = 'Strength: —';
      entropyText.textContent = 'Entropy: — bits';
      return;
    }
    passwordEl.textContent = pw;
    const charsetSize = new Set(buildCharset()).size || 0;
    const bits = estimateEntropy(pw, charsetSize);
    entropyText.textContent = bits + ' bits';

    // Map entropy to meter % and label
    const safeBits = Math.min(bits, 128);
    const pct = Math.round((safeBits/128)*100);
    meterBar.style.width = pct + '%';

    let label = 'Very weak';
    if(bits >= 80) label = 'Excellent';
    else if(bits >= 60) label = 'Strong';
    else if(bits >= 40) label = 'Fair';
    else if(bits >= 28) label = 'Weak';
    strengthText.textContent = 'Strength: ' + label;
  }

  generateBtn.addEventListener('click', () => {
    const pw = generatePassword();
    updateUI(pw);
  });

  regenBtn.addEventListener('click', () => {
    const pw = generatePassword();
    updateUI(pw);
  });

  shuffleBtn.addEventListener('click', () => {
    if(!lastPassword){ alert('No password yet. Click Generate first.'); return; }
    // Shuffle lastPassword characters
    let arr = Array.from(lastPassword);
    for(let i=arr.length-1;i>0;i--){
      const j = secureRandomInt(i+1);
      [arr[i], arr[j]] = [arr[j], arr[i]];
    }
    lastPassword = arr.join('');
    updateUI(lastPassword);
  });

  copyBtn.addEventListener('click', async () => {
    if(!lastPassword){ alert('No password yet. Click Generate first.'); return; }
    try {
      await navigator.clipboard.writeText(lastPassword);
      copyBtn.textContent = 'Copied!';
      setTimeout(()=> copyBtn.textContent = 'Copy', 1200);
    } catch(e){
      alert('Could not copy to clipboard: ' + e);
    }
  });

  tipsBtn.addEventListener('click', () => {
    rulesBlock.style.display = rulesBlock.style.display === 'none' ? 'block' : 'none';
  });

  // Initialize UI
  updateUI('');
})();
</script>
</body>
</html>
