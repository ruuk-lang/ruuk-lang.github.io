---
layout: default
title: Ruuk
---

<div class="hero">
  <div class="eyebrow">a language designed around how humans reason</div>
  <h1>Close enough to read.<br>Precise enough to<br><em>enforce.</em></h1>
  <p>Ruuk is a statically typed functional language whose syntax is grounded in cognitive linguistics — the structures humans naturally use to reason about events, roles, and outcomes. Parameter roles are prepositions. Outcomes are named domain concepts. State transitions read like state diagrams.</p>
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

<div class="features">
  <div class="feature">
    <div class="feature-label">design principle</div>
    <h3>Code that reads like the domain it models</h3>
    <p>Ruuk's syntax uses structures the brain already has. Prepositions name what each argument does. Outcomes are domain vocabulary, not error codes. The cold read is faster because the gap between the domain model and the code is smaller — for humans and language models alike.</p>
  </div>
  <div class="feature">
    <div class="feature-label">outcomes</div>
    <h3>Named failure modes, not exceptions</h3>
    <p>Operations declare every domain-meaningful result — not just success and error, but <em>AlreadyExists</em>, <em>InsufficientInventory</em>, <em>CarrierUnavailable</em>. The compiler rejects any call site that leaves one unhandled.</p>
  </div>
  <div class="feature">
    <div class="feature-label">developer · compiler · AI · domain expert</div>
    <h3>Four parties, one shared artifact</h3>
    <p>You define the structure. AI fills the implementation. The compiler verifies the contract. And the clinician, compliance officer, or analyst can read the outcome declarations and confirm they match the spec — without reading implementation code.</p>
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
