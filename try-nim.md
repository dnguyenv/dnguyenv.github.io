---
layout: page
title: "Try NIM"
permalink: /try-nim/
description: "A no-install, no-proxy way to try NVIDIA's NIM API. Paste your key, pick a model, type a prompt, copy the command, run it."
---

<p style="color: var(--fg-muted); margin-bottom: 1.5rem;">
A small companion to <a href="/2026/05/the-bill-that-quietly-became-optional/">this post</a>.
Paste your <a href="https://build.nvidia.com/settings/api-keys" target="_blank" rel="noopener">NVIDIA NIM API key</a>,
pick a model, type a prompt, and copy the command into your terminal. Your key never leaves
your browser. This page does not make API calls; it generates a request you run locally,
so the only host that ever sees your key is NVIDIA.
</p>

<div id="trynim-app" class="trynim">

  <div class="trynim-row">
    <label for="trynim-key" class="trynim-label">NIM API key</label>
    <div class="trynim-key-wrap">
      <input type="password" id="trynim-key" autocomplete="off" spellcheck="false"
             placeholder="nvapi-..." class="trynim-input" />
      <button type="button" id="trynim-show" class="trynim-btn-mini" aria-label="Show or hide key">show</button>
      <button type="button" id="trynim-clear" class="trynim-btn-mini" aria-label="Clear stored key">clear</button>
    </div>
    <p class="trynim-hint" id="trynim-key-status">Stored only in this browser's localStorage on this domain.</p>
  </div>

  <div class="trynim-row">
    <label for="trynim-model" class="trynim-label">Model</label>
    <select id="trynim-model" class="trynim-input">
      <option value="nvidia/nemotron-3-super-120b-a12b">nvidia / nemotron-3-super-120b-a12b, agentic flagship, 1M context</option>
      <option value="openai/gpt-oss-120b">openai / gpt-oss-120b, open-weights reasoning</option>
      <option value="deepseek-ai/deepseek-v4-pro">deepseek-ai / deepseek-v4-pro, coding, 1M context</option>
      <option value="moonshotai/kimi-k2-thinking">moonshotai / kimi-k2-thinking, reasoning</option>
      <option value="minimaxai/minimax-m2.7">minimaxai / minimax-m2.7</option>
      <option value="z-ai/glm5.1">z-ai / glm5.1</option>
      <option value="qwen/qwen3-coder-480b-a35b-instruct">qwen / qwen3-coder-480b-a35b-instruct, code flagship</option>
      <option value="meta/llama-3.3-70b-instruct">meta / llama-3.3-70b-instruct</option>
    </select>
  </div>

  <div class="trynim-row">
    <label for="trynim-prompt" class="trynim-label">Prompt</label>
    <textarea id="trynim-prompt" class="trynim-input trynim-textarea"
              placeholder="Explain hybrid Mamba-Transformer in one paragraph."
              rows="4" spellcheck="false"></textarea>
  </div>

  <div class="trynim-row trynim-toolbar">
    <div class="trynim-tabs" role="tablist" aria-label="Output format">
      <button type="button" class="trynim-tab is-active" data-mode="curl" role="tab" aria-selected="true">curl</button>
      <button type="button" class="trynim-tab" data-mode="python" role="tab" aria-selected="false">Python</button>
    </div>
    <button type="button" id="trynim-copy" class="trynim-btn">Copy command</button>
  </div>

  <pre class="trynim-output"><code id="trynim-out"></code></pre>

  <p class="trynim-hint">
    Run the command in any terminal. The first time, you may need
    <code>pip install openai</code> for the Python version. The curl version
    needs nothing beyond curl, which ships with macOS, Linux, and recent Windows.
  </p>

</div>

<style>
.trynim {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
  background: var(--bg-soft);
  border: 1px solid var(--border);
  border-radius: 6px;
  padding: 1.5rem;
  margin: 1.5rem 0;
}
.trynim-row { display: flex; flex-direction: column; gap: 0.4rem; }
.trynim-label {
  font-family: var(--mono);
  font-size: 0.8rem;
  color: var(--fg-muted);
  letter-spacing: 0.04em;
  text-transform: uppercase;
}
.trynim-input {
  width: 100%;
  background: var(--bg);
  border: 1px solid var(--border-mid);
  border-radius: 4px;
  color: var(--fg);
  font-family: var(--mono);
  font-size: 0.9rem;
  padding: 0.6rem 0.75rem;
  outline: none;
}
.trynim-input:focus { border-color: var(--fg-muted); }
.trynim-textarea { resize: vertical; min-height: 5.5rem; line-height: 1.55; }
.trynim-key-wrap { display: flex; gap: 0.4rem; align-items: stretch; }
.trynim-key-wrap .trynim-input { flex: 1; }
.trynim-btn-mini {
  background: transparent;
  border: 1px solid var(--border-mid);
  color: var(--fg-muted);
  font-family: var(--mono);
  font-size: 0.75rem;
  padding: 0 0.7rem;
  border-radius: 4px;
  cursor: pointer;
  letter-spacing: 0.03em;
}
.trynim-btn-mini:hover { color: var(--fg); border-color: var(--fg-muted); }
.trynim-btn {
  background: var(--fg);
  color: var(--bg);
  border: none;
  font-family: var(--mono);
  font-size: 0.85rem;
  padding: 0.55rem 1rem;
  border-radius: 4px;
  cursor: pointer;
  letter-spacing: 0.03em;
  font-weight: 500;
}
.trynim-btn:hover { opacity: 0.85; }
.trynim-btn:disabled { opacity: 0.5; cursor: not-allowed; }
.trynim-toolbar {
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
}
.trynim-tabs { display: flex; gap: 0.25rem; }
.trynim-tab {
  background: transparent;
  border: 1px solid var(--border-mid);
  color: var(--fg-muted);
  font-family: var(--mono);
  font-size: 0.8rem;
  padding: 0.4rem 0.85rem;
  border-radius: 4px;
  cursor: pointer;
}
.trynim-tab.is-active { color: var(--bg); background: var(--fg); border-color: var(--fg); }
.trynim-output {
  background: var(--code-bg);
  border: 1px solid var(--border);
  border-radius: 4px;
  padding: 1rem;
  margin: 0;
  overflow-x: auto;
  font-family: var(--mono);
  font-size: 0.85rem;
  line-height: 1.55;
  white-space: pre;
}
.trynim-output code { background: transparent; padding: 0; color: var(--code-fg); }
.trynim-hint {
  font-size: 0.8rem;
  color: var(--fg-subtle);
  margin: 0;
  line-height: 1.55;
}
.trynim-hint code {
  background: var(--code-bg);
  padding: 0.05rem 0.35rem;
  border-radius: 3px;
  font-size: 0.8em;
}
</style>

<script>
// SECURITY-REVIEW: This script handles a user-supplied NVIDIA API key.
// Invariants:
//   1. The key lives only in window.localStorage on this origin.
//   2. The page makes zero network calls. No fetch, no XHR, no img/script src.
//      The key is rendered into the displayed command text (textContent only)
//      so the user can copy it. It never touches a remote host from this page.
//   3. The user's prompt is JSON-encoded via JSON.stringify before being placed
//      inside the rendered curl heredoc, so shell metacharacters (backticks,
//      $VAR, quotes, newlines) cannot break out of the JSON body.
//   4. The rendered command is written to the DOM via .textContent only.
//      Never .innerHTML.
(function () {
  'use strict';

  var STORAGE_KEY = 'trynim.apiKey.v1';
  var BASE_URL = 'https://integrate.api.nvidia.com/v1';

  var keyInput = document.getElementById('trynim-key');
  var keyStatus = document.getElementById('trynim-key-status');
  var showBtn = document.getElementById('trynim-show');
  var clearBtn = document.getElementById('trynim-clear');
  var modelSel = document.getElementById('trynim-model');
  var promptEl = document.getElementById('trynim-prompt');
  var outEl = document.getElementById('trynim-out');
  var copyBtn = document.getElementById('trynim-copy');
  var tabs = document.querySelectorAll('.trynim-tab');

  var mode = 'curl';

  function loadKey() {
    try {
      var v = window.localStorage.getItem(STORAGE_KEY);
      if (v) {
        keyInput.value = v;
        setStatus('Loaded saved key from this browser. Click clear to remove.');
      }
    } catch (e) {
      setStatus('localStorage unavailable; key will not persist between visits.');
    }
  }

  function saveKey(v) {
    try {
      if (v) {
        window.localStorage.setItem(STORAGE_KEY, v);
      } else {
        window.localStorage.removeItem(STORAGE_KEY);
      }
    } catch (e) { /* ignore quota/private mode */ }
  }

  function setStatus(msg) {
    keyStatus.textContent = msg;
  }

  function buildBody(model, prompt) {
    return {
      model: model,
      messages: [{ role: 'user', content: prompt }]
    };
  }

  function curlCommand(key, model, prompt) {
    var keyDisplay = key && key.length > 0 ? key : 'YOUR_NVIDIA_API_KEY';
    var body = JSON.stringify(buildBody(model, prompt), null, 2);
    return [
      'curl -s ' + BASE_URL + '/chat/completions \\',
      '  -H "Authorization: Bearer ' + keyDisplay + '" \\',
      '  -H "Content-Type: application/json" \\',
      "  -d @- <<'NIMJSON'",
      body,
      'NIMJSON'
    ].join('\n');
  }

  function pythonSnippet(key, model, prompt) {
    var keyLine = key && key.length > 0
      ? 'api_key=' + JSON.stringify(key) + ','
      : 'api_key=os.environ["NVIDIA_API_KEY"],';
    var promptLit = JSON.stringify(prompt);
    var modelLit = JSON.stringify(model);
    return [
      'import os',
      'from openai import OpenAI',
      '',
      'client = OpenAI(',
      '    base_url=' + JSON.stringify(BASE_URL) + ',',
      '    ' + keyLine,
      ')',
      '',
      'resp = client.chat.completions.create(',
      '    model=' + modelLit + ',',
      '    messages=[{"role": "user", "content": ' + promptLit + '}],',
      ')',
      'print(resp.choices[0].message.content)'
    ].join('\n');
  }

  function render() {
    var key = (keyInput.value || '').trim();
    var model = modelSel.value;
    var prompt = (promptEl.value || '').trim() ||
      'Explain hybrid Mamba-Transformer in one paragraph.';
    var text = mode === 'python'
      ? pythonSnippet(key, model, prompt)
      : curlCommand(key, model, prompt);
    outEl.textContent = text;
  }

  function setMode(next) {
    mode = next;
    tabs.forEach(function (t) {
      var active = t.getAttribute('data-mode') === next;
      t.classList.toggle('is-active', active);
      t.setAttribute('aria-selected', active ? 'true' : 'false');
    });
    render();
  }

  keyInput.addEventListener('input', function () {
    saveKey((keyInput.value || '').trim());
    render();
  });

  showBtn.addEventListener('click', function () {
    var hidden = keyInput.type === 'password';
    keyInput.type = hidden ? 'text' : 'password';
    showBtn.textContent = hidden ? 'hide' : 'show';
  });

  clearBtn.addEventListener('click', function () {
    keyInput.value = '';
    saveKey('');
    setStatus('Cleared from this browser.');
    render();
  });

  modelSel.addEventListener('change', render);
  promptEl.addEventListener('input', render);

  tabs.forEach(function (t) {
    t.addEventListener('click', function () { setMode(t.getAttribute('data-mode')); });
  });

  copyBtn.addEventListener('click', function () {
    var text = outEl.textContent || '';
    var done = function () {
      var orig = copyBtn.textContent;
      copyBtn.textContent = 'Copied';
      setTimeout(function () { copyBtn.textContent = orig; }, 1200);
    };
    if (navigator.clipboard && navigator.clipboard.writeText) {
      navigator.clipboard.writeText(text).then(done, function () {
        fallbackCopy(text); done();
      });
    } else {
      fallbackCopy(text); done();
    }
  });

  function fallbackCopy(text) {
    var ta = document.createElement('textarea');
    ta.value = text;
    ta.setAttribute('readonly', '');
    ta.style.position = 'absolute';
    ta.style.left = '-9999px';
    document.body.appendChild(ta);
    ta.select();
    try { document.execCommand('copy'); } catch (e) { /* noop */ }
    document.body.removeChild(ta);
  }

  loadKey();
  render();
})();
</script>
