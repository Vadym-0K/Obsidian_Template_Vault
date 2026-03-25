```dataviewjs
const STATE_NOTE_PATH = "3 Resources/Profile State.md";
const DAILY_FOLDER = "4 Archives/Journal/Daily Notes";
const DAILY_FORMAT = "YYYY-MM-DD";

const XP = { easy: 1, normal: 3, hard: 10 };
const XP_LOGGED_DAY = 1;
const XP_BIRTHDAY_BONUS = 500;

// reuse theme from profile card if present
const UI = window.__gpTheme ?? {
  panel: "rgba(0,0,0,0.14)",
  panel2: "rgba(0,0,0,0.10)",
  border: "rgba(0,0,0,0.20)",
  title: "#8be4b5",
};

function panel(el) {
  el.style.background = UI.panel;
  el.style.border = `1px solid ${UI.border}`;
  el.style.borderRadius = "10px";
  el.style.padding = "12px";
}
function hRow(el) {
  el.style.display = "flex";
  el.style.gap = "10px";
  el.style.alignItems = "center";
  el.style.flexWrap = "wrap";
}
function flatBtn(el) {
  el.style.padding = "8px 10px";
  el.style.borderRadius = "8px";
  el.style.background = UI.panel2;
  el.style.border = `1px solid ${UI.border}`;
  el.style.cursor = "pointer";
  el.style.userSelect = "none";
  el.style.fontWeight = "700";
  el.style.transition = "background .12s ease";
  el.onmouseover = () => (el.style.background = "rgba(255,255,255,0.06)");
  el.onmouseout = () => (el.style.background = UI.panel2);
}
function escapeRe(s) { return s.replace(/[.*+?^${}()|[\]\\]/g, "\\$&"); }

function readInline(content, key) {
  const safe = escapeRe(key);
  const r = new RegExp(`^\\s*${safe}::\\s*(.*?)\\s*$`, "m");
  const m = content.match(r);
  if (!m) return "";
  let v = (m[1] ?? "").trim();
  v = v.replace(/^"(.*)"$/, "$1").replace(/^'(.*)'$/, "$1");
  return v;
}
function readInlineNumber(content, key, defVal = 0) {
  const raw = readInline(content, key);
  const n = Number(raw);
  return Number.isFinite(n) ? n : defVal;
}
function setOrInsertInlineField(content, key, value) {
  const safe = escapeRe(key);
  const r = new RegExp(`^\\s*${safe}::\\s*.*$`, "m");
  const line = `${key}:: ${value}`;
  if (r.test(content)) return content.replace(r, line);
  return `${line}\n${content}`;
}

function initialStateText() {
  return `Name:: "Vadym Kharchenko"
DOB:: "1998-10-05"

XP_Total:: 0
XP_LastLogin:: ""
XP_Streak:: 0

Tasks_Easy:: 0
Tasks_Normal:: 0
Tasks_Hard:: 0

XP_BirthdayBonusYear:: 0
`;
}

async function ensureStateFile() {
  let f = app.vault.getAbstractFileByPath(STATE_NOTE_PATH);
  if (!f) f = await app.vault.create(STATE_NOTE_PATH, initialStateText());
  return f;
}

async function loadState() {
  const f = await ensureStateFile();
  const text = await app.vault.read(f);
  return {
    DOB: readInline(text, "DOB") || "1998-10-05",
    XP_Total: readInlineNumber(text, "XP_Total", 0),
    XP_LastLogin: readInline(text, "XP_LastLogin") || "",
    XP_Streak: readInlineNumber(text, "XP_Streak", 0),
    Tasks_Easy: readInlineNumber(text, "Tasks_Easy", 0),
    Tasks_Normal: readInlineNumber(text, "Tasks_Normal", 0),
    Tasks_Hard: readInlineNumber(text, "Tasks_Hard", 0),
    XP_BirthdayBonusYear: readInlineNumber(text, "XP_BirthdayBonusYear", 0),
    __text: text,
    __file: f
  };
}

async function saveState(s) {
  let text = s.__text;

  // ensure birthday bonus tracking field exists
  text = setOrInsertInlineField(text, "XP_BirthdayBonusYear", Math.max(0, Math.floor(s.XP_BirthdayBonusYear || 0)));

  text = setOrInsertInlineField(text, "XP_Total", Math.max(0, s.XP_Total));
  text = setOrInsertInlineField(text, "XP_LastLogin", `"${s.XP_LastLogin || ""}"`);
  text = setOrInsertInlineField(text, "XP_Streak", Math.max(0, s.XP_Streak));
  text = setOrInsertInlineField(text, "Tasks_Easy", Math.max(0, s.Tasks_Easy));
  text = setOrInsertInlineField(text, "Tasks_Normal", Math.max(0, s.Tasks_Normal));
  text = setOrInsertInlineField(text, "Tasks_Hard", Math.max(0, s.Tasks_Hard));

  await app.vault.modify(s.__file, text);
}

async function getDailyFile(dateMoment) {
  const name = dateMoment.format(DAILY_FORMAT) + ".md";
  const path = `${DAILY_FOLDER}/${name}`;
  let file = app.vault.getAbstractFileByPath(path);
  if (!file) file = await app.vault.create(path, "");
  return file;
}

async function ensureLoggedTrueToday() {
  const file = await getDailyFile(moment().startOf("day"));
  let content = await app.vault.read(file);

  if (/^\s*logged::\s*true\s*$/m.test(content)) return;

  if (/^\s*logged::\s*/m.test(content)) content = content.replace(/^\s*logged::\s*.*$/m, "logged:: true");
  else content = `logged:: true\n` + content;

  await app.vault.modify(file, content);
}

function isBirthdayToday(dobStr) {
  const dob = moment(dobStr, "YYYY-MM-DD", true);
  if (!dob.isValid()) return false;
  const t = moment().startOf("day");
  return dob.month() === t.month() && dob.date() === t.date();
}

async function maybeApplyBirthdayBonus(s) {
  const year = Number(moment().format("YYYY"));
  if (!isBirthdayToday(s.DOB)) return { applied: false, year };

  // Apply only once per year
  if (Number(s.XP_BirthdayBonusYear || 0) === year) return { applied: false, year };

  s.XP_Total += XP_BIRTHDAY_BONUS;
  s.XP_BirthdayBonusYear = year;
  return { applied: true, year };
}

// ---------- UI ----------
const root = this.container;
root.innerHTML = "";

// 1) OUTER WRAPPER (centers the whole block)
const wrapper = document.createElement("div");
wrapper.style.maxWidth = "760px";  // optional: limits width
wrapper.style.margin = "0 auto";   // centers it
root.appendChild(wrapper);

// 2) INNER CARD (your existing panel)
const card = document.createElement("div");
panel(card);
wrapper.appendChild(card);

const title = document.createElement("div");
title.textContent = "Actions";
title.style.fontWeight = "900";
title.style.color = UI.title;
title.style.marginBottom = "10px";
card.appendChild(title);

const row = document.createElement("div");
hRow(row);
card.appendChild(row);

function makeBtn(label) {
  const b = document.createElement("div");
  b.textContent = label;
  flatBtn(b);
  return b;
}

const easyBtn = makeBtn(`Easy +${XP.easy}`);
const normalBtn = makeBtn(`Task +${XP.normal}`);
const hardBtn = makeBtn(`Hard +${XP.hard}`);
const logBtn = makeBtn(`Log today +${XP_LOGGED_DAY}`);

row.appendChild(easyBtn);
row.appendChild(normalBtn);
row.appendChild(hardBtn);
row.appendChild(logBtn);

const hint = document.createElement("div");
hint.style.marginTop = "10px";
hint.style.opacity = "0.75";
hint.style.fontSize = "12px";
card.appendChild(hint);

async function rerenderProfile() {
  if (window.__gpRenderProfile) await window.__gpRenderProfile();
}

async function refreshHint() {
  const s = await loadState();
  hint.textContent = `XP: ${s.XP_Total} • Last login: ${s.XP_LastLogin || "—"} • Birthday bonus: ${isBirthdayToday(s.DOB) ? "today" : "not today"}`;
}
await refreshHint();

async function mutate(mutator) {
  const s = await loadState();
  mutator(s);
  const b = await maybeApplyBirthdayBonus(s);
  await saveState(s);
  await refreshHint();
  await rerenderProfile();
  return b;
}

easyBtn.onclick = async () => {
  await mutate((s) => { s.XP_Total += XP.easy; s.Tasks_Easy += 1; });
  new Notice(`+${XP.easy} XP`);
};

normalBtn.onclick = async () => {
  await mutate((s) => { s.XP_Total += XP.normal; s.Tasks_Normal += 1; });
  new Notice(`+${XP.normal} XP`);
};

hardBtn.onclick = async () => {
  await mutate((s) => { s.XP_Total += XP.hard; s.Tasks_Hard += 1; });
  new Notice(`+${XP.hard} XP`);
};

logBtn.onclick = async () => {
  const today = moment().startOf("day");
  const todayStr = today.format("YYYY-MM-DD");

  await ensureLoggedTrueToday();

  const bonus = await mutate((s) => {
    const last = s.XP_LastLogin ? moment(s.XP_LastLogin, "YYYY-MM-DD", true) : null;
    const already = last && last.isValid() && last.isSame(today, "day");
    if (already) return;

    s.XP_Total += XP_LOGGED_DAY;

    if (last && last.isValid() && last.isSame(moment(today).subtract(1, "day"), "day")) {
      s.XP_Streak = (s.XP_Streak || 0) + 1;
    } else {
      s.XP_Streak = 1;
    }

    s.XP_LastLogin = todayStr;
  });

  if (bonus.applied) new Notice(`Logged today (+${XP_LOGGED_DAY}) + Birthday bonus (+${XP_BIRTHDAY_BONUS})`);
  else new Notice(`Logged today (+${XP_LOGGED_DAY} XP if not already logged)`);
};
```