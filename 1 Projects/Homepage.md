
[[TODO]] · [[TOFIX]] · [[WORK]] · [[ARMY]] · [[Financial Dashboard]] 
___
```dataviewjs
const STATE_NOTE_PATH = "3 Resources/Profile State.md";
const AVATAR_PATH = "4 Archives/Attachments/wallpaper_berserk.jpg";
const AVATAR_SIZE = 150;

// ---------- FLAT THEME ----------
const UI = {
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

function pill(text) {
  const el = document.createElement("div");
  el.textContent = text;
  el.style.padding = "6px 10px";
  el.style.borderRadius = "999px";
  el.style.background = UI.panel2;
  el.style.border = `1px solid ${UI.border}`;
  el.style.fontSize = "12px";
  el.style.fontWeight = "700";
  el.style.opacity = "0.95";
  return el;
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

function levelFromXp(xp) {
  xp = Math.max(0, Math.floor(Number(xp) || 0));
  let level = 1;
  let need = 25;
  let remaining = xp;
  while (remaining >= need) {
    remaining -= need;
    level += 1;
    need = 25 + (level - 1) * 25;
    if (level > 500) break;
  }
  return { level, xpIntoLevel: remaining, xpForNext: need };
}

function ageFromDob(dobStr) {
  const dob = moment(dobStr, "YYYY-MM-DD", true);
  if (!dob.isValid()) return null;
  return moment().diff(dob, "years");
}
function daysUntilBirthday(dobStr) {
  const dob = moment(dobStr, "YYYY-MM-DD", true);
  if (!dob.isValid()) return null;

  const today = moment().startOf("day");
  let next = moment(dob).year(today.year()).startOf("day");
  if (next.isBefore(today, "day")) next = next.add(1, "year");
  return next.diff(today, "days");
}

function fileUrlFromPath(path) {
  const f = app.vault.getAbstractFileByPath(path);
  if (!f) return "";
  return app.vault.getResourcePath(f);
}

async function loadStateText() {
  const f = app.vault.getAbstractFileByPath(STATE_NOTE_PATH);
  if (!f) return "";
  return await app.vault.read(f);
}

// Expose a renderer so other sections can force an update.
const mount = this.container;
mount.innerHTML = "";

window.__gpTheme = UI;

window.__gpRenderProfile = async () => {
  const text = await loadStateText();

  const Name = readInline(text, "Name") || "Vadym Kharchenko";
  const DOB = readInline(text, "DOB") || "1998-10-05";

  const XP_Total = readInlineNumber(text, "XP_Total", 0);
  const XP_Streak = readInlineNumber(text, "XP_Streak", 0);

  const Tasks_Easy = readInlineNumber(text, "Tasks_Easy", 0);
  const Tasks_Normal = readInlineNumber(text, "Tasks_Normal", 0);
  const Tasks_Hard = readInlineNumber(text, "Tasks_Hard", 0);

  const age = ageFromDob(DOB);
  const daysToBday = daysUntilBirthday(DOB);

  const lvl = levelFromXp(XP_Total);
  const pct = Math.max(0, Math.min(1, lvl.xpIntoLevel / lvl.xpForNext));

  mount.innerHTML = "";

  const card = document.createElement("div");
  panel(card);
  card.style.display = "grid";
  card.style.gridTemplateColumns = `${AVATAR_SIZE}px 1fr`;
  card.style.gap = "12px";
  card.style.alignItems = "center";

  if (window.matchMedia("(max-width: 520px)").matches) {
    card.style.gridTemplateColumns = "1fr";
    card.style.justifyItems = "center";
  }

  const avatarWrap = document.createElement("div");
  avatarWrap.style.width = `${AVATAR_SIZE}px`;
  avatarWrap.style.height = `${AVATAR_SIZE}px`;
  avatarWrap.style.borderRadius = "999px";
  avatarWrap.style.overflow = "hidden";
  avatarWrap.style.border = `2px solid ${UI.title}`;
  avatarWrap.style.background = UI.panel2;

  const avatarImg = document.createElement("img");
  avatarImg.src = fileUrlFromPath(AVATAR_PATH);
  avatarImg.style.width = "100%";
  avatarImg.style.height = "100%";
  avatarImg.style.objectFit = "cover";
  avatarWrap.appendChild(avatarImg);

  const info = document.createElement("div");
  if (window.matchMedia("(max-width: 520px)").matches) info.style.textAlign = "center";

  const title = document.createElement("div");
  title.textContent = Name;
  title.style.fontSize = "22px";
  title.style.fontWeight = "900";
  title.style.color = UI.title;

  const meta = document.createElement("div");
  meta.style.marginTop = "6px";
  meta.style.opacity = "0.85";
  meta.style.display = "flex";
  meta.style.flexWrap = "wrap";
  meta.style.gap = "12px";
  meta.innerHTML = `<div>DOB: ${DOB}</div><div>Age: ${age ?? "?"}</div><div>Days until birthday: ${daysToBday ?? "?"}</div>`;

  const levelRow = document.createElement("div");
  levelRow.style.marginTop = "10px";
  levelRow.style.display = "flex";
  levelRow.style.justifyContent = "space-between";
  levelRow.style.alignItems = "baseline";
  levelRow.style.flexWrap = "wrap";
  levelRow.style.gap = "8px";
  levelRow.innerHTML = `
    <div style="font-weight:900">Level ${lvl.level} <span style="opacity:.7">(${Math.floor(XP_Total)} XP)</span></div>
    <div style="opacity:.75">Progress: ${lvl.xpIntoLevel}/${lvl.xpForNext}</div>
  `;

  const barWrap = document.createElement("div");
  barWrap.style.marginTop = "6px";
  barWrap.style.height = "8px";
  barWrap.style.borderRadius = "999px";
  barWrap.style.background = "rgba(255,255,255,0.06)";
  barWrap.style.border = `1px solid ${UI.border}`;
  barWrap.style.overflow = "hidden";

  const bar = document.createElement("div");
  bar.style.height = "100%";
  bar.style.width = `${pct * 100}%`;
  bar.style.background = UI.title;
  barWrap.appendChild(bar);

  const pills = document.createElement("div");
  pills.style.marginTop = "10px";
  hRow(pills);
  if (window.matchMedia("(max-width: 520px)").matches) pills.style.justifyContent = "center";

  pills.appendChild(pill(`Streak: ${XP_Streak}`));
  pills.appendChild(pill(`Easy: ${Tasks_Easy}`));
  pills.appendChild(pill(`Task: ${Tasks_Normal}`));
  pills.appendChild(pill(`Hard: ${Tasks_Hard}`));

  info.appendChild(title);
  info.appendChild(meta);
  info.appendChild(levelRow);
  info.appendChild(barWrap);
  info.appendChild(pills);

  card.appendChild(avatarWrap);
  card.appendChild(info);

  mount.appendChild(card);
};

await window.__gpRenderProfile();
```

![[Add Points]]

![[Badge Showcase]]

___
```dataviewjs

const page = dv.page(moment().format("YYYY-MM-DD"));

const hydration = page?.hydration ?? 0;

const goal = page?.hydration_goal ?? 80;

  

const pct = Math.min(100, Math.round((hydration / goal) * 100));

  

const container = this.container;

  

// === Outer wrapper ===

const wrap = document.createElement("div");

wrap.style.display = "flex";

wrap.style.justifyContent = "center";

wrap.style.margin = "30px 0";

  

// === Circle container ===

const SIZE = 220; // bigger circle

const RADIUS = 95; // bigger radius

const STROKE = 14; // thicker ring

  

const circle = document.createElement("div");

circle.style.position = "relative";

circle.style.width = `${SIZE}px`;

circle.style.height = `${SIZE}px`;

  

// === SVG circular progress ===

circle.innerHTML = `

<svg width="${SIZE}" height="${SIZE}">

<circle

cx="${SIZE/2}" cy="${SIZE/2}" r="${RADIUS}"

stroke="#303030"

stroke-width="${STROKE}"

fill="none"

/>

<circle

cx="${SIZE/2}" cy="${SIZE/2}" r="${RADIUS}"

stroke="#8be4b5"

stroke-width="${STROKE}"

fill="none"

stroke-linecap="round"

stroke-dasharray="${2 * Math.PI * RADIUS}"

stroke-dashoffset="${(1 - pct / 100) * (2 * Math.PI * RADIUS)}"

style="transition: stroke-dashoffset 0.4s ease;"

/>

</svg>

`;

  

// === Center icon + text ===

const center = document.createElement("div");

center.style.position = "absolute";

center.style.top = "0";

center.style.left = "0";

center.style.width = "100%";

center.style.height = "100%";

center.style.display = "flex";

center.style.flexDirection = "column";

center.style.alignItems = "center";

center.style.justifyContent = "center";

center.style.fontWeight = "700";

  

center.innerHTML = `

<div style="font-size:48px; margin-bottom:6px; color:#8be4b5;">💧</div>

<div style="font-size:20px; color:#dcdcdc;">${hydration} / ${goal} oz</div>

`;

  

circle.appendChild(center);

wrap.appendChild(circle);

container.appendChild(wrap);

```
```dataviewjs
// === Quick Add Hydration (preset + custom) ===
const DAILY_FOLDER = "4 Archives/Journal/Daily Notes";
const DAILY_FORMAT = "YYYY-MM-DD";
const amounts = [4, 8, 12, 16, 20];

const c = this.container;
c.innerHTML = "";
c.style.marginBottom = "16px";

const row = document.createElement("div");
row.style.display = "flex";
row.style.gap = "12px";
row.style.alignItems = "center";
row.style.flexWrap = "wrap";
c.appendChild(row);

function styleBtn(el) {
  el.style.fontSize = "18px";
  el.style.cursor = "pointer";
  el.style.padding = "6px 12px";
  el.style.borderRadius = "8px";
  el.style.background = "#1f1f1f";
  el.style.border = "1px solid #00000040";
  el.style.color = "#dcdcdc";
  el.style.transition = "0.15s";
  el.style.userSelect = "none";
  el.onmouseover = () => (el.style.background = "#2a2a2a");
  el.onmouseout  = () => (el.style.background = "#1f1f1f");
}

async function getDailyFile(dateMoment) {
  const path = `${DAILY_FOLDER}/${dateMoment.format(DAILY_FORMAT)}.md`;
  let file = app.vault.getAbstractFileByPath(path);
  if (!file) file = await app.vault.create(path, "");
  return file;
}

async function addHydration(amount) {
  const a = Number(amount);
  if (!Number.isFinite(a) || a <= 0) return new Notice("Enter a valid oz amount (e.g. 12 or 7.5)");

  const file = await getDailyFile(moment());
  let text = await app.vault.read(file);

  const re = /hydration::\s*([0-9]+(?:\.[0-9]+)?)/;
  const m = text.match(re);
  const cur = m ? Number(m[1]) : 0;
  const upd = Math.round((cur + a) * 10) / 10;

  text = m ? text.replace(re, `hydration:: ${upd}`) : `hydration:: ${upd}\n` + text;

  await app.vault.modify(file, text);
  new Notice(`+${a} oz (Total: ${upd} oz)`);
}

// preset buttons
amounts.forEach(a => {
  const b = document.createElement("div");
  b.textContent = `+${a} oz`;
  styleBtn(b);
  b.onclick = () => addHydration(a);
  row.appendChild(b);
});

// custom input group (input + "oz" suffix)
const inputWrap = document.createElement("div");
inputWrap.style.display = "flex";
inputWrap.style.alignItems = "center";
inputWrap.style.background = "#1f1f1f";
inputWrap.style.border = "1px solid #00000040";
inputWrap.style.borderRadius = "8px";
inputWrap.style.overflow = "hidden";

const input = document.createElement("input");
input.type = "number";
input.step = "0.1";
input.min = "0";
input.placeholder = "0.0";
input.style.fontSize = "18px";
input.style.padding = "6px 10px";
input.style.width = "90px";
input.style.border = "0";
input.style.outline = "none";
input.style.background = "transparent";
input.style.color = "#dcdcdc";

const suffix = document.createElement("div");
suffix.textContent = "oz";
suffix.style.padding = "6px 10px";
suffix.style.borderLeft = "1px solid #00000040";
suffix.style.color = "#dcdcdc";
suffix.style.opacity = "0.85";
suffix.style.fontWeight = "700";

inputWrap.appendChild(input);
inputWrap.appendChild(suffix);
row.appendChild(inputWrap);

// add button (same row)
const addBtn = document.createElement("div");
addBtn.textContent = "+ Add";
styleBtn(addBtn);

const submit = async () => {
  await addHydration(input.value);
  input.value = "";
};

addBtn.onclick = submit;
input.addEventListener("keydown", (e) => { if (e.key === "Enter") submit(); });

row.appendChild(addBtn);
```
___
```dataviewjs

const tasks = dv.page("1 Projects/TODO").file.tasks

.filter(t => t.section?.subpath === "Today Tasks");

const done = tasks.filter(t => t.completed).length;

const total = tasks.length;

const pct = total === 0 ? 0 : Math.round((done / total) * 100);

  

dv.header(2, `🗓️ Today Tasks`);


dv.taskList(tasks, false);

```
```dataviewjs
const page = dv.page("1 Projects/TODO");
const tasks = page.file.tasks;

// Exact section matcher
function count(section) {
    return tasks.filter(t => t.section?.subpath === section).length;
}

const data = [
    ["Today Tasks", count("Today Tasks")],
    ["High Priority", count("High Priority")],
    ["Medium Priority", count("Medium Priority")],
    ["Low Priority", count("Low Priority")],
    ["Weekend Tasks", count("Weekend Tasks")],
    ["Army Todo", count("Army Todo")],   // ← FIXED
    ["Work", count("Work")],
    ["Daily Template", count("Daily Template")],
    ["Monthly Checklist", count("Monthly Checklist")]
];

dv.header(2, "📊 Task Load Overview");

let html = `<div style="display:flex;flex-direction:column;gap:6px;">`;

for (let [label, value] of data) {
    const width = Math.min(100, value * 5); // scale factor
    html += `
    <div>
        <strong style="color:#8be4b5;">${label}</strong> — ${value}
        <div style="width:100%;background:#222;border-radius:6px;height:8px;margin-top:3px;">
            <div style="width:${width}%;height:100%;background:#8be4b5;border-radius:6px;"></div>
        </div>
    </div>`;
}

html += `</div>`;

dv.el("div", html);
```




```dataviewjs
const folder = "4 Archives/Journal/Daily Notes";

// === DATE INFO ===
const today = moment().startOf("day");
const todayFormatted = today.format("dddd, MMMM D, YYYY");
const dayOfYear = today.dayOfYear();

// === COLLECT EXACTLY 365 DAYS ENDING TODAY ===
let days = [];
for (let i = 364; i >= 0; i--) {
    const date = moment(today).subtract(i, "days");
    const filename = date.format("YYYY-MM-DD");
    const page = dv.page(`${folder}/${filename}`);

    days.push({
        date: date,
        exists: !!page
    });
}

// === COLORS (dark theme friendly) ===
const COLOR_FILLED = "#5bd38a";   // matte green
const COLOR_EMPTY = "#303030";    // lighter dark gray
const COLOR_TODAY = "#8be4b5";    // mint highlight
const COLOR_BORDER = "#00000030"; // subtle border

const container = this.container;

// === HEADER ===
const header = document.createElement("div");
header.style.textAlign = "center";
header.style.marginBottom = "12px";
header.style.fontSize = "20px";
header.style.fontWeight = "700";
header.style.color = "#8be4b5";

header.innerHTML = `
🗂️ Journal Tracker<br>
<span style="font-size:14px; font-weight:500; color:#cccccc;">
📅 ${todayFormatted} — Day ${dayOfYear} of ${today.year()}
</span>
`;

container.appendChild(header);

// === HEATMAP GRID ===
// Full width, responsive, today ends bottom-right
const heatmap = document.createElement("div");
heatmap.style.display = "grid";
heatmap.style.gridTemplateColumns = "repeat(auto-fill, minmax(18px, 1fr))";
heatmap.style.gap = "6px";
heatmap.style.width = "100%";
heatmap.style.marginTop = "10px";

days.forEach(d => {
    const box = document.createElement("div");

    box.style.width = "100%";
    box.style.aspectRatio = "1 / 1";
    box.style.borderRadius = "4px";
    box.style.border = `1px solid ${COLOR_BORDER}`;

    if (d.date.isSame(today, "day")) {
        box.style.background = COLOR_TODAY;
        box.title = `Today — ${d.date.format("YYYY-MM-DD")}`;
    } else {
        box.style.background = d.exists ? COLOR_FILLED : COLOR_EMPTY;
        box.title = d.date.format("YYYY-MM-DD");
    }

    heatmap.appendChild(box);
});

container.appendChild(heatmap);

// === STREAK ===
let streak = 0;
for (let i = days.length - 1; i >= 0; i--) {
    if (days[i].exists) streak++;
    else break;
}

const footer = document.createElement("div");
footer.style.marginTop = "16px";
footer.style.fontSize = "16px";
footer.style.fontWeight = "600";
footer.style.color = COLOR_FILLED;
footer.style.textAlign = "center";

footer.innerHTML = `
🔥 Current streak: ${streak} days
`;

container.appendChild(footer);
```



```dataviewjs
const folder = "4 Archives/Journal/Daily Notes";

// === COLLECT 365 DAYS OF MOOD ===
const today = moment().startOf("day");

let days = [];
for (let i = 364; i >= 0; i--) {
    const date = moment(today).subtract(i, "days");
    const filename = date.format("YYYY-MM-DD");
    const page = dv.page(`${folder}/${filename}`);

    days.push({
        date,
        mood: page?.mood ?? null
    });
}

// === BRIGHT COLORS (1–5) ===
const COLORS = {
    1: "#ff4d4d",  // red
    2: "#ffb84d",  // amber
    3: "#ffe44d",  // yellow
    4: "#b6ff4d",  // lime
    5: "#5bff4d"   // green
};

const COLOR_EMPTY = "#303030";
const COLOR_BORDER = "#00000030";

const container = this.container;

// === TITLE ONLY ===
const header = document.createElement("div");
header.style.textAlign = "center";
header.style.marginBottom = "12px";
header.style.fontSize = "20px";
header.style.fontWeight = "700";
header.style.color = "#8be4b5";

header.innerHTML = `🧠 Mood Tracker`;
container.appendChild(header);

// === HEATMAP GRID ===
const heatmap = document.createElement("div");
heatmap.style.display = "grid";
heatmap.style.gridTemplateColumns = "repeat(auto-fill, minmax(18px, 1fr))";
heatmap.style.gap = "6px";
heatmap.style.width = "100%";

days.forEach(d => {
    const box = document.createElement("div");

    box.style.width = "100%";
    box.style.aspectRatio = "1 / 1";
    box.style.borderRadius = "4px";
    box.style.border = `1px solid ${COLOR_BORDER}`;

    // === TODAY uses TODAY'S mood color ===
    if (d.date.isSame(today, "day")) {
        if (d.mood) {
            box.style.background = COLORS[d.mood];
            box.title = `Today — Mood: ${d.mood}`;
        } else {
            box.style.background = COLOR_EMPTY;
            box.title = `Today — No mood logged`;
        }
    }
    // === Past days ===
    else if (d.mood) {
        box.style.background = COLORS[d.mood];
        box.title = `${d.date.format("YYYY-MM-DD")} — Mood: ${d.mood}`;
    } else {
        box.style.background = COLOR_EMPTY;
        box.title = `${d.date.format("YYYY-MM-DD")} — No mood logged`;
    }

    heatmap.appendChild(box);
});

container.appendChild(heatmap);
```


```dataviewjs

```

```dataviewjs
const folder = "4 Archives/Journal/Daily Notes";

// === COLLECT 365 DAYS OF LOGINS ===
const today = moment().startOf("day");

let days = [];
for (let i = 364; i >= 0; i--) {
  const date = moment(today).subtract(i, "days");
  const filename = date.format("YYYY-MM-DD");
  const page = dv.page(`${folder}/${filename}`);

  const logged = page?.logged === true || String(page?.logged ?? "").toLowerCase() === "true";

  days.push({
    date,
    logged
  });
}

const COLOR_LOGGED = "#8be4b5";
const COLOR_TODAY = "#5bff4d";
const COLOR_EMPTY = "#303030";
const COLOR_BORDER = "#00000030";

const container = this.container;

// === TITLE ONLY ===
const header = document.createElement("div");
header.style.textAlign = "center";
header.style.marginBottom = "12px";
header.style.fontSize = "20px";
header.style.fontWeight = "700";
header.style.color = "#8be4b5";

header.innerHTML = `🗓️ Login Tracker`;
container.appendChild(header);

// === HEATMAP GRID ===
const heatmap = document.createElement("div");
heatmap.style.display = "grid";
heatmap.style.gridTemplateColumns = "repeat(auto-fill, minmax(18px, 1fr))";
heatmap.style.gap = "6px";
heatmap.style.width = "100%";

days.forEach(d => {
  const box = document.createElement("div");

  box.style.width = "100%";
  box.style.aspectRatio = "1 / 1";
  box.style.borderRadius = "4px";
  box.style.border = `1px solid ${COLOR_BORDER}`;

  // === TODAY ===
  if (d.date.isSame(today, "day")) {
    if (d.logged) {
      box.style.background = COLOR_TODAY;
      box.title = `Today — Logged`;
    } else {
      box.style.background = COLOR_EMPTY;
      box.title = `Today — Not logged`;
    }
  }
  // === Past days ===
  else if (d.logged) {
    box.style.background = COLOR_LOGGED;
    box.title = `${d.date.format("YYYY-MM-DD")} — Logged`;
  } else {
    box.style.background = COLOR_EMPTY;
    box.title = `${d.date.format("YYYY-MM-DD")} — Not logged`;
  }

  heatmap.appendChild(box);
});

container.appendChild(heatmap);
```

```dataviewjs
const folder = "4 Archives/Journal/Daily Notes";
const today = moment().startOf("day");

const COLORS_EMPTY = "#303030";
const COLOR_BORDER = "#00000030";

function colorForSleep(h) {
  const n = Number(h);
  if (!Number.isFinite(n)) return COLORS_EMPTY;
  if (n < 4) return "#ff4d4d";
  if (n < 6) return "#ffb84d";
  if (n < 7) return "#ffe44d";
  if (n < 8) return "#b6ff4d";
  return "#5bff4d";
}

let days = [];
for (let i = 364; i >= 0; i--) {
  const date = moment(today).subtract(i, "days");
  const filename = date.format("YYYY-MM-DD");
  const page = dv.page(`${folder}/${filename}`);

  const raw = page?.sleep ?? null;
  const sleep = Number(raw);

  // treat 0 or missing as "unset"
  days.push({
    date,
    sleep: (Number.isFinite(sleep) && sleep > 0) ? sleep : null
  });
}

// title
const container = this.container;
container.innerHTML = "";

const header = document.createElement("div");
header.style.textAlign = "center";
header.style.marginBottom = "12px";
header.style.fontSize = "20px";
header.style.fontWeight = "700";
header.style.color = "#8be4b5";
header.innerHTML = `😴 Sleep Tracker`;
container.appendChild(header);

// grid
const heatmap = document.createElement("div");
heatmap.style.display = "grid";
heatmap.style.gridTemplateColumns = "repeat(auto-fill, minmax(18px, 1fr))";
heatmap.style.gap = "6px";
heatmap.style.width = "100%";

days.forEach(d => {
  const box = document.createElement("div");
  box.style.width = "100%";
  box.style.aspectRatio = "1 / 1";
  box.style.borderRadius = "4px";
  box.style.border = `1px solid ${COLOR_BORDER}`;

  const bg = (d.sleep == null) ? COLORS_EMPTY : colorForSleep(d.sleep);
  box.style.background = bg;

  const ds = d.date.format("YYYY-MM-DD");
  box.title = (d.sleep == null) ? `${ds} — No sleep logged` : `${ds} — Sleep: ${d.sleep}h`;

  heatmap.appendChild(box);
});

container.appendChild(heatmap);
```
