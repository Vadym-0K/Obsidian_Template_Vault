
```dataviewjs
await (async () => {
  const PROFILE_PAGE = "3 Resources/Profile State";
  const FIN_DASH_PAGE = "3 Resources/Finances/Financial Dashboard";
  const TX_PATH = "3 Resources/Finances/Transaction.md";
  const DAILY_FOLDER = "4 Archives/Journal/Daily Notes";

  const P = dv.page(PROFILE_PAGE);
  const FD = dv.page(FIN_DASH_PAGE) ?? {};
  if (!P) { dv.paragraph(`Missing profile: ${PROFILE_PAGE}`); return; }

  const num = (v, d=0) => (Number.isFinite(Number(v)) ? Number(v) : d);

  const fmtUSD = (n) => {
    const x = Number(n);
    if (!Number.isFinite(x)) return "$0";
    return "$" + Math.round(x).toLocaleString("en-US");
  };

  function badgeColor(kind, unlocked){
    if (!unlocked) return { bg:"rgba(0,0,0,0.10)", border:"#00000040", text:"#9a9a9a", glow:"none", bar:"rgba(255,255,255,0.18)" };
    if (kind === "money")  return { bg:"rgba(0,0,0,0.10)", border:"#2dff9a", text:"#d9fff1", glow:"0 0 0 2px rgba(45,255,154,.14)", bar:"#2dff9a" };
    if (kind === "tasks")  return { bg:"rgba(0,0,0,0.10)", border:"#5bff4d", text:"#e6ffe4", glow:"0 0 0 2px rgba(91,255,77,.14)", bar:"#5bff4d" };
    if (kind === "xp")     return { bg:"rgba(0,0,0,0.10)", border:"#ffb84d", text:"#fff1d6", glow:"0 0 0 2px rgba(255,184,77,.14)", bar:"#ffb84d" };
    if (kind === "login")  return { bg:"rgba(0,0,0,0.10)", border:"#5bc0de", text:"#d9f6ff", glow:"0 0 0 2px rgba(91,192,222,.14)", bar:"#5bc0de" };
    if (kind === "sleep")  return { bg:"rgba(0,0,0,0.10)", border:"#b88cff", text:"#efe4ff", glow:"0 0 0 2px rgba(184,140,255,.14)", bar:"#b88cff" };
    if (kind === "stress") return { bg:"rgba(0,0,0,0.10)", border:"#ff6b6b", text:"#ffe2e2", glow:"0 0 0 2px rgba(255,107,107,.12)", bar:"#ff6b6b" };
    return { bg:"rgba(0,0,0,0.10)", border:"#777", text:"#fff", glow:"none", bar:"#777" };
  }

  // Requirement progression: req(level) = base * growth^level (infinite)
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
    const matches = text.match(/\{[^]*?\}/g) || [];
    const out = [];
    for (const raw of matches) { try { out.push(JSON.parse(raw)); } catch(_){} }
    return out.filter(Boolean);
  }

  // --- DAILY: sleep streak ---
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

  // --- DAILY: stress score (14d) from mood ---
  async function computeStressScore(days=14) {
    // mood 1..5 -> stress 4..0
    const today = moment().startOf("day");
    let score = 0;
    let logged = 0;

    for (let i = 0; i < days; i++) {
      const d = moment(today).subtract(i, "days");
      const page = dv.page(`${DAILY_FOLDER}/${d.format("YYYY-MM-DD")}`);
      const mood = Number(page?.mood ?? null);
      if (Number.isFinite(mood)) {
        const m = Math.max(1, Math.min(5, mood));
        score += (5 - m);
        logged++;
      }
    }
    return { score, logged, days };
  }

  // ---- profile stats ----
  const xpTotal = num(P.XP_Total, 0);
  const loginStreak = num(P.XP_Streak, 0);

  const tasksEasy = num(P.Tasks_Easy, 0);
  const tasksNormal = num(P.Tasks_Normal, 0);
  const tasksHard = num(P.Tasks_Hard, 0);
  const tasksTotal = tasksEasy + tasksNormal + tasksHard;
  const tasksScore = tasksEasy*1 + tasksNormal*3 + tasksHard*5;

  // ---- finance stats ----
  const balance = num(FD.Balance ?? FD.balance ?? FD.CurrentBalance ?? FD.current_balance, 0);

  const tx = await loadTransactionsFromFile(TX_PATH);
  const txCount = tx.length;
  const incomeTotal = tx.filter(t => String(t.type).toLowerCase()==="income").reduce((s,t)=>s+(Number(t.amount)||0),0);
  const expenseTotal = tx.filter(t => String(t.type).toLowerCase()==="expense").reduce((s,t)=>s+Math.abs(Number(t.amount)||0),0);
  const volumeTotal = incomeTotal + expenseTotal;
  const netCashflow = incomeTotal - expenseTotal;

  // ---- daily derived ----
  const sleepStreak = await computeSleepLoggedStreak();
  const stress14 = await computeStressScore(14);

  // ---- levels ----
  const bLogin = levelInfo(loginStreak, 3, 1.35);
  const bSleep = levelInfo(sleepStreak, 3, 1.35);
  const bStress = levelInfo(stress14.score, 6, 1.35); // 6 stress points -> lvl1 (tune)
  const bTasks = levelInfo(tasksScore, 25, 1.35);
  const bXP    = levelInfo(xpTotal, 10, 1.35);

  const bBal   = levelInfo(balance, 100, 1.30);
  const bTx    = levelInfo(txCount, 5, 1.35);
  const bVol   = levelInfo(volumeTotal, 50, 1.35);
  const bNet   = levelInfo(Math.max(0, netCashflow), 50, 1.35);

  function renderBadge({ title, subtitle, icon, kind, unlocked, level, pct }) {
    const c = badgeColor(kind, unlocked);

    const card = dv.el("div", "", {
      style: `
        background:${c.bg};
        border:1px solid #00000040;
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
    barWrap.style.cssText=`margin-top:12px;height:8px;border-radius:999px;border:1px solid #00000040;background:rgba(0,0,0,.18);overflow:hidden;`;

    const fill = barWrap.createDiv();
    fill.style.cssText=`height:100%;width:${Math.round((pct ?? 0)*100)}%;background:${c.bar};opacity:${unlocked ? .95 : .35};`;

    return card;
  }

  const badges = [
    { kind:"login", icon:"⚡", title:"LOGIN STREAK", unlocked: loginStreak>0, level:bLogin.level, pct:bLogin.pct,
      subtitle:`Now: ${loginStreak}d • Next: ${Math.ceil(bLogin.nextReq)}d` },

    { kind:"sleep", icon:"😴", title:"SLEEP LOG STREAK", unlocked: sleepStreak>0, level:bSleep.level, pct:bSleep.pct,
      subtitle:`Now: ${sleepStreak}d • Next: ${Math.ceil(bSleep.nextReq)}d` },

    { kind:"stress", icon:"🧠", title:"STRESS (14D)", unlocked: stress14.logged>0, level:bStress.level, pct:bStress.pct,
      subtitle:`Now: ${stress14.score} pts (${stress14.logged}/${stress14.days} logged) • Next: ${Math.ceil(bStress.nextReq)} pts` },

    { kind:"tasks", icon:"🗡", title:"TASK SCORE", unlocked: tasksTotal>0, level:bTasks.level, pct:bTasks.pct,
      subtitle:`Now: ${tasksScore} • Next: ${Math.ceil(bTasks.nextReq)}` },

    { kind:"xp", icon:"★", title:"XP TOTAL", unlocked: xpTotal>0, level:bXP.level, pct:bXP.pct,
      subtitle:`Now: ${xpTotal} • Next: ${Math.ceil(bXP.nextReq)}` },

    { kind:"money", icon:"🏦", title:"BALANCE KEEPER", unlocked: balance>0, level:bBal.level, pct:bBal.pct,
      subtitle:`Now: ${fmtUSD(balance)} • Next: ${fmtUSD(bBal.nextReq)}` },

    { kind:"money", icon:"🧾", title:"TRANSACTION LOGGER", unlocked: txCount>0, level:bTx.level, pct:bTx.pct,
      subtitle:`Now: ${txCount} • Next: ${Math.ceil(bTx.nextReq)}` },

    { kind:"money", icon:"📦", title:"VOLUME TRACKED", unlocked: volumeTotal>0, level:bVol.level, pct:bVol.pct,
      subtitle:`Now: ${fmtUSD(volumeTotal)} • Next: ${fmtUSD(bVol.nextReq)}` },

    { kind:"money", icon:"📈", title:"POSITIVE CASHFLOW", unlocked: netCashflow>0, level:bNet.level, pct:bNet.pct,
      subtitle:`Now: ${fmtUSD(netCashflow)} • Next: ${fmtUSD(bNet.nextReq)}` },
  ];

  // ---- header ----
  const header = dv.el("div", "", { style: "text-align:center;margin: 10px 0 18px 0;" });
  header.createDiv({ text: `🏅 ${P.Name ?? "Profile"} — Achievements` })
    .style.cssText = "font-size:22px;font-weight:900;color:#8be4b5;margin-bottom:10px;";
  header.createDiv({ text: `Balance: ${fmtUSD(balance)} • TX: ${txCount} • Volume: ${fmtUSD(volumeTotal)} • Net: ${fmtUSD(netCashflow)}` })
    .style.cssText = "opacity:.85;font-weight:800;";
  dv.container.appendChild(header);

  // ---- grid spacing ----
  const grid = dv.el("div", "", {
    style: `
      display:grid;
      grid-template-columns: repeat(auto-fit, minmax(320px, 1fr));
      gap:18px;
      width:100%;
      align-items:start;
    `
  });

  badges.forEach(b => grid.appendChild(renderBadge(b)));
  dv.container.appendChild(grid);

  const hint = dv.el("div","", {style:"margin-top:16px;opacity:.7;font-size:12px;"});
  hint.textContent = `Sources: ${TX_PATH} (parsed ${txCount} JSON objects), ${FIN_DASH_PAGE} (Balance), daily notes mood/sleep. Stress uses stress=5-mood over last 14 days.`;
  dv.container.appendChild(hint);
})();
```


