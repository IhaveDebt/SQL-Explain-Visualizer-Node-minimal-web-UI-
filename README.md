#!/usr/bin/env node
/**
 * SQL Explain Visualizer
 *
 * Small Node server that accepts a JSON EXPLAIN plan (Postgres-style JSON),
 * walks the plan tree, generates human-friendly advice and a minimal HTML
 * visualization (tree) served at /.
 *
 * Usage:
 *   1) npm init -y
 *   2) npm install express body-parser
 *   3) node src/server.js
 *
 * POST a JSON plan to /plan with `{"plan": {...}}` or use the UI.
 *
 * This file is a single-file prototype: server + inline HTML UI.
 */

const express = require("express");
const bodyParser = require("body-parser");

const app = express();
app.use(bodyParser.json({ limit: "2mb" }));

function adviceFromPlan(plan) {
  const adv = [];
  function walk(node) {
    if (!node || typeof node !== "object") return;
    const type = node["Node Type"] || node["nodeType"] || "";
    if (/Seq Scan/i.test(type)) {
      adv.push(`Sequential scan on ${node["Relation Name"] || node["relation"] || "table"} — consider an index for filters: ${node["Filter"] || node["filter"] || "<unknown>"}`);
    }
    if (/Index Scan/i.test(type) || /Index Only Scan/i.test(type)) {
      adv.push(`Index scan on ${node["Relation Name"] || node["relation"] || "<table>"} using ${node["Index Name"] || node["indexName"] || "<index>"} — usually good.`);
    }
    if (node["Sort Key"] || node["Sort Keys"]) {
      adv.push(`Sort step present (sort keys: ${JSON.stringify(node["Sort Key"] || node["Sort Keys"])}) — consider ordering indexes or pushdown.`);
    }
    if (node["Hash Join"] || /Hash Join/i.test(type)) {
      adv.push("Hash Join detected — good for large unsorted inputs but check memory/hash size.");
    }
    (node["Plans"] || node["plans"] || []).forEach(walk);
  }
  walk(plan);
  // remove duplicates and return
  return [...new Set(adv)];
}

function planToTreeHtml(plan) {
  function toNode(n) {
    if (!n) return "";
    const type = n["Node Type"] || n["nodeType"] || "Node";
    const cost = n["Total Cost"] || n["Cost"] || n["cost"] || "";
    const label = `${type}${cost ? " (cost: " + cost + ")" : ""}`;
    const children = (n["Plans"] || n["plans"] || []);
    let html = `<li><details open><summary>${escapeHtml(label)}</summary><div class="meta">`;
    // show some interesting keys
    ["Relation Name","Index Name","Filter","Sort Key","Actual Rows","Actual Time"].forEach(k => {
      if (n[k]) html += `<div><strong>${escapeHtml(k)}:</strong> ${escapeHtml(String(n[k]))}</div>`;
    });
    html += `</div>`;
    if (children && children.length) {
      html += "<ul>";
      for (const c of children) html += toNode(c);
      html += "</ul>";
    }
    html += "</details></li>";
    return html;
  }
  return `<ul class="plan-root">${toNode(plan)}</ul>`;
}

function escapeHtml(s) {
  return (s || "").toString().replace(/[&<>"']/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[c]));
}

app.get("/", (req, res) => {
  res.type("html").send(`<!doctype html>
<html>
<head>
<meta charset="utf-8">
<title>SQL Explain Visualizer</title>
<style>
body{font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial; margin:20px}
.container{display:flex;gap:16px}
.panel{flex:1;border:1px solid #ddd;padding:12px;border-radius:6px}
textarea{width:100%;height:280px;font-family:monospace}
.plan-root{list-style:none;padding-left:0}
.meta{font-size:12px;color:#333;margin:6px 0}
.summary{background:#f6f8fa;padding:8px;border-radius:6px}
</style>
</head>
<body>
<h1>SQL Explain Visualizer</h1>
<div class="container">
  <div class="panel">
    <h3>Paste EXPLAIN (ANALYZE, FORMAT JSON) output (JSON)</h3>
    <textarea id="planIn" placeholder='{"Plan": {...}}'></textarea>
    <div style="margin-top:8px"><button id="btn">Visualize</button> <button id="ex">Example</button></div>
  </div>
  <div class="panel">
    <h3>Plan visualization</h3>
    <div id="vis" class="summary">No plan yet.</div>
    <h4>Advice</h4>
    <div id="advice" class="summary"></div>
  </div>
</div>
<script>
document.getElementById('btn').addEventListener('click', async () => {
  try {
    const plan = JSON.parse(document.getElementById('planIn').value);
    const r = await fetch('/plan',{method:'POST',headers:{'content-type':'application/json'},body: JSON.stringify(plan)});
    const j = await r.json();
    document.getElementById('vis').innerHTML = j.html;
    document.getElementById('advice').innerText = j.advice.join('\\n') || "(no advice)";
  } catch(e){
    alert('Invalid JSON: '+e.message);
  }
});
document.getElementById('ex').addEventListener('click', () => {
  const ex = {"Plan":{"Node Type":"Aggregate","Plans":[{"Node Type":"Seq Scan","Relation Name":"orders","Filter":"status = 'open'","Total Cost":1200},{"Node Type":"Index Scan","Relation Name":"users","Index Name":"users_pkey","Total Cost":10}],"Total Cost":1210}};
  document.getElementById('planIn').value = JSON.stringify(ex,null,2);
});
</script>
</body>
</html>`);
});

app.post("/plan", (req, res) => {
  const payload = req.body;
  const plan = payload.Plan || payload.plan || payload;
  const advice = adviceFromPlan(plan);
  const html = planToTreeHtml(plan);
  res.json({ advice, html });
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log("SQL Explain Visualizer listening on http://localhost:" + PORT));
