
```dataviewjs
await (async()=>{
  const DP="3 Resources/Fuel/Fuel Data.md", VP="3 Resources/Fuel/Vehicles.md", FT=["Regular","Midgrade","Premium","Diesel","E85"];
  const N=v=>Number.isFinite(+v)?+v:null, R=(x,d=2)=>Math.round(x*10**d)/10**d;
  const F=p=>app.vault.getAbstractFileByPath(p), RD=async p=>{const f=F(p);return f?app.vault.read(f):""}, EN=async p=>F(p)||app.vault.create(p,"");
  const J=s=>{const m=s.match(/\{[^]*?\}/g)||[];return m.map(x=>{try{return JSON.parse(x)}catch{return null}}).filter(Boolean)};
  const veh=async()=>{const t=await RD(VP), o=J(t).map(x=>x.name??x.vehicle??x.title).filter(Boolean).map(String), l=t.split("\n").map(x=>x.trim()).filter(x=>x.startsWith("- ")).map(x=>x.slice(2).trim()).filter(Boolean); const a=[...new Set([...o,...l])]; return a.length?a:["Civic"]};
  const rec=async()=>J(await RD(DP)).filter(x=>x?.date).map(x=>({...x,date:String(x.date)}));
  const lastO=(rs,v)=>{let m=null; for(const r of rs) if(r.vehicle===v && Number.isFinite(+r.odometer)) m=Math.max(m??-Infinity,+r.odometer); return m==null||m===-Infinity?null:m};

  const vs=await veh(), rs=await rec(), root=this.container; root.innerHTML=""; root.style.marginBottom="16px";
  const cssI="width:100%;padding:8px 10px;border-radius:10px;border:1px solid #00000040;background:#1f1f1f;color:#dcdcdc;outline:none;font-size:14px;";
  const cssL="font-size:12px;opacity:.75;font-weight:800;margin-bottom:6px;";
  const box=(lab,el,h="")=>{const d=document.createElement("div"); d.innerHTML=`<div style="${cssL}">${lab}</div>`; d.appendChild(el); if(h){const s=document.createElement("div"); s.textContent=h; s.style.cssText="margin-top:6px;font-size:12px;opacity:.6;"; d.appendChild(s);} return d};
  const mkI=(type,ph,step)=>{const i=document.createElement("input"); i.type=type; i.placeholder=ph||""; if(step) i.step=step; i.style.cssText=cssI; return i};
  const mkS=(arr)=>{const s=document.createElement("select"); s.style.cssText=cssI; arr.forEach(v=>{const o=document.createElement("option"); o.value=v; o.textContent=v; s.appendChild(o)}); return s};

  const card=document.createElement("div"); card.style.cssText="padding:14px;border-radius:12px;border:1px solid #00000040;background:rgba(0,0,0,0.08);"; root.appendChild(card);
  const h=document.createElement("div"); h.textContent="⛽ Refueling"; h.style.cssText="font-size:18px;font-weight:900;color:#8be4b5;margin-bottom:10px;"; card.appendChild(h);
  const grid=document.createElement("div"); grid.style.cssText="display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:12px;"; card.appendChild(grid);

  const vSel=mkS(vs); vSel.value=vs.includes("Civic")?"Civic":vs[0];
  const dInp=mkI("date"); dInp.value=moment().format("YYYY-MM-DD");
  const oInp=mkI("number","e.g. 123456");
  const gInp=mkI("number","e.g. 10.2","0.01");
  const fSel=mkS(FT); fSel.value="Regular";
  const pInp=mkI("number","e.g. 3.299","0.001");
  const tInp=mkI("number","e.g. 33.56","0.01");
  const mOut=mkI("text","Auto"); mOut.disabled=true; mOut.style.opacity=".75";

  const odoHint=document.createElement("div"); odoHint.style.cssText="margin-top:6px;font-size:12px;opacity:.6;";
  const odoBox=document.createElement("div"); odoBox.innerHTML=`<div style="${cssL}">Odometer</div>`; odoBox.appendChild(oInp); odoBox.appendChild(odoHint);

  const msg=document.createElement("div"); msg.style.cssText="margin-top:8px;font-size:12px;opacity:.75;white-space:pre-line;"; card.appendChild(msg);

  grid.appendChild(box("Vehicle",vSel));
  grid.appendChild(box("Date",dInp));
  grid.appendChild(odoBox);
  grid.appendChild(box("Gallons (gal)",gInp));
  grid.appendChild(box("Fuel Type",fSel));
  grid.appendChild(box("Price / gal",pInp));
  grid.appendChild(box("Total Cost",tInp));
  grid.appendChild(box("MPG (auto)",mOut));

  const upd=()=>{
    const v=vSel.value, last=lastO(rs,v);
    odoHint.textContent=last==null?"Last: —":`Last: ${last}`;
    const g=N(gInp.value), p=N(pInp.value), t=N(tInp.value), o=N(oInp.value);

    if(g!=null && p!=null && (t==null||tInp.value==="")) tInp.value=""+R(g*p,2);
    else if(g!=null && t!=null && (p==null||pInp.value==="")) pInp.value=""+R(t/g,3);

    let mpg=null;
    if(last!=null && o!=null && g!=null && g>0 && o>last) mpg=(o-last)/g;
    mOut.value=mpg==null?"":""+R(mpg,1);

    msg.textContent=(last==null?`Last odometer for ${v}: (none yet)`:`Last odometer for ${v}: ${last}`)+`\n`+(mpg==null?"MPG: needs odometer + gallons + previous odometer (and odo must increase).":"");
  };

  [gInp,pInp,tInp,oInp].forEach(x=>x.addEventListener("input",upd));
  vSel.addEventListener("change",upd);
  upd();

  const row=document.createElement("div"); row.style.cssText="display:flex;gap:12px;flex-wrap:wrap;margin-top:12px;"; card.appendChild(row);
  const btn=(txt,fn)=>{const b=document.createElement("div"); b.textContent=txt; b.style.cssText="cursor:pointer;padding:8px 12px;border-radius:10px;border:1px solid #00000040;background:#1f1f1f;color:#dcdcdc;font-weight:900;user-select:none;"; b.onclick=fn; row.appendChild(b); return b};

  btn("Save Refuel", async()=>{
    const vehicle=vSel.value, date=dInp.value||moment().format("YYYY-MM-DD");
    const odometer=N(oInp.value), gallons=N(gInp.value); let pricePerGallon=N(pInp.value), totalCost=N(tInp.value);
    const filled=[gallons!=null,pricePerGallon!=null,totalCost!=null].filter(Boolean).length;
    if(!vehicle) return new Notice("Pick a vehicle.");
    if(filled<2) return new Notice("Enter at least 2 of: gallons, price/gal, total cost.");

    if(gallons!=null && pricePerGallon!=null && totalCost==null) totalCost=gallons*pricePerGallon;
    if(gallons!=null && totalCost!=null && pricePerGallon==null) pricePerGallon=totalCost/gallons;

    const last=lastO(rs,vehicle);
    let mpg=null; if(last!=null && odometer!=null && gallons!=null && gallons>0 && odometer>last) mpg=(odometer-last)/gallons;

    const rec={date,vehicle,odometer:odometer??null,gallons:gallons??null,fuelType:fSel.value,pricePerGallon:pricePerGallon==null?null:R(pricePerGallon,3),totalCost:totalCost==null?null:R(totalCost,2),mpg:mpg==null?null:R(mpg,1)};
    const file=await EN(DP), prev=await app.vault.read(file);
    await app.vault.modify(file, prev + (prev.endsWith("\n")||prev.length===0?"":"\n") + `- ${JSON.stringify(rec)}\n`);
    new Notice(`Saved refuel: ${vehicle} • ${date}`);
  });

  btn("Clear", ()=>{oInp.value=gInp.value=pInp.value=tInp.value=mOut.value=""; upd();});
})();
```

___
```dataviewjs
const P="Fuel/Fuel Data.md", DEF="Civic";
const USD=n=>(+n||0).toLocaleString(undefined,{style:"currency",currency:"USD"});
const F=(n,d=1)=>Number.isFinite(+n)?(+n).toFixed(d):"—";
const M=d=>String(d).slice(0,7), S=a=>a.reduce((s,x)=>s+x,0);
const B="#5bc0de", G="#5bff4d", R="#ff6b6b";
const C=(cur,prev,lowBetter=0)=>!Number.isFinite(+cur)||!Number.isFinite(+prev)||+cur===+prev?B:((lowBetter?(+cur<+prev):(+cur>+prev))?G:R);

const parse=raw=>{
  const a=[];
  for(const l of raw.split("\n")){
    const t=l.trim(); if(!t) continue;
    const j=(t.startsWith("- ")?t.slice(2):t).trim(); if(!j.startsWith("{")) continue;
    try{
      const o=JSON.parse(j); if(!o?.date||!o?.vehicle) continue;
      a.push({d:""+o.date,t:o.time?""+o.time:"",v:""+o.vehicle,o:+o.odometer,g:+o.gallons,ppg:o.pricePerGallon==null?null:+o.pricePerGallon,c:o.totalCost==null?null:+o.totalCost});
    }catch{}
  }
  a.sort((x,y)=>((x.t?x.d+" "+x.t:x.d)<(y.t?y.d+" "+y.t:y.d)?-1:1));
  return a;
};

const f=app.vault.getAbstractFileByPath(P); if(!f){dv.paragraph("Fuel file missing: "+P); return;}
const all=parse(await app.vault.read(f)); if(!all.length){dv.paragraph("No fuel entries found."); return;}

const vs=[...new Set(all.map(x=>x.v))], pv=dv.current()?.vehicle?""+dv.current().vehicle:"";
const V=(pv&&vs.includes(pv))?pv:(vs.includes(DEF)?DEF:all.at(-1).v);
const r=all.filter(x=>x.v===V); dv.header(2,`⛽ Fuel — ${V}`);

let po=null, mpg=[];
for(const x of r){
  if(!Number.isFinite(x.o)||!Number.isFinite(x.g)||x.g<=0){ if(Number.isFinite(x.o)) po=x.o; continue; }
  if(po!=null && x.o>po) mpg.push({m:M(x.d),mpg:(x.o-po)/x.g,d:x.d});
  po=x.o;
}
const avgAll=mpg.length?S(mpg.map(x=>x.mpg))/mpg.length:null, lastM=mpg.at(-1)?.mpg ?? null;
const last=r.at(-1)||{}, prev=r.at(-2)||{};
const lastP=last.ppg, prevP=prev.ppg;

const bm=new Map(), get=m=>bm.get(m)||bm.set(m,{c:0,g:0,ps:0,pn:0,ms:0,mn:0}).get(m);
for(const x of r){ const m=M(x.d); if(!m) continue; const a=get(m); if(Number.isFinite(x.c)) a.c+=x.c; if(Number.isFinite(x.g)) a.g+=x.g; if(Number.isFinite(x.ppg)) (a.ps+=x.ppg,a.pn++); }
for(const x of mpg){ const a=get(x.m); (a.ms+=x.mpg,a.mn++); }

const ms=[...bm.keys()].sort(), cal=moment().format("YYYY-MM");
const cm=bm.has(cal)?cal:ms.at(-1), pm=moment(cm,"YYYY-MM").subtract(1,"month").format("YYYY-MM");
const a=bm.get(cm)||{c:0,g:0,ps:0,pn:0,ms:0,mn:0}, b=bm.get(pm)||{};
const cmM=a.mn?a.ms/a.mn:null, pmM=b.mn?b.ms/b.mn:null;
const cmP=a.pn?a.ps/a.pn:null, pmP=b.pn?b.ps/b.pn:null;

const wrap=dv.el("div",""); Object.assign(wrap.style,{display:"grid",gridTemplateColumns:"repeat(auto-fit,minmax(190px,1fr))",gap:"10px",margin:"10px 0 16px 0"});
const KPI=(t,v,s="",col="")=>{
  const d=document.createElement("div"); Object.assign(d.style,{padding:"12px",border:"1px solid rgba(255,255,255,0.12)",borderRadius:"14px",background:"rgba(30,30,30,0.65)"});
  d.innerHTML=`<div style="opacity:.85;font-size:12px">${t}</div><div style="font-size:22px;font-weight:800;margin-top:6px;${col?`color:${col}`:""}">${v}</div>${s?`<div style="opacity:.75;font-size:12px;margin-top:6px">${s}</div>`:""}`;
  wrap.appendChild(d);
};

KPI("Average MPG (all time)", avgAll==null?"—":F(avgAll,1));
KPI("Last MPG (computed)", lastM==null?"—":F(lastM,1), last.d?`Date: ${last.d}`:"", B);
KPI("Last price / gal", lastP==null?"—":USD(lastP), prevP==null?"":`Prev: ${USD(prevP)}`, C(lastP,prevP,1));

KPI(`Avg MPG (${cm})`, cmM==null?"—":F(cmM,1), pmM==null?`Prev: — (${pm})`:`Prev: ${F(pmM,1)} (${pm})`, C(cmM,pmM,0));
KPI(`Avg price / gal (${cm})`, cmP==null?"—":USD(cmP), pmP==null?`Prev: — (${pm})`:`Prev: ${USD(pmP)} (${pm})`, C(cmP,pmP,1));
KPI(`Cost (${cm})`, USD(a.c||0), (b.c==null)?`Prev: — (${pm})`:`Prev: ${USD(b.c||0)} (${pm})`, C(a.c,(b.c??NaN),1));
KPI(`Gallons (${cm})`, (a.g||0).toFixed(2), (b.g==null)?`Prev: — (${pm})`:`Prev: ${(b.g||0).toFixed(2)} (${pm})`, C(a.g,(b.g??NaN),1));
```

```dataviewjs
await (async () => {
  const P="Fuel/Fuel Data.md", DEF="Civic";
  const B="#5bc0de", G="#5bff4d", R="#ff6b6b", GRID="rgba(255,255,255,.12)", TXT="rgba(255,255,255,.85)";
  const QC=(c,w=900,h=320,bg="transparent")=>"https://quickchart.io/chart?"+new URLSearchParams({c:JSON.stringify(c),width:""+w,height:""+h,backgroundColor:bg});
  const M=d=>String(d).slice(0,7);
  const col=(cur,prev,lowBetter=0)=>!Number.isFinite(+cur)||!Number.isFinite(+prev)||+cur===+prev?B:((lowBetter?(+cur<+prev):(+cur>+prev))?G:R);

  const parse=raw=>{
    const a=[];
    for(const l of raw.split("\n")){
      const t=l.trim(); if(!t) continue;
      const j=(t.startsWith("- ")?t.slice(2):t).trim(); if(!j.startsWith("{")) continue;
      try{
        const o=JSON.parse(j); if(!o?.date||!o?.vehicle) continue;
        a.push({d:""+o.date,t:o.time?""+o.time:"",v:""+o.vehicle,o:+o.odometer,g:+o.gallons,ppg:o.pricePerGallon==null?null:+o.pricePerGallon,c:o.totalCost==null?null:+o.totalCost});
      }catch{}
    }
    a.sort((x,y)=>((x.t?x.d+" "+x.t:x.d)<(y.t?y.d+" "+y.t:y.d)?-1:1));
    return a;
  };

  const f=app.vault.getAbstractFileByPath(P); if(!f){dv.paragraph("Fuel file missing: "+P); return;}
  const all=parse(await app.vault.read(f)); if(!all.length){dv.paragraph("No fuel entries found."); return;}

  const vs=[...new Set(all.map(x=>x.v))], pv=dv.current()?.vehicle?""+dv.current().vehicle:"";
  const V=(pv&&vs.includes(pv))?pv:(vs.includes(DEF)?DEF:all.at(-1).v);
  const r=all.filter(x=>x.v===V);

  // MPG from odometer deltas
  let po=null; const mpg=[];
  for(const x of r){
    if(!Number.isFinite(x.o)||!Number.isFinite(x.g)||x.g<=0){ if(Number.isFinite(x.o)) po=x.o; continue; }
    if(po!=null && x.o>po) mpg.push({d:x.d,y:+(((x.o-po)/x.g).toFixed(2))});
    po=x.o;
  }

  // monthly agg: cost + avg price
  const mm=new Map(), get=m=>mm.get(m)||mm.set(m,{cost:0,ps:0,pn:0}).get(m);
  for(const x of r){
    const m=M(x.d); if(!m) continue;
    const a=get(m);
    if(Number.isFinite(x.c)) a.cost+=x.c;
    if(Number.isFinite(x.ppg)) (a.ps+=x.ppg,a.pn++);
  }
  const months=[...mm.keys()].sort();
  const mCost=months.map(m=>+((mm.get(m).cost||0).toFixed(2)));
  const mPpg=months.map(m=>{const a=mm.get(m); return a.pn?+((a.ps/a.pn).toFixed(3)):null;});

  dv.header(2,`⛽ Fuel — ${V}`);

  // small month/price line
  const lastM=months.at(-1), prevM=months.length>1?months.at(-2):null;
  const lastP=lastM?mPpg.at(-1):null, prevP=prevM?mPpg.at(-2):null;
  const s=document.createElement("div");
  s.style.cssText="opacity:.85;font-weight:800;margin:6px 0 10px 0;";
  s.innerHTML=`<span style="color:${B};">Month:</span> ${lastM??"—"} • <span style="color:${B};">Avg price:</span> <span style="color:${col(lastP,prevP,1)};">${lastP==null?"—":"$"+lastP.toFixed(3)}</span>`;
  dv.container.appendChild(s);

  const baseOpt=(title)=>({plugins:{legend:{display:false},title:{display:true,text:title}},scales:{x:{ticks:{autoSkip:true,maxTicksLimit:14,color:TXT},grid:{color:GRID}},y:{ticks:{color:TXT},grid:{color:GRID}}}});
  const img=(cfg,h=320)=>dv.el("img","",{attr:{src:QC(cfg,900,h)}});

  // Chart 1: MPG line
  if(mpg.length>=2){
    img({type:"line",data:{labels:mpg.map(x=>x.d),datasets:[{data:mpg.map(x=>x.y),borderColor:B,backgroundColor:"rgba(91,192,222,.15)",fill:true,pointRadius:2,tension:.25}]},options:baseOpt("MPG (computed from odometer)")});
  } else dv.paragraph("Not enough data to chart MPG.");

  // Chart 2: Monthly avg price/gal line
  if(months.length>=2 && mPpg.some(v=>v!=null)){
    img({type:"line",data:{labels:months,datasets:[{data:mPpg,borderColor:G,backgroundColor:"rgba(91,255,77,.12)",fill:true,pointRadius:2,tension:.25,spanGaps:true}]},options:baseOpt("Average Price / gal (Monthly)")});
  } else dv.paragraph("Not enough data to chart monthly price/gal.");

  // Chart 3: Monthly cost bar
  if(months.length>=1){
    img({type:"bar",data:{labels:months,datasets:[{data:mCost,backgroundColor:"rgba(91,192,222,.30)",borderColor:B,borderWidth:1}]},options:{...baseOpt("Cost by Month"),scales:{...baseOpt("").scales,y:{...baseOpt("").scales.y,beginAtZero:true}}}});
  }
})();
```


