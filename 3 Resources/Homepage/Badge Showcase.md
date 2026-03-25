```dataviewjs
await (async () => {
  // ---- paths ----
  const PROFILE_PAGE = "3 Resources/Profile State";
  const FIN_DASH_PAGE = "3 Resources/Finances/Financial Dashboard";
  const TX_PATH = "3 Resources/Finances/Transaction.md";
  const DAILY_FOLDER = "4 Archives/Journal/Daily Notes";

  const MAX_SHOW = 5; // shelf shows top N

  const P = dv.page(PROFILE_PAGE);
  const FD = dv.page(FIN_DASH_PAGE) ?? {};
  if (!P) { dv.paragraph(`Missing profile: ${PROFILE_PAGE}`); return; }

  const num = (v, d=0) => (Number.isFinite(Number(v)) ? Number(v) : d);
  const fmtUSD = (n) => "$" + Math.round(Number(n) || 0).toLocaleString("en-US");

  function badgeColor(kind, unlocked){
    if (!unlocked) return { bg:"rgba(0,0,0,0.10)", border:"#00000040", text:"#9a9a9a" };
    if (kind === "money")  return { bg:"rgba(0,0,0,0.10)", border:"#2dff9a", text:"#d9fff1" };
    if (kind === "tasks")  return { bg:"rgba(0,0,0,0.10)", border:"#5bff4d", text:"#e6ffe4" };
    if (kind === "xp")     return { bg:"rgba(0,0,0,0.10)", border:"#ffb84d", text:"#fff1d6" };
    if (kind === "login")  return { bg:"rgba(0,0,0,0.10)", border:"#5bc0de", text:"#d9f6ff" };
    if (kind === "sleep")  return { bg:"rgba(0,0,0,0.10)", border:"#b88cff", text:"#efe4ff" };
    if (kind === "stress") return { bg:"rgba(0,0,0,0.10)", border:"#ff6b6b", text:"#ffe2e2" };
    return { bg:"rgba(0,0,0,0.10)", border:"#777", text:"#fff" };
  }

  // infinite levels
  function levelInfo(value, base, growth){
    const v = Math.max(0, Number(value) || 0);
    const b = Math.max(1e-9, Number(base) || 1);
    const g = Math.max(1.000001, Number(growth) || 1.25);

    let lvl = 0;
    if (v >= b) {
      lvl = Math.floor(Math.log(v / b) / Math.log(g));
      if (!Number.isFinite(lvl) || lvl < 0) lvl = 0;
    }
    const nextReq = b * Math.pow(g, lvl + 1);
    return { level: lvl, nextReq };
  }

  async function loadTransactionsFromFile(path) {
    const f = app.vault.getAbstractFileByPath(path);
    if (!f) return [];
    const text = await app.vault.read(f);
    const matches = text.match(/\{[^]*?\}/g) || [];
    const out = [];
    for (const raw of matches) { try { out.push(JSON.parse(raw)); } catch(_){} }
    return out.filter(Boolean);
  }

  // ---- Daily stats (sleep + stress from mood) ----
  async function computeSleepLoggedStreak() {
    const today = moment().startOf("day");
    let streak = 0;
    for (let i = 0; i < 800; i++) {
      const d = moment(today).subtract(i, "days");
      const page = dv.page(`${DAILY_FOLDER}/${d.format("YYYY-MM-DD")}`);
      const sleep = Number(page?.sleep ?? null);
      if (Number.isFinite(sleep) && sleep > 0) streak++;
      else break;
    }
    return streak;
  }

  async function computeStressScore14d() {
    // Based on mood (assume mood 1..5). Convert to stress 0..4 as: stress = 5 - mood.
    // Score = sum(stress) over last 14 days with mood present.
    const today = moment().startOf("day");
    let score = 0;
    let logged = 0;

    for (let i = 0; i < 14; i++) {
      const d = moment(today).subtract(i, "days");
      const page = dv.page(`${DAILY_FOLDER}/${d.format("YYYY-MM-DD")}`);
      const mood = Number(page?.mood ?? null);
      if (Number.isFinite(mood)) {
        const m = Math.max(1, Math.min(5, mood));
        const stress = 5 - m; // mood 5 => 0 stress, mood 1 => 4 stress
        score += stress;
        logged++;
      }
    }
    return { score, logged };
  }

  const sleepStreak = await computeSleepLoggedStreak();
  const stress14 = await computeStressScore14d();

  // ---- core stats (profile + finance) ----
  const xpTotal = num(P.XP_Total, 0);
  const loginStreak = num(P.XP_Streak, 0);

  const tasksEasy = num(P.Tasks_Easy, 0);
  const tasksNormal = num(P.Tasks_Normal, 0);
  const tasksHard = num(P.Tasks_Hard, 0);
  const tasksTotal = tasksEasy + tasksNormal + tasksHard;
  const tasksScore = tasksEasy*1 + tasksNormal*3 + tasksHard*5;

  const balance = num(FD.Balance ?? FD.balance ?? FD.CurrentBalance ?? FD.current_balance, 0);

  const tx = await loadTransactionsFromFile(TX_PATH);
  const txCount = tx.length;
  const incomeTotal = tx.filter(t => String(t.type).toLowerCase()==="income").reduce((s,t)=>s+(Number(t.amount)||0),0);
  const expenseTotal = tx.filter(t => String(t.type).toLowerCase()==="expense").reduce((s,t)=>s+Math.abs(Number(t.amount)||0),0);
  const volumeTotal = incomeTotal + expenseTotal;

  // ---- badges (include sleep + stress) ----
  let badges = [
    { key:"login", kind:"login", icon:"⚡", title:"Login", valueText:`${loginStreak}d`, unlocked: loginStreak>0, ...levelInfo(loginStreak, 3, 1.35), isMoney:false },

    { key:"sleep", kind:"sleep", icon:"😴", title:"Sleep Log", valueText:`${sleepStreak}d`, unlocked: sleepStreak>0, ...levelInfo(sleepStreak, 3, 1.35), isMoney:false },

    // Stress: higher stress score => higher level (you can invert if you want "calm" instead)
    { key:"stress", kind:"stress", icon:"🧠", title:"Stress (14d)", valueText:`${stress14.score} pts`, unlocked: stress14.logged>0, ...levelInfo(stress14.score, 6, 1.35), isMoney:false },

    { key:"tasks", kind:"tasks", icon:"🗡", title:"Tasks", valueText:`${tasksScore}`, unlocked: tasksTotal>0, ...levelInfo(tasksScore, 25, 1.35), isMoney:false },
    { key:"xp",    kind:"xp",    icon:"★", title:"XP",    valueText:`${xpTotal}`, unlocked: xpTotal>0, ...levelInfo(xpTotal, 10, 1.35), isMoney:false },

    { key:"bal",   kind:"money", icon:"🏦", title:"Balance", valueText:fmtUSD(balance), unlocked: balance>0, ...levelInfo(balance, 100, 1.30), isMoney:true },
    { key:"tx",    kind:"money", icon:"🧾", title:"TX", valueText:`${txCount}`, unlocked: txCount>0, ...levelInfo(txCount, 5, 1.35), isMoney:false },
    { key:"vol",   kind:"money", icon:"📦", title:"Volume", valueText:fmtUSD(volumeTotal), unlocked: volumeTotal>0, ...levelInfo(volumeTotal, 50, 1.35), isMoney:true },
  ];

  // show top N: unlocked first, then highest level
  badges = badges
    .sort((a,b) => (b.unlocked - a.unlocked) || (b.level - a.level))
    .slice(0, MAX_SHOW);

  // ---- UI: shelf ----
  const container = document.createElement("div");
  container.style.cssText = `
    margin: 10px 0 16px 0;
    padding: 12px;
    border-radius: 12px;
    border: 1px solid #00000040;
    background: rgba(0,0,0,0.08);
  `;

  const head = document.createElement("div");
  head.style.cssText = "display:flex;align-items:baseline;justify-content:space-between;gap:12px;margin-bottom:10px;";
  container.appendChild(head);

  const hL = document.createElement("div");
  hL.textContent = "🏆 Badge Shelf";
  hL.style.cssText = "font-weight:900;color:#8be4b5;font-size:18px;";
  head.appendChild(hL);

  const hR = document.createElement("div");
  hR.textContent = `Top ${MAX_SHOW}`;
  hR.style.cssText = "opacity:.7;font-weight:800;font-size:12px;";
  head.appendChild(hR);

  const grid = document.createElement("div");
  grid.style.cssText = `
    display:grid;
    grid-template-columns: repeat(auto-fit, minmax(170px, 1fr));
    gap:12px;
    width:100%;
  `;
  container.appendChild(grid);

  function chip(b){
    const c = badgeColor(b.kind, b.unlocked);

    const el = document.createElement("div");
    el.style.cssText = `
      border-radius: 12px;
      padding: 10px 12px;
      background: ${c.bg};
      border: 1px solid ${c.border};
      image-rendering: pixelated;
      font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
    `;

    const top = document.createElement("div");
    top.style.cssText = "display:flex;align-items:center;justify-content:space-between;gap:10px;margin-bottom:6px;";
    el.appendChild(top);

    const left = document.createElement("div");
    left.style.cssText = "display:flex;align-items:center;gap:10px;min-width:0;";
    top.appendChild(left);

    const icon = document.createElement("div");
    icon.textContent = b.icon;
    icon.style.cssText = `
      width:34px;height:34px;border-radius:10px;
      display:flex;align-items:center;justify-content:center;
      border:1px solid ${c.border};
      background: rgba(255,255,255,0.05);
      font-size:18px;
      filter: ${b.unlocked ? "none" : "grayscale(1) opacity(.65)"};
      flex: 0 0 auto;
    `;
    left.appendChild(icon);

    const text = document.createElement("div");
    text.style.minWidth = "0";
    left.appendChild(text);

    const title = document.createElement("div");
    title.textContent = b.title;
    title.style.cssText = `font-weight:900;color:${c.text};white-space:nowrap;overflow:hidden;text-overflow:ellipsis;`;
    text.appendChild(title);

    const val = document.createElement("div");
    val.textContent = b.valueText;
    val.style.cssText = `margin-top:2px;font-size:12px;color:${c.text};opacity:${b.unlocked ? .85 : .65};`;
    text.appendChild(val);

    const lvl = document.createElement("div");
    lvl.textContent = `LV.${b.level}`;
    lvl.style.cssText = `
      padding: 4px 8px;
      border-radius: 999px;
      border: 1px solid ${c.border};
      color: ${c.text};
      font-weight: 900;
      font-size: 11px;
      opacity: ${b.unlocked ? .95 : .55};
      white-space: nowrap;
      flex: 0 0 auto;
    `;
    top.appendChild(lvl);

    const sub = document.createElement("div");
    const nextText =
      (b.title === "TX" || !b.isMoney) ? `${Math.ceil(b.nextReq)}` : fmtUSD(b.nextReq);

    sub.textContent = `Next: ${nextText}`;
    sub.style.cssText = `font-size:12px;opacity:${b.unlocked ? .80 : .60};color:${c.text};`;
    el.appendChild(sub);

    return el;
  }

  badges.forEach(b => grid.appendChild(chip(b)));

  dv.container.appendChild(container);
})();
```
