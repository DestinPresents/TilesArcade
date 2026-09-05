const $ = s => document.querySelector(s);
const screens = {home:$("#homeScreen"), map:$("#mapScreen"), game:$("#gameScreen")};

const state = {
  level: +(localStorage.getItem("te_level") || 1),
  coins: +(localStorage.getItem("te_coins") || 40),
  streak: +(localStorage.getItem("te_streak") || 3),
  powerups: JSON.parse(localStorage.getItem("te_powerups") || '{"undo":3,"shuffle":3,"hint":3,"extra":1}')
};

const icons = ["🍎","🍌","🍊","🍇","🍓","🥝","🍉","🌸","🍋","🥥"];
let tiles = [], tray = [], history = [], target = 12, matched = 0, timer = 60, timerId = null, trayMax = 7;

function save(){
  localStorage.setItem("te_level", state.level);
  localStorage.setItem("te_coins", state.coins);
  localStorage.setItem("te_streak", state.streak);
  localStorage.setItem("te_powerups", JSON.stringify(state.powerups));
}
function show(name){Object.values(screens).forEach(x=>x.classList.remove("active")); screens[name].classList.add("active");}
function updateTop(){ $("#coinCount").textContent=state.coins; $("#streak").textContent="x"+state.streak; $("#homeLevel").textContent=state.level; }

function buildMap(){
  const map=$("#levelMap"); map.innerHTML="";
  for(let n=1;n<=30;n++){
    const b=document.createElement("button");
    b.className="level-node "+(n===state.level?"current ":"")+(n>state.level?"locked":"");
    b.textContent=n>state.level?"🔒 "+n:n;
    if(n<=state.level)b.onclick=()=>startLevel(n);
    map.appendChild(b);
  }
}

function startLevel(level=state.level){
  state.level=level; save(); show("game");
  target = 12 + Math.min(12, Math.floor((level-1)/4)*3);
  matched=0; tray=[]; history=[]; trayMax=7; timer=62-Math.min(20,Math.floor(level/5)*3);
  $("#gameLevel").textContent=level; $("#targetCount").textContent=target; $("#timer").textContent=timer;
  generateLevel(); renderTray(); updateProgress();
  clearInterval(timerId); timerId=setInterval(()=>{timer--;$("#timer").textContent=timer;if(timer<=0){clearInterval(timerId);lose("⏰ Time's Up!");}},1000);
}
function generateLevel(){
  const board=$("#board"); board.innerHTML="";
  tiles=[];
  const count=target;
  // Every icon appears in groups of 3.
  const needed=Math.ceil(count/3)*3;
  let pool=[];
  for(let i=0;i<needed;i++) pool.push(icons[(i+Math.floor(state.level/3))%icons.length]);
  pool.sort(()=>Math.random()-.5);

  const cols=4, rows=Math.ceil(needed/4);
  const w=board.clientWidth, h=board.clientHeight;
  for(let i=0;i<needed;i++){
    const col=i%cols, row=Math.floor(i/cols);
    const x=10+col*((w-85)/3)+(row%2?8:0);
    const y=12+row*56;
    const t={id:i,icon:pool[i],x,y,removed:false,z:100+i};
    tiles.push(t);
  }
  // Add a few offset "cover" tiles so the board feels layered.
  if(state.level>=3){
    const extra = Math.min(8, Math.floor(state.level/3));
    for(let i=0;i<extra;i++){
      const base=tiles[(i*3+2)%tiles.length];
      base.x+= (i%2?14:-10); base.y+=10; base.z+=20;
    }
  }
  renderBoard();
}
function renderBoard(){
  const board=$("#board");
  board.querySelectorAll(".tile").forEach(x=>x.remove());
  const visible=tiles.filter(t=>!t.removed);
  visible.forEach(t=>{
    const el=document.createElement("button");
    el.className="tile"; el.dataset.id=t.id; el.style.left=t.x+"px";el.style.top=t.y+"px";el.style.zIndex=t.z;
    // Tile is blocked when another tile overlaps its centre.
    const blocked=visible.some(o=>o.id!==t.id && o.z>t.z && overlap(t,o));
    if(blocked)el.classList.add("blocked");
    el.textContent=t.icon;
    if(!blocked)el.onclick=()=>pickTile(t.id);
    board.appendChild(el);
  });
}
function overlap(a,b){
  return Math.abs((a.x+33)-(b.x+33))<48 && Math.abs((a.y+33)-(b.y+33))<48;
}
function pickTile(id){
  const t=tiles.find(x=>x.id===id); if(!t||t.removed)return;
  const el=document.querySelector(`[data-id="${id}"]`);
  history.push({t:{...t},tray:[...tray],matched});
  t.removed=true; tray.push(t.icon);
  if(el)el.classList.add("removing");
  setTimeout(()=>{
    const groups={}; tray.forEach((x,i)=>(groups[x]??=[]).push(i));
    let clear=[];
    Object.values(groups).forEach(arr=>{while(arr.length>=3){clear.push(...arr.splice(0,3));}});
    if(clear.length){
      const clearSet=new Set(clear);
      tray=tray.filter((_,i)=>!clearSet.has(i)); matched+=clear.length;
      state.coins+=clear.length; save();
    }
    renderTray(); renderBoard(); updateProgress();
    if(matched>=target){clearInterval(timerId); win();}
    else if(tray.length>=trayMax){clearInterval(timerId);lose("💥 Tray Full!");}
  },120);
}
function renderTray(){
  const trayEl=$("#tray");trayEl.innerHTML="";
  for(let i=0;i<trayMax;i++){const s=document.createElement("div");s.className="slot "+(tray[i]?"filled":"");s.textContent=tray[i]||"";trayEl.appendChild(s)}
}
function updateProgress(){
  $("#matchedCount").textContent=matched;$("#progressBar").style.width=Math.min(100,matched/target*100)+"%";
  $("#coinCount").textContent=state.coins;
  $("#undoCount").textContent=state.powerups.undo;$("#shuffleCount").textContent=state.powerups.shuffle;
  $("#hintCount").textContent=state.powerups.hint;$("#extraCount").textContent=state.powerups.extra;
}
function win(){
  state.coins+=25; state.level=Math.max(state.level+1,state.level); save(); updateTop();
  modal(`<h2>🎉 Level Complete!</h2><p>You cleared all the tiles.</p><p>🪙 <b>+25 Coins</b></p><button class="action" onclick="closeModal();startLevel(state.level)">NEXT LEVEL ▶</button>`);
}
function lose(title){
  modal(`<h2>${title}</h2><p>Don't worry! Try the level again.</p><button class="action" onclick="closeModal();startLevel(state.level)">TRY AGAIN</button>`);
}
function modal(html){$("#modalContent").innerHTML=html;$("#modal").classList.remove("hidden")}
function closeModal(){$("#modal").classList.add("hidden")}

$("#playBtn").onclick=()=>startLevel(state.level);
$("#mapBtn").onclick=()=>{buildMap();show("map")};
$("#mapBtn2").onclick=()=>{buildMap();show("map")};
$("#mapBack").onclick=()=>show("home");
$("#gameBack").onclick=()=>{clearInterval(timerId);updateTop();show("home")};
$("#modalClose").onclick=closeModal;
$("#settingsBtn").onclick=()=>modal(`<h2>⚙️ Settings</h2><p>Sound and music controls will be added in the next build.</p><button class="action" onclick="closeModal()">OK</button>`);
$("#shopBtn").onclick=$("#shopBtn2").onclick=()=>modal(`<h2>🛒 Explorer Shop</h2><p>Coins: 🪙 <b>${state.coins}</b></p><p>Power-up shop is ready to connect.</p><button class="action" onclick="closeModal()">CLOSE</button>`);

$("#undoBtn").onclick=()=>{
  if(!state.powerups.undo||!history.length)return;
  state.powerups.undo--;const h=history.pop();const original=tiles.find(t=>t.id===h.t.id);
  Object.assign(original,h.t);tray=h.tray;matched=h.matched;save();renderTray();renderBoard();updateProgress();
};
$("#shuffleBtn").onclick=()=>{
  if(!state.powerups.shuffle)return;
  state.powerups.shuffle--;const alive=tiles.filter(t=>!t.removed);const positions=alive.map(t=>({x:t.x,y:t.y})).sort(()=>Math.random()-.5);
  alive.forEach((t,i)=>{t.x=positions[i].x;t.y=positions[i].y});save();renderBoard();updateProgress();
};
$("#hintBtn").onclick=()=>{
  if(!state.powerups.hint)return;
  state.powerups.hint--;const alive=tiles.filter(t=>!t.removed);
  const counts={};alive.forEach(t=>counts[t.icon]=(counts[t.icon]||0)+1);
  const candidate=alive.find(t=>counts[t.icon]>=2 && !document.querySelector(`[data-id="${t.id}"]`).classList.contains("blocked"));
  if(candidate){const el=document.querySelector(`[data-id="${candidate.id}"]`);el.animate([{transform:"scale(1)"},{transform:"scale(1.22)"},{transform:"scale(1)"}],500)}
  save();updateProgress();
};
$("#extraBtn").onclick=()=>{
  if(!state.powerups.extra||trayMax>=8)return;
  state.powerups.extra--;trayMax=8;save();renderTray();updateProgress();
};

updateTop();buildMap();
