
```dataviewjs
await (async () => {
  const PROFILE_PAGE_A = "Life Game Profile/Profile State";
  const PROFILE_PAGE_B = "Life Game Profile/Profile State.md";

  const FIN_DASH_PAGE = "Fianances/Financial Dashboard";
  const FIN_DASH_MD   = "Fianances/Financial Dashboard.md";
  const TX_PATH = "Fianances/Transaction.md";

  const GAME_LOG_PATH = "Life Game Profile/Game Log.md";
  const SKILLS = ["Strength","Cardio","Focus","Discipline","Social"];

  const P = dv.page(PROFILE_PAGE_A) ?? dv.page(PROFILE_PAGE_B);
  const FD = dv.page(FIN_DASH_PAGE) ?? {};
  if (!P) { dv.paragraph(`Missing profile: tried "${PROFILE_PAGE_A}" and "${PROFILE_PAGE_B}"`); return; }

  const num = (v, d=0) => (Number.isFinite(Number(v)) ? Number(v) : d);

  // --- colors ---
  const POS="#2dff9a", NEG="#ff6b6b", NEU="rgba(255,255,255,.92)";
  const tone = (n)=> (Number(n)||0) > 0 ? POS : (Number(n)||0) < 0 ? NEG : NEU;
  const fmtUSD = (n) => {
    const x = Number(n);
    if (!Number.isFinite(x)) return "$0";
    return "$" + Math.round(x).toLocaleString("en-US");
  };

  function badgeColor(kind, unlocked){
    if (!unlocked) return { bg:"rgba(0,0,0,0.10)", border:"#00000040", text:"#9a9a9a", glow:"none", bar:"rgba(255,255,255,0.18)" };
    if (kind === "money") return { bg:"rgba(0,0,0,0.10)", border:"#2dff9a", text:"#d9fff1", glow:"0 0 0 2px rgba(45,255,154,.14)", bar:"#2dff9a" };
    if (kind === "tasks") return { bg:"rgba(0,0,0,0.10)", border:"#5bff4d", text:"#e6ffe4", glow:"0 0 0 2px rgba(91,255,77,.14)", bar:"#5bff4d" };
    if (kind === "xp")    return { bg:"rgba(0,0,0,0.10)", border:"#ffb84d", text:"#fff1d6", glow:"0 0 0 2px rgba(255,184,77,.14)", bar:"#ffb84d" };
    if (kind === "login") return { bg:"rgba(0,0,0,0.10)", border:"#5bc0de", text:"#d9f6ff", glow:"0 0 0 2px rgba(91,192,222,.14)", bar:"#5bc0de" };
    if (kind === "skill") return { bg:"rgba(0,0,0,0.10)", border:"#c4b5fd", text:"#f1ecff", glow:"0 0 0 2px rgba(196,181,253,.14)", bar:"#c4b5fd" };
    return { bg:"rgba(0,0,0,0.10)", border:"#777", text:"#fff", glow:"none", bar:"#777" };
  }

  function levelInfo(value, base, growth){
    const v = Math.max(0, Number(value) || 0);
    const b = Math.max(1e-9, Number(base) || 1);
    const g = Math.max(1.000001, Number(growth) || 1.25);
    let lvl = 0;
    if (v >= b) {
      lvl = Math.floor(Math.log(v / b) / Math.log(g));
      if (!Number.isFinite(lvl) || lvl < 0) lvl = 0;
    }
    const curReq = b * Math.pow(g, lvl);
    const nextReq = b * Math.pow(g, lvl + 1);
    const pct = nextReq <= curReq ? 1 : (v - curReq) / (nextReq - curReq);
    return { level: lvl, curReq, nextReq, pct: Math.max(0, Math.min(1, pct)) };
  }

  async function loadTransactionsFromFile(path) {
    const f = app.vault.getAbstractFileByPath(path);
    if (!f) return [];
    const text = await app.vault.read(f);
    const out = [];
    for (const line of text.split("\n")) {
      const t = line.trim();
      if (!t) continue;
      const json = (t.startsWith("- ") ? t.slice(2) : t).trim();
      if (!json.startsWith("{")) continue;
      try { const o = JSON.parse(json); if (o && typeof o === "object") out.push(o); } catch {}
    }
    return out;
  }

  async function loadGameEvents(path){
    const f = app.vault.getAbstractFileByPath(path);
    if (!f) return [];
    const raw = await app.vault.read(f);
    const ymd = s => String(s).slice(0,10);
    const out = [];
    for (const line of raw.split("\n")) {
      const t = line.trim();
      if (!t) continue;
      const j = (t.startsWith("- ") ? t.slice(2) : t).trim();
      if (!j.startsWith("{")) continue;
      try {
        const o = JSON.parse(j);
        if (!o?.date) continue;
        out.push({ date: ymd(o.date), kind: String(o.kind||""), delta: Number(o.delta)||0, note: String(o.note||""), stat: o.stat ? String(o.stat) : null });
      } catch {}
    }
    out.sort((a,b)=>a.date<b.date?-1:a.date>b.date?1:0);
    return out;
  }

  // --- finance getters (prefer window.__FIN from summary page) ---
  async function readFileText(pathWithExt){
    const f = app.vault.getAbstractFileByPath(pathWithExt);
    if (!f) return "";
    return await app.vault.read(f);
  }
  function escapeRe(s){ return s.replace(/[.*+?^${}()|[\]\\]/g,"\\$&"); }
  function readInlineNumberFromText(text, key){
    const m = text.match(new RegExp(`^\\s*${escapeRe(key)}\\s*::\\s*([^\\n]+)\\s*$`, "mi"));
    if (!m) return null;
    const raw = String(m[1]).trim().replace(/[$,]/g,"");
    const n = Number(raw);
    return Number.isFinite(n) ? n : null;
  }
  async function getFinanceNumber(key, fallbacks = []) {
    const w = window.__FIN ? Number(window.__FIN[key]) : NaN;
    if (Number.isFinite(w)) return w;

    const dvVal = num(FD[key] ?? FD[key.toLowerCase()] ?? null, NaN);
    if (Number.isFinite(dvVal)) return dvVal;

    const txt = await readFileText(FIN_DASH_MD);
    const v0 = readInlineNumberFromText(txt, key);
    if (Number.isFinite(v0)) return v0;

    for (const k of fallbacks) {
      const v = readInlineNumberFromText(txt, k);
      if (Number.isFinite(v)) return v;
    }
    return 0;
  }

  function renderBadge({ title, subtitle, icon, kind, unlocked, level, pct }) {
    const c = badgeColor(kind, unlocked);

    const card = dv.el("div", "", {
      style: `
        background:${c.bg};
        border:1px solid ${c.border};
        border-radius:12px;
        padding:14px;
        box-shadow:${c.glow};
        image-rendering: pixelated;
        font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
      `
    });

    const top = card.createDiv();
    top.style.cssText="display:flex;align-items:center;justify-content:space-between;gap:14px;";

    const left = top.createDiv();
    left.style.cssText="display:flex;align-items:center;gap:12px;min-width:0;";

    const tile = left.createDiv();
    tile.style.cssText=`
      width:42px;height:42px;border-radius:10px;overflow:hidden;
      border:1px solid ${c.border};
      background:${unlocked ? "rgba(255,255,255,0.06)" : "rgba(255,255,255,0.03)"};
      display:flex;align-items:center;justify-content:center;
      font-size:20px;
      filter:${unlocked ? "none" : "grayscale(1) opacity(.65)"};
      flex: 0 0 auto;
    `;
    tile.textContent = icon ?? "■";

    const textBox = left.createDiv();
    textBox.style.minWidth="0";

    const t = textBox.createDiv();
    t.textContent = title;
    t.style.cssText=`font-weight:900;color:${c.text};font-size:14px;letter-spacing:.3px;`;

    const s = textBox.createDiv();
    s.textContent = subtitle ?? "";
    s.style.cssText=`margin-top:6px;font-size:12px;color:${c.text};opacity:${unlocked ? .85 : .65};`;

    const lvl = top.createDiv();
    lvl.textContent = `LV.${level}`;
    lvl.style.cssText=`
      font-size:12px;font-weight:900;
      padding:6px 10px;border-radius:999px;
      border:1px solid ${c.border};
      color:${c.text};
      opacity:${unlocked ? .95 : .55};
      flex: 0 0 auto;
    `;

    const barWrap = card.createDiv();
    barWrap.style.cssText=`
      margin-top:12px;
      height:8px;
      border-radius:999px;
      border:1px solid #00000040;
      background:rgba(0,0,0,.18);
      overflow:hidden;
      margin-bottom:18px;
    `;

    const fill = barWrap.createDiv();
    fill.style.cssText=`height:100%;width:${Math.round((pct ?? 0)*100)}%;background:${c.bar};opacity:${unlocked ? .95 : .35};`;

    return card;
  }

  // ---- profile stats ----
  const xpTotal = num(P.XP_Total, 0);
  const loginStreak = num(P.XP_Streak, 0);
  const tasksEasy = num(P.Tasks_Easy, 0);
  const tasksNormal = num(P.Tasks_Normal, 0);
  const tasksHard = num(P.Tasks_Hard, 0);
  const tasksTotal = tasksEasy + tasksNormal + tasksHard;
  const tasksScore = tasksEasy*1 + tasksNormal*3 + tasksHard*5;

  // ---- finance stats for header + badge ----
  const balance = await getFinanceNumber("Balance", ["CurrentBalance","NetWorth","Cash"]);
  const netMonth = await getFinanceNumber("NetMonth", ["NetThisMonth","Net_Month"]);
  const incomeMonth = await getFinanceNumber("IncomeMonth", ["IncomeThisMonth","Income_Month"]);
  const expenseMonth = await getFinanceNumber("ExpenseMonth", ["ExpensesThisMonth","Expense_Month"]);

  // ---- tx-derived stats ----
  const tx = await loadTransactionsFromFile(TX_PATH);
  const txCount = tx.length;
  const incomeTotal = tx.filter(t => String(t.type).toLowerCase()==="income").reduce((s,t)=>s+(Number(t.amount)||0),0);
  const expenseTotal = tx.filter(t => String(t.type).toLowerCase()==="expense").reduce((s,t)=>s+Math.abs(Number(t.amount)||0),0);
  const volumeTotal = incomeTotal + expenseTotal;
  const netCashflow = incomeTotal - expenseTotal;

  // ---- skill stats ----
  const gameEv = await loadGameEvents(GAME_LOG_PATH);
  const statEvents = gameEv.filter(e => e.kind === "stat" && (e.stat || e.note));
  const statName = e => e.stat || String(e.note||"").replace(/^STAT:/,"").trim();

  const skillTotals = Object.fromEntries(SKILLS.map(s=>[s,0]));
  for (const e of statEvents) {
    const s = statName(e);
    if (!SKILLS.includes(s)) continue;
    skillTotals[s] += Number(e.delta)||0;
  }

  const wk = d => moment(d,"YYYY-MM-DD").format("GGGG-[W]WW");
  const thisW = moment().format("GGGG-[W]WW");
  const weekTotals = Object.fromEntries(SKILLS.map(s=>[s,0]));
  for (const e of statEvents) {
    const s = statName(e);
    if (!SKILLS.includes(s)) continue;
    if (wk(e.date) === thisW) weekTotals[s] += Number(e.delta)||0;
  }

  // ---- badge levels ----
  const bLogin = levelInfo(loginStreak, 3, 1.35);
  const bTasks = levelInfo(tasksScore, 25, 1.35);
  const bXP    = levelInfo(xpTotal, 10, 1.35);

  const bBal   = levelInfo(balance, 100, 1.30);
  const bTx    = levelInfo(txCount, 5, 1.35);
  const bVol   = levelInfo(volumeTotal, 50, 1.35);
  const bNet   = levelInfo(Math.max(0, netCashflow), 50, 1.35);

  const skillBadges = SKILLS.map(skill => {
    const total = skillTotals[skill] || 0;
    const week = weekTotals[skill] || 0;
    const b = levelInfo(total, 3, 1.35);
    const iconMap = {Strength:"💪",Cardio:"🏃",Focus:"🧠",Discipline:"🛡️",Social:"🫂"};
    return {
      kind:"skill",
      icon: iconMap[skill] ?? "◆",
      title: `${skill.toUpperCase()} MASTERY`,
      unlocked: total > 0,
      level: b.level,
      pct: b.pct,
      subtitle: `All-time: ${total.toFixed(1)} • This week: ${week.toFixed(1)} • Next: ${Math.ceil(b.nextReq)}`
    };
  });

  const badges = [
    { kind:"login", icon:"⚡", title:"LOGIN STREAK", unlocked: loginStreak>0, level:bLogin.level, pct:bLogin.pct,
      subtitle:`Now: ${loginStreak}d • Next: ${Math.ceil(bLogin.nextReq)}d` },

    { kind:"tasks", icon:"🗡", title:"TASK SCORE", unlocked: tasksTotal>0, level:bTasks.level, pct:bTasks.pct,
      subtitle:`Now: ${tasksScore} • Next: ${Math.ceil(bTasks.nextReq)}` },

    { kind:"xp", icon:"★", title:"XP TOTAL", unlocked: xpTotal>0, level:bXP.level, pct:bXP.pct,
      subtitle:`Now: ${xpTotal} • Next: ${Math.ceil(bXP.nextReq)}` },

    ...skillBadges,

    { kind:"money", icon:"🏦", title:"BALANCE KEEPER", unlocked: balance>0, level:bBal.level, pct:bBal.pct,
      subtitle:`Now: ${fmtUSD(balance)} • Next: ${fmtUSD(bBal.nextReq)}` },

    { kind:"money", icon:"🧾", title:"TRANSACTION LOGGER", unlocked: txCount>0, level:bTx.level, pct:bTx.pct,
      subtitle:`Now: ${txCount} • Next: ${Math.ceil(bTx.nextReq)}` },

    { kind:"money", icon:"📦", title:"VOLUME TRACKED", unlocked: volumeTotal>0, level:bVol.level, pct:bVol.pct,
      subtitle:`Now: ${fmtUSD(volumeTotal)} • Next: ${fmtUSD(bVol.nextReq)}` },

    { kind:"money", icon:"📈", title:"POSITIVE CASHFLOW (ALL)", unlocked: netCashflow>0, level:bNet.level, pct:bNet.pct,
      subtitle:`Now: ${fmtUSD(netCashflow)} • Next: ${fmtUSD(bNet.nextReq)}` },
  ];

  // --- header stats row (compact, content-sized, more spacing) ---
const statsRow = dv.el("div", "", {
  style: `
    display:flex;
    flex-wrap:wrap;
    gap:30px;                 /* more space between pills */
    justify-content:center;
    align-items:flex-start;
    margin:0 0 18px 0;
  `
});

function pill(icon, label, value, color){
  const d = document.createElement("div");
  d.style.cssText = `
    display:inline-flex;      /* shrink to content */
    width:fit-content;        /* shrink to content */
    flex:0 0 auto;            /* don't stretch */
    align-items:center;
    gap:10px;

    padding:8px 10px;         /* smaller box */
    border-radius:999px;
    border:1px solid rgba(255,255,255,0.12);
    background:rgba(30,30,30,0.55);

    font-weight:800;
    line-height:1.15;
  `;

  d.innerHTML = `
    <div style="font-size:16px;line-height:1;flex:0 0 auto">${icon}</div>
    <div style="text-align:left;min-width:0">
      <div style="font-size:11px;opacity:.80;font-weight:900;white-space:nowrap">${label}</div>
      <div style="font-size:14px;font-weight:900;color:${color};white-space:nowrap">${value}</div>
    </div>
  `;
  return d;
}

statsRow.appendChild(pill("🏦","Balance", fmtUSD(balance), tone(balance)));
statsRow.appendChild(pill("📅","Month net", fmtUSD(netMonth), tone(netMonth)));
statsRow.appendChild(pill("⬆️","Income (month)", fmtUSD(incomeMonth), tone(incomeMonth)));
statsRow.appendChild(pill("⬇️","Expenses (month)", fmtUSD(expenseMonth), tone(-expenseMonth)));

dv.container.appendChild(statsRow);

  // ---- grid ----
  const grid = dv.el("div", "---");

  badges.forEach(b => grid.appendChild(renderBadge(b)));
  dv.container.appendChild(grid);

  // ---- REMOVED bottom hint block (per request) ----
})();
```
