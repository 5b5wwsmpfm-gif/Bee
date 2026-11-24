# Bee
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1,maximum-scale=1,user-scalable=no,viewport-fit=cover">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="theme-color" content="#0b1a1c">
<title>Пчела против ножей</title>
<link href="https://fonts.googleapis.com/css2?family=Press+Start+2P&display=swap" rel="stylesheet">
<style>
*{margin:0;padding:0;box-sizing:border-box;-webkit-tap-highlight-color:transparent;user-select:none;-webkit-user-select:none}
html,body{width:100%;height:100%;overflow:hidden;background:#05100f;touch-action:none;font-family:'Press Start 2P',monospace;color:#f5c518}
canvas{position:fixed;inset:0;width:100%;height:100%;image-rendering:pixelated;image-rendering:crisp-edges}
.screen{position:fixed;inset:0;display:flex;flex-direction:column;align-items:center;justify-content:space-between;
 padding:calc(env(safe-area-inset-top) + 6vh) 16px calc(env(safe-area-inset-bottom) + 20vh);pointer-events:none;text-align:center}
.center{justify-content:center;background:rgba(4,12,12,.6);padding-bottom:calc(env(safe-area-inset-bottom) + 20px)}
.hidden{display:none!important}
h1,h2{line-height:1.5;color:#f5c518;text-shadow:3px 3px 0 #000,-2px -2px 0 #000,2px -2px 0 #000,-2px 2px 0 #000}
h1{font-size:clamp(20px,7.5vw,42px)}
h2{font-size:clamp(18px,6vw,34px)}
p,.best,.hint{text-shadow:2px 2px 0 #000;font-size:clamp(10px,3.4vw,16px);line-height:1.8}
.hint{color:#d8dde0;font-size:clamp(8px,2.6vw,12px);margin-top:10px}
.bottom,.panel{display:flex;flex-direction:column;align-items:center;gap:12px}
.btn{pointer-events:auto;font-family:inherit;font-size:clamp(14px,4.6vw,22px);padding:16px 30px;background:#f5c518;color:#111;border:none;
 box-shadow:0 0 0 4px #0a0a0a,6px 6px 0 4px #0a0a0a;cursor:pointer;margin:6px}
.btn:active{transform:translate(4px,4px);box-shadow:0 0 0 4px #0a0a0a,2px 2px 0 4px #0a0a0a}
.btn.sec{background:#d8dde0}
#hud{position:fixed;left:0;right:0;top:0;display:flex;justify-content:space-between;align-items:center;
 padding:calc(env(safe-area-inset-top) + 10px) 12px 0;pointer-events:none}
#hearts img{width:28px;height:28px;image-rendering:pixelated;margin-right:4px}
#score{font-size:clamp(14px,4.5vw,22px);text-shadow:3px 3px 0 #000}
#newrec{color:#ff6b6b}
</style>
</head>
<body>
<canvas id="c"></canvas>

<div id="hud" class="hidden"><div id="hearts"></div><div id="score">0</div></div>

<div id="menu" class="screen">
  <h1>ПЧЕЛА<br>ПРОТИВ НОЖЕЙ</h1>
  <div class="bottom">
    <div class="best">РЕКОРД: <span id="bestM">0</span></div>
    <button class="btn" id="play">ИГРАТЬ</button>
    <div class="hint">Води пальцем по экрану,<br>чтобы уворачиваться</div>
  </div>
</div>

<div id="over" class="screen center hidden">
  <div class="panel">
    <h2>ИГРА<br>ОКОНЧЕНА</h2>
    <p>СЧЁТ: <span id="sc">0</span></p>
    <p>РЕКОРД: <span id="bs">0</span></p>
    <p id="newrec" class="hidden">НОВЫЙ РЕКОРД!</p>
    <button class="btn" id="again">ЕЩЁ РАЗ</button>
    <button class="btn sec" id="tomenu">МЕНЮ</button>
  </div>
</div>

<script>
const $=id=>document.getElementById(id);
const cvs=$('c'),ctx=cvs.getContext('2d');
let W=160,H=284,bg;

function rng(a){return()=>{a|=0;a=a+0x6D2B79F5|0;let t=Math.imul(a^a>>>15,1|a);t=t+Math.imul(t^t>>>7,61|t)^t;return((t^t>>>14)>>>0)/4294967296;};}
function fromRows(rows,pal){const h=rows.length,w=rows[0].length,c=document.createElement('canvas');c.width=w;c.height=h;const g=c.getContext('2d');
 for(let y=0;y<h;y++)for(let x=0;x<w;x++){const p=pal[rows[y][x]];if(p){g.fillStyle=p;g.fillRect(x,y,1,1);}}return c;}

/* ===== СПРАЙТЫ ===== */
const knifeImg=fromRows([
"..KKK..",".KGGGK.","..KHK..","..KhK..","..KHK..","..KhK..","..KHK..",
"KKKKKKK","KgggggK","KKKKKKK",
".KWSsK.",".KWSsK.",".KWSsK.",".KWSsK.",".KWSsK.",".KWSsK.",".KWSsK.",".KWSsK.",".KWSsK.",
"..KWK..","..KSK..","...K..."
],{K:'#0a0a0a',G:'#7d858c',H:'#7a4a2a',h:'#4e2e1a',g:'#3a4048',W:'#ffffff',S:'#d4dbe0',s:'#8f9aa3'});

const heartRows=[".KK.KK.","KRWKRRK","KRRRRRK","KRRRRRK",".KRRRK.","..KRK..","...K..."];
const heartFull=fromRows(heartRows,{K:'#0a0a0a',R:'#d8283a',W:'#ff9a9a'}).toDataURL();
const heartEmpty=fromRows(heartRows,{K:'#0a0a0a',R:'#3a1a1e',W:'#3a1a1e'}).toDataURL();

function makeBee(fr){
 const S=30,m=new Array(S*S).fill(0);
 const el=(x,y,cx,cy,a,b,ang)=>{const dx=x+.5-cx,dy=y+.5-cy,c=Math.cos(ang),s=Math.sin(ang),u=dx*c+dy*s,v=-dx*s+dy*c;
   return u*u/(a*a)+v*v/(b*b)<=1?[u,v]:null;};
 const w1=fr?[9,10,6.5,2.3,.55]:[10,9,6.5,3.2,.9], w2=fr?[13.5,8,5.2,2.2,.95]:[14,7,5.5,2.8,1.3];
 for(let y=0;y<S;y++)for(let x=0;x<S;x++){
  let col=0,r;
  if(r=el(x,y,9.5,19,8,5.5,-.45)){const[u,v]=r;const b=Math.min(5,Math.max(0,Math.floor((u+8)/3.2)));
    if(b%2)col=v<-2.5?'#3b3b3b':'#141414';
    else col=b==0?(v>1.5?'#b8650c':'#e8951a'):(v<-2.8?'#ffe27a':v>2?'#e0951c':'#f5c518');}
  if(r=el(x,y,14.5,15,3.5,3.5,0))col=r[1]<-1.5?'#3a3a3a':'#1e1e1e';
  for(const w of [w1,w2]) if(r=el(x,y,...w))col=((x*2+y)%6==0)?'#9ec3d3':(r[1]>1?'#c6dde6':'#eaf6fa');
  if(r=el(x,y,20,16,5.5,5.5,0)){const[dx,dy]=r;col=(dx<-1.5&&dy<-1.5)?'#ffe27a':(dx+dy>4?'#e0951c':'#f5c518');}
  if(el(x,y,21.5,15.5,2.4,2.4,0))col='#0a0a0a';
  if(x==21&&y==14)col='#ffffff';
  m[y*S+x]=col;
 }
 const o=m.slice();
 for(let y=0;y<S;y++)for(let x=0;x<S;x++){if(o[y*S+x])continue;
  if((x>0&&o[y*S+x-1])||(x<S-1&&o[y*S+x+1])||(y>0&&o[(y-1)*S+x])||(y<S-1&&o[(y+1)*S+x]))m[y*S+x]='#0a0a0a';}
 [[19,10],[19,9],[18,8],[18,7],[18,6],[17,5],[22,10],[23,9],[23,8],[24,7],[25,6],[26,5],
  [7,26],[7,27],[6,28],[10,26],[11,27],[11,28],[14,23],[15,24],[16,25]].forEach(([x,y])=>m[y*S+x]='#0a0a0a');
 const c=document.createElement('canvas');c.width=S;c.height=S;const g=c.getContext('2d');
 for(let i=0;i<S*S;i++)if(m[i]){g.fillStyle=m[i];g.fillRect(i%S,Math.floor(i/S),1,1);}
 return c;
}
const beeImgs=[makeBee(0),makeBee(1)];

/* ===== ФОН (лес) ===== */
function makeBg(){
 const c=document.createElement('canvas');c.width=W;c.height=H;const g=c.getContext('2d');const R=rng(1337);
 const id=g.createImageData(W,H),d=id.data;
 const pal=[[7,17,19],[10,26,28],[15,38,40],[22,52,54],[30,68,69],[40,86,85],[52,104,100],[66,120,114]];
 const cx=W/2,cy=H*.4;
 for(let y=0;y<H;y++)for(let x=0;x<W;x++){
  const dx=(x-cx)/(W*.62),dy=(y-cy)/(H*.55);
  let v=(1-Math.sqrt(dx*dx+dy*dy))*pal.length*1.15+(((x+y)&1)?.3:-.3);
  const p=pal[Math.max(0,Math.min(pal.length-1,Math.floor(v)))],i=(y*W+x)*4;
  d[i]=p[0];d[i+1]=p[1];d[i+2]=p[2];d[i+3]=255;}
 g.putImageData(id,0,0);
 const trunk=(x,w,col)=>{g.fillStyle=col;
  for(let y=0;y<H;y++){const ww=Math.round(w+(y/H)*w*.4);g.fillRect(Math.round(x-ww/2),y,ww,1);}
  const nb=2+Math.floor(R()*3);
  for(let i=0;i<nb;i++){let by=H*.1+R()*H*.5,bx=x,dir=R()<.5?-1:1,len=6+R()*14;
   for(let j=0;j<len;j++){g.fillRect(Math.round(bx),Math.round(by),2,1);bx+=dir;by-=.6+R()*.3;}}};
 for(let i=0;i<9;i++)trunk(R()*W,2+R()*3,'rgba(14,40,42,0.55)');
 for(let i=0;i<6;i++)trunk(R()*W,4+R()*4,'rgba(8,26,28,0.7)');
 trunk(W*.2+R()*6,7,'rgba(5,14,15,0.9)');trunk(W*.8-R()*6,7,'rgba(5,14,15,0.9)');
 trunk(W*.04+R()*4,13,'#060f10');trunk(W*.96-R()*4,13,'#060f10');
 // верхние кроны
 g.fillStyle='#061314';
 for(let i=0;i<34;i++){const bx=R()*W,by=R()*H*.07,r=3+R()*9;for(let yy=-r;yy<=r;yy++){const ww=Math.round(Math.sqrt(r*r-yy*yy));g.fillRect(Math.round(bx-ww),Math.round(by+yy),ww*2,1);}}
 // коричневые ветки-арка
 const vine=(sx,sy,dir)=>{let x=sx,y=sy;const len=W*.48;
  for(let j=0;j<len;j++){const p=j/len;g.fillStyle='#2a1c12';g.fillRect(Math.round(x),Math.round(y),2,2);
   if(j%3==0){g.fillStyle='#4b3322';g.fillRect(Math.round(x),Math.round(y),1,1);}
   x+=dir*(.8+R()*.4);y-=(H*.5/len)*(1.6-1.2*p)+Math.sin(j*.4)*.3;}};
 vine(1,H*.58,1);vine(W-3,H*.58,-1);vine(1,H*.45,1);vine(W-3,H*.45,-1);
 // земля
 const gy=Math.round(H*.8);
 g.fillStyle='#0f2218';g.fillRect(0,gy,W,H-gy);
 g.fillStyle='#132c1e';
 for(let i=0;i<9;i++){const bx=R()*W,by=gy-R()*5,r=3+R()*6;for(let yy=-r;yy<=0;yy++){const ww=Math.round(Math.sqrt(r*r-yy*yy));g.fillRect(Math.round(bx-ww),Math.round(by+yy),ww*2,1);}}
 for(let i=0;i<W*(H-gy)*.5;i++){const x=Math.floor(R()*W),y=gy+Math.floor(R()*(H-gy)),lit=1-Math.abs(x-W/2)/(W/2);
  const cols=lit>.5?['#2f5a37','#24492f','#1b3a26']:['#1b3a26','#14301f','#0d1f16'];g.fillStyle=cols[Math.floor(R()*3)];g.fillRect(x,y,2,1);}
 for(let x=0;x<W;x++){const h=1+Math.floor(R()*5);g.fillStyle=R()<.5?'#1b3a26':'#24492f';g.fillRect(x,gy-h,1,h);}
 return c;
}

function resize(){
 const asp=innerHeight/innerWidth;
 if(asp>=1.2){W=160;H=Math.round(160*asp);}else{H=240;W=Math.round(240/asp);}
 cvs.width=W;cvs.height=H;ctx.imageSmoothingEnabled=false;bg=makeBg();
 if(bee){bee.x=Math.min(W-10,bee.x);bee.y=Math.min(H-10,bee.y);}
}

/* ===== ЗВУК ===== */
let ac;
function beep(f1,f2,dur,type='square',vol=.07){try{ac=ac||new(window.AudioContext||window.webkitAudioContext)();
 const o=ac.createOscillator(),g=ac.createGain(),t=ac.currentTime;o.type=type;o.frequency.setValueAtTime(f1,t);
 o.frequency.exponentialRampToValueAtTime(f2,t+dur);g.gain.setValueAtTime(vol,t);g.gain.exponentialRampToValueAtTime(.001,t+dur);
 o.connect(g);g.connect(ac.destination);o.start();o.stop(t+dur);}catch(e){}}

/* ===== СОСТОЯНИЕ ===== */
let state='menu',anim=0,time=0,lives=5,invuln=0,spawnT=0,shake=0,flash=0,deadT=0,lastScore=-1;
let bee=null,knives=[],warnings=[],parts=[];
let best=+(localStorage.getItem('beeBest')||0);
$('bestM').textContent=best;
const motes=Array.from({length:16},()=>({x:Math.random()*400,y:Math.random()*600,vx:(Math.random()-.5)*4,vy:-2-Math.random()*4,ph:Math.random()*6}));

function updHearts(){const h=$('hearts');h.innerHTML='';for(let i=0;i<5;i++){const im=new Image();im.src=i<lives?heartFull:heartEmpty;h.appendChild(im);}}

function startGame(){
 lives=5;time=0;knives=[];warnings=[];parts=[];invuln=1;spawnT=.8;lastScore=-1;
 bee={x:W/2,y:H*.7,face:1,rot:0,vy:0};state='play';
 $('menu').classList.add('hidden');$('over').classList.add('hidden');$('hud').classList.remove('hidden');
 updHearts();beep(400,900,.15);
}
function gameOver(){
 state='over';const s=Math.floor(time*10);let rec=false;
 if(s>best){best=s;localStorage.setItem('beeBest',best);rec=true;}
 $('sc').textContent=s;$('bs').textContent=best;$('bestM').textContent=best;
 $('newrec').classList.toggle('hidden',!rec);
 $('hud').classList.add('hidden');$('over').classList.remove('hidden');
}
function toMenu(){state='menu';knives=[];warnings=[];parts=[];
 $('over').classList.add('hidden');$('hud').classList.add('hidden');$('menu').classList.remove('hidden');}

$('play').onclick=startGame;$('again').onclick=startGame;$('tomenu').onclick=toMenu;

/* ===== УПРАВЛЕНИЕ ===== */
let drag=null;const keys={};
addEventListener('pointerdown',e=>{if(state==='play')drag={x:e.clientX,y:e.clientY};});
addEventListener('pointermove',e=>{if(!drag||state!=='play')return;const k=W/innerWidth*1.3;
 moveBee((e.clientX-drag.x)*k,(e.clientY-drag.y)*k);drag={x:e.clientX,y:e.clientY};});
addEventListener('pointerup',()=>drag=null);addEventListener('pointercancel',()=>drag=null);
addEventListener('keydown',e=>keys[e.key]=true);addEventListener('keyup',e=>keys[e.key]=false);
document.addEventListener('touchmove',e=>e.preventDefault(),{passive:false});
function moveBee(dx,dy){bee.x=Math.max(10,Math.min(W-10,bee.x+dx));bee.y=Math.max(12,Math.min(H-10,bee.y+dy));
 if(dx>.3)bee.face=1;else if(dx<-.3)bee.face=-1;}

/* ===== ЛОГИКА ===== */
function spawn(){
 const sp=Math.min(175,70+time*1.6),r=Math.random();
 if(time>12&&r<.25){const left=Math.random()<.5;warnings.push({x:left?3:W-5,y:24+Math.random()*(H-70),t:.7,left,sp:sp*1.1});}
 else if(time>25&&r<.42){const x=10+Math.random()*(W-20),y=-15,dx=bee.x-x,dy=bee.y-y,d=Math.hypot(dx,dy);knives.push({x,y,vx:dx/d*sp,vy:dy/d*sp});}
 else if(time>40&&r<.52){const x=W/2+(Math.random()-.5)*W*.6;for(let i=-2;i<=2;i++){const a=Math.PI/2+i*.28;knives.push({x,y:-15,vx:Math.cos(a)*sp*.8,vy:Math.sin(a)*sp*.8});}}
 else knives.push({x:6+Math.random()*(W-12),y:-15,vx:(Math.random()-.5)*20,vy:sp*(.85+Math.random()*.3)});
}
function hit(){
 lives--;updHearts();invuln=1.4;shake=.3;flash=.25;
 for(let i=0;i<16;i++)parts.push({x:bee.x,y:bee.y,vx:(Math.random()-.5)*120,vy:(Math.random()-.5)*120,life:.6,col:Math.random()<.5?'#f5c518':'#ffffff'});
 if(navigator.vibrate)navigator.vibrate(80);beep(300,60,.25,'sawtooth',.1);
 if(lives<=0){state='dying';deadT=1.2;bee.vy=-60;drag=null;}
}
function update(dt){
 anim+=dt;
 for(const m of motes){m.x=(m.x+m.vx*dt+W)%W;m.y+=m.vy*dt;if(m.y<0)m.y=H;}
 if(state==='play'){
  time+=dt;
  let kx=0,ky=0;
  if(keys.ArrowLeft||keys.a)kx--;if(keys.ArrowRight||keys.d)kx++;if(keys.ArrowUp||keys.w)ky--;if(keys.ArrowDown||keys.s)ky++;
  if(kx||ky)moveBee(kx*110*dt,ky*110*dt);
  spawnT-=dt;if(spawnT<=0){spawn();spawnT=Math.max(.22,.85-time*.012);}
  const s=Math.floor(time*10);if(s!==lastScore){$('score').textContent=s;lastScore=s;}
  if(invuln>0)invuln-=dt;
 }
 if(state==='dying'){bee.vy+=300*dt;bee.y+=bee.vy*dt;bee.rot+=6*dt;deadT-=dt;if(deadT<=0)gameOver();}
 for(let i=warnings.length-1;i>=0;i--){const w=warnings[i];w.t-=dt;
  if(w.t<=0){knives.push({x:w.left?-15:W+15,y:w.y,vx:w.left?w.sp:-w.sp,vy:0});warnings.splice(i,1);}}
 for(let i=knives.length-1;i>=0;i--){const k=knives[i];k.x+=k.vx*dt;k.y+=k.vy*dt;
  if(k.x<-40||k.x>W+40||k.y>H+40||k.y<-60){knives.splice(i,1);continue;}
  if(state==='play'&&invuln<=0){
   const sp=Math.hypot(k.vx,k.vy),dx=k.vx/sp,dy=k.vy/sp;
   const cs=[[bee.x+5*bee.face,bee.y,5],[bee.x-6*bee.face,bee.y+3,5.5]];
   let h=false;
   for(let t=-9;t<=11&&!h;t+=2){const px=k.x+dx*t,py=k.y+dy*t;
    for(const[cx,cy,r]of cs)if((px-cx)**2+(py-cy)**2<r*r){h=true;break;}}
   if(h){knives.splice(i,1);hit();}
  }}
 for(let i=parts.length-1;i>=0;i--){const p=parts[i];p.x+=p.vx*dt;p.y+=p.vy*dt;p.vy+=150*dt;p.life-=dt;if(p.life<=0)parts.splice(i,1);}
 if(shake>0)shake-=dt;if(flash>0)flash-=dt;
}

/* ===== ОТРИСОВКА ===== */
function drawKnife(x,y,a,s=1){ctx.save();ctx.translate(Math.round(x),Math.round(y));ctx.rotate(a-Math.PI/2);ctx.scale(s,s);ctx.drawImage(knifeImg,-3,-11);ctx.restore();}
function drawBee(x,y,s,face,rot){const fr=Math.floor(anim*20)%2;ctx.save();ctx.translate(Math.round(x),Math.round(y));
 if(rot)ctx.rotate(rot);ctx.scale(face*s,s);ctx.drawImage(beeImgs[fr],-15,-17);ctx.restore();}
function render(){
 ctx.save();
 if(shake>0)ctx.translate(Math.round((Math.random()-.5)*4),Math.round((Math.random()-.5)*4));
 ctx.drawImage(bg,0,0);
 for(const m of motes){if(Math.sin(anim*2+m.ph)>0){ctx.fillStyle='#8fc7b4';ctx.fillRect(Math.round(m.x),Math.round(m.y),1,1);}}
 if(state==='menu'){
  drawBee(W/2,H*.4+Math.sin(anim*3)*3,2,1,0);
  const fan=[[.13,.70,-.35,1.4],[.2,.86,-.85,2],[.4,.95,-1.35,1.7]];
  fan.forEach(([x,y,a,s],i)=>{const b=Math.sin(anim*2+i)*1.5;
   drawKnife(x*W,y*H+b,a,s);drawKnife((1-x)*W,y*H+b,Math.PI-a,s);});
 }else{
  for(const k of knives)drawKnife(k.x,k.y,Math.atan2(k.vy,k.vx));
  if(!(invuln>0&&state==='play'&&Math.floor(invuln*12)%2))drawBee(bee.x,bee.y+Math.sin(anim*8)*1,1,bee.face,bee.rot);
  for(const p of parts){ctx.fillStyle=p.col;ctx.fillRect(Math.round(p.x),Math.round(p.y),1,1);}
  for(const w of warnings)if(Math.floor(w.t*10)%2){ctx.fillStyle='#ff3b3b';ctx.fillRect(w.x,w.y-4,2,5);ctx.fillRect(w.x,w.y+3,2,2);}
 }
 ctx.restore();
 if(flash>0){ctx.fillStyle=`rgba(220,30,40,${flash*1.4})`;ctx.fillRect(0,0,W,H);}
}

let last=performance.now();
function loop(now){const dt=Math.min(.05,(now-last)/1000);last=now;update(dt);render();requestAnimationFrame(loop);}
addEventListener('resize',resize);
resize();requestAnimationFrame(loop);
</script>
</body>
</html>
