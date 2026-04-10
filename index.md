---
layout: default
title: Ruuk
---

<div class="hero">
  <div class="eyebrow">a language for complex business domains</div>
  <h1>Every failure mode<br>named. Every case<br><em>enforced.</em></h1>
  <p>Ruuk is a statically typed functional language for teams where the cost of a missed edge case outlasts the sprint that caused it. Operations declare all their outcomes — the compiler enforces every call site handles every one. State machines enforce which operations are valid in which states.</p>
  <p class="hero-sub">Built for teams using AI: the compiler is the unconditional third party that verifies what you write, what you review, and what AI generates.</p>
</div>

<div class="features-showcase">
  <div class="showcases-label">novel features — built to compose</div>

  <div class="code-block">
    <div class="code-block-label">operations + outcomes</div>
<pre><span class="cm">-- the contract: declare all failure modes up front</span>
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

<span class="cm">-- every outcome must be handled — but todo defers without hiding</span>
<span class="fn">createUser</span> req <span class="kw">to</span> store
|> <span class="kw">on</span> <span class="ty">Created</span> user            -> <span class="fn">notifyWelcome</span> user
|> <span class="kw">on</span> <span class="ty">AlreadyExists</span>           -> <span class="kw">todo</span>
|> <span class="kw">on</span> <span class="ty">ValidationFailed</span> errors -> <span class="ty">Err</span> (<span class="ty">BadRequest</span> errors)
<span class="cm">-- warning: AlreadyExists handler is todo  (not an error — but not silent)</span></pre>
  </div>

  <div class="showcase-pair">
    <div class="code-block">
      <div class="code-block-label">projections + field accounting</div>
<pre><span class="cm">-- derive all API shapes from one canonical type</span>
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
    { name = u.name; email = u.email; role = u.role }</pre>
    </div>

    <div class="code-block">
      <div class="code-block-label">state machines + typestate</div>
<pre><span class="kw">resource</span> <span class="ty">Connection</span>&lt;<span class="ty">Idle</span> | <span class="ty">InTransaction</span> | <span class="ty">Closed</span>&gt;

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
<span class="cm">-- committing outside a transaction, orphan states</span></pre>
    </div>
  </div>
</div>

<div class="features">
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
  <div class="feature">
    <div class="feature-label">typestate</div>
    <h3>Invalid states are type errors</h3>
    <p>Resources move through explicitly declared states. <em>Order&lt;Pending&gt;</em> can be approved. <em>Order&lt;Delivered&gt;</em> cannot. Transitions are declared, compiler-verified, and exhaustively checked at the whole-program level.</p>
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
