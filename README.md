
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>字里行间</title>
<style>
  html,body{margin:0;padding:0;background:#0b0b12;height:100%;overflow:hidden;
    font-family:"KaiTi","STKaiti","SimSun",serif;}
  #wrap{position:fixed;inset:0;display:flex;align-items:center;justify-content:center;}
  canvas{background:#0b0b12;image-rendering:auto;box-shadow:0 0 60px rgba(0,0,0,.8);}
</style>
</head>
<body>
<div id="wrap"><canvas id="cv" width="960" height="540"></canvas></div>
<script src="chaizi.js"></script>
<script>
"use strict";
/* =========================================================
   字里行间 · 叙事解谜冒险 + 轻经营
   白天做菜维持"字"的日常，夜晚探索推动"人"的觉醒。
   ========================================================= */
const cv = document.getElementById('cv'), ctx = cv.getContext('2d');
const W = 960, H = 540, GROUND = 432;
const INK = '#e9e2d0', INK_DIM = '#9a927e', RED = '#c8443a', GOLD = '#d8a94a';
const FONT = '"KaiTi","STKaiti","SimSun",serif';

/* ---------- 自适应 ---------- */
function fit(){
  const s = Math.min(innerWidth/W, innerHeight/H);
  cv.style.width = W*s+'px'; cv.style.height = H*s+'px';
}
addEventListener('resize', fit); fit();

/* ---------- 音效 ---------- */
let AC = null, MUTE = false;
function beep(f=440, d=0.08, type='sine', vol=0.12){
  if(MUTE) return;
  try{
    AC = AC || new (window.AudioContext||window.webkitAudioContext)();
    const o = AC.createOscillator(), g = AC.createGain();
    o.type = type; o.frequency.value = f;
    g.gain.setValueAtTime(vol, AC.currentTime);
    g.gain.exponentialRampToValueAtTime(0.001, AC.currentTime+d);
    o.connect(g); g.connect(AC.destination);
    o.start(); o.stop(AC.currentTime+d);
  }catch(e){}
}

/* ---------- 全局状态 ---------- */
const G = {
  state:'title',      // title intro cook cooksum night final over
  day:1, coins:0, awaken:0,
  inv:{},             // 笔画库存: 一 丨 囗 口 小
  items:{},           // 火 巾 墨块 钥 墨笔
  flags:{},           // 一次性标记
  form:'人',
  time:0,
};
function girlStage(){
  const t=[0,2,4,7,10]; let s=0;
  for(let i=0;i<t.length;i++) if(G.awaken>=t[i]) s=i;
  return Math.min(s,4);
}
function addAwaken(n, why){
  const before = girlStage();
  G.awaken += n;
  if(girlStage()>before) toast('她好像……多出了几根线条。');
  beep(660,0.15,'sine',0.08);
}
function invAdd(ch,n=1){ G.inv[ch]=(G.inv[ch]||0)+n; }

/* ---------- 存档 ---------- */
const SAVE_KEY='zljh_save_v1';
function save(phase){
  localStorage.setItem(SAVE_KEY, JSON.stringify({
    day:G.day, phase, coins:G.coins, awaken:G.awaken,
    inv:G.inv, items:G.items, flags:G.flags
  }));
}
function loadSave(){
  try{ return JSON.parse(localStorage.getItem(SAVE_KEY)); }catch(e){ return null; }
}
function clearSave(){ localStorage.removeItem(SAVE_KEY); }

/* ---------- 拆字（CSV + 额外表） ---------- */
const EXTRA_DEC = { '灯':['火','丁'], '味':['口','未'], '锁':['小','金','贝'], '墙':['土','啬'] };
const EXTRA_COM = { '门\u0001口':'问' };
/* 内置拆字表（与 chaizi.csv 一致）：fetch 失败（如 file:// 直开）时兜底，防止卡关 */
const BUILTIN_DEC = { '好':['女','子'],'明':['日','月'],'休':['亻','木'],'林':['木','木'],'森':['木','木','木'],
  '鲜':['鱼','羊'],'男':['田','力'],'尖':['小','大'],'尘':['小','土'],'歪':['不','正'],'泪':['氵','目'],
  '功':['工','力'],'妈':['女','马'],'们':['亻','门'],'叶':['口','十'],'回':['囗','口'],'早':['日','十'],
  '古':['十','口'],'杏':['木','口'],'呆':['口','木'] };
function chai(ch){
  const a = (typeof decompose==='function') ? decompose(ch) : [];
  if(a && a.length) return a;
  if(EXTRA_DEC[ch]) return EXTRA_DEC[ch].slice();
  return BUILTIN_DEC[ch] ? BUILTIN_DEC[ch].slice() : [];
}
function he(parts){
  if(typeof combine==='function'){
    const c = combine(parts); if(c) return c;
  }
  return EXTRA_COM[parts.join('\u0001')] || '';
}

/* ---------- 输入 ---------- */
const keys = {};
let mouse = {x:0,y:0,down:false,clicked:false};
addEventListener('keydown', e=>{
  if([' ','ArrowLeft','ArrowRight','ArrowUp','ArrowDown'].includes(e.key)) e.preventDefault();
  keys[e.key.toLowerCase()] = true;
  if(e.repeat) return;
  onKey(e.key.toLowerCase());
});
addEventListener('keyup', e=>{ keys[e.key.toLowerCase()] = false; });
cv.addEventListener('mousemove', e=>{
  const r = cv.getBoundingClientRect();
  mouse.x = (e.clientX-r.left)/r.width*W;
  mouse.y = (e.clientY-r.top)/r.height*H;
});
cv.addEventListener('mousedown', ()=>{ mouse.down=true; mouse.clicked=true; });
addEventListener('mouseup', ()=>{ mouse.down=false; });

/* ---------- 立即模式 UI 按钮 ---------- */
let uiRects = [];
function uiBegin(){ uiRects = []; }
function btn(x,y,w,h,label,opt={}){
  const hov = mouse.x>=x && mouse.x<=x+w && mouse.y>=y && mouse.y<=y+h;
  ctx.save();
  ctx.globalAlpha = opt.disabled?0.35:1;
  ctx.fillStyle = hov ? 'rgba(200,68,58,0.25)' : 'rgba(233,226,208,0.08)';
  ctx.strokeStyle = hov ? RED : INK_DIM;
  ctx.lineWidth = 1.5;
  roundRect(x,y,w,h,6); ctx.fill(); ctx.stroke();
  ctx.fillStyle = hov ? '#fff' : INK;
  ctx.font = (opt.size||22)+'px '+FONT;
  ctx.textAlign='center'; ctx.textBaseline='middle';
  ctx.fillText(label, x+w/2, y+h/2+1);
  ctx.restore();
  if(hov && mouse.clicked && !opt.disabled){ beep(520,0.06,'triangle',0.1); opt.fn && opt.fn(); return true; }
  return false;
}
function roundRect(x,y,w,h,r){
  ctx.beginPath();
  ctx.moveTo(x+r,y); ctx.arcTo(x+w,y,x+w,y+h,r); ctx.arcTo(x+w,y+h,x,y+h,r);
  ctx.arcTo(x,y+h,x,y,r); ctx.arcTo(x,y,x+w,y,r); ctx.closePath();
}
function hitRect(x,y,w,h,fn){
  if(mouse.x>=x && mouse.x<=x+w && mouse.y>=y && mouse.y<=y+h && mouse.clicked){ fn(); return true; }
  return false;
}

/* ---------- 文字绘制 ---------- */
function CH(ch,x,y,size,color=INK,align='center',alpha=1,glow=0){
  ctx.save();
  ctx.globalAlpha = alpha;
  ctx.font = size+'px '+FONT;
  ctx.textAlign = align; ctx.textBaseline = 'middle';
  if(glow){ ctx.shadowColor = color; ctx.shadowBlur = glow; }
  ctx.fillStyle = color;
  ctx.fillText(ch,x,y);
  ctx.restore();
}
function toast(t){ toasts.push({t, ttl:3.2}); }
let toasts = [];
function drawToasts(dt){
  toasts.forEach(o=>o.ttl-=dt);
  toasts = toasts.filter(o=>o.ttl>0);
  ctx.save(); ctx.textAlign='center';
  toasts.forEach((o,i)=>{
    ctx.globalAlpha = Math.min(1,o.ttl);
    ctx.font = '20px '+FONT; ctx.fillStyle = GOLD;
    ctx.fillText(o.t, W/2, 86+i*28);
  });
  ctx.restore();
}

/* ---------- 对话系统 ---------- */
let dlg = null; // {lines:[{sp,tx,choices}], i, ch, cb}
function say(lines, cb){ dlg = {lines, i:0, ch:0, cb}; }
function dlgAdvance(){
  if(!dlg) return;
  const L = dlg.lines[dlg.i];
  if(dlg.ch < L.tx.length){ dlg.ch = L.tx.length; return; } // 先补完
  if(L.choices) return; // 选项必须点击
  dlg.i++; dlg.ch = 0;
  beep(700,0.03,'square',0.04);
  if(dlg.i>=dlg.lines.length){ const cb=dlg.cb; dlg=null; cb&&cb(); }
}
function drawDlg(dt){
  if(!dlg) return;
  const L = dlg.lines[dlg.i];
  dlg.ch = Math.min(L.tx.length, dlg.ch + dt*22);
  ctx.save();
  ctx.fillStyle='rgba(8,8,14,0.88)';
  ctx.strokeStyle=INK_DIM; ctx.lineWidth=1.5;
  roundRect(60,382,840,140,8); ctx.fill(); ctx.stroke();
  if(L.sp){
    ctx.fillStyle = L.sp==='系统' ? '#7fb3d8' : (L.sp==='她' ? '#e8a0b4' : GOLD);
    ctx.font='22px '+FONT; ctx.textAlign='left';
    ctx.fillText(L.sp, 84, 406);
  }
  ctx.fillStyle=INK; ctx.font='23px '+FONT; ctx.textAlign='left';
  wrapText(L.tx.slice(0,Math.floor(dlg.ch)), 84, L.sp?440:414, 800, 32);
  if(dlg.ch>=L.tx.length){
    if(L.choices){
      L.choices.forEach((c,i)=>{
        btn(220+i*180, 466, 150, 40, c.t, {fn:()=>{ const f=c.fn; dlg=null; f&&f(); }});
      });
    }else{
      ctx.fillStyle=INK_DIM; ctx.font='16px '+FONT; ctx.textAlign='right';
      ctx.fillText('空格 / E 继续 ▸', 884, 508);
    }
  }
  ctx.restore();
}
function wrapText(t,x,y,maxw,lh){
  let line='',yy=y;
  for(const c of t){
    if(ctx.measureText(line+c).width>maxw || c==='\n'){ ctx.fillText(line,x,yy); line=(c==='\n'?'':c); yy+=lh; }
    else line+=c;
  }
  ctx.fillText(line,x,yy);
}

/* ---------- 拆字覆盖层 ---------- */
let chaiUI = null; // {glyph,parts:[{ch}],note,onTake}
function openChai(glyph, onTake, note){
  const parts = chai(glyph);
  if(!parts.length){ say([{sp:'',tx:'「'+glyph+'」字浑然一体，拆不开。'}]); return; }
  chaiUI = {glyph, parts: parts.map(ch=>({ch})), onTake, note: note||'点击一个偏旁，把它借走。'};
  beep(300,0.12,'sine',0.1);
}
function drawChai(){
  if(!chaiUI) return;
  ctx.save();
  ctx.fillStyle='rgba(5,5,10,0.92)'; ctx.fillRect(0,0,W,H);
  CH(chaiUI.glyph, 240, 250, 150, INK, 'center', 1, 20);
  CH('→', 420, 250, 60, INK_DIM);
  chaiUI.parts.forEach((p,i)=>{
    const x = 560+i*140, y = 250;
    const hov = Math.abs(mouse.x-x)<50 && Math.abs(mouse.y-y)<60;
    CH(p.ch, x, y, 110, hov?GOLD:INK, 'center', 1, hov?24:0);
    CH(String(i+1), x, y+86, 20, INK_DIM);
    if(hov && mouse.clicked){
      const f = chaiUI.onTake; const part = p.ch;
      chaiUI = null; beep(760,0.1,'triangle',0.12);
      f && f(part);
    }
  });
  ctx.fillStyle=INK_DIM; ctx.font='20px '+FONT; ctx.textAlign='center';
  ctx.fillText(chaiUI.note, W/2, 420);
  ctx.fillText('（鼠标点击偏旁，或按 1/2/3 选择；Esc 取消）', W/2, 452);
  if(mouse.clicked && mouse.y>380) chaiUI=null;
  ctx.restore();
}

/* ---------- 火柴人女孩 ---------- */
function drawGirl(x, y, stage, t, scale=1){
  ctx.save();
  ctx.translate(x,y); ctx.scale(scale,scale);
  ctx.strokeStyle='#f2e3d5'; ctx.lineWidth=2.4; ctx.lineCap='round';
  const sway = Math.sin(t*1.6)*1.5;
  // 头
  ctx.beginPath(); ctx.arc(0,-52,13,0,Math.PI*2); ctx.stroke();
  // 身体
  ctx.beginPath(); ctx.moveTo(0,-39); ctx.lineTo(0,-8); ctx.stroke();
  // 手臂
  ctx.beginPath(); ctx.moveTo(-11,-30+sway*0.3); ctx.lineTo(0,-28); ctx.lineTo(11,-30-sway*0.3); ctx.stroke();
  // 腿
  ctx.beginPath(); ctx.moveTo(0,-8); ctx.lineTo(-8,10); ctx.moveTo(0,-8); ctx.lineTo(8,10); ctx.stroke();
  if(stage>=1){ // 头发
    ctx.strokeStyle='#e8c9a0'; ctx.lineWidth=1.8;
    for(let i=-1;i<=1;i++){
      ctx.beginPath();
      ctx.moveTo(i*7,-62);
      ctx.bezierCurveTo(i*12+sway,-58, i*14+sway*1.6,-40, i*11+sway*1.8,-26-(stage>=4?8:0));
      ctx.stroke();
    }
  }
  if(stage>=2){ // 眼睛
    ctx.fillStyle='#fff';
    ctx.beginPath(); ctx.arc(-4.5,-53,1.8,0,7); ctx.arc(4.5,-53,1.8,0,7); ctx.fill();
    ctx.fillStyle='#f9d976';
    ctx.beginPath(); ctx.arc(-4.5,-53,0.8,0,7); ctx.arc(4.5,-53,0.8,0,7); ctx.fill();
  }
  if(stage>=3){ // 嘴 + 手指
    ctx.strokeStyle='#f2e3d5'; ctx.lineWidth=1.6;
    ctx.beginPath(); ctx.arc(0,-47,3.5,0.25,Math.PI-0.25); ctx.stroke();
    for(const s of [-1,1]){
      ctx.beginPath();
      ctx.moveTo(s*11,-30-s*sway*0.3);
      ctx.lineTo(s*14,-27-s*sway*0.3);
      ctx.moveTo(s*11,-30-s*sway*0.3);
      ctx.lineTo(s*14,-31-s*sway*0.3);
      ctx.stroke();
    }
  }
  if(stage>=4){ // 衣裙褶皱 + 腮红
    ctx.strokeStyle='#d9b8c4'; ctx.lineWidth=1.4;
    ctx.beginPath(); ctx.moveTo(-9,-12); ctx.lineTo(-13,8); ctx.moveTo(9,-12); ctx.lineTo(13,8); ctx.stroke();
    ctx.beginPath(); ctx.moveTo(-13,8); ctx.lineTo(13,8); ctx.stroke();
    ctx.fillStyle='rgba(232,120,140,0.35)';
    ctx.beginPath(); ctx.arc(-7,-49,2.4,0,7); ctx.arc(7,-49,2.4,0,7); ctx.fill();
  }
  ctx.restore();
}

/* =========================================================
   白天 · 烹饪经营
   ========================================================= */
const TRAY = ['生','食','鱼','菜','羊','火','水','刀','盐'];
const RECIPES = {'火生':'熟','水鱼':'汤','刀菜':'丝','火食':'炒','盐食':'咸','水菜':'煮','羊鱼':'鲜'};
const DAYS = {
  1:[
    {want:'熟', a:'生', b:'火', cust:'贩', line:'来份熟的。生的我吃不来。'},
    {want:'汤', a:'鱼', b:'水', cust:'更', line:'打更了一夜，想喝口鱼汤。'},
    {want:'炒', a:'食', b:'火', cust:'官', line:'小炒一份。火候——你看着办。'},
  ],
  2:[
    {want:'丝', a:'菜', b:'刀', cust:'妈', line:'菜切成丝，孩子才肯吃。'},
    {want:'咸', a:'食', b:'盐', cust:'们', line:'嘴里淡出个鸟了，来点咸的！'},
    {want:'鲜', a:'鱼', b:'羊', cust:'官', line:'听说「鱼」和「羊」凑在一起，是个了不得的字。'},
  ],
  3:[
    {want:'煮', a:'菜', b:'水', cust:'休', line:'清水煮菜就好。我靠着树，吃不得油。'},
    {want:'熟', a:'生', b:'火', cust:'乞', line:'做熟就行……谢谢你。'},
    {special:true, cust:'客', line:'老板，给我来一个——「味」字。'},
  ],
  4:[
    {want:'炒', a:'食', b:'火', cust:'官', line:'最后一日了？炒个菜吧。'},
    {want:'鲜', a:'鱼', b:'羊', cust:'妈', line:'再来个鲜的，孩子惦记好几天了。'},
    {want:'汤', a:'鱼', b:'水', cust:'更', line:'汤也要。夜还长。'},
  ],
};
let cook = null;
function startCook(){
  cook = { orders: DAYS[G.day].slice(), idx:0, phase:'serve', chosen:[], t:0, gauge:0,
           earned:0, msg:null, stamp:null, result:null, wrongFlash:0 };
  G.state='cook';
}
function cookOrder(){ return cook.orders[cook.idx]; }
function drawCook(dt){
  cook.t += dt;
  // 背景：饭馆
  const g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0,'#241a12'); g.addColorStop(1,'#120d09');
  ctx.fillStyle=g; ctx.fillRect(0,0,W,H);
  CH('饭', 100, 90, 70, 'rgba(216,169,74,0.25)');
  CH('馆', 880, 90, 70, 'rgba(216,169,74,0.25)');
  // 灶台
  ctx.fillStyle='rgba(0,0,0,0.35)'; ctx.fillRect(330,300,300,190);
  CH('鼎', 480, 360, 120, '#8a7a5e');
  // 火光
  const fireLevel = cook.phase==='fire' ? (0.5+0.5*Math.sin(cook.t*3))*0.5+0.3 : 0.3;
  ctx.save();
  ctx.globalAlpha = fireLevel;
  const fg = ctx.createRadialGradient(480,430,4,480,430,70);
  fg.addColorStop(0,'#ffb347'); fg.addColorStop(0.5,'#c8443a'); fg.addColorStop(1,'rgba(200,68,58,0)');
  ctx.fillStyle=fg; ctx.beginPath(); ctx.arc(480,430,70,0,7); ctx.fill();
  ctx.restore();
  CH('火', 480, 445, 40+fireLevel*16, '#ffb347', 'center', 0.9, 12);

  const od = cookOrder();
  if(!od){ return; }

  // 客人
  CH(od.cust, 180, 250, 120, INK, 'center', 1, 6);
  ctx.fillStyle='rgba(233,226,208,0.08)';
  roundRect(60,80,330,86,8); ctx.fill();
  ctx.strokeStyle=INK_DIM; ctx.stroke();
  ctx.fillStyle=INK; ctx.font='20px '+FONT; ctx.textAlign='left';
  wrapText('“'+od.line+'”', 78, 108, 300, 28);

  if(cook.phase==='serve' || cook.phase==='pick'){
    // 菜台
    ctx.fillStyle=INK_DIM; ctx.font='18px '+FONT; ctx.textAlign='center';
    ctx.fillText('案板（点两个字下锅）', 480, 195);
    cook.chosen.forEach((c,i)=> CH(c, 440+i*80, 245, 54, GOLD));
    if(cook.chosen.length===0) CH('＋', 480, 245, 40, 'rgba(233,226,208,0.3)');
    // 错误提示
    if(cook.wrongFlash>0){
      cook.wrongFlash-=dt;
      CH('坏', 480, 245, 54, RED, 'center', Math.min(1,cook.wrongFlash));
    }
    // 食材架
    TRAY.forEach((c,i)=>{
      const x = 640 + (i%3)*96, y = 180 + Math.floor(i/3)*80;
      btn(x, y, 80, 64, c, {size:34, disabled: cook.chosen.includes(c) || cook.phase!=='pick',
        fn:()=>{
          cook.chosen.push(c);
          if(cook.chosen.length===2) resolveCook();
        }});
    });
    if(cook.phase==='serve'){ cook.phase='pick'; }
  }
  else if(cook.phase==='fire'){
    CH(cook.result, 480, 245, 70, GOLD, 'center', 1, 16);
    // 火候条
    cook.gauge += dt*2.2;
    const pos = 0.5 + 0.48*Math.sin(cook.gauge*2);
    const bx=180, bw=600, by=492;
    ctx.fillStyle='rgba(233,226,208,0.1)'; ctx.fillRect(bx,by,bw,16);
    // 区域：中心绿，两边黄，两端红
    ctx.fillStyle='rgba(90,180,90,0.5)'; ctx.fillRect(bx+bw*0.42,by,bw*0.16,16);
    ctx.fillStyle='rgba(216,169,74,0.45)'; ctx.fillRect(bx+bw*0.28,by,bw*0.14,16); ctx.fillRect(bx+bw*0.58,by,bw*0.14,16);
    ctx.fillStyle='rgba(200,68,58,0.5)'; ctx.fillRect(bx,by,bw*0.28,16); ctx.fillRect(bx+bw*0.72,by,bw*0.28,16);
    ctx.fillStyle='#fff'; ctx.fillRect(bx+pos*bw-2,by-6,4,28);
    ctx.fillStyle=INK; ctx.font='20px '+FONT; ctx.textAlign='center';
    ctx.fillText('按「空格」出锅 —— 绿区完美，红区会烧焦', 480, by-18);
  }
  else if(cook.phase==='stamp'){
    CH(cook.result, 480, 245, 70, cook.stamp==='焦' ? '#5a5348' : GOLD, 'center', 1, 12);
    const col = cook.stamp==='完美' ? '#7ec87e' : (cook.stamp==='尚可' ? GOLD : RED);
    ctx.save();
    ctx.translate(560,200); ctx.rotate(-0.25);
    ctx.strokeStyle=col; ctx.lineWidth=3; ctx.globalAlpha=0.9;
    roundRect(-70,-30,140,60,6); ctx.stroke();
    CH(cook.stamp, 0, 2, 38, col);
    ctx.restore();
    ctx.fillStyle=INK_DIM; ctx.font='18px '+FONT; ctx.textAlign='center';
    ctx.fillText('空格 继续', 480, 510);
  }

  // 顶部信息
  ctx.fillStyle=INK_DIM; ctx.font='19px '+FONT; ctx.textAlign='left';
  ctx.fillText('第'+G.day+'天 · 第'+(cook.idx+1)+'/'+cook.orders.length+'位客人', 24, 30);
  ctx.textAlign='right'; ctx.fillStyle=GOLD;
  ctx.fillText('贝 ×'+G.coins, W-24, 30);
  drawAwakenMeter();
}
function drawAwakenMeter(){
  ctx.textAlign='right'; ctx.font='19px '+FONT;
  ctx.fillStyle='#e8a0b4';
  ctx.fillText('觉醒 '+G.awaken, W-24, 58);
  ctx.textAlign='left'; ctx.fillStyle=INK_DIM;
  ctx.fillText('她的样子：'+'●'.repeat(girlStage())+'○'.repeat(4-girlStage()), 24, 58);
}
function resolveCook(){
  const [a,b] = cook.chosen;
  const od = cookOrder();
  const key = [a,b].sort().join('');
  let r = RECIPES[key] || he([a,b]) || he([b,a]);
  if(od.special){
    // 「味」字订单 —— 剧情事件
    cook.result = r || '坏';
    cook.phase='stamp'; cook.stamp='无味';
    G.flags.weiSeen = true;
    cook.after = ()=> say([
      {sp:'客',tx:'……（他尝了一口，摇摇头）'},
      {sp:'客',tx:'吃不出味道。'},
      {sp:'',tx:'他走了。你看着他剩下的半碗菜，忽然发现——'},
      {sp:'',tx:'「味」字的那个「口」，空空的。像谁张着嘴，却说不出话。'},
      {sp:'',tx:'（夜里，去看看自家的招牌吧。）'},
    ], ()=>{ addAwaken(1); nextOrder(); });
    return;
  }
  if(r === od.want){
    cook.result = r; cook.phase='fire'; cook.gauge=0;
    beep(500,0.1,'triangle');
  }else{
    cook.chosen=[]; cook.wrongFlash=1.2;
    G.coins = Math.max(0, G.coins-1);
    toast('做坏了……客人摇摇头走了（贝 -1）');
    beep(180,0.2,'sawtooth',0.1);
    nextOrder();
  }
}
function nextOrder(){
  cook.idx++; cook.chosen=[];
  if(cook.idx>=cook.orders.length){
    // 打烊
    G.state='cooksum';
  }else{
    cook.phase='pick';
  }
}
function cookStamp(){
  // 火候判定
  const pos = 0.5 + 0.48*Math.sin(cook.gauge*2);
  const d = Math.abs(pos-0.5);
  const od = cookOrder();
  if(d<=0.08){ cook.stamp='完美'; G.coins+=3; cook.earned+=3; beep(880,0.15,'sine',0.12); }
  else if(d<=0.22){ cook.stamp='尚可'; G.coins+=1; cook.earned+=1; beep(620,0.12,'sine',0.1); }
  else{
    cook.stamp='焦';
    if(!G.flags.jiaoFirst){
      G.flags.jiaoFirst = true;
      cook.after = ()=> say([
        {sp:'',tx:'火太大了。菜焦了。客人皱着眉走了。'},
        {sp:'我',tx:'……'},
        {sp:'',tx:'可你盯着锅底那一点焦黑，忽然觉得——那才是这口锅今天说出的、唯一一句真话。'},
      ], ()=>{ addAwaken(1); nextOrder(); });
    }else{
      cook.after = ()=>{ addAwaken(1); toast('又是焦的。你心里那点火，亮了一下。'); nextOrder(); };
    }
  }
  cook.phase='stamp';
  if(cook.stamp!=='焦') cook.after = ()=> nextOrder();
}
function drawCookSum(){
  ctx.fillStyle='#0d0a08'; ctx.fillRect(0,0,W,H);
  CH('打 烊', W/2, 150, 72, GOLD, 'center', 1, 16);
  ctx.fillStyle=INK; ctx.font='24px '+FONT; ctx.textAlign='center';
  ctx.fillText('今日收入：贝 ×'+ (cook?cook.earned:0) + '　　结余：贝 ×'+G.coins, W/2, 250);
  ctx.fillStyle=INK_DIM; ctx.font='21px '+FONT;
  const lines = [
    '客人都走了。街上安静下来。',
    G.flags.weiSeen && G.day===3 ? '那个没吃到「味」的客人，一直在你脑子里。' : '你把灶台擦了一遍又一遍。',
    '夜色漫上来。该去街上走走了。',
  ];
  lines.forEach((l,i)=> ctx.fillText(l, W/2, 310+i*34));
  btn(W/2-90, 440, 180, 48, '入 夜 →', {fn:()=> startNight(G.day)});
}

/* =========================================================
   夜晚 · 探索解谜
   ========================================================= */
let N = null;
function startNight(id){
  G.form='人';
  N = { id, t:0, px:80, cam:0, hotspots:[], gaps:[], bridges:[], parts:[],
        girlX:0, girlTalked:false, exitX:0, hint:'', done:false };
  G.state='night';
  NIGHT_BUILDERS[id]();
  save('night');
  toast(N.hint);
}
function solidAt(x){
  // 缺口处无桥则跌落不可通行
  for(const gp of N.gaps){
    if(x>gp.x0+6 && x<gp.x1-6){
      const br = N.bridges.find(b=> b.ttl>0 && x>b.x0 && x<b.x1);
      if(!br) return false;
    }
  }
  return true;
}
function nearGapEdge(){
  for(const gp of N.gaps){
    if(Math.abs(N.px-gp.x0)<46 || Math.abs(N.px-gp.x1)<46) return gp;
  }
  return null;
}
function addHotspot(h){ N.hotspots.push(h); }
function flame(x,y,n=6){
  for(let i=0;i<n;i++) N.parts.push({x:x+(Math.random()-0.5)*30, y:y-Math.random()*10,
    vy:-20-Math.random()*30, vx:(Math.random()-0.5)*14, ttl:1+Math.random(), fire:true});
}

/* ---- 各夜建造器 ---- */
const NIGHT_BUILDERS = {};
/* ---- 第一夜：拆字 · 借笔 · 留白 ---- */
NIGHT_BUILDERS[1] = function(){
  N.w = 1750; N.lit=false; N.hint='A/D 移动 · E 互动 · 这条街，晚上是另一个样子。';
  N.gaps = [{x0:930, x1:1040}];
  N.girlX = 1300; N.exitX = 1660;
  addHotspot({x:300, g:()=> G.flags.dengChai?'丁':'灯', size:110, label:'路灯',
    act(){
      if(!G.flags.dengChai){
        openChai('灯', p=>{
          if(p==='火'){ G.flags.dengChai=true; G.items.火=true;
            say([{sp:'',tx:'你把「灯」字的火旁摘了下来。火苗躺在你手心，温温的。'},
                 {sp:'',tx:'「灯」瘦成了「丁」，光暗了一半。'}]);
          }else{ say([{sp:'',tx:'「丁」字被你捏了一下，叮当作响。你把它放了回去。'}]); }
        });
      }else say([{sp:'',tx:'「丁」孤零零地立着，像一根钉子。'}]);
    }});
  addHotspot({x:580, g:()=> G.flags.chaiLit?'灰':'柴', size:90, label:'柴堆',
    act(){
      if(G.flags.chaiLit){ say([{sp:'',tx:'火烧得正好。灰烬里，有什么在闪。'}]); return; }
      if(!G.items.火){ say([{sp:'',tx:'一堆「柴」。缺一把火。'}]); return; }
      G.flags.chaiLit = true; G.items.火=false; N.lit=true;
      flame(580,GROUND-40,16);
      beep(200,0.4,'sawtooth',0.08);
      say([{sp:'',tx:'火舌舔上「柴」字，整条街亮了起来。'},
           {sp:'',tx:'原来这条街这么长。原来夜晚不只有黑。'}], ()=> addAwaken(1));
    }});
  addHotspot({x:640, g:()=>'一', size:40, label:'灰烬里的笔画', cond:()=> G.flags.chaiLit && !G.flags.pickYi,
    act(){
      G.flags.pickYi=true; invAdd('一');
      say([{sp:'',tx:'灰烬里剩下一横。像谁没写完的一句话。'},
           {sp:'',tx:'获得笔画「一」。它可以加在别的字上——包括你自己。（按 1 试试）'}]);
    }});
  addHotspot({x:1300, g:()=>'', size:0, label:'她', girl:true});
  addHotspot({x:1660, g:()=>'睡', size:70, label:'回家睡觉',
    act(){ say([{sp:'',tx:'今夜就到这里吧。'}], ()=>{ G.day=2; dayIntro(); }); }});
  N.firstGapMsg=false;
};
/* ---- 第二夜：加笔 · 变形 ---- */
NIGHT_BUILDERS[2] = function(){
  N.w = 1950; N.lit=true; N.hint='按 1 加「一」成「大」· 按 2 加「丨」成「个」· 按 0 变回「人」';
  N.girlX = 1720; N.exitX = 1880;
  addHotspot({x:260, g:()=> G.flags.zaoChai?'日':'早', size:90, label:'茶铺招牌',
    act(){
      if(G.flags.zaoChai){ say([{sp:'',tx:'招牌上只剩一个「日」，天亮得没道理。'}]); return; }
      openChai('早', p=>{
        if(p==='十'){ G.flags.zaoChai=true; invAdd('一'); invAdd('丨');
          say([{sp:'',tx:'你抽走「十」，轻轻掰开——成了一横一竖。'},
               {sp:'',tx:'获得「一」和「丨」。现在你可以给自己加笔了。（1：大 / 2：个 / 0：还原）'}]);
        }else{ say([{sp:'',tx:'你把「日」捧在手心，烫得赶紧挂了回去。'}]); }
      });
    }});
  addHotspot({x:640, g:()=>'官', size:100, label:'衙役',
    act(){
      const f = G.form;
      if(f==='囚'){
        if(!G.flags.qiuSee){ G.flags.qiuSee=true;
          say([{sp:'官',tx:'（他满意地点头）这就对了。字，就该待在格子里。'},
               {sp:'',tx:'你心里咯噔一下——被夸“规矩”，原来这么难受。'}], ()=> addAwaken(1));
        }else say([{sp:'官',tx:'乖。'}]);
      }
      else if(f==='大') say([{sp:'官',tx:'（他仰着头，声音发虚）……这位爷，里边请？'}]);
      else if(f==='个') say([{sp:'官',tx:'（他低头找了半天）哪儿来的小不点儿？'}]);
      else say([{sp:'官',tx:'衙门重地，闲人规矩点。'}]);
    }});
  N.blockX = 930; N.blockCond = ()=> !G.flags.wallDown;
  addHotspot({x:980, g:()=> G.flags.wallDown?'塌':'墙', size:130, label:'高墙',
    act(){
      if(G.flags.wallDown){ say([{sp:'',tx:'墙塌了一个角，刚好能过。'}]); return; }
      if(G.form!=='大'){ say([{sp:'',tx:'墙很高。比三个「人」摞起来还高。'},
        {sp:'我',tx:'如果我再「大」一点……'}]); return; }
      G.flags.wallDown=true;
      say([{sp:'',tx:'你伸手——「大」字的手，刚好够到墙头。'},
           {sp:'',tx:'墙皮簌簌地掉，露出后面藏着的东西。'}]);
    }});
  addHotspot({x:1120, g:()=> G.flags.huiChai?(G.flags.huiLeft||'囗'):'回', size:80, label:'风筝', cond:()=> G.flags.wallDown,
    act(){
      if(G.flags.huiChai){ say([{sp:'',tx:'「'+G.flags.huiLeft+'」空空地悬着。'}]); return; }
      openChai('回', p=>{
        G.flags.huiChai=true;
        if(p==='囗'){ G.flags.huiLeft='口'; invAdd('囗');
          say([{sp:'',tx:'获得「囗」。按 3，它会把你围成「囚」——小心，也会看清一些东西。'}]);
        }else{ G.flags.huiLeft='囗'; invAdd('口');
          say([{sp:'',tx:'获得「口」。一张嘴。门若有了嘴，就会说话。'}]);
        }
      }, '只能拿走一个——选哪个？');
    }});
  addHotspot({x:1420, g:()=>'穴', size:56, label:'窄洞',
    act(){
      if(G.flags.caveDone){ say([{sp:'',tx:'洞空了。'}]); return; }
      if(G.form!=='个'){ say([{sp:'',tx:'洞太窄了。只有「个」那样瘦的字，才进得去。'}]); return; }
      G.flags.caveDone=true; G.items.墨块=true;
      say([{sp:'',tx:'洞里很黑。你摸到一块没干的墨。'},
           {sp:'',tx:'它在你掌心微微发热，像一颗很小的心脏。'}], ()=> addAwaken(1));
    }});
  addHotspot({x:1720, g:()=>'', size:0, label:'她', girl:true});
  addHotspot({x:1880, g:()=>'睡', size:70, label:'回家睡觉',
    act(){ say([{sp:'',tx:'今夜就到这里吧。'}], ()=>{ G.day=3; dayIntro(); }); }});
};
/* ---- 第三夜：门 + 口 = 问 ---- */
NIGHT_BUILDERS[3] = function(){
  N.w = 1750; N.lit=true; N.hint='你回头看了一眼自己的招牌——那个「味」字。';
  N.girlX = 1330; N.exitX = 1660;
  addHotspot({x:220, g:()=> G.flags.weiChai?'未':'味', size:90, label:'自家招牌',
    act(){
      if(G.flags.weiChai){ say([{sp:'',tx:'招牌成了「未」。未来，也未尝。'}]); return; }
      openChai('味', p=>{
        if(p==='口'){ G.flags.weiChai=true; invAdd('口');
          say([{sp:'',tx:'原来「味」字的口，一直挂在自己头顶上。'},
               {sp:'',tx:'那个找你要「味」的客人……他的口，又丢在哪了呢？'}], ()=> addAwaken(1));
        }else{ say([{sp:'',tx:'「未」是还没长成的树。你让它继续长着。'}]); }
      });
    }});
  N.blockX = 600; N.blockCond = ()=> !G.flags.menOpen;
  addHotspot({x:660, g:()=> G.flags.menWen?(G.flags.menOpen?'開':'问'):'门', size:150, label:'大门',
    act(){
      if(G.flags.menOpen){ say([{sp:'',tx:'门开着，像一个愿意倾听的人。'}]); return; }
      if(!G.flags.menWen){
        if((G.inv['口']||0)<1){ say([{sp:'',tx:'门上没有嘴。它听不见你。'},
          {sp:'',tx:'（需要一张「口」）'}]); return; }
        G.inv['口']--; G.flags.menWen=true;
        beep(400,0.3,'sine',0.12);
        say([{sp:'',tx:'你把「口」按在门上。笔画蠕动、咬合——'},
             {sp:'',tx:'「门」变成了「问」。门，睁开眼睛看着你。'}], askRiddle);
        return;
      }
      askRiddle();
    }});
  addHotspot({x:1010, g:()=> G.flags.xiuChai?'亻':'休', size:80, label:'靠着树的人',
    act(){
      if(G.flags.xiuChai){ say([{sp:'',tx:'「亻」独自站着，站得笔直，像在赌气。'}]); return; }
      openChai('休', p=>{
        if(p==='木'){ G.flags.xiuChai=true;
          say([{sp:'',tx:'你抽走了他靠着的「木」。他晃了晃，只剩一个孤零零的「亻」。'},
               {sp:'休',tx:'……喂。'},
               {sp:'',tx:'你赶紧把「木」还了回去。他重新靠上去，装作什么都没发生。'},
               {sp:'',tx:'可你记住了他晃的那一下——原来每个字，都靠着点什么才站得稳。'}], ()=> addAwaken(1));
        }else{ say([{sp:'',tx:'「亻」就是他本人。总不能把人拆走。'}]); }
      });
    }});
  addHotspot({x:1330, g:()=>'', size:0, label:'她', girl:true});
  addHotspot({x:1660, g:()=>'睡', size:70, label:'回家睡觉', cond:()=> G.flags.menOpen,
    act(){ say([{sp:'',tx:'今夜就到这里吧。'}], ()=>{ G.day=4; dayIntro(); }); }});
};
function askRiddle(){
  say([{sp:'问',tx:'答对我的问题，我就开。\n什么东西——越拆越多？',
        choices:[
          {t:'「柴」', fn:()=>{ G.flags.menTry=true; say([{sp:'问',tx:'柴越拆越少。再想想。'}]); }},
          {t:'「字」', fn:()=>{
            G.flags.menOpen=true;
            const first = !G.flags.menTry;
            G.coins+=2;
            say([{sp:'问',tx:'……对。字拆成偏旁，偏旁还能再拆。'},
                 {sp:'问',tx:'门缝里有东西滚出来——两枚「贝」。拿去吧，是我攒的。'},
                 {sp:'',tx:'门缓缓开了。门后那条街，比白天宽得多。'}],
                 ()=>{ if(first) addAwaken(1); });
          }},
          {t:'「梦」', fn:()=>{ G.flags.menTry=true; say([{sp:'问',tx:'梦越拆越醒。不对。'}]); }},
        ]}]);
}
/* ---- 第四夜：锁 · 贝 · 留白之桥 ---- */
NIGHT_BUILDERS[4] = function(){
  N.w = 1950; N.lit=true; N.hint='F：在留白处画一笔（会慢慢消失）· 今天后半夜，街要到头了。';
  N.gaps = [{x0:960, x1:1060},{x0:1150, x1:1280}];
  N.girlX = 1560; N.exitX = 1870;
  addHotspot({x:400, g:()=> G.flags.suoChai?(G.flags.suoOpen?'贝':'金'):'锁', size:100, label:'当铺的锁',
    act(){
      if(!G.flags.suoChai){
        openChai('锁', p=>{
          if(p==='小'){ G.flags.suoChai=true; invAdd('小');
            say([{sp:'',tx:'「锁」散了一地：小、金、贝。你捡起「小」。'},
                 {sp:'',tx:'剩下的「金」还在锁孔里咬着，得用东西换它松口。'}]);
          }else if(p==='贝'){ say([{sp:'',tx:'这枚「贝」焊死在锁上——它只认外来的贝。'}]); }
          else{ say([{sp:'',tx:'「金」纹丝不动。'}]); }
        });
        return;
      }
      if(G.flags.suoOpen){ say([{sp:'',tx:'锁开了，像一句说完的话。'}]); return; }
      if(G.coins>=2){
        G.coins-=2; G.flags.suoOpen=true; G.items.钥=true;
        beep(900,0.2,'triangle',0.12);
        say([{sp:'',tx:'你放进两枚白天攒的「贝」。「金」咂咂嘴，吐出一把「钥」。'}]);
      }else if(G.items.墨块){
        G.items.墨块=false; G.flags.suoOpen=true; G.items.钥=true;
        say([{sp:'',tx:'你没有贝。你掏出那块温热的墨——「金」盯着它看了很久，忽然让开了。'},
             {sp:'',tx:'“这个比贝值钱。”它瓮声瓮气地说。一把「钥」落在你手里。'}]);
      }else{
        say([{sp:'',tx:'「金」指着旁边刻的小字：两枚贝，换一把钥。'},
             {sp:'',tx:'（白天做菜可以赚贝。完美出锅 +3，尚可 +1。）'}]);
      }
    }});
  addHotspot({x:620, g:()=> G.flags.boxOpen?'箱':'箱', size:90, label:'木箱',
    act(){
      if(G.flags.boxOpen){ say([{sp:'',tx:'箱子空了，但你觉得它是满的。'}]); return; }
      if(!G.items.钥){ say([{sp:'',tx:'锁着。'}]); return; }
      G.flags.boxOpen=true; G.items.墨笔=true;
      say([{sp:'',tx:'箱子里没有金银。只有一笔——一捺，悬在半空，还没干。'},
           {sp:'我',tx:'……'},
           {sp:'',tx:'你忽然鼻子一酸。这一笔，你好像丢了很多年。'},
           {sp:'',tx:'获得「未干的一笔」。'}], ()=> addAwaken(1));
    }});
  addHotspot({x:1560, g:()=>'', size:0, label:'她', girl:true});
  addHotspot({x:1870, g:()=>'→', size:70, label:'走向后半夜', cond:()=> N.girlTalked,
    act(){ startNight(5); }});
};
/* ---- 第五夜：走出 ---- */
NIGHT_BUILDERS[5] = function(){
  N.w = 2500; N.lit=true; N.hint='';
  N.girlX = -1; N.exitX = 2380; N.walkWith = true;
  G.items.巾 = false;
  say([{sp:'',tx:'你解下围裙——一个「巾」字。叠好，放在灶台上。'},
       {sp:'她',tx:'你不后悔？'},
       {sp:'我',tx:'我做了这么多年厨师，一直在等一个「味」字。后来我才知道，它不在锅里。'},
       {sp:'',tx:'走吧。'}]);
  N.scenery = [
    {x:200,g:'灯',s:90,c:'#f0d080'},{x:380,g:'店',s:100,c:'#6a6255'},
    {x:560,g:'家',s:100,c:'#c89a6a'},{x:760,g:'林',s:90,c:'#5c7a5c'},
    {x:950,g:'酒',s:80,c:'#7a6a9a'},{x:1120,g:'官',s:100,c:'#8a8a9a'},
    {x:1330,g:'森',s:100,c:'#4c6a4c'},{x:1520,g:'梦',s:90,c:'#9a7ab8'},
    {x:1730,g:'云',s:80,c:'#e8e8f0'},{x:1950,g:'路',s:90,c:'#8a8272'},
  ];
};

/* ---- 夜晚引擎 ---- */
function nightUpdate(dt){
  N.t += dt;
  if(dlg || chaiUI) return;
  let mv = 0;
  if(keys['a']||keys['arrowleft']) mv=-1;
  if(keys['d']||keys['arrowright']) mv=1;
  if(mv!==0 && G.form!=='囚'){
    const sp = G.form==='大' ? 150 : (G.form==='个' ? 200 : 180);
    let nx = N.px + mv*sp*dt;
    // 墙体 / 大门阻挡
    if(N.blockX && N.blockCond && N.blockCond() && nx>N.blockX && N.px<=N.blockX) nx = N.px;
    if(nx>30 && nx<N.w-20 && solidAt(nx)) N.px = nx;
    else if(!solidAt(nx) && !N.gapWarned){
      N.gapWarned = true;
      say([{sp:'',tx:'前面是留白——什么都没有。'},
           {sp:'',tx:'（按 F，在没有字的地方画一笔。它会慢慢消失，抓紧时间。）'}],
           ()=> setTimeout(()=>N.gapWarned=false, 1000));
    }
  }
  // 女孩同行（第五夜）
  if(N.walkWith) N.girlX = N.px - 46;
  // 相机
  N.cam = Math.max(0, Math.min(N.w-W, N.px - 400));
  // 桥衰减
  N.bridges.forEach(b=> b.ttl-=dt);
  N.bridges = N.bridges.filter(b=> b.ttl>0);
  // 粒子
  N.parts.forEach(p=>{ p.ttl-=dt; p.x+=p.vx*dt; p.y+=p.vy*dt; });
  N.parts = N.parts.filter(p=>p.ttl>0);
  // 第五夜：走到世界尽头 → 终局
  if(N.id===5 && N.px>2320 && !N.fin){ N.fin=true; startFinal(); }
}
function nightInteract(){
  if(dlg){ dlgAdvance(); return; }
  if(chaiUI) return;
  // 女孩
  if(!N.girlTalked && N.girlX>0 && Math.abs(N.px-N.girlX)<70){
    girlTalk(); return;
  }
  for(const h of N.hotspots){
    if(h.girl) continue;
    if(h.cond && !h.cond()) continue;
    if(Math.abs(N.px-h.x)<70){ h.act(); return; }
  }
}
function girlTalk(){
  const s = girlStage();
  const done = ()=>{ N.girlTalked=true; };
  if(N.id===1){
    say([
      {sp:'',tx:'她站在留白边上，空白的脸朝着你。没有眼睛，你却觉得被看见了。'},
      {sp:'我',tx:'打烊了。这条街，晚上不太平。'},
      {sp:'她',tx:'我知道。我只是想看看——字都睡了以后，街是什么样子。'},
      {sp:'她',tx:'你看那边。没有字的地方，是不是也可以是路？'},
    ], ()=>{ addAwaken(1); done(); });
  }else if(N.id===2){
    const lines = [
      {sp:'我',tx:'你变样子了。'},
      {sp:'她',tx:'嗯。每天长一点点。有时候是一根头发，有时候是一根手指。'},
      {sp:'她',tx:'火柴人最简单，也最容易被忘掉。我想变得精细一点，精细到没有人能忽略我。'},
    ];
    if(s<2) lines.push({sp:'她',tx:'你好像总是站着不动。字站久了，会忘了自己还能变。'});
    else lines.push({sp:'她',tx:'你今天……好像和昨天不太一样。笔画松了些。'});
    say(lines, done);
  }else if(N.id===3){
    const lines = [
      {sp:'',tx:'她看见你，笑了一下。笑容像刚学会的笔画，生涩，但真实。'},
      {sp:'我',tx:'你今天……有嘴了。'},
      {sp:'她',tx:'嗯，长的。你猜我学会的第一个词是什么？'},
      {sp:'她',tx:'是「不」。'},
    ];
    if(G.flags.xiuChai) lines.push(
      {sp:'她',tx:'我看见你抽走了他的树，又还了回去。'},
      {sp:'她',tx:'不规矩，但有心。这样长出来的笔画，才是活的。'});
    say(lines, ()=>{ addAwaken(1); done(); });
  }else if(N.id===4){
    say([
      {sp:'她',tx:'我一开始也不知道怎么长。有一天，我看见天上飘过去一个「云」字。'},
      {sp:'她',tx:'我问它：你怎么能变？它说——我本来就不是云。我是水变的，以后还会变成雨，变成雪。'},
      {sp:'她',tx:'它说，字不是定死的，是活出来的。'},
      {sp:'我',tx:'……字不是定死的，是活出来的。'},
      {sp:'她',tx:'所以。你想不想去看看，你的笔画之外，还有什么？'},
    ], ()=>{ addAwaken(1); done(); });
  }else{ done(); }
}
function transformTo(f){
  if(G.state!=='night' || dlg || chaiUI) return;
  if(G.form===f) return;
  const need = { '大':'一', '个':'丨', '囚':'囗' };
  // 还原当前
  if(G.form!=='人'){ invAdd(need[G.form]); G.form='人'; }
  if(f==='人'){ beep(400,0.08); return; }
  const c = need[f];
  if((G.inv[c]||0)<1){ toast('缺少笔画「'+c+'」'); beep(160,0.1,'sawtooth',0.08); return; }
  G.inv[c]--; G.form=f;
  beep(520,0.15,'sine',0.12);
  if(f==='囚'){
    say([{sp:'',tx:'「囗」合拢。你成了「囚」。四面都是墙，墙的每一笔都是你自己。'}],
      ()=> setTimeout(()=>{ if(G.form==='囚'){ invAdd('囗'); G.form='人'; } }, 2600));
  }
  const key = 'tf_'+f;
  if(!G.flags[key]){ G.flags[key]=true; addAwaken(1); }
}
function drawLiubai(){
  const gp = nearGapEdge();
  if(!gp){ toast('这里没有留白可画。'); return; }
  N.bridges.push({x0:gp.x0-8, x1:gp.x1+8, ttl:35});
  beep(700,0.2,'sine',0.1);
  G.flags.lbCount = (G.flags.lbCount||0)+1;
  if(G.flags.lbCount<=3){
    say([{sp:'',tx:'你在空无一物的地方落笔。笔画悬着，微微发颤——它不属于任何字。'},
         {sp:'',tx:'它正在慢慢变淡。自由是有代价的。'}], ()=> addAwaken(1));
  }else{
    toast('你又画了一笔。它仍在慢慢变淡。');
  }
}
function drawNight(dt){
  // 背景
  const g = ctx.createLinearGradient(0,0,0,H);
  if(N.lit){ g.addColorStop(0,'#1a1a2e'); g.addColorStop(1,'#0d0d18'); }
  else{ g.addColorStop(0,'#0a0a14'); g.addColorStop(1,'#050508'); }
  ctx.fillStyle=g; ctx.fillRect(0,0,W,H);
  // 星
  ctx.save(); ctx.globalAlpha=0.5;
  for(let i=0;i<40;i++){
    const sx=(i*197+ N.id*53)%W, sy=(i*89)%200+20;
    ctx.fillStyle = i%7? '#555':'#888';
    ctx.fillRect(sx,sy,1.6,1.6);
  }
  ctx.restore();
  ctx.save();
  ctx.translate(-N.cam,0);
  // 地面
  ctx.strokeStyle='#3a362c'; ctx.lineWidth=3;
  ctx.beginPath();
  let lastX=0;
  const segs = [[0, N.w]];
  N.gaps.forEach(gp=>{ segs.push([gp.x0,gp.x1]); });
  // 简化：分段画
  let cur=0;
  const sorted=[...N.gaps].sort((a,b)=>a.x0-b.x0);
  ctx.beginPath();
  let px0=0;
  ctx.moveTo(0,GROUND);
  for(const gp of sorted){ ctx.lineTo(gp.x0,GROUND); ctx.moveTo(gp.x1,GROUND); }
  ctx.lineTo(N.w,GROUND);
  ctx.stroke();
  // 缺口（深渊）
  for(const gp of N.gaps){
    ctx.fillStyle='rgba(0,0,0,0.85)';
    ctx.fillRect(gp.x0,GROUND,gp.x1-gp.x0,H-GROUND);
    // 留白微光
    ctx.save();
    ctx.globalAlpha = 0.25+0.15*Math.sin(N.t*2);
    ctx.setLineDash([6,8]);
    ctx.strokeStyle='#8fb3d8'; ctx.lineWidth=1.5;
    ctx.strokeRect(gp.x0+6,GROUND+8,gp.x1-gp.x0-12,40);
    ctx.restore();
    CH('留白', (gp.x0+gp.x1)/2, GROUND+64, 18, 'rgba(143,179,216,0.4)');
  }
  // 桥
  for(const b of N.bridges){
    const a = Math.min(1, b.ttl/8);
    ctx.save();
    ctx.globalAlpha = a;
    ctx.strokeStyle='#e8dfc8'; ctx.lineWidth=5; ctx.lineCap='round';
    ctx.shadowColor='#e8dfc8'; ctx.shadowBlur=14;
    ctx.beginPath(); ctx.moveTo(b.x0,GROUND-2);
    ctx.quadraticCurveTo((b.x0+b.x1)/2, GROUND-14, b.x1, GROUND-2);
    ctx.stroke();
    ctx.restore();
  }
  // 第五夜街景
  if(N.scenery) for(const s of N.scenery) CH(s.g, s.x, GROUND-s.s*0.55, s.s, s.c, 'center', 0.9);
  // 热点
  for(const h of N.hotspots){
    if(h.girl) continue;
    if(h.cond && !h.cond()) continue;
    const gl = h.g();
    if(gl) CH(gl, h.x, GROUND-h.size*0.55, h.size, INK, 'center', 0.95, h.label==='大门'?8:0);
    // 交互提示
    if(Math.abs(N.px-h.x)<70){
      ctx.save();
      ctx.globalAlpha = 0.8+0.2*Math.sin(N.t*5);
      CH('E', h.x, GROUND-(h.size||60)-36, 20, GOLD);
      if(h.label){ ctx.font='16px '+FONT; ctx.textAlign='center'; ctx.fillStyle=INK_DIM; ctx.fillText(h.label, h.x, GROUND-(h.size||60)-14); }
      ctx.restore();
    }
  }
  // 粒子（火）
  for(const p of N.parts){
    ctx.save(); ctx.globalAlpha=Math.min(1,p.ttl);
    ctx.fillStyle = p.fire? '#ffb347':'#e8dfc8';
    ctx.beginPath(); ctx.arc(p.x,p.y,2.2,0,7); ctx.fill(); ctx.restore();
  }
  // 女孩
  if(N.girlX>0){
    drawGirl(N.girlX, GROUND-12, girlStage(), N.t);
    if(!N.girlTalked && Math.abs(N.px-N.girlX)<70) CH('E', N.girlX, GROUND-120, 20, '#e8a0b4');
  }
  // 玩家
  const formSize = G.form==='大'?92:(G.form==='个'?46:64);
  const formCol = G.form==='囚' ? '#c8b090' : '#f0e8d8';
  CH(G.form, N.px, GROUND-formSize*0.55, formSize, formCol, 'center', 1, 10);
  if(N.walkWith===undefined && G.items.巾!==false && N.id<5){}
  ctx.restore();

  // 暗角（第一夜未点火）
  if(!N.lit){
    const sx = N.px - N.cam;
    const rg = ctx.createRadialGradient(sx,GROUND-40,60, sx,GROUND-40,240);
    rg.addColorStop(0,'rgba(4,4,10,0)');
    rg.addColorStop(1,'rgba(4,4,10,0.97)');
    ctx.fillStyle=rg; ctx.fillRect(0,0,W,H);
  }
  // HUD
  ctx.save();
  ctx.fillStyle='rgba(8,8,14,0.7)';
  roundRect(12,12,300,84,8); ctx.fill();
  ctx.fillStyle=INK_DIM; ctx.font='17px '+FONT; ctx.textAlign='left';
  ctx.fillText('第'+['','一','二','三','四','五'][N.id]+'夜', 24, 34);
  let invStr = Object.entries(G.inv).filter(([k,v])=>v>0).map(([k,v])=>k+'×'+v).join(' ');
  ctx.fillStyle=GOLD;
  ctx.fillText('笔画: '+(invStr||'（空）'), 24, 58);
  ctx.fillStyle='#e8a0b4';
  ctx.fillText('觉醒 '+G.awaken+' · 她 '+'●'.repeat(girlStage())+'○'.repeat(4-girlStage()), 24, 82);
  ctx.textAlign='right'; ctx.fillStyle=GOLD; ctx.font='17px '+FONT;
  ctx.fillText('贝 ×'+G.coins, W-20, 30);
  ctx.fillStyle=INK_DIM;
  ctx.fillText('1大 2个 3囚 0还原 · E互动 · F留白', W-20, H-16);
  ctx.restore();
}

/* =========================================================
   白天开场卡
   ========================================================= */
function dayIntro(){
  G.state='intro';
  save('intro');
}
const INTRO_LINES = {
  1:['卯时。','你把自己拼成一个「厨」字的模样，','走进那间用「屋」字搭成的饭馆。'],
  2:['日子很规矩。就像你写的字一样规矩。','没有人觉得有什么不对。','——除了你心里那个被硌着的地方。'],
  3:['昨天夜里的事，像一滴没干的墨。','你切菜的时候，刀落在案板上，','声音特别脆。'],
  4:['她在慢慢长。','而你心里那个声音，','怎么也灭不掉。'],
};
function drawIntro(){
  ctx.fillStyle='#0d0b09'; ctx.fillRect(0,0,W,H);
  CH('第'+['','一','二','三','四'][G.day]+'天', W/2, 140, 60, GOLD, 'center', 1, 14);
  ctx.fillStyle=INK; ctx.font='24px '+FONT; ctx.textAlign='center';
  (INTRO_LINES[G.day]||[]).forEach((l,i)=> ctx.fillText(l, W/2, 240+i*40));
  btn(W/2-90, 420, 180, 48, '开门营业 →', {fn:startCook});
}

/* =========================================================
   终局：画一笔
   ========================================================= */
let FIN = null;
function startFinal(){
  FIN = { p:0, sys1:false, sys2:false, done:false, shake:0, revealT:0 };
  G.state='final';
  say([{sp:'她',tx:'路到头了。再往前，就没有字了。'},
       {sp:'我',tx:'那就写一个新的。'},
       {sp:'',tx:'按住「空格」——在你身旁的留白里，画一笔。'}]);
}
function drawFinal(dt){
  if(FIN.shake>0) FIN.shake-=dt;
  const shx = FIN.shake>0 ? (Math.random()-0.5)*8 : 0;
  ctx.save(); ctx.translate(shx,0);
  ctx.fillStyle='#050508'; ctx.fillRect(-20,0,W+40,H);
  CH('人', 380, 300, 130, INK, 'center', 1, 10);
  // 正在画的一笔（一捺）
  const p = FIN.p;
  if(p>0){
    ctx.save();
    ctx.strokeStyle = p>=1 ? GOLD : '#c8443a';
    ctx.lineWidth = 7; ctx.lineCap='round';
    ctx.shadowColor = ctx.strokeStyle; ctx.shadowBlur = 20;
    ctx.beginPath();
    ctx.moveTo(480, 250);
    const ex = 480 + 130*p, ey = 250 + 130*p + Math.sin(p*3)*6;
    ctx.quadraticCurveTo(480+60*p, 250+90*p, ex, ey);
    ctx.stroke();
    ctx.restore();
  }
  if(!FIN.done && !dlg){
    // 按住空格画
    if(keys[' ']){
      FIN.p = Math.min(1, FIN.p + dt*0.22);
      if(FIN.p>=0.3 && !FIN.sys1){ FIN.sys1=true;
        say([{sp:'系统',tx:'警告：该笔画无法识别。'}]); }
      if(FIN.p>=0.62 && !FIN.sys2){ FIN.sys2=true; FIN.shake=0.8;
        say([{sp:'系统',tx:'这不是字。请停止。'}]); }
    }
    if(FIN.p>=1 && !FIN.done){
      FIN.done=true; FIN.revealT=0; beep(180,1.2,'sine',0.1);
      say([{sp:'系统',tx:'……'},
           {sp:'系统',tx:'……正在重新识别……'},
           {sp:'系统',tx:'这是你的字。'}],
        ()=>{ G.state='over'; clearSave(); });
    }
  }
  ctx.fillStyle=INK_DIM; ctx.font='20px '+FONT; ctx.textAlign='center';
  if(!FIN.done && !dlg) ctx.fillText('按住 空格 画下去'+'　'+'▮'.repeat(Math.ceil(FIN.p*10))+'▯'.repeat(10-Math.ceil(FIN.p*10)), W/2, 470);
  ctx.restore();
}
function drawOver(dt){
  FIN.revealT += dt;
  ctx.fillStyle='#050508'; ctx.fillRect(0,0,W,H);
  const glowA = Math.min(1, FIN.revealT/2);
  CH('人', 400, 250, 120, INK, 'center', 1, 10);
  ctx.save();
  ctx.globalAlpha = glowA;
  ctx.strokeStyle=GOLD; ctx.lineWidth=7; ctx.lineCap='round';
  ctx.shadowColor=GOLD; ctx.shadowBlur=26;
  ctx.beginPath(); ctx.moveTo(478,200); ctx.quadraticCurveTo(540,292,610,338); ctx.stroke();
  ctx.restore();
  if(FIN.revealT>1.5){
    CH('字 里 行 间', W/2, 90, 46, GOLD, 'center', glowA, 16);
    ctx.fillStyle=INK; ctx.font='22px '+FONT; ctx.textAlign='center'; ctx.globalAlpha=glowA;
    ctx.fillText('她站在你旁边，头发被风吹起来，', W/2, 380);
    ctx.fillText('像一笔刚刚落下的、还没有干透的墨。', W/2, 412);
    ctx.fillStyle=INK_DIM; ctx.font='19px '+FONT;
    ctx.fillText('觉醒值 '+G.awaken+' · 剩余贝 ×'+G.coins+' · 谢谢游玩', W/2, 458);
    ctx.globalAlpha=1;
    if(FIN.revealT>3) btn(W/2-90, 486, 180, 40, '再玩一次', {fn:()=>{ resetGame(); }});
  }
}
function resetGame(){
  G.day=1; G.coins=0; G.awaken=0; G.inv={}; G.items={}; G.flags={}; G.form='人';
  clearSave();
  G.state='title';
}

/* =========================================================
   标题
   ========================================================= */
function drawTitle(){
  const t = G.time;
  ctx.fillStyle='#0b0b12'; ctx.fillRect(0,0,W,H);
  // 漂浮的字
  const fs = ['人','厨','火','字','味','云','灯','门','锁','墨','刀','贝'];
  fs.forEach((c,i)=>{
    const x = (i*173 + Math.sin(t*0.3+i)*30)%W;
    const y = 60 + (i*97)%420 + Math.cos(t*0.4+i*2)*16;
    CH(c, x<0?x+W:x, y, 26+ (i%3)*8, 'rgba(233,226,208,'+(0.05+0.04*(i%3))+')');
  });
  CH('字 里 行 间', W/2, 170, 78, INK, 'center', 1, 22);
  ctx.fillStyle=INK_DIM; ctx.font='21px '+FONT; ctx.textAlign='center';
  ctx.fillText('字不是定死的，是活出来的。', W/2, 240);
  // 主角 + 她
  CH('人', W/2-70, 340, 66, INK, 'center', 1, 8);
  drawGirl(W/2+70, 352, 3, t, 0.9);
  const has = loadSave();
  btn(W/2-190, 420, 170, 52, '新的开始', {fn:()=>{
    resetGame(); G.state='intro'; dayIntro();
  }});
  btn(W/2+20, 420, 170, 52, '继续', {disabled:!has, fn:()=>{
    const s = loadSave(); if(!s) return;
    G.day=s.day; G.coins=s.coins; G.awaken=s.awaken;
    G.inv=s.inv||{}; G.items=s.items||{}; G.flags=s.flags||{};
    if(s.phase==='night') startNight(G.day===4 && s.flags.menOpen? 4: G.day);
    else dayIntro();
  }});
  ctx.fillStyle='#55504a'; ctx.font='16px '+FONT;
  ctx.fillText('键盘 A/D 移动 · E 互动 · 空格 确认 · 需要音频与键盘', W/2, 516);
}

/* =========================================================
   主循环 / 按键分发
   ========================================================= */
function onKey(k){
  if(G.state==='title') return;
  if(dlg){
    const L = dlg.lines[dlg.i];
    if(L && L.choices && dlg.ch>=L.tx.length && ['1','2','3','4'].includes(k)){
      const c = L.choices[parseInt(k)-1];
      if(c){ dlg=null; c.fn && c.fn(); }
      return;
    }
    if(k===' '||k==='e'||k==='enter') dlgAdvance();
    return;
  }
  if(chaiUI){
    if(k==='escape'){ chaiUI=null; return; }
    if(['1','2','3'].includes(k)){
      const p = chaiUI.parts[parseInt(k)-1];
      if(p){ const f=chaiUI.onTake, part=p.ch; chaiUI=null; beep(760,0.1,'triangle',0.12); f&&f(part); }
    }
    return;
  }
  if(G.state==='cook'){
    if(cook.phase==='fire' && k===' ') cookStamp();
    else if(cook.phase==='stamp' && (k===' '||k==='e')){
      const a = cook.after; cook.after=null;
      if(a) a(); else nextOrder();
    }
    return;
  }
  if(G.state==='night'){
    if(k==='e') nightInteract();
    else if(k==='f') drawLiubai();
    else if(k==='1') transformTo('大');
    else if(k==='2') transformTo('个');
    else if(k==='3') transformTo('囚');
    else if(k==='0') transformTo('人');
    else if(k==='m'){ MUTE=!MUTE; toast(MUTE?'已静音':'声音开启'); }
    // 第五夜走到尽头
    if(N.id===5 && N.px>2320 && !N.fin){ N.fin=true; startFinal(); }
    return;
  }
  if(G.state==='final'){ return; }
}
let lastTs = 0;
function loop(ts){
  const dt = Math.min(0.05, (ts-lastTs)/1000 || 0.016);
  lastTs = ts;
  G.time += dt;
  uiBegin();
  switch(G.state){
    case 'title': drawTitle(); break;
    case 'intro': drawIntro(); break;
    case 'cook': drawCook(dt); break;
    case 'cooksum': drawCookSum(); break;
    case 'night': nightUpdate(dt); drawNight(dt); break;
    case 'final': drawFinal(dt); break;
    case 'over': drawOver(dt); break;
  }
  drawDlg(dt);
  drawChai();
  drawToasts(dt);
  mouse.clicked = false;
  requestAnimationFrame(loop);
}

/* ---------- 启动 ---------- */
(async function(){
  try{ await loadChaizi('chaizi.csv'); }
  catch(e){ console.warn('拆字表加载失败，使用内置表', e); }
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
