
```dataviewjs
await (async () => {
  const root = this.container;

  const LOG_PATH="Life Game Profile/Game Log.md";
  const STATE_PATH="Life Game Profile/Profile State.md";
  const SKILLS=["Strength","Cardio","Focus","Discipline","Social"];
  const SKC={Strength:"#ffb86b",Cardio:"#5bc0de",Focus:"#c4b5fd",Discipline:"#8be4b5",Social:"#ff8fd1"};

  // theme (uses your gp theme if present)
  const UI = window.__gpTheme ?? {
    panel: "rgba(0,0,0,0.14)",
    panel2: "rgba(0,0,0,0.10)",
    border: "rgba(0,0,0,0.20)",
    title: "#8be4b5",
  };

  const TXT="rgba(255,255,255,.92)";
  const GRID="rgba(255,255,255,.18)";
  const AXIS="rgba(255,255,255,.28)";

  const F=p=>app.vault.getAbstractFileByPath(p);
  const read=async p=>{const f=F(p);return f?app.vault.read(f):""};
  const ensure=async(p,init="")=>F(p)||app.vault.create(p,init);
  const ymd=s=>String(s).slice(0,10);
  const sum=a=>a.reduce((s,x)=>s+x,0);

  const parse=raw=>{
    const a=[];
    for(const l of raw.split("\n")){
      const t=l.trim(); if(!t) continue;
      const j=(t.startsWith("- ")?t.slice(2):t).trim(); if(!j.startsWith("{")) continue;
      try{
        const o=JSON.parse(j); if(!o?.date) continue;
        a.push({date:ymd(o.date),delta:+o.delta||0,kind:String(o.kind||""),note:String(o.note||""),stat:o.stat?String(o.stat):null});
      }catch{}
    }
    a.sort((x,y)=>x.date<y.date?-1:x.date>y.date?1:0);
    return a;
  };

  const readInline=(txt,key)=>{
    const m=txt.match(new RegExp(`^\\s*${key}::\\s*(.*?)\\s*$`,"m"));
    return m?String(m[1]).replace(/^["']|["']$/g,"").trim():"";
  };
  const readNum=(txt,key,d=0)=>{const n=+readInline(txt,key);return Number.isFinite(n)?n:d};

  // ---- SVG helpers ----
  const svgEl = (w,h,inner,extra="") =>
    `<svg viewBox="0 0 ${w} ${h}" preserveAspectRatio="xMidYMid meet" ${extra} xmlns="http://www.w3.org/2000/svg">${inner}</svg>`;
  const line = (x1,y1,x2,y2,stroke=GRID,sw=1) => `<line x1="${x1}" y1="${y1}" x2="${x2}" y2="${y2}" stroke="${stroke}" stroke-width="${sw}" />`;
  const text = (x,y,t,fill=TXT,size=12,anchor="start",weight=600) =>
    `<text x="${x}" y="${y}" fill="${fill}" font-size="${size}" font-weight="${weight}" text-anchor="${anchor}" font-family="ui-sans-serif, system-ui">${t}</text>`;
  const polyline = (pts,stroke,sw=2,fill="none",opacity=1) =>
    `<polyline points="${pts.map(p=>p.join(",")).join(" ")}" stroke="${stroke}" stroke-width="${sw}" fill="${fill}" opacity="${opacity}" stroke-linejoin="round" stroke-linecap="round" />`;
  const circle = (cx,cy,r,fill="#fff",stroke="none",sw=0,op=1) =>
    `<circle cx="${cx}" cy="${cy}" r="${r}" fill="${fill}" stroke="${stroke}" stroke-width="${sw}" opacity="${op}" />`;

  const wrapSvg = (html, maxW=760) => {
    const d=document.createElement("div");
    d.style.cssText=`max-width:${maxW}px;margin:10px auto;`;
    d.innerHTML = html.replace("<svg", `<svg style="width:100%;height:auto;display:block;"`);
    return d;
  };

  function makeLineChart({labels, series, width=760, height=260, title=""}) {
    const padL=46, padR=12, padT=26, padB=34;
    const W=width, H=height, innerW=W-padL-padR, innerH=H-padT-padB;
    const all = series.flatMap(s=>s.values);
    const minV = Math.min(0, ...all);
    const maxV = Math.max(1, ...all);
    const y = v => padT + (maxV - v) / (maxV - minV) * innerH;
    const x = i => padL + (labels.length<=1?0:(i/(labels.length-1))*innerW);

    let g="";
    if(title) g += text(padL, 18, title, TXT, 14, "start", 900);

    const ticks=4;
    for(let i=0;i<=ticks;i++){
      const yy=padT + (i/ticks)*innerH;
      g += line(padL, yy, W-padR, yy, GRID, 1);
    }
    g += line(padL, padT, padL, H-padB, AXIS, 1.2);
    g += line(padL, H-padB, W-padR, H-padB, AXIS, 1.2);

    const maxXTicks=8;
    const step=Math.max(1, Math.ceil(labels.length / maxXTicks));
    for(let i=0;i<labels.length;i+=step){
      g += text(x(i), H-12, labels[i], "rgba(255,255,255,.75)", 10, "middle", 650);
    }

    for(const s of series){
      const pts=s.values.map((v,i)=>[x(i),y(v)]);
      g += polyline(pts, s.color, 2.4);
      for(const [px,py] of pts) g += circle(px,py,2.6,"#ffffff","rgba(255,255,255,.6)",1);
    }

    // legend (wrap)
    let lx = padL;
    let ly = H - padB + 18;
    for (const s of series) {
      g += `<rect x="${lx}" y="${ly-10}" width="10" height="8" rx="2" fill="${s.color}"/>`;
      g += text(lx+14, ly-3, s.name, TXT, 10, "start", 700);
      lx += (s.name.length*6.2 + 34);
      if (lx > W - 140) { lx = padL; ly += 14; }
    }

    return svgEl(W,H,g);
  }

  function makeRadar({valuesBySkill, width=760, height=420, title="Skills (RPG Radar)"}) {
    const W=width,H=height;
    const cx=W/2, cy=H/2+10;
    const r=Math.min(W,H)*0.33;

    const vals=SKILLS.map(s=>Math.max(0,valuesBySkill[s]||0));
    const maxV=Math.max(1,...vals);
    const norm=vals.map(v=>v/maxV);

    const ang=i=>(-Math.PI/2)+i*(2*Math.PI/SKILLS.length);
    const pt=(i,k=1)=>[cx+Math.cos(ang(i))*r*k, cy+Math.sin(ang(i))*r*k];

    let g="";
    g += text(20, 24, title, TXT, 16, "start", 900);

    const rings=5;
    for(let j=1;j<=rings;j++){
      const k=j/rings;
      const pts=SKILLS.map((_,i)=>pt(i,k));
      g += polyline([...pts,pts[0]], GRID, 1.3);
    }

    for(let i=0;i<SKILLS.length;i++){
      const [x2,y2]=pt(i,1);
      g += line(cx,cy,x2,y2,AXIS,1.5);
      const [lx,ly]=pt(i,1.18);
      g += text(lx, ly, SKILLS[i], TXT, 12, "middle", 800);
    }

    const polyPts=SKILLS.map((_,i)=>pt(i,norm[i]));
    g += `<polygon points="${polyPts.map(p=>p.join(",")).join(" ")}" fill="rgba(139,228,181,.22)"/>`;
    g += polyline([...polyPts,polyPts[0]], "rgba(255,255,255,.85)", 2.0);
    for(const [px,py] of polyPts) g += circle(px,py,3.0,"#ffffff","rgba(255,255,255,.7)",1);

    return svgEl(W,H,g);
  }

  // ---- load data ----
  await ensure(LOG_PATH,"");
  const ev=parse(await read(LOG_PATH));
  if(!ev.length){ dv.paragraph("No log events yet."); return; }

  const stateTxt = await read(STATE_PATH);
  const xpNow = readNum(stateTxt,"XP_Total",0);

  const byDay=new Map();
  for(const e of ev) byDay.set(e.date,(byDay.get(e.date)||0)+(+e.delta||0));
  const days=[...byDay.keys()].sort();
  const dayDelta=days.map(d=>byDay.get(d));
  let xp=xpNow-sum(dayDelta);
  const xpTrend=[]; for(const d of days){xp+=byDay.get(d);xpTrend.push(+xp.toFixed(0));}

  const statEvents=ev.filter(e=>e.kind==="stat"&&(e.stat||e.note));
  const statName=e=>e.stat||String(e.note||"").replace(/^STAT:/,"").trim();

  const totals=Object.fromEntries(SKILLS.map(s=>[s,0]));
  for(const e of statEvents){const s=statName(e); if(SKILLS.includes(s)) totals[s]+=(+e.delta||0);}
  const totalStatsAll=Object.values(totals).reduce((a,b)=>a+b,0);

  const wk=d=>moment(d,"YYYY-MM-DD").format("GGGG-[W]WW");
  const byWeek=new Map();
  const getW=w=>byWeek.get(w)||byWeek.set(w,Object.assign({__total:0},Object.fromEntries(SKILLS.map(s=>[s,0])))).get(w);
  for(const e of statEvents){
    const s=statName(e); if(!SKILLS.includes(s)) continue;
    const w=wk(e.date), o=getW(w), v=(+e.delta||0);
    o[s]+=v; o.__total+=v;
  }
  const weeks=[...byWeek.keys()].sort();
  const weekTotal=weeks.map(w=>byWeek.get(w).__total);
  const weekSeries=Object.fromEntries(SKILLS.map(s=>[s,weeks.map(w=>byWeek.get(w)[s])]));
  const thisWeek=moment().format("GGGG-[W]WW");
  const thisWeekTotal=(byWeek.get(thisWeek)?.__total)??0;

  const bestSkill=SKILLS.slice().sort((a,b)=>(totals[b]||0)-(totals[a]||0))[0];
  const fmt1 = n => (Math.round((Number(n)||0)*10)/10).toFixed(1);

  // ===== PRETTY TOP SECTION (no cards; styled panel + 2 columns) =====
  const top=document.createElement("div");
  top.style.cssText = `
    max-width:760px;margin:0 auto 12px auto;
    border:1px solid ${UI.border};
    background:${UI.panel};
    border-radius:14px;
    padding:14px;
  `;

  const topTitle=document.createElement("div");
  topTitle.textContent="Stats";
  topTitle.style.cssText=`font-weight:900;color:${UI.title};margin-bottom:10px;letter-spacing:.2px;`;
  top.appendChild(topTitle);

  const grid=document.createElement("div");
  grid.style.cssText="display:grid;grid-template-columns:1fr 1fr;gap:10px;";
  top.appendChild(grid);

  const item=(label,value,sub="")=>{
    const d=document.createElement("div");
    d.style.cssText=`padding:10px 12px;border:1px solid ${UI.border};background:${UI.panel2};border-radius:12px;`;
    d.innerHTML = `
      <div style="opacity:.82;font-size:12px;color:${TXT}">${label}</div>
      <div style="font-size:22px;font-weight:900;margin-top:4px;color:${TXT}">${value}</div>
      ${sub?`<div style="opacity:.72;font-size:12px;margin-top:6px;color:${TXT}">${sub}</div>`:""}
    `;
    return d;
  };

  grid.appendChild(item("Stats (this week)", fmt1(thisWeekTotal), "Weekly level (sum of stat gains)"));
  grid.appendChild(item("XP (current)", String(xpNow), "From Profile State"));
  grid.appendChild(item("Total stats (all time)", fmt1(totalStatsAll), "Sum of all stat deltas"));
  grid.appendChild(item("Top skill", `${bestSkill} (${fmt1(totals[bestSkill]||0)})`, "Highest total stat"));

  // mobile 1-col
  if (window.innerWidth < 520) grid.style.gridTemplateColumns="1fr";

  root.appendChild(top);

  // ===== Keep the rest (table + graphs) =====
  dv.header(3,"Skill totals (all time)");
  dv.table(["Skill","Total"], SKILLS.map(s=>[s,fmt1(totals[s])]));

  root.appendChild(wrapSvg(makeRadar({valuesBySkill: totals, width:760, height:420}), 760));
  root.appendChild(wrapSvg(makeLineChart({
    labels: weeks,
    series: [{name:'Weekly total stats ("Level")', color:"#8be4b5", values: weekTotal}],
    width: 760, height: 280, title: 'Weekly Stat Total ("Level")'
  }), 760));
  root.appendChild(wrapSvg(makeLineChart({
    labels: weeks,
    series: SKILLS.map(s=>({name:s, color:SKC[s], values: weekSeries[s]})),
    width: 760, height: 320, title:"Weekly Skill Trends"
  }), 760));
  root.appendChild(wrapSvg(makeLineChart({
    labels: days,
    series: [{name:"XP", color:"#8be4b5", values: xpTrend}],
    width: 760, height: 280, title:"XP Trend (by day)"
  }), 760));

  const last=ev.slice(-12).reverse().map(e=>[e.date,e.delta,e.kind,e.note]);
  dv.header(3,"Recent events");
  dv.table(["Date","Δ","Kind","Note"], last);
})();
```

