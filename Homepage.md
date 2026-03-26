
[[TODO]] · [[TOFIX]] · [[WORK]] · [[ARMY]] · [[Financial Dashboard]] · [[Fuel]]
___
```dataviewjs
const STATE_NOTE_PATH = "Life Game Profile/Profile State.md";
const AVATAR_PATH = "Data/Resources/wallpaper_berserk.jpg";
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


```dataviewjs
const STATE_NOTE_PATH="Life Game Profile/Profile State.md",DAILY_FOLDER="Data/Journal",DAILY_FORMAT="YYYY-MM-DD",GAME_LOG_PATH="Life Game Profile/Game Log.md";

const XP={easy:1,normal:3,hard:10},XP_LOGGED_DAY=1,XP_BIRTHDAY_BONUS=500;
const XP_MESSUP=5,MESSUP_LABEL=`Mess-up -${XP_MESSUP}`,MESSUP_COLOR="#ff6b6b";

// activity scaling (STATS ONLY)
const ACT_MULT={easy:1,normal:1.5,hard:2};

// activities (base deltas BEFORE multiplier)
const WORK_XP=10;
const ACTIVITIES=[
  {id:"gym",label:"Strength",stats:{Strength:1}},
  {id:"run",label:"Cardio",stats:{Cardio:1}},
  {id:"deep",label:"Focus",stats:{Focus:1}},
  {id:"journal",label:"Discipline",stats:{Discipline:1}},
  {id:"social",label:"Social",stats:{Social:1}},
  {id:"mess",label:"Mess-up",stats:{Discipline:-1}},
  {id:"work",label:`Work (+${WORK_XP} XP + stats)`,xp:WORK_XP,stats:{Cardio:1,Focus:1,Discipline:1,Social:1}},
];

const UI=window.__gpTheme??{panel:"rgba(0,0,0,0.14)",panel2:"rgba(0,0,0,0.10)",border:"rgba(0,0,0,0.20)",title:"#8be4b5"};

const panel=e=>{e.style.background=UI.panel;e.style.border=`1px solid ${UI.border}`;e.style.borderRadius="10px";e.style.padding="12px"};
const hRow=e=>{e.style.display="flex";e.style.gap="10px";e.style.alignItems="center";e.style.flexWrap="wrap"};
const flatBtn=e=>{e.style.padding="8px 10px";e.style.borderRadius="8px";e.style.background=UI.panel2;e.style.border=`1px solid ${UI.border}`;e.style.cursor="pointer";e.style.userSelect="none";e.style.fontWeight="700";e.style.transition="background .12s ease";e.onmouseover=()=>e.style.background="rgba(255,255,255,0.06)";e.onmouseout=()=>e.style.background=UI.panel2};
const esc=s=>s.replace(/[.*+?^${}()|[\]\\]/g,"\\$&");
const read=(txt,key)=>{const m=txt.match(new RegExp(`^\\s*${esc(key)}::\\s*(.*?)\\s*$`,"m"));return m?String(m[1]).replace(/^["']|["']$/g,"").trim():""};
const readN=(txt,key,d=0)=>{const n=+read(txt,key);return Number.isFinite(n)?n:d};
const setF=(txt,key,val)=>{const r=new RegExp(`^\\s*${esc(key)}::\\s*.*$`,"m"),line=`${key}:: ${val}`;return r.test(txt)?txt.replace(r,line):`${line}\n${txt}`};

function initialStateText(){return`Name:: "Vadym Kharchenko"
DOB:: "1998-10-05"

XP_Total:: 0
XP_LastLogin:: ""
XP_Streak:: 0

Tasks_Easy:: 0
Tasks_Normal:: 0
Tasks_Hard:: 0

XP_BirthdayBonusYear:: 0
`}

async function ensureFile(path,init=""){let f=app.vault.getAbstractFileByPath(path);if(!f)f=await app.vault.create(path,init);return f}
async function ensureStateFile(){return ensureFile(STATE_NOTE_PATH,initialStateText())}
async function loadState(){const f=await ensureStateFile(),txt=await app.vault.read(f);return{DOB:read(txt,"DOB")||"1998-10-05",XP_Total:readN(txt,"XP_Total",0),XP_LastLogin:read(txt,"XP_LastLogin")||"",XP_Streak:readN(txt,"XP_Streak",0),Tasks_Easy:readN(txt,"Tasks_Easy",0),Tasks_Normal:readN(txt,"Tasks_Normal",0),Tasks_Hard:readN(txt,"Tasks_Hard",0),XP_BirthdayBonusYear:readN(txt,"XP_BirthdayBonusYear",0),__text:txt,__file:f}}
async function saveState(s){let t=s.__text;t=setF(t,"XP_BirthdayBonusYear",Math.max(0,Math.floor(s.XP_BirthdayBonusYear||0)));t=setF(t,"XP_Total",Math.max(0,s.XP_Total));t=setF(t,"XP_LastLogin",`"${s.XP_LastLogin||""}"`);t=setF(t,"XP_Streak",Math.max(0,s.XP_Streak));t=setF(t,"Tasks_Easy",Math.max(0,s.Tasks_Easy));t=setF(t,"Tasks_Normal",Math.max(0,s.Tasks_Normal));t=setF(t,"Tasks_Hard",Math.max(0,s.Tasks_Hard));await app.vault.modify(s.__file,t)}
async function getDailyFile(m){return ensureFile(`${DAILY_FOLDER}/${m.format(DAILY_FORMAT)}.md`,"")}
async function ensureLoggedTrueToday(){const f=await getDailyFile(moment().startOf("day"));let t=await app.vault.read(f);if(/^\s*logged::\s*true\s*$/m.test(t))return;t=/^\s*logged::\s*/m.test(t)?t.replace(/^\s*logged::\s*.*$/m,"logged:: true"):`logged:: true\n${t}`;await app.vault.modify(f,t)}
const isBday=dobStr=>{const d=moment(dobStr,"YYYY-MM-DD",true);if(!d.isValid())return false;const t=moment().startOf("day");return d.month()===t.month()&&d.date()===t.date()}
async function maybeBdayBonus(s){const y=+moment().format("YYYY");if(!isBday(s.DOB))return{applied:false,year:y};if(+s.XP_BirthdayBonusYear===y)return{applied:false,year:y};s.XP_Total+=XP_BIRTHDAY_BONUS;s.XP_BirthdayBonusYear=y;return{applied:true,year:y}}

// UPDATED: logEvent supports extra payload
async function logEvent(delta,kind,note="",extra={}){const f=await ensureFile(GAME_LOG_PATH,"");const prev=await app.vault.read(f);const rec={date:moment().format("YYYY-MM-DD"),ts:moment().format("YYYY-MM-DD HH:mm"),delta,kind,note,...extra};await app.vault.modify(f,prev+(prev.endsWith("\n")||prev.length===0?"":"\n")+`- ${JSON.stringify(rec)}\n`)}

async function mutate(mut){const s=await loadState();mut(s);const b=await maybeBdayBonus(s);await saveState(s);await refreshHint();if(window.__gpRenderProfile)await window.__gpRenderProfile();return b}

const scaleStats=(stats,m)=>{const o={};for(const[k,v]of Object.entries(stats||{}))o[k]=Math.round(v*m*10)/10;return o}

let selected=null;

// ---------- UI ----------
const root=this.container;root.innerHTML="";
const wrapper=document.createElement("div");wrapper.style.maxWidth="760px";wrapper.style.margin="0 auto";root.appendChild(wrapper);
const card=document.createElement("div");panel(card);wrapper.appendChild(card);

const title=document.createElement("div");title.textContent="Actions";title.style.fontWeight="900";title.style.color=UI.title;title.style.marginBottom="10px";card.appendChild(title);

const row=document.createElement("div");hRow(row);card.appendChild(row);

const makeBtn=label=>{const b=document.createElement("div");b.textContent=label;flatBtn(b);return b}
const makeDangerBtn=label=>{const b=makeBtn(label);b.style.border=`1px solid ${MESSUP_COLOR}55`;b.style.color=MESSUP_COLOR;b.onmouseover=()=>b.style.background="rgba(255,107,107,0.10)";b.onmouseout=()=>b.style.background=UI.panel2;return b}

const easyBtn=makeBtn(`Easy +${XP.easy}`),normalBtn=makeBtn(`Task +${XP.normal}`),hardBtn=makeBtn(`Hard +${XP.hard}`),logBtn=makeBtn(`Log today +${XP_LOGGED_DAY}`),messBtn=makeDangerBtn(MESSUP_LABEL);
row.appendChild(easyBtn);row.appendChild(normalBtn);row.appendChild(hardBtn);row.appendChild(logBtn);row.appendChild(messBtn);

// activity picker UNDER buttons (dark-mode friendly)
const pickRow=document.createElement("div");hRow(pickRow);pickRow.style.marginTop="10px";card.appendChild(pickRow);

const pickBtn=document.createElement("div");pickBtn.textContent="Pick activity…";flatBtn(pickBtn);pickRow.appendChild(pickBtn);

let open=false;
const menu=document.createElement("div");
menu.style.cssText=`display:none;margin-top:8px;border:1px solid ${UI.border};border-radius:10px;overflow:hidden;background:${UI.panel2};`;
card.appendChild(menu);

const menuItem=(txt,danger=false)=>{const it=document.createElement("div");it.textContent=txt;it.style.cssText=`padding:10px 12px;cursor:pointer;user-select:none;font-weight:800;color:${danger?MESSUP_COLOR:"#dcdcdc"};border-bottom:1px solid ${UI.border};`;it.onmouseover=()=>it.style.background="rgba(255,255,255,0.06)";it.onmouseout=()=>it.style.background="transparent";return it}

for(const a of ACTIVITIES){
  const it=menuItem(a.label,a.id==="mess");
  it.onclick=()=>{selected=a;pickBtn.textContent=a.label;open=false;menu.style.display="none"};
  menu.appendChild(it);
}
if(menu.lastChild)menu.lastChild.style.borderBottom="0";

pickBtn.onclick=()=>{open=!open;menu.style.display=open?"block":"none"};

const hint=document.createElement("div");hint.style.marginTop="10px";hint.style.opacity="0.75";hint.style.fontSize="12px";card.appendChild(hint);

async function refreshHint(){const s=await loadState();hint.textContent=`XP: ${s.XP_Total} • Last login: ${s.XP_LastLogin||"—"} • Birthday bonus: ${isBday(s.DOB)?"today":"not today"}`}
await refreshHint();

async function applySelectedActivity(mult,label){
  if(!selected) return;

  // bundle (readable)
  const scaled=scaleStats(selected.stats,mult);
  await logEvent(0,"activity",`${selected.id} (${label} x${mult})`,{activity:selected.id,mult,difficulty:label,stats:scaled});

  // per-stat (chart-friendly)
  for(const [stat,delta] of Object.entries(scaled)){
    if(!delta) continue;
    await logEvent(delta,"stat",`STAT:${stat}`,{stat,delta,activity:selected.id,mult,difficulty:label});
  }

  // Work XP is NOT scaled (stats only). Work XP still applies if selected.
  if(selected.id==="work" && selected.xp){
    await mutate(s=>{s.XP_Total+=selected.xp});
    await logEvent(+selected.xp,"work","Work",{activity:"work"});
  }
}

// Buttons + logging
easyBtn.onclick=async()=>{await mutate(s=>{s.XP_Total+=XP.easy;s.Tasks_Easy+=1});await logEvent(+XP.easy,"task","Easy");await applySelectedActivity(ACT_MULT.easy,"Easy");new Notice(`+${XP.easy} XP`)};
normalBtn.onclick=async()=>{await mutate(s=>{s.XP_Total+=XP.normal;s.Tasks_Normal+=1});await logEvent(+XP.normal,"task","Normal");await applySelectedActivity(ACT_MULT.normal,"Normal");new Notice(`+${XP.normal} XP`)};
hardBtn.onclick=async()=>{await mutate(s=>{s.XP_Total+=XP.hard;s.Tasks_Hard+=1});await logEvent(+XP.hard,"task","Hard");await applySelectedActivity(ACT_MULT.hard,"Hard");new Notice(`+${XP.hard} XP`)};

logBtn.onclick=async()=>{
  const today=moment().startOf("day"),todayStr=today.format("YYYY-MM-DD");
  await ensureLoggedTrueToday();
  const bonus=await mutate(s=>{
    const last=s.XP_LastLogin?moment(s.XP_LastLogin,"YYYY-MM-DD",true):null;
    const already=last&&last.isValid()&&last.isSame(today,"day");
    if(already) return;
    s.XP_Total+=XP_LOGGED_DAY;
    if(last&&last.isValid()&&last.isSame(moment(today).subtract(1,"day"),"day")) s.XP_Streak=(s.XP_Streak||0)+1; else s.XP_Streak=1;
    s.XP_LastLogin=todayStr;
  });
  const s=await loadState();
  const justLogged=s.XP_LastLogin===todayStr;
  if(justLogged) await logEvent(+XP_LOGGED_DAY,"login","Logged today");
  if(bonus.applied) await logEvent(+XP_BIRTHDAY_BONUS,"bonus","Birthday");
  new Notice(bonus.applied?`Logged today (+${XP_LOGGED_DAY}) + Birthday bonus (+${XP_BIRTHDAY_BONUS})`:`Logged today (+${XP_LOGGED_DAY} XP if not already logged)`);
};

messBtn.onclick=async()=>{await mutate(s=>{s.XP_Total=Math.max(0,(s.XP_Total||0)-XP_MESSUP)});await logEvent(-XP_MESSUP,"penalty","Mess-up");new Notice(`-${XP_MESSUP} XP`)};
```

```dataviewjs
await (async () => {
  // ---- paths ----
  const PROFILE_PAGE = "Life Game Profile/Profile State";
  const FIN_DASH_PAGE = "Fianances/Financial Dashboard";
  const TX_PATH = "Fianances/Transaction";
  const DAILY_FOLDER = "Data/Journal";

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
[[Personal Stats]] · [[Achievements]]
___
```dataviewjs
const todayName = moment().format("YYYY-MM-DD"); 
const page = dv.pages().where(p => p.file?.name === todayName).first();
const water = Number(page?.water ?? page?.hydration ?? 0) || 0;
const other = Number(page?.other_liquid ?? 0) || 0;
const goal  = Number(page?.hydration_goal ?? 80) || 80;

const total = water + other;
const pct = Math.max(0, Math.min(100, Math.round((total / goal) * 100)));

const SIZE=220, R=95, ST=14, CIRC=2*Math.PI*R;
const clamp01=x=>Math.max(0,Math.min(1,x));
const wLen=CIRC*clamp01(water/goal);
const oLen=CIRC*clamp01(other/goal);

// --- recommended ratio (edit these) ---
const REC_WATER = 0.80; // 80%
const REC_OTHER = 0.20; // 20%
const recWaterOz = Math.round(goal * REC_WATER * 10) / 10;
const recOtherOz = Math.round(goal * REC_OTHER * 10) / 10;

const dWater = Math.round((water - recWaterOz) * 10) / 10;
const dOther = Math.round((other - recOtherOz) * 10) / 10;
const sign = (x)=> x>0 ? `+${x}` : `${x}`;

const wrap = document.createElement("div");
wrap.style.cssText="display:flex;flex-direction:column;align-items:center;margin:24px 0;";

const circle = document.createElement("div");
circle.style.cssText=`position:relative;width:${SIZE}px;height:${SIZE}px;`;

circle.innerHTML = `
<svg width="${SIZE}" height="${SIZE}" viewBox="0 0 ${SIZE} ${SIZE}">
  <g transform="rotate(-90 ${SIZE/2} ${SIZE/2})">
    <circle cx="${SIZE/2}" cy="${SIZE/2}" r="${R}" stroke="#303030" stroke-width="${ST}" fill="none" />
    <circle cx="${SIZE/2}" cy="${SIZE/2}" r="${R}"
      stroke="#8be4b5" stroke-width="${ST}" fill="none" stroke-linecap="round"
      stroke-dasharray="${wLen} ${CIRC-wLen}" stroke-dashoffset="0" />
    <circle cx="${SIZE/2}" cy="${SIZE/2}" r="${R}"
      stroke="#5bc0de" stroke-width="${ST}" fill="none" stroke-linecap="butt"
      stroke-dasharray="${oLen} ${CIRC-oLen}" stroke-dashoffset="${-wLen}" />
  </g>
</svg>
`;

const center = document.createElement("div");
center.style.cssText=`
  position:absolute;inset:0;
  display:flex;flex-direction:column;align-items:center;justify-content:center;
  font-weight:800;text-align:center;
  pointer-events:none;
`;

const inner = document.createElement("div");
inner.style.cssText=`width:70%;max-width:150px;line-height:1.15;`;

const line1 = document.createElement("div");
line1.textContent = "💧";
line1.style.cssText="font-size:18px;margin-bottom:6px;color:#8be4b5;";

const mainText = `${total} / ${goal} oz (${pct}%)`;
const line2 = document.createElement("div");
line2.textContent = mainText;
line2.style.cssText=`font-size:${mainText.length>16?14:18}px;color:#dcdcdc;white-space:normal;word-break:break-word;`;

inner.appendChild(line1);
inner.appendChild(line2);
center.appendChild(inner);

circle.appendChild(center);
wrap.appendChild(circle);

// details + recommended ratio (outside)
const details = document.createElement("div");
details.style.cssText="margin-top:10px;font-size:12px;opacity:.85;color:#dcdcdc;text-align:center;line-height:1.45;";
details.innerHTML = `
  <div>
    <span style="color:#8be4b5;font-weight:900;">Water:</span> ${water} oz
    &nbsp;•&nbsp;
    <span style="color:#5bc0de;font-weight:900;">Other:</span> ${other} oz
  </div>
  <div style="opacity:.8;margin-top:4px;">
    Recommended split: <b>${Math.round(REC_WATER*100)}%</b> water / <b>${Math.round(REC_OTHER*100)}%</b> other
    <br/>
    Target: <span style="color:#8be4b5;font-weight:900;">${recWaterOz} oz</span> water (${sign(dWater)}),
    <span style="color:#5bc0de;font-weight:900;">${recOtherOz} oz</span> other (${sign(dOther)})
  </div>
`;
wrap.appendChild(details);

this.container.innerHTML="";
this.container.appendChild(wrap);
```
```dataviewjs
// === Quick Add Hydration (preset + custom) ===
const DAILY_FOLDER = "Data/Journal";
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

function styleBtn(el, active=false) {
  el.style.fontSize = "18px";
  el.style.cursor = "pointer";
  el.style.padding = "6px 12px";
  el.style.borderRadius = "8px";
  el.style.background = active ? "#2a2a2a" : "#1f1f1f";
  el.style.border = "1px solid #00000040";
  el.style.color = "#dcdcdc";
  el.style.transition = "0.15s";
  el.style.userSelect = "none";
  el.onmouseover = () => (el.style.background = "#2a2a2a");
  el.onmouseout  = () => (el.style.background = active ? "#2a2a2a" : "#1f1f1f");
}

async function getDailyFile(dateMoment) {
  const path = `${DAILY_FOLDER}/${dateMoment.format(DAILY_FORMAT)}.md`;
  let file = app.vault.getAbstractFileByPath(path);
  if (!file) file = await app.vault.create(path, "");
  return file;
}

function upsertNumberField(text, key, nextVal) {
  const re = new RegExp(`^${key}::\\s*([0-9]+(?:\\.[0-9]+)?)\\s*$`, "m");
  const m = text.match(re);
  return m ? text.replace(re, `${key}:: ${nextVal}`) : `${key}:: ${nextVal}\n` + text;
}
function readNumberField(text, key) {
  const re = new RegExp(`^${key}::\\s*([0-9]+(?:\\.[0-9]+)?)\\s*$`, "m");
  const m = text.match(re);
  return m ? Number(m[1]) : 0;
}

// --- selection (Water vs Other) ---
let mode = "water"; // "water" | "other"

const selWrap = document.createElement("div");
selWrap.style.display = "flex";
selWrap.style.border = "1px solid #00000040";
selWrap.style.borderRadius = "10px";
selWrap.style.overflow = "hidden";
row.appendChild(selWrap);

function mkSelBtn(label, value, color) {
  const b = document.createElement("div");
  b.textContent = label;
  b.style.fontSize = "16px";
  b.style.fontWeight = "900";
  b.style.padding = "6px 10px";
  b.style.cursor = "pointer";
  b.style.userSelect = "none";
  b.style.background = "#1f1f1f";
  b.style.color = "#dcdcdc";
  b.style.borderRight = "1px solid #00000040";
  b.onclick = () => { mode = value; refreshSel(); };
  b.dataset.value = value;
  b.dataset.color = color;
  return b;
}
const waterBtn = mkSelBtn("Water", "water", "#8be4b5");
const otherBtn = mkSelBtn("Other", "other", "#5bc0de");
otherBtn.style.borderRight = "0";
selWrap.appendChild(waterBtn);
selWrap.appendChild(otherBtn);

function refreshSel() {
  [waterBtn, otherBtn].forEach(b => {
    const active = b.dataset.value === mode;
    b.style.background = active ? "#2a2a2a" : "#1f1f1f";
    b.style.color = active ? b.dataset.color : "#dcdcdc";
  });
}
refreshSel();

async function addHydration(amount) {
  const a = Number(amount);
  if (!Number.isFinite(a) || a <= 0) return new Notice("Enter a valid oz amount (e.g. 12 or 7.5)");

  const file = await getDailyFile(moment());
  let text = await app.vault.read(file);

  const key = (mode === "other") ? "other_liquid" : "water";

  const cur = readNumberField(text, key);
  const upd = Math.round((cur + a) * 10) / 10;
  text = upsertNumberField(text, key, upd);

  // keep hydration:: as total (water + other)
  const w = (key === "water") ? upd : readNumberField(text, "water");
  const o = (key === "other_liquid") ? upd : readNumberField(text, "other_liquid");
  const total = Math.round((w + o) * 10) / 10;
  text = upsertNumberField(text, "hydration", total);

  await app.vault.modify(file, text);
  new Notice(`+${a} oz to ${mode} (Total: ${upd} oz) • Hydration: ${total} oz`);
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

const tasks = dv.page("TODO").file.tasks

.filter(t => t.section?.subpath === "Today Tasks");

const done = tasks.filter(t => t.completed).length;

const total = tasks.length;

const pct = total === 0 ? 0 : Math.round((done / total) * 100);

  

dv.header(2, `🗓️ Today Tasks`);


dv.taskList(tasks, false);

```

```dataviewjs
const page = dv.page("TODO");
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
const folder = "Data/Journal";

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
const folder = "Data/Journal";

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
const folder = "Data/Journal";

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
const folder = "Data/Journal";
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

[[Vault Changelog]] · 