---
startingBalance: 0
---

```dataviewjs
const filePath = "3 Resources/Finances/Transaction.md";
const categories = ["Food", "Rent", "Gas", "Shopping", "Subscriptions", "Other"];

const container = this.container;
container.style.display = "flex";
container.style.flexDirection = "column";
container.style.gap = "12px";
container.style.marginBottom = "20px";

function styledInput(el) {
  el.style.padding = "6px";
  el.style.borderRadius = "6px";
  el.style.background = "#1f1f1f";
  el.style.border = "1px solid #00000040";
  el.style.color = "#dcdcdc";
  el.style.width = "200px";
}

const typeSelect = document.createElement("select");
["expense", "income"].forEach((t) => {
  const opt = document.createElement("option");
  opt.value = t;
  opt.textContent = t;
  typeSelect.appendChild(opt);
});
styledInput(typeSelect);

const categorySelect = document.createElement("select");
categories.forEach((c) => {
  const opt = document.createElement("option");
  opt.value = c;
  opt.textContent = c;
  categorySelect.appendChild(opt);
});
styledInput(categorySelect);

const amountInput = document.createElement("input");
amountInput.type = "number";
amountInput.step = "0.01";
amountInput.placeholder = "Amount";
styledInput(amountInput);

const noteInput = document.createElement("input");
noteInput.type = "text";
noteInput.placeholder = "Note (optional)";
styledInput(noteInput);

const btn = document.createElement("div");
btn.textContent = "Add Transaction";
btn.style.padding = "8px 12px";
btn.style.background = "#8be4b5";
btn.style.color = "#000";
btn.style.borderRadius = "6px";
btn.style.cursor = "pointer";
btn.style.textAlign = "center";
btn.style.fontWeight = "600";
btn.style.width = "fit-content";

btn.onclick = async () => {
  const amount = parseFloat(amountInput.value);
  if (Number.isNaN(amount)) return new Notice("Enter amount");

  const file = app.vault.getAbstractFileByPath(filePath);
  if (!file) return new Notice("Transactions file missing");

  const tx = {
    date: moment().format("YYYY-MM-DD"),
    type: typeSelect.value,
    amount: amount,
    category: categorySelect.value,
    note: (noteInput.value ?? "").replace(/\n/g, " ").trim(),
  };

  let content = await app.vault.read(file);
  if (!content.endsWith("\n") && content.length > 0) content += "\n";
  content += `- ${JSON.stringify(tx)}\n`;

  await app.vault.modify(file, content);
  new Notice("Transaction added");

  amountInput.value = "";
  noteInput.value = "";
};

container.appendChild(typeSelect);
container.appendChild(categorySelect);
container.appendChild(amountInput);
container.appendChild(noteInput);
container.appendChild(btn);
```
```dataviewjs
const activeFile = app.workspace.getActiveFile();
let startingBalance = Number(dv.current().startingBalance ?? 0) || 0;

const row = dv.el("div", "");
row.style.display = "flex";
row.style.gap = "10px";
row.style.alignItems = "center";
row.style.flexWrap = "wrap";
row.style.margin = "10px 0 18px 0";

const label = document.createElement("div");
label.textContent = "Starting balance";
label.style.opacity = "0.85";
label.style.fontWeight = "700";

const input = document.createElement("input");
input.type = "number";
input.step = "0.01";
input.value = String(startingBalance);
input.style.padding = "10px";
input.style.borderRadius = "12px";
input.style.border = "1px solid rgba(255,255,255,0.12)";
input.style.background = "rgba(30,30,30,0.65)";
input.style.color = "var(--text-normal)";
input.style.width = "200px";

const btn = document.createElement("button");
btn.textContent = "Save";
btn.style.padding = "10px 12px";
btn.style.borderRadius = "12px";
btn.style.border = "1px solid rgba(255,255,255,0.12)";
btn.style.cursor = "pointer";

const hint = document.createElement("div");
hint.style.opacity = "0.7";
hint.style.fontSize = "12px";
hint.textContent = "Stored in frontmatter: startingBalance";

row.appendChild(label);
row.appendChild(input);
row.appendChild(btn);
row.appendChild(hint);

btn.onclick = async () => {
  const newVal = Number(input.value);
  if (!Number.isFinite(newVal)) return new Notice("Enter a valid number");
  if (!activeFile) return new Notice("No active file");

  const content = await app.vault.read(activeFile);
  const fm = content.match(/^---\n([\s\S]*?)\n---\n?/);

  let next;
  if (!fm) {
    next = `---\nstartingBalance: ${newVal}\n---\n\n` + content;
  } else {
    const body = fm[1];
    if (/^\s*startingBalance\s*:/m.test(body)) {
      const body2 = body.replace(/^\s*startingBalance\s*:\s*.*$/m, `startingBalance: ${newVal}`);
      next = content.replace(/^---\n([\s\S]*?)\n---\n?/, `---\n${body2}\n---\n`);
    } else {
      const body2 = body + `\nstartingBalance: ${newVal}`;
      next = content.replace(/^---\n([\s\S]*?)\n---\n?/, `---\n${body2}\n---\n`);
    }
  }

  await app.vault.modify(activeFile, next);
  new Notice("Saved. Reopen note (or enable Dataview auto-refresh) to recalc.");
};
```

---

## Summary

```dataviewjs
const filePath = "3 Resources/Finances/Transaction.md";

function money(n) {
  const v = Number(n) || 0;
  return v.toLocaleString(undefined, { style: "currency", currency: "USD" });
}

function parseJsonLineTransactions(raw) {
  const txs = [];
  for (const line of raw.split("\n")) {
    const t = line.trim();
    if (!t) continue;
    const json = t.startsWith("- ") ? t.slice(2).trim() : t.trim();
    if (!json.startsWith("{")) continue;
    try {
      const o = JSON.parse(json);
      if (!o?.date || !o?.type) continue;
      txs.push({
        date: String(o.date),
        ts: o.ts ? String(o.ts) : "",
        type: String(o.type),
        amount: Number(o.amount) || 0,
        category: String(o.category ?? "Other"),
        note: String(o.note ?? ""),
      });
    } catch {}
  }
  return txs;
}

function sum(arr) { return arr.reduce((s, x) => s + x, 0); }
function yyyymm(d) { return String(d).slice(0, 7); }

const file = app.vault.getAbstractFileByPath(filePath);
if (!file) {
  dv.paragraph("Transactions file missing: " + filePath);
} else {
  const raw = await app.vault.read(file);
  const txs = parseJsonLineTransactions(raw).slice().sort((a, b) => {
    const ad = a.ts?.trim() ? a.ts : a.date;
    const bd = b.ts?.trim() ? b.ts : b.date;
    return ad < bd ? -1 : ad > bd ? 1 : 0;
  });

  if (!txs.length) {
    dv.paragraph("No transactions found.");
  } else {
    const startingBalance = Number(dv.current().startingBalance ?? 0) || 0;

    const incomeTotal = sum(txs.filter(t => t.type === "income").map(t => t.amount));
    const expenseTotal = sum(txs.filter(t => t.type === "expense").map(t => t.amount));
    const balance = startingBalance + (incomeTotal - expenseTotal);

    const currentMonth = moment().format("YYYY-MM");
    const txMonth = txs.filter(t => yyyymm(t.date) === currentMonth);
    const incomeMonth = sum(txMonth.filter(t => t.type === "income").map(t => t.amount));
    const expenseMonth = sum(txMonth.filter(t => t.type === "expense").map(t => t.amount));
    const netMonth = incomeMonth - expenseMonth;

    const kpiWrap = dv.el("div", "");
    kpiWrap.style.display = "grid";
    kpiWrap.style.gridTemplateColumns = "repeat(auto-fit, minmax(190px, 1fr))";
    kpiWrap.style.gap = "10px";
    kpiWrap.style.margin = "10px 0 16px 0";

    function kpi(title, value, sub = "") {
      const card = document.createElement("div");
      card.style.padding = "12px";
      card.style.border = "1px solid rgba(255,255,255,0.12)";
      card.style.borderRadius = "14px";
      card.style.background = "rgba(30,30,30,0.65)";

      const t = document.createElement("div");
      t.style.opacity = "0.85";
      t.style.fontSize = "12px";
      t.textContent = title;

      const v = document.createElement("div");
      v.style.fontSize = "22px";
      v.style.fontWeight = "800";
      v.style.marginTop = "6px";
      v.textContent = value;

      const s = document.createElement("div");
      s.style.opacity = "0.75";
      s.style.fontSize = "12px";
      s.style.marginTop = "6px";
      s.textContent = sub;

      card.appendChild(t);
      card.appendChild(v);
      if (sub) card.appendChild(s);
      kpiWrap.appendChild(card);
    }

    kpi("Current balance", money(balance), `Starting: ${money(startingBalance)}`);
    kpi("Income (all time)", money(incomeTotal));
    kpi("Expenses (all time)", money(expenseTotal));
    kpi("Net (this month)", money(netMonth), currentMonth);
    kpi("Expenses (this month)", money(expenseMonth), currentMonth);
  }
}
```

---

## Custom date range stats

```dataviewjs
const filePath = "3 Resources/Finances/Transaction.md";

// ---------- helpers ----------
function money(n) {
  const v = Number(n) || 0;
  return v.toLocaleString(undefined, { style: "currency", currency: "USD" });
}

function parseJsonLineTransactions(raw) {
  const txs = [];
  for (const line of raw.split("\n")) {
    const t = line.trim();
    if (!t) continue;
    const json = t.startsWith("- ") ? t.slice(2).trim() : t.trim();
    if (!json.startsWith("{")) continue;
    try {
      const o = JSON.parse(json);
      if (!o?.date || !o?.type) continue;
      txs.push({
        date: String(o.date), // YYYY-MM-DD
        ts: o.ts ? String(o.ts) : "",
        type: String(o.type),
        amount: Number(o.amount) || 0,
        category: String(o.category ?? "Other"),
        note: String(o.note ?? ""),
      });
    } catch {}
  }
  return txs;
}

function sum(arr) {
  return arr.reduce((s, x) => s + x, 0);
}

function inRange(dateStr, fromStr, toStr) {
  // all YYYY-MM-DD so lexicographic compare is safe
  return dateStr >= fromStr && dateStr <= toStr;
}

// ---------- UI ----------
const wrap = this.container;
wrap.style.display = "flex";
wrap.style.flexDirection = "column";
wrap.style.gap = "10px";

const controls = document.createElement("div");
controls.style.display = "flex";
controls.style.gap = "10px";
controls.style.flexWrap = "wrap";
controls.style.alignItems = "end";

function styleInput(el) {
  el.style.padding = "10px";
  el.style.borderRadius = "12px";
  el.style.border = "1px solid rgba(255,255,255,0.12)";
  el.style.background = "rgba(30,30,30,0.65)";
  el.style.color = "var(--text-normal)";
}

function labeled(labelText, inputEl) {
  const box = document.createElement("div");
  box.style.display = "flex";
  box.style.flexDirection = "column";
  box.style.gap = "6px";

  const lab = document.createElement("div");
  lab.textContent = labelText;
  lab.style.opacity = "0.85";
  lab.style.fontSize = "12px";
  lab.style.fontWeight = "700";

  box.appendChild(lab);
  box.appendChild(inputEl);
  return box;
}

const fromInput = document.createElement("input");
fromInput.type = "date";
fromInput.value = moment().startOf("month").format("YYYY-MM-DD");
styleInput(fromInput);

const toInput = document.createElement("input");
toInput.type = "date";
toInput.value = moment().format("YYYY-MM-DD");
styleInput(toInput);

const runBtn = document.createElement("button");
runBtn.textContent = "Calculate";
runBtn.style.padding = "10px 12px";
runBtn.style.borderRadius = "12px";
runBtn.style.border = "1px solid rgba(255,255,255,0.12)";
runBtn.style.cursor = "pointer";

controls.appendChild(labeled("From", fromInput));
controls.appendChild(labeled("To", toInput));
controls.appendChild(runBtn);
wrap.appendChild(controls);

const out = document.createElement("div");
wrap.appendChild(out);

// ---------- Load data ----------
const file = app.vault.getAbstractFileByPath(filePath);
if (!file) {
  dv.paragraph("Transactions file missing: " + filePath);
} else {
  const raw = await app.vault.read(file);
  const all = parseJsonLineTransactions(raw);

  const calc = () => {
    out.innerHTML = "";

    const from = fromInput.value;
    const to = toInput.value;

    if (!from || !to) {
      out.textContent = "Pick both dates.";
      return;
    }
    if (from > to) {
      out.textContent = "`From` must be <= `To`.";
      return;
    }

    const txs = all
      .filter(t => t.date && inRange(t.date, from, to))
      .slice()
      .sort((a, b) => (a.ts?.trim() ? a.ts : a.date).localeCompare(b.ts?.trim() ? b.ts : b.date));

    if (!txs.length) {
      out.textContent = `No transactions from ${from} to ${to}.`;
      return;
    }

    const income = sum(txs.filter(t => t.type === "income").map(t => t.amount));
    const expenses = sum(txs.filter(t => t.type === "expense").map(t => t.amount));
    const net = income - expenses;

    // category totals (expenses only)
    const catMap = new Map();
    for (const t of txs) {
      if (t.type !== "expense") continue;
      catMap.set(t.category, (catMap.get(t.category) ?? 0) + t.amount);
    }
    const catRows = [...catMap.entries()]
      .sort((a, b) => b[1] - a[1])
      .slice(0, 10)
      .map(([c, v]) => [c, money(v)]);

    // render numbers
    const kpi = document.createElement("div");
    kpi.style.display = "grid";
    kpi.style.gridTemplateColumns = "repeat(auto-fit, minmax(190px, 1fr))";
    kpi.style.gap = "10px";
    kpi.style.marginTop = "8px";

    function card(title, value) {
      const d = document.createElement("div");
      d.style.padding = "12px";
      d.style.border = "1px solid rgba(255,255,255,0.12)";
      d.style.borderRadius = "14px";
      d.style.background = "rgba(30,30,30,0.65)";

      const t = document.createElement("div");
      t.style.opacity = "0.85";
      t.style.fontSize = "12px";
      t.style.fontWeight = "700";
      t.textContent = title;

      const v = document.createElement("div");
      v.style.fontSize = "22px";
      v.style.fontWeight = "800";
      v.style.marginTop = "6px";
      v.textContent = value;

      d.appendChild(t);
      d.appendChild(v);
      kpi.appendChild(d);
    }

    card("Transactions", String(txs.length));
    card("Income", money(income));
    card("Expenses", money(expenses));
    card("Net", money(net));

    out.appendChild(kpi);

    // optional table
    const tableHost = document.createElement("div");
    tableHost.style.marginTop = "10px";
    out.appendChild(tableHost);

    // Use dv.table into a temp container by swapping dv.container context
    const prev = dv.container;
    dv.container = tableHost;

    dv.header(4, "Top expense categories (range)");
    if (catRows.length) dv.table(["Category", "Total"], catRows);
    else dv.paragraph("No expenses in this range.");

    dv.container = prev;
  };

  runBtn.onclick = calc;

  // auto-run once
  calc();
}
```

___
## Charts — Balance

### Ending balance by month (line)

```dataviewjs
const filePath = "3 Resources/Finances/Transaction.md";

function quickChartUrl(config, w = 900, h = 320, bg = "transparent") {
  const base = "https://quickchart.io/chart";
  const params = new URLSearchParams({
    c: JSON.stringify(config),
    width: String(w),
    height: String(h),
    backgroundColor: bg,
  });
  return `${base}?${params.toString()}`;
}

function parseJsonLineTransactions(raw) {
  const txs = [];
  for (const line of raw.split("\n")) {
    const t = line.trim();
    if (!t) continue;
    const json = t.startsWith("- ") ? t.slice(2).trim() : t.trim();
    if (!json.startsWith("{")) continue;
    try {
      const o = JSON.parse(json);
      if (!o?.date || !o?.type) continue;
      txs.push({
        date: String(o.date),
        ts: o.ts ? String(o.ts) : "",
        type: String(o.type),
        amount: Number(o.amount) || 0,
      });
    } catch {}
  }
  return txs;
}
function signedAmount(t) { return (t.type === "expense" ? -1 : 1) * t.amount; }
function yyyymm(d) { return String(d).slice(0, 7); }

const file = app.vault.getAbstractFileByPath(filePath);
if (!file) {
  dv.paragraph("Transactions file missing: " + filePath);
} else {
  const raw = await app.vault.read(file);
  const txs = parseJsonLineTransactions(raw).slice().sort((a, b) => {
    const ad = a.ts?.trim() ? a.ts : a.date;
    const bd = b.ts?.trim() ? b.ts : b.date;
    return ad < bd ? -1 : ad > bd ? 1 : 0;
  });

  if (!txs.length) {
    dv.paragraph("No transactions found.");
  } else {
    const startingBalance = Number(dv.current().startingBalance ?? 0) || 0;

    let running = startingBalance;
    const monthEnd = new Map(); // YYYY-MM -> ending balance

    for (const t of txs) {
      running += signedAmount(t);
      const m = yyyymm(t.date);
      if (m) monthEnd.set(m, Number(running.toFixed(2)));
    }

    const months = [...monthEnd.keys()].sort();
    const data = months.map(m => monthEnd.get(m));

    const cfg = {
      type: "line",
      data: {
        labels: months,
        datasets: [{
          label: "Balance",
          data,
          borderColor: "#8be4b5",
          backgroundColor: "rgba(139,228,181,0.15)",
          fill: true,
          pointRadius: 2,
          tension: 0.25,
        }]
      },
      options: {
        plugins: {
          legend: { display: false },
          title: { display: true, text: "Ending Balance (Monthly)" }
        },
        scales: {
          x: { ticks: { autoSkip: true, maxTicksLimit: 14 } },
          y: { beginAtZero: false }
        }
      }
    };

    dv.el("img", "", { attr: { src: quickChartUrl(cfg, 900, 320) } });
  }
}
```

---

## Charts — Monthly cashflow

### Income vs expenses by month (bar)

```dataviewjs
const filePath = "3 Resources/Finances/Transaction.md";

function quickChartUrl(config, w = 900, h = 320, bg = "transparent") {
  const base = "https://quickchart.io/chart";
  const params = new URLSearchParams({
    c: JSON.stringify(config),
    width: String(w),
    height: String(h),
    backgroundColor: bg,
  });
  return `${base}?${params.toString()}`;
}

function parseJsonLineTransactions(raw) {
  const txs = [];
  for (const line of raw.split("\n")) {
    const t = line.trim();
    if (!t) continue;
    const json = t.startsWith("- ") ? t.slice(2).trim() : t.trim();
    if (!json.startsWith("{")) continue;
    try {
      const o = JSON.parse(json);
      if (!o?.date || !o?.type) continue;
      txs.push({
        date: String(o.date),
        ts: o.ts ? String(o.ts) : "",
        type: String(o.type),
        amount: Number(o.amount) || 0,
      });
    } catch {}
  }
  return txs;
}

function yyyymm(d) { return String(d).slice(0, 7); }

const file = app.vault.getAbstractFileByPath(filePath);
if (!file) {
  dv.paragraph("Transactions file missing: " + filePath);
} else {
  const raw = await app.vault.read(file);
  const txs = parseJsonLineTransactions(raw);

  if (!txs.length) {
    dv.paragraph("No transactions found.");
  } else {
    const monthMap = new Map(); // month -> {income, expense}

    for (const t of txs) {
      const m = yyyymm(t.date);
      if (!monthMap.has(m)) monthMap.set(m, { income: 0, expense: 0 });
      const v = monthMap.get(m);
      if (t.type === "income") v.income += t.amount;
      else v.expense += t.amount;
    }

    const months = [...monthMap.keys()].sort();
    const incomeByMonth = months.map(m => Number(monthMap.get(m).income.toFixed(2)));
    const expenseByMonth = months.map(m => Number(monthMap.get(m).expense.toFixed(2)));

    const cfg = {
      type: "bar",
      data: {
        labels: months,
        datasets: [
          { label: "Income", data: incomeByMonth, backgroundColor: "rgba(139,228,181,0.80)" },
          { label: "Expenses", data: expenseByMonth, backgroundColor: "rgba(255,99,132,0.70)" },
        ]
      },
      options: {
        plugins: { title: { display: true, text: "Income vs Expenses by Month" } },
        scales: { x: { stacked: false }, y: { beginAtZero: true } }
      }
    };

    dv.el("img", "", { attr: { src: quickChartUrl(cfg, 900, 320) } });
  }
}
```


---

## Latest transactions

```dataviewjs
const filePath = "3 Resources/Finances/Transaction.md";

function money(n) {
  const v = Number(n) || 0;
  return v.toLocaleString(undefined, { style: "currency", currency: "USD" });
}

function parseJsonLineTransactions(raw) {
  const txs = [];
  for (const line of raw.split("\n")) {
    const t = line.trim();
    if (!t) continue;
    const json = t.startsWith("- ") ? t.slice(2).trim() : t.trim();
    if (!json.startsWith("{")) continue;
    try {
      const o = JSON.parse(json);
      if (!o?.date || !o?.type) continue;
      txs.push({
        date: String(o.date),
        ts: o.ts ? String(o.ts) : "",
        type: String(o.type),
        amount: Number(o.amount) || 0,
        category: String(o.category ?? "Other"),
        note: String(o.note ?? ""),
      });
    } catch {}
  }
  return txs;
}

const file = app.vault.getAbstractFileByPath(filePath);
if (!file) {
  dv.paragraph("Transactions file missing: " + filePath);
} else {
  const raw = await app.vault.read(file);
  const txs = parseJsonLineTransactions(raw).slice().sort((a, b) => {
    const ad = a.ts?.trim() ? a.ts : a.date;
    const bd = b.ts?.trim() ? b.ts : b.date;
    return ad < bd ? -1 : ad > bd ? 1 : 0;
  });

  if (!txs.length) {
    dv.paragraph("No transactions found.");
  } else {
    dv.table(
      ["date", "type", "amount", "category", "note"],
      txs.slice().reverse().slice(0, 15).map(t => [t.date, t.type, money(t.amount), t.category, t.note])
    );
  }
}
```