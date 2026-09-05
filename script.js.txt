const $=s=>document.querySelector(s), $$=s=>[...document.querySelectorAll(s)];
const icons=["🍓","🌸","🍄","🎈","🍊","🍧","🍉","🥝","🌺","🍋","🍀","🧁"];
const screens={home:$("#home"),map:$("#map"),game:$("#game")};
const state={
 level:+localStorage.getItem("tq_level")||21,
 coins:+localStorage.getItem("tq_coins")||340,
 streak:+localStorage.getItem("tq_streak")||3,
 lastDaily:localStorage.getItem("tq_daily")||"",
 pu:JSON.parse(localStorage.getItem("tq_pu")||'{"undo":3,"shuffle":3,"hint":3}'),
 sound:localStorage.getItem("tq_sound")!=="0"
};
let board=[],tray=[],history=[],matched=0,target=18,time=90,timerId=null,levelSeed=0;

function save(){localStorage.setItem("tq_level",state.level);localStorage.setItem("tq_coins",state.coins);localStorage.setItem("tq_streak",state.streak);localStorage.setItem("tq_daily",state.lastDaily);localStorage.setItem("tq_pu",JSON.stringify(state.pu));localStorage.setItem("tq_sound",state.sound?"1":"0")}
function show(n){Object.values(screens).forEach(x=>x.classList.remove("active"));screens[n].classList.add("active")}
function updateCoins(){ $("#coinCount").textContent=state.coins; $("#homeLevel").textContent=state.level;$("#playLevel").textContent=state.level;$("#gameLevel").textContent=state.level;$("#streakText").textContent=state.streak+" days"; }
function dailyAvailable(){return state.lastDaily!==new Date().toISOString().slice(0,10)}
function claimDaily(){
 if(!dailyAvailable()){modal(`<h2>🪙 Already Claimed</h2><p>Come back tomorrow for your next coin reward.</p><button class="modal-action" onclick="closeModal()">OK</button>`);return}
 state.coins+=50;state.lastDaily=new Date().toISOString().slice(0,10);save();updateCoins();
 modal(`<h2>🎁 Daily Reward!</h2><p>You received</p><div style="font-size:38px">🪙 <b>+50</b></div><button class="modal-action" onclick="closeModal()">AWESOME!</button>`);
}
function buildPreview(){
 const p=$("#previewGrid");p.innerHTML="";
 const arr=[...icons].sort(()=>Math.random()-.5).slice(0,12);
 arr.forEach(x=>{const d=document.createElement("div");d.textContent=x;p.appendChild(d)});
}
function buildMap(){
 const m=$("#mapPath");m.innerHTML="";
 for(let n=1;n<=60;n++){
  const b=document.createElement("button");
  b.className="map-node "+(n<state.level?"done " :"")+(n===state.level?"current ":"")+(n>state.level?"locked":"");
  b.textContent=n>state.level?"🔒 "+n:(n<state.level?"✓ "+n:n);
  if(n<=state.level)b.onclick=()=>startLevel(n);
  m.appendChild(b);
 }
}
function startLevel(level=state.level){
 state.level=level;save();show("game");levelSeed=level;
 target=Math.max(12,Math.min(30,12+Math.floor((level-1)/5)*3));
 matched=0;tray=[];history=[];time=Math.max(55,95-Math.floor(level/8)*3);
 $("#target").textContent=target;$("#matched").textContent=0;$("#time").textContent=time;$("#progress").style.width="0%";updatePowerups();
 generateSolvableLevel();
 clearInterval(timerId);timerId=setInterval(()=>{time--;$("#time").textContent=time;if(time<=0){clearInterval(timerId);lose("⏰ Time's Up!")}},1000);
}
function shuffle(a){return a.sort(()=>Math.random()-.5)}
function generateSolvableLevel(){
 const boardEl=$("#board");boardEl.innerHTML="";board=[];
 const groups=Math.ceil(target/3), needed=groups*3;
 const iconsPool=[];
 for(let g=0;g<groups;g++){const icon=icons[(g+levelSeed*2)%icons.length];for(let k=0;k<3;k++)iconsPool.push(icon)}
 // Create groups from bottom to top. Every top group is fully playable, guaranteeing a solution.
 const cols=4, positions=[];
 const w=Math.max(300,boardEl.clientWidth),h=Math.max(260,boardEl.clientHeight);
 const xs=[10,(w-61)/3,(w-61)/3*2,w-71], ys=[];
 const rows=Math.ceil(Math.min(needed,24)/4);
 for(let r=0;r<rows;r++)ys.push(8+r*54);
 for(let r=0;r<rows;r++)for(let c=0;c<4;c++)positions.push({x:xs[c],y:ys[r]});
 // Build groups and assign three positions. Newer group is higher z.
 for(let g=0;g<groups;g++){
   const pos=[];
   // Spread the three tiles; for harder levels allow overlap with earlier layers.
   const base=(g*3)%Math.max(1,positions.length-2);
   for(let k=0;k<3;k++){
     let p=positions[(base+k*2+g)%positions.length];
     p={x:Math.min(w-66,p.x+(g%3-1)*4),y:Math.min(h-66,p.y+(g%2)*3)};
     pos.push(p);
   }
   pos.forEach((p,k)=>board.push({id:board.length,icon:iconsPool[g*3+k],x:p.x,y:p.y,z:g*10+k,group:g,removed:false}));
 }
 // Guarantee no accidental overlap among same top group; visual depth comes from group z ordering.
 renderBoard();
}
function overlaps(a,b){
 return Math.abs(a.x+30.5-(b.x+30.5))<48 && Math.abs(a.y+30.5-(b.y+30.5))<48;
}
function isBlocked(t){
 return board.some(o=>!o.removed&&o.id!==t.id&&o.z>t.z&&overlaps(t,o));
}
function renderBoard(){
 const el=$("#board");el.querySelectorAll(".tile").forEach(x=>x.remove());
 board.filter(t=>!t.removed).forEach(t=>{
   const b=document.createElement("button");b.className="tile";b.dataset.id=t.id;b.style.left=t.x+"px";b.style.top=t.y+"px";b.style.setProperty("--z",t.z);
   if(isBlocked(t))b.classList.add("blocked"); else b.onclick=()=>pick(t.id);
   b.textContent=t.icon;el.appendChild(b);
 });
}
function record(){history.push({board:board.map(t=>({...t})),tray:[...tray],matched})}
function pick(id){
 const t=board.find(x=>x.id===id);if(!t||t.removed||isBlocked(t))return;
 record();t.removed=true;tray.push(t.icon);
 const b=document.querySelector(`[data-id="${id}"]`);if(b)b.classList.add("pop");
 setTimeout(()=>{
   // Remove every complete triple; because every level is generated in triples,
   // a selected tile always has two partners somewhere in the current solvable stack.
   let changed=true;
   while(changed){
     changed=false;
     for(const icon of [...new Set(tray)]){
       const inds=tray.map((x,i)=>x===icon?i:-1).filter(i=>i>=0);
       if(inds.length>=3){const take=new Set(inds.slice(0,3));tray=tray.filter((_,i)=>!take.has(i));matched+=3;state.coins+=3;changed=true}
     }
   }
   save();renderTray();renderBoard();updateProgress();
   if(matched>=target){clearInterval(timerId);win()}
   else if(tray.length>=7){clearInterval(timerId);lose("💥 Tray Full!")}
 },110);
}
function renderTray(){const t=$("#tray");t.innerHTML="";for(let i=0;i<7;i++){const s=document.createElement("div");s.className="slot"+(tray[i]?" filled":"");s.textContent=tray[i]||"";t.appendChild(s)}}
function updateProgress(){$("#matched").textContent=matched;$("#progress").style.width=Math.min(100,matched/target*100)+"%";updatePowerups();updateCoins()}
function updatePowerups(){["undo","shuffle","hint"].forEach(k=>$("#"+k).querySelector("b").textContent=state.pu[k])}
function win(){
 state.coins+=25;
 if(state.level>=state.level)state.level=Math.min(60,state.level+1);
 save();updateCoins();
 modal(`<div style="font-size:58px">🎉</div><h2>Level Complete!</h2><p>Beautifully done!<br><span style="font-size:28px">🪙</span> <b>+25 Coins</b></p><button class="modal-action" onclick="closeModal();startLevel(state.level)">NEXT LEVEL ▶</button>`);
}
function lose(title){modal(`<div style="font-size:52px">🐣</div><h2>${title}</h2><p>Take another try. You can do it!</p><button class="modal-action" onclick="closeModal();startLevel(state.level)">TRY AGAIN</button>`)}
function modal(html){$("#modalBody").innerHTML=html;$("#modal").classList.remove("hidden")}
function closeModal(){$("#modal").classList.add("hidden")}
function useUndo(){
 if(!state.pu.undo||!history.length)return;
 state.pu.undo--;const h=history.pop();board=h.board.map(t=>({...t}));tray=[...h.tray];matched=h.matched;save();renderTray();renderBoard();updateProgress();
}
function useShuffle(){
 if(!state.pu.shuffle)return;
 state.pu.shuffle--;
 const alive=board.filter(t=>!t.removed), positions=alive.map(t=>({x:t.x,y:t.y}));shuffle(positions);alive.forEach((t,i)=>{t.x=positions[i].x;t.y=positions[i].y});
 save();renderBoard();updatePowerups();
}
function useHint(){
 if(!state.pu.hint)return;
 const avail=board.filter(t=>!t.removed&&!isBlocked(t));if(!avail.length)return;
 // Prefer an available tile whose icon already exists in tray, otherwise a pair in board.
 const candidate=avail.find(t=>tray.includes(t.icon))||avail[0];state.pu.hint--;
 const el=document.querySelector(`[data-id="${candidate.id}"]`);if(el){el.classList.add("hint");setTimeout(()=>el.classList.remove("hint"),1300)}
 save();updatePowerups();
}
function openShop(){
 modal(`<div class="shop-title">🛒 Coin Shop</div><div class="shop-balance">Your balance: 🪙 <b>${state.coins}</b></div>
 <div class="pack-grid">
  <div class="pack"><span class="coins">🪙</span><div class="pack-info"><strong>100 Coins</strong><small>Starter Pack</small></div><button class="buy" onclick="buyPack(100,49)">₹49</button></div>
  <div class="pack"><span class="coins">🪙</span><div class="pack-info"><strong>500 Coins</strong><small>Best Value</small></div><button class="buy" onclick="buyPack(500,199)">₹199</button></div>
  <div class="pack"><span class="coins">🪙</span><div class="pack-info"><strong>1,000 Coins</strong><small>Big Pack</small></div><button class="buy" onclick="buyPack(1000,299)">₹299</button></div>
 </div><p style="font-size:11px;color:#7895a4;margin-top:12px">Payments can be connected to your preferred payment gateway for the production version.</p>`);
}
function buyPack(coins,price){
 modal(`<h2>🪙 ${coins.toLocaleString()} Coins</h2><p>Price: <b>₹${price}</b></p><p>This demo shop is ready for payment-gateway integration. No payment is processed by this web build.</p><button class="modal-action" onclick="closeModal()">OK</button>`);
}
function settings(){
 modal(`<h2>⚙ Settings</h2>
 <div class="setting-row">🔊 Sound <button class="toggle ${state.sound?"":"off"}" id="soundToggle">${state.sound?"ON":"OFF"}</button></div>
 <div class="setting-row">❓ How to Play <span>›</span></div>
 <div class="setting-row">ℹ About <span>v1.0</span></div>
 <button class="modal-action" onclick="closeModal()">DONE</button>`);
 $("#soundToggle").onclick=()=>{state.sound=!state.sound;save();settings()};
}
$("#playBtn").onclick=()=>startLevel(state.level);
$("#homeMap").onclick=$("#mapBtn").onclick=()=>{buildMap();show("map")};
$("#mapHome").onclick=()=>show("home");
$("#gameHome").onclick=()=>{clearInterval(timerId);updateCoins();show("home")};
$("#homeShop").onclick=$("#shopTop").onclick=openShop;
$("#dailyBtn").onclick=claimDaily;
$("#settingsBtn").onclick=settings;
$("#closeModal").onclick=closeModal;
$("#undo").onclick=useUndo;$("#shuffle").onclick=useShuffle;$("#hint").onclick=useHint;
updateCoins();buildPreview();buildMap();updatePowerups();
