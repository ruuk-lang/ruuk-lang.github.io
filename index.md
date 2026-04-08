---
layout: default
title: Ruuk
---

<div class="hero">
  <div class="eyebrow">a language for human minds</div>
  <h1>Code designed around<br><em>cognitive science</em>,<br>not syntax aesthetics.</h1>
  <p>Ruuk is a statically typed, functional language built on the premise that the right starting point for language design is not syntax or type theory — but how human minds actually encode and process meaning.</p>
</div>

<div class="code-sample">
<pre><span class="cm">-- outcome types enumerate all failure modes explicitly</span>
<span class="kw">let</span> <span class="fn">parse_date</span> : <span class="ty">String</span> -> <span class="ty">Result</span>[<span class="ty">Date</span>, <span class="ty">ParseError</span>]
<span class="kw">let</span> <span class="fn">parse_date</span> input =
  <span class="kw">match</span> <span class="fn">tokenize</span> input <span class="kw">with</span>
  | <span class="ty">Ok</span> tokens -> <span class="fn">build_date</span> tokens
  | <span class="ty">Err</span> (<span class="ty">InvalidFormat</span> msg) -> <span class="ty">Err</span> (<span class="ty">BadInput</span> msg)
  | <span class="ty">Err</span> (<span class="ty">OutOfRange</span> _)   -> <span class="ty">Err</span> <span class="ty">DateOutOfRange</span></pre>
</div>

<div class="features">
  <div class="feature">
    <div class="feature-label">design principle</div>
    <h3>Cognitive linguistics first</h3>
    <p>Features derived from how humans naturally structure experience — not invented from type theory.</p>
  </div>
  <div class="feature">
    <div class="feature-label">type system</div>
    <h3>HM inference + exhaustive matching</h3>
    <p>ML/F# tradition. Statically typed, expression-oriented, immutable by default.</p>
  </div>
  <div class="feature">
    <div class="feature-label">for the cold read</div>
    <h3>Intent is explicit</h3>
    <p>Parameters name their role. Operations enumerate outcomes. State is declared, not implied.</p>
  </div>
</div>

<div class="post-list">
  <div class="post-list-heading">recent posts</div>
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