<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:031316,55:0a3b45,100:22d3ee&height=235&section=header&text=consoleAI&fontSize=46&fontColor=ecfeff&fontAlignY=40&desc=A%20multi-model%20chat%20client%20that%20fits%20in%20a%20terminal&descSize=17&descAlignY=62&animation=twinkling" alt="consoleAI" width="100%">

<img src="https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=500&size=19&duration=3200&pause=900&color=67e8f9&center=true&vCenter=true&width=700&lines=Pure%20stdlib.%20Python%203.9%2B.%20No%20pip%20install.;Seven%20providers%2C%20one%20prompt%2C%20live%20model%20switching;Tools%20with%20DNS%20pinning%2C%20because%20fetch_url%20can%20reach%20anything" alt="Typing animation" />

<br />

<img src="https://img.shields.io/badge/Fork%20of-olumolu%2FconsoleAI-22d3ee?style=flat-square&labelColor=031316" alt="Fork of olumolu/consoleAI" />
<img src="https://img.shields.io/badge/Licence-GPL--2.0-67e8f9?style=flat-square&labelColor=031316" alt="GPL-2.0" />
<img src="https://img.shields.io/badge/Dependencies-zero-0891b2?style=flat-square&labelColor=031316" alt="Zero dependencies" />
<img src="https://img.shields.io/badge/Python-3.9%2B-06b6d4?style=flat-square&labelColor=031316" alt="Python 3.9+" />
<img src="https://img.shields.io/badge/Lines-5%2C154-22d3ee?style=flat-square&labelColor=031316" alt="5154 lines" />

</div>

<br />

<div align="center">

<sub>A <b>fork</b>. The client was written upstream by <a href="https://github.com/olumolu/consoleAI">olumolu</a>; this copy is kept current and carries its own additions.</sub>

</div>

---

<div align="center">

## What it is

</div>

One Python file, no dependencies, and a chat interface in your terminal that can talk to **Gemini, OpenRouter, Groq, Together, Cerebras, Novita and Ollama**. You pick a provider on the command line, filter the model list, and start typing. Swap models mid-conversation without losing history.

The constraint — **standard library only** — is what makes it interesting. No <code>requests</code>, no <code>rich</code>, no <code>tiktoken</code>. Everything is <code>urllib</code>, ANSI escapes and hand-written parsers: Markdown rendering, LaTeX-to-Unicode, HTML-to-text extraction, a search client, and an SSRF-safe fetcher.

<div align="center">

<sub><b>5,154 lines</b> total: <b>3,716</b> in <code>ai.py</code> and <b>1,438</b> in <code>ai.sh</code>, the bash original kept for environments where Python is not an option.</sub>

</div>

---

<div align="center">

## The loop

</div>

<ol>
<li><b>Start with a provider.</b> <code>python ai.py gemini</code> — the model list is fetched live and filtered by any argument you pass.</li>
<li><b>Talk.</b> Responses stream token by token, colour-coded: your input, the answer, reasoning, tool activity, errors. History is kept to a configurable window.</li>
<li><b>Attach, switch, save.</b> <code>/upload</code> an image for a vision model, <code>/model</code> to change model mid-thread, <code>/save</code> and <code>/load</code> to keep a session in <code>~/.chat_sessions/</code>.</li>
<li><b>Let it use tools.</b> When tools are on, the model can call six of them, up to <b>10 iterations</b> per turn, each with a <b>60 s</b> execution timeout.</li>
</ol>

<div align="center">

## What is inside

</div>

<ul>
<li><b>Seven providers, one interface</b> — each with its own endpoint, token accounting and streaming format, behind one REPL.</li>
<li><b>Markdown in a terminal</b> — bold, italics, inline code, fenced blocks, with ANSI applied after parsing so escape codes never break a code block.</li>
<li><b>LaTeX to Unicode</b> — Greek letters, superscripts, subscripts and fractions rendered as characters (<code>α²</code>, <code>π/2</code>) instead of raw TeX.</li>
<li><b>Vision</b> — attach an image by path, inspect it, clear it; sent as inline data on models that accept it.</li>
<li><b>Thinking output</b> — reasoning tokens can be shown or hidden with <code>/togglethinking</code>, for models that emit them.</li>
<li><b>Six tools</b> — <code>get_time</code>, <code>calculator</code>, <code>web_search</code>, <code>web_research</code>, <code>fetch_url</code>, <code>wikipedia</code>; Gemini additionally gets Google Search grounding when tools are enabled.</li>
<li><b>Multi-line input</b> — trailing <code>\</code> continues a line; <code>/paste</code> takes bulk text until <code>---</code>.</li>
<li><b>Session files</b> — named JSON sessions under <code>~/.chat_sessions/</code>, loadable months later.</li>
</ul>

---

<div align="center">

## Stack

</div>

<ul>
<li><b>Python 3.9+ standard library only</b> — <code>urllib</code>, <code>json</code>, <code>socket</code>, <code>ipaddress</code>, <code>http.cookiejar</code>, <code>html.parser</code>, <code>re</code>, <code>threading</code>.</li>
<li><b>ANSI</b> — colour and layout, with a visible-length helper that strips escape sequences before measuring, so padded columns line up.</li>
<li><b><code>ai.sh</code></b> — the bash implementation, same providers, no Python.</li>
<li><b>GPL-2.0</b> — inherited with the fork.</li>
</ul>

---

<div align="center">

## Under the hood

</div>

<details>
<summary><b>A tool that fetches arbitrary URLs needs a real guard</b></summary>

<br />

<p><code>fetch_url</code> is the most dangerous thing in the file: a model can ask it to open any address. The defence is DNS-level, not string-level, because blocking <code>localhost</code> by name is trivial to bypass.</p>

<pre><code>def _resolve_and_validate(hostname):
    blocked_hosts = {"localhost", "127.0.0.1", "0.0.0.0", "::1",
                     "metadata.google.internal", "metadata.google.com"}
    infos = socket.getaddrinfo(hostname, None, socket.AF_UNSPEC, socket.SOCK_STREAM)
    for _family, _, _, _, sockaddr in infos:
        if not _is_ip_blocked(sockaddr[0]):
            return sockaddr[0]
    raise ValueError(f"All IPs for {hostname} resolve to private/reserved addresses")</code></pre>

<p>Three properties make this worth trusting. It blocks by <b>resolved IP</b>, using <code>ipaddress</code> — private, loopback, reserved, link-local, multicast, <code>169.254.*</code>, <code>0.0.0.0</code> and IPv6 site-local — so a hostname pointing at an internal address is caught regardless of what it is called. It blocks <b>cloud metadata hostnames</b> by name, because <code>metadata.google.internal</code> is the classic credential-theft target. And <b>redirects are validated again</b>:</p>

<pre><code>class _SSRFSafeRedirectHandler(urllib.request.HTTPRedirectHandler):
    def redirect_request(self, req, fp, code, msg, headers, newurl):
        ok, err = _validate_url(newurl)
        if not ok: raise urllib.error.URLError(f"Redirect blocked: {err}")
        return super().redirect_request(req, fp, code, msg, headers, newurl)</code></pre>

<p>Without that last piece, a public URL that 302s to <code>127.0.0.1</code> defeats the whole check. The opener is built once with this handler and a cookie jar, so cookies behave like a browser's across a session.</p>

</details>

<details>
<summary><b>Search without an API key</b></summary>

<br />

<p><code>web_search</code> and <code>web_research</code> hit Startpage, extract structured results, then do the part a search API would normally do:</p>

<ul>
<li><b>Unwrap</b> redirect URLs (<code>_unwrap_result_url</code>) and skip known-bad ones (<code>_should_skip_result</code>).</li>
<li><b>Score</b> each result — term overlap in title, snippet and body, plus a <b>domain quality bonus</b> — and re-rank.</li>
<li><b>Fetch the top sources in batches</b>, extract readable text with a purpose-built <code>HTMLParser</code>, and pull the best excerpts by chunking the body (1,400 characters with 200 overlap) and scoring each chunk against the query.</li>
</ul>

<p>That pipeline is why <code>web_research</code> returns an answer with citations rather than ten links and an apology: the model receives excerpts, not URLs.</p>

</details>

<details>
<summary><b>Terminal text is a parser problem</b></summary>

<br />

<p>Markdown rendering happens on raw text, then ANSI codes are applied to the result — applying colour first and parsing after would have escape sequences inside the syntax you are trying to match. Code blocks are protected from inline formatting so a <code>*</code> inside a fenced block stays a <code>*</code>.</p>

<p>The same discipline shows up in <code>_visible_len()</code>: it strips ANSI before measuring, because a padded column calculated on the raw string is wrong by the length of its own colour codes.</p>

</details>

<details>
<summary><b>Tools are a loop with a ceiling</b></summary>

<br />

<p>A model that can call tools can call them forever. Two constants stop that:</p>

<pre><code>MAX_TOOL_ITERATIONS = 10
TOOL_EXEC_TIMEOUT   = 60</code></pre>

<p>Ten rounds of call-and-observe per turn, each tool capped at a minute. Past the limit the loop ends and the model is told to answer with what it has — which is a better failure than a terminal that appears to hang.</p>

<p>Tool activity is colour-coded separately from the answer, so it is always obvious whether the text on screen is the model talking or the program reporting.</p>

</details>

---

<div align="center">

## Setup

</div>

<pre><code>git clone https://github.com/Jivaansh-Yadav/consoleAI.git
cd consoleAI

export GEMINI_API_KEY="…"        # or edit the API_KEYS dict in ai.py
python ai.py gemini              # list models
python ai.py gemini flash        # filter, then chat</code></pre>

<p>Or with no environment variables at all — fill the <code>API_KEYS</code> dictionary at the top of <code>ai.py</code>. Ollama needs no key; leave it blank and point at a local server.</p>

<div align="center">

<sub><b>Credit.</b> Upstream: <a href="https://github.com/olumolu/consoleAI">olumolu/consoleAI</a>, GPL-2.0. This fork keeps the licence and the attribution.</sub>

</div>

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:031316,55:0a3b45,100:22d3ee&height=110&section=footer&reversal=true" alt="" width="100%">
