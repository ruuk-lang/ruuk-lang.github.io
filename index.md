---
layout: default
title: Ruuk
---

<div class="hero">
  <div class="eyebrow">a language designed around how humans reason</div>
  <h1>Close enough to read.<br>Precise enough to<br><em>enforce.</em></h1>
  <p>Ruuk is a statically typed functional language whose syntax is built from the structural vocabulary domain experts and developers already share — prepositions for roles, named concepts for outcomes, transitions for state. The gap between what a domain means and what the code says is smaller than in any mainstream language.</p>
  <p class="hero-sub">Code this close to the domain is faster to review, harder to misread, and — because LLMs are language models — better for AI collaboration too. The same design that makes ruuk readable makes it precise enough for the compiler to enforce exhaustive outcome handling, valid state transitions, and complete field accounting.</p>
</div>

<div class="problem-section">
  <p>AI generates implementation quickly. What it can't generate is the design intent behind it — which outcomes matter, which states are valid, which fields should never cross a trust boundary. In most languages, that intent lives in comments or nobody's memory. Ruuk makes it structural.</p>
</div>

<div class="features-showcase">
  <div class="showcases-label">novel features — built to compose</div>

  <div class="code-block">
    <div class="tab-bar">
      <button class="tab-btn active" data-tab="tab-ops">operations + outcomes</button>
      <button class="tab-btn" data-tab="tab-proj">projections</button>
      <button class="tab-btn" data-tab="tab-state">state machines</button>
      <button class="tab-btn" data-tab="tab-todo">todo</button>
    </div>

    <div class="tab-pane active" id="tab-ops"><pre><span class="cm">-- the contract: declare all failure modes up front</span>
<span class="kw">op</span> <span class="fn">CreateUser</span>
    <span class="kw">payload</span> req  : <span class="ty">CreateUserRequest</span>
    <span class="kw">to</span>      store: <span class="ty">UserStore</span>
    <span class="kw">outcomes</span>
        | <span class="ty">Created</span>          <span class="kw">of</span> <span class="ty">User</span>
        | <span class="ty">AlreadyExists</span>
        | <span class="ty">ValidationFailed</span> <span class="kw">of</span> <span class="ty">List</span>[<span class="ty">ValidationError</span>]

<span class="cm">-- the implementation: a plain function returning declared outcomes</span>
<span class="kw">let</span> <span class="fn">createUser</span> req store =
    <span class="kw">match</span> <span class="fn">validate</span> req <span class="kw">with</span>
    | <span class="ty">Err</span> errors -> <span class="ty">ValidationFailed</span> errors
    | <span class="ty">Ok</span> _      ->
        <span class="kw">match</span> <span class="fn">lookup</span> req.email <span class="kw">from</span> store <span class="kw">with</span>
        | <span class="ty">Some</span> _ -> <span class="ty">AlreadyExists</span>
        | <span class="ty">None</span>   -> <span class="ty">Created</span> (<span class="fn">insert</span> req <span class="kw">to</span> store)

<span class="cm">-- every call site must handle every outcome — the compiler enforces it</span>
<span class="fn">createUser</span> req <span class="kw">to</span> store
|> <span class="kw">on</span> <span class="ty">Created</span> user            -> <span class="fn">notifyWelcome</span> user
|> <span class="kw">on</span> <span class="ty">AlreadyExists</span>           -> <span class="ty">Err</span> <span class="ty">Conflict</span>
|> <span class="kw">on</span> <span class="ty">ValidationFailed</span> errors -> <span class="ty">Err</span> (<span class="ty">BadRequest</span> errors)</pre></div>

    <div class="tab-pane" id="tab-proj"><pre><span class="cm">-- derive all API shapes from one canonical type</span>
<span class="kw">type</span> <span class="ty">User</span> = {
    id           : <span class="ty">Guid</span>
    name         : <span class="ty">String</span>
    email        : <span class="ty">String</span>
    passwordHash : <span class="ty">String</span>
    role         : <span class="ty">Role</span>
}

<span class="kw">type</span> <span class="ty">UserProfile</span>       = <span class="ty">User</span> <span class="kw">without</span> { passwordHash }
<span class="kw">type</span> <span class="ty">CreateUserRequest</span> = <span class="ty">User</span> <span class="kw">without</span> { id }
<span class="kw">type</span> <span class="ty">UserSummary</span>       = <span class="ty">User</span> <span class="kw">only</span>    { id; name }

<span class="cm">-- field accounting: the compiler verifies every field</span>
<span class="cm">-- is kept, dropped, or transformed — nothing falls through</span>
<span class="kw">transform</span> <span class="fn">toProfile</span> (u: <span class="ty">User</span>) : <span class="ty">UserProfile</span> =
    { name = u.name; email = u.email; role = u.role }</pre></div>

    <div class="tab-pane" id="tab-state"><pre><span class="kw">resource</span> <span class="ty">Connection</span>&lt;<span class="ty">Idle</span> | <span class="ty">InTransaction</span> | <span class="ty">Closed</span>&gt;

<span class="kw">op</span> <span class="fn">beginTransaction</span>
    <span class="kw">subject</span> conn: <span class="ty">Connection</span>&lt;<span class="ty">Idle</span>&gt;
    <span class="kw">performs</span> <span class="ty">Connection</span>.<span class="ty">Idle</span> -> <span class="ty">Connection</span>.<span class="ty">InTransaction</span>
    <span class="kw">outcomes</span>
        | <span class="ty">Begun</span>   <span class="kw">of</span> <span class="ty">Connection</span>&lt;<span class="ty">InTransaction</span>&gt;
        | <span class="ty">Timeout</span>

<span class="kw">let</span> <span class="fn">beginTransaction</span> conn =
    <span class="kw">match</span> <span class="fn">tryBegin</span> conn <span class="kw">with</span>
    | <span class="ty">Ok</span> conn -> <span class="ty">Begun</span> conn
    | <span class="ty">Err</span> _   -> <span class="ty">Timeout</span>

<span class="kw">op</span> <span class="fn">commit</span>
    <span class="kw">subject</span> conn: <span class="ty">Connection</span>&lt;<span class="ty">InTransaction</span>&gt;
    <span class="kw">performs</span> <span class="ty">Connection</span>.<span class="ty">InTransaction</span> -> <span class="ty">Connection</span>.<span class="ty">Idle</span>
    <span class="kw">outcomes</span>
        | <span class="ty">Committed</span>  <span class="kw">of</span> <span class="ty">Connection</span>&lt;<span class="ty">Idle</span>&gt;
        | <span class="ty">Failed</span>     <span class="kw">of</span> <span class="ty">DbError</span>

<span class="kw">let</span> <span class="fn">commit</span> conn =
    <span class="kw">match</span> <span class="fn">tryCommit</span> conn <span class="kw">with</span>
    | <span class="ty">Ok</span> conn -> <span class="ty">Committed</span> conn
    | <span class="ty">Err</span> e   -> <span class="ty">Failed</span> e

<span class="cm">-- compiler catches: querying a closed connection,</span>
<span class="cm">-- committing outside a transaction, orphan states</span></pre></div>

    <div class="tab-pane" id="tab-todo"><pre><span class="cm">-- todo is a keyword — the compiler tracks it, unlike a comment</span>
<span class="kw">match</span> notification <span class="kw">with</span>
| <span class="ty">Email</span>   -> <span class="fn">sendEmail</span> notification
| <span class="ty">SMS</span>     -> <span class="fn">sendSMS</span> notification
| <span class="ty">Push</span>    -> <span class="kw">todo</span>
| <span class="ty">Webhook</span> -> <span class="kw">todo</span>
<span class="cm">-- warning: todo in Push arm</span>
<span class="cm">-- warning: todo in Webhook arm</span>

<span class="cm">-- works in outcome pipelines too</span>
<span class="fn">createUser</span> req <span class="kw">to</span> store
|> <span class="kw">on</span> <span class="ty">Created</span> user            -> <span class="fn">notifyWelcome</span> user
|> <span class="kw">on</span> <span class="ty">AlreadyExists</span>           -> <span class="kw">todo</span>
|> <span class="kw">on</span> <span class="ty">ValidationFailed</span> errors -> <span class="ty">Err</span> (<span class="ty">BadRequest</span> errors)
<span class="cm">-- warning: AlreadyExists handler is todo</span>
<span class="cm">-- the program compiles — but nothing slips through silently</span></pre></div>
  </div>
</div>

<div class="feature-sections">

  <div class="feature-section">
    <div class="feature-section-eyebrow">design principle</div>
    <h2>Code that reads like the domain it models</h2>
    <p>Ruuk borrows directly from the structural vocabulary domain experts and developers already use. Prepositions express roles — <em>transferFunds from account to reserve via gateway</em>. Named concepts express outcomes — <em>InsuranceDenied</em>, <em>ContraindicationDetected</em> — not error codes. State transitions read like the protocol descriptions domain experts already write. These aren't new conventions to learn; they're the ones already in use.</p>
    <p>This makes cold reads faster: the reviewer handed AI-generated code, the fifth developer to maintain a system, the architect verifying a design — all spend less time reconstructing intent because less intent was lost in translation.</p>
    <div class="agent-callout">
      <strong>For AI agents:</strong> richer semantic metadata at every declaration site. Parameter roles give agents structured information about what each argument <em>does</em> — not just its type but its function in the operation. Agents generate code that fits the semantic frame rather than inferring roles from variable names.
    </div>
  </div>

  <div class="feature-section">
    <div class="feature-section-eyebrow">outcomes</div>
    <h2>Every failure mode named. Every case enforced.</h2>
    <p>Operations declare all their outcomes — not a binary Ok/Err, but every domain-meaningful result: <em>AlreadyExists</em>, <em>InsufficientInventory</em>, <em>InsuranceDenied</em>. The compiler rejects any call site that leaves one unhandled. The design decision — which outcomes exist and what they mean — is syntactic, not buried in implementation or held in someone's memory.</p>
    <p>This is the difference between a system that handles the cases someone thought of on the day they wrote it, and a system where the compiler prevents the gap from existing at all.</p>
    <div class="agent-callout">
      <strong>For AI agents:</strong> conventional languages require running integration tests to discover a missing error path — seconds of latency, noisy output to parse. Ruuk's compiler rejects missing outcome handlers in milliseconds with a precise diagnostic: <em>"outcome AlreadyExists unhandled at line 42."</em> The agent's search space is pruned before any code runs.
    </div>
  </div>

  <div class="feature-section">
    <div class="feature-section-eyebrow">developer · compiler · AI · domain expert</div>
    <h2>The contract layer anyone can read</h2>
    <p>Ruuk separates declaration from implementation. The <em>op</em> declaration names every outcome. The <em>resource</em> declaration names every valid state. Type projections name which fields cross which trust boundary. These encode design intent — and they are written to be readable without implementation knowledge.</p>
    <p>A clinician can review an <em>op Prescribe</em> declaration and confirm it captures the protocol. A compliance officer can read outcome sets and verify regulatory coverage. An architect can review the full interface layer without reading a single function body. The compiler then enforces that whatever AI generates must honor those declarations.</p>
  </div>

  <div class="feature-section">
    <div class="feature-section-eyebrow">for AI agents</div>
    <h2>Declared invariants, not discovered ones</h2>
    <p>In a conventional language, an agent discovers invariants by running code and observing failures. It learns that <em>createUser</em> can fail with <em>AlreadyExists</em> by hitting that case in a test. The feedback loop is: generate → test (seconds to minutes) → parse noisy output → infer the constraint → fix.</p>
    <p>In ruuk, invariants are declared — in operation outcome sets, state machine transitions, projection rules — and the compiler enforces them statically. The agent doesn't need to discover that <em>AlreadyExists</em> is a possible outcome; it's in the declaration. The feedback loop is: generate → compile (milliseconds) → structured diagnostic with exact location and constraint → fix.</p>
    <p>This is what a better agent environment looks like: not a smarter model, but a tighter feedback loop that makes any model converge faster toward structurally correct code.</p>
  </div>

</div>

<div class="status">
  <p>Ruuk is under active development and not yet available for general use. The posts below track progress and explore the ideas behind the language.</p>
</div>

<div class="post-list">
  <div class="post-list-heading">posts</div>
  {% for post in site.posts %}
  <a href="{{ post.url }}" style="text-decoration:none">
    <div class="post-card">
      <h3>{{ post.title }}</h3>
      {% if post.excerpt %}<p>{{ post.excerpt | strip_html | truncatewords: 30 }}</p>{% endif %}
      <div class="post-meta">{{ post.date | date: "%B %d, %Y" }}</div>
    </div>
  </a>
  {% endfor %}
</div>

<script>
  document.querySelectorAll('.tab-btn').forEach(function(btn) {
    btn.addEventListener('click', function() {
      var target = btn.getAttribute('data-tab');
      document.querySelectorAll('.tab-btn').forEach(function(b) { b.classList.remove('active'); });
      document.querySelectorAll('.tab-pane').forEach(function(p) { p.classList.remove('active'); });
      btn.classList.add('active');
      document.getElementById(target).classList.add('active');
    });
  });
</script>
