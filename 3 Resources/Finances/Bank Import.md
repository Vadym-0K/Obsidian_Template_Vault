
```dataviewjs
const filePath = "3 Resources/Finances/Transaction.md";

// ---------------- helpers ----------------
function pad2(n){ return String(n).padStart(2, "0"); }

function normalizeDateLong(s) {
  // Accept: "Feb 28, 2026"
  const m = s.match(/^(Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+(\d{1,2}),\s*(\d{4})$/i);
  if (!m) return null;
  const mon = m[1].toLowerCase();
  const day = Number(m[2]);
  const year = Number(m[3]);

  const map = {jan:1,feb:2,mar:3,apr:4,may:5,jun:6,jul:7,aug:8,sep:9,oct:10,nov:11,dec:12};
  const mm = map[mon];
  if (!mm || !(day>=1 && day<=31) || !(year>=1900 && year<=2100)) return null;

  return `${year}-${pad2(mm)}-${pad2(day)}`;
}

function parseAmount(s) {
  // "$645.66" or "-$645.66"
  const cleaned = (s || "")
    .replace(/\s/g, "")
    .replace(/,/g, "")
    .replace(/USD/i, "");

  const m = cleaned.match(/^(-?)\$(\d+(?:\.\d{1,2})?)$/);
  if (!m) return null;

  const sign = m[1] === "-" ? -1 : 1;
  const val = Number(m[2]);
  if (!Number.isFinite(val)) return null;

  return sign * val;
}

function topLevelCategory(catPath) {
  // "Shopping & Entertainment: Electronics" -> "Shopping & Entertainment"
  // "Finance: Bank of America Credit Card Payment" -> "Finance"
  const s = (catPath || "").trim();
  if (!s) return "Other";
  const idx = s.indexOf(":");
  return (idx === -1 ? s : s.slice(0, idx)).trim();
}

function parseLine(line) {
  // Your pasted lines are "Date<TAB or spaces>Payee...<TAB>Account...<TAB>CategoryPath<TAB>Amount"
  // But there is noisy "on Feb 28, 2026collapsed". We'll strip that.
  let l = line.trim();
  if (!l) return null;

  // Try split by tab first
  let parts = l.split("\t").map(x => x.trim()).filter(Boolean);
  if (parts.length < 4) {
    // Fallback: split by 2+ spaces
    parts = l.split(/\s{2,}/).map(x => x.trim()).filter(Boolean);
  }
  if (parts.length < 2) return null;

  // Extract date from beginning: "Feb 28, 2026"
  const dateMatch = l.match(/^(Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+\d{1,2},\s*\d{4}/i);
  if (!dateMatch) return null;

  const dateStr = dateMatch[0];
  const date = normalizeDateLong(dateStr);
  if (!date) return null;

  // Amount is at end: "$..." or "-$..."
  const amountMatch = l.match(/(-?\$\d[\d,]*\.\d{2})\s*$/);
  if (!amountMatch) return null;

  const signed = parseAmount(amountMatch[1].replace(",", ""));
  if (signed === null) return null;

  const type = signed < 0 ? "expense" : "income";
  const amount = Math.abs(signed);

  // Remove trailing amount from line
  const withoutAmount = l.replace(/(-?\$\d[\d,]*\.\d{2})\s*$/, "").trim();

  // Remove the leading date
  let rest = withoutAmount.slice(dateStr.length).trim();

  // Remove the noisy "on Feb xx, yyyycollapsed" fragments
  rest = rest.replace(/\bon\s+(Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+\d{1,2},\s*\d{4}collapsed\b/gi, "").trim();

  // Heuristic: Category path is usually the last "X: Y" chunk before amount.
  // Find the last occurrence of something like "Something: Something"
  const catPathMatch = rest.match(/([A-Za-z0-9,&/()'"\-\s]+:\s*[A-Za-z0-9,&/()'"\-\s]+)\s*$/);
  const catPath = catPathMatch ? catPathMatch[1].trim() : "";

  // Payee: try to take text between date and the "on ..." or before account/category.
  // We'll take the first segment up to the first "D3bit" / "Cred1t" marker if present.
  let payee = rest;
  const acctIdx = payee.search(/\b(D3bit|Cred1t)\b/i);
  if (acctIdx !== -1) payee = payee.slice(0, acctIdx).trim();
  // If we extracted a category path from end, remove it from payee as well
  if (catPath && payee.endsWith(catPath)) payee = payee.slice(0, -catPath.length).trim();

  payee = payee.replace(/\s+/g, " ").trim();
  const category = topLevelCategory(catPath || "");

  return {
    date,
    ts: `${date} 00:00:00`,
    type,
    amount,
    category: category || "Other",
    note: `${payee}${catPath ? " — " + catPath : ""}`.trim(),
  };
}

function parseAll(text) {
  const txs = [];
  for (const line of text.split("\n")) {
    const tx = parseLine(line);
    if (tx) txs.push(tx);
  }
  return txs;
}

// ---------------- UI ----------------
const root = this.container;
root.style.display = "flex";
root.style.flexDirection = "column";
root.style.gap = "10px";

const ta = document.createElement("textarea");
ta.placeholder = "Paste your exported rows here...";
ta.style.width = "100%";
ta.style.height = "240px";
ta.style.padding = "10px";
ta.style.borderRadius = "10px";

const btnRow = document.createElement("div");
btnRow.style.display = "flex";
btnRow.style.gap = "10px";
btnRow.style.flexWrap = "wrap";

const previewBtn = document.createElement("button");
previewBtn.textContent = "Preview parsed";

const appendBtn = document.createElement("button");
appendBtn.textContent = "Append to Transaction.md";
appendBtn.style.background = "#8be4b5";
appendBtn.style.border = "1px solid rgba(0,0,0,0.2)";
appendBtn.style.padding = "8px 10px";
appendBtn.style.borderRadius = "8px";
appendBtn.style.cursor = "pointer";

btnRow.appendChild(previewBtn);
btnRow.appendChild(appendBtn);

const info = document.createElement("div");
info.style.opacity = "0.8";
info.style.fontSize = "12px";

root.appendChild(ta);
root.appendChild(btnRow);
root.appendChild(info);

let parsed = [];

previewBtn.onclick = () => {
  parsed = parseAll(ta.value);
  info.textContent = parsed.length ? `Parsed ${parsed.length} rows.` : "No rows parsed. Make sure lines include 'Feb 28, 2026' and end with $amount.";
  if (parsed.length) {
    dv.table(
      ["date", "type", "amount", "category", "note"],
      parsed.slice(0, 40).map(t => [t.date, t.type, t.amount, t.category, t.note])
    );
    if (parsed.length > 40) dv.paragraph(`Showing first 40 of ${parsed.length}.`);
  }
};

appendBtn.onclick = async () => {
  parsed = parsed.length ? parsed : parseAll(ta.value);
  if (!parsed.length) return new Notice("Nothing to append");

  const file = app.vault.getAbstractFileByPath(filePath);
  if (!file) return new Notice("Transactions file missing: " + filePath);

  let content = await app.vault.read(file);
  if (!content.endsWith("\n") && content.length > 0) content += "\n";

  const lines = parsed.map(t => `- ${JSON.stringify(t)}`).join("\n") + "\n";
  await app.vault.modify(file, content + lines);

  new Notice(`Appended ${parsed.length} transactions`);
};
```