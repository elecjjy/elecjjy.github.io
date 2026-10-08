[Uploading motor3.html…]()
<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>3상 유도전동기 결선 실습 (PC·VR)</title>
<style>
  :root{
    --bg:#141a22; --panel:#1d2733; --panel2:#26333f; --line:#3a4a5a;
    --text:#e8eef3; --dim:#8fa3b3; --mint:#7fd6c2; --warn:#ffb454; --err:#ff6b57; --ok:#4be08f;
    --font:"Pretendard","Noto Sans KR","Malgun Gothic","Apple SD Gothic Neo",system-ui,sans-serif;
  }
  *{box-sizing:border-box}
  html,body{margin:0;height:100%;overflow:hidden;background:var(--bg);color:var(--text);font-family:var(--font)}
  canvas{display:block;position:fixed;inset:0;width:100%;height:100%;touch-action:none}
  #hud{position:fixed;left:0;right:0;top:0;padding:calc(env(safe-area-inset-top,0px) + 10px) 14px 0;pointer-events:none;display:flex;gap:12px;align-items:flex-start;justify-content:space-between}
  #title{font-size:15px;font-weight:700;letter-spacing:-.01em;background:rgba(20,26,34,.82);border:1px solid var(--line);border-radius:10px;padding:8px 12px;backdrop-filter:blur(6px)}
  #title small{display:block;font-weight:400;color:var(--dim);font-size:12px;margin-top:2px}
  #status{max-width:min(520px,52vw);background:rgba(20,26,34,.88);border:1px solid var(--line);border-left:4px solid var(--dim);border-radius:10px;padding:8px 12px;backdrop-filter:blur(6px);font-size:13px;line-height:1.5}
  #status b{display:block;font-size:14px;margin-bottom:2px}
  #status.ok{border-left-color:var(--ok)} #status.warn{border-left-color:var(--warn)} #status.err{border-left-color:var(--err)} #status.info{border-left-color:var(--mint)}
  #status span{color:var(--dim)}
  #bar{position:fixed;left:0;right:0;bottom:0;padding:10px 12px calc(env(safe-area-inset-bottom,0px) + 12px);display:flex;flex-wrap:wrap;gap:8px;justify-content:center;pointer-events:none}
  .btn{pointer-events:auto;font:600 14px var(--font);color:var(--text);background:var(--panel);border:1px solid var(--line);border-radius:10px;padding:10px 14px;cursor:pointer;min-height:42px}
  .btn:hover{background:var(--panel2);border-color:var(--mint)}
  .btn:focus-visible{outline:2px solid var(--mint);outline-offset:2px}
  .btn.primary{background:var(--mint);color:#0d1b17;border-color:var(--mint)}
  .btn.primary:hover{background:#9ae7d6}
  .btn:disabled{opacity:.5;cursor:default}
  #modal{position:fixed;inset:0;background:rgba(5,9,13,.7);display:none;align-items:center;justify-content:center;padding:16px;z-index:10}
  #modal.show{display:flex}
  #sheet{background:var(--panel);border:1px solid var(--line);border-radius:14px;max-width:640px;width:100%;max-height:86vh;overflow:auto;padding:20px 22px;line-height:1.65;font-size:14px}
  #sheet h2{margin:0 0 10px;font-size:18px}
  #sheet h3{margin:16px 0 6px;font-size:14px;color:var(--mint)}
  #sheet table{border-collapse:collapse;width:100%;font-size:13px;margin:6px 0}
  #sheet th,#sheet td{border:1px solid var(--line);padding:5px 8px;text-align:left}
  #sheet th{background:var(--panel2)}
  #sheet code{background:var(--panel2);padding:1px 5px;border-radius:4px}
  #sheet .row{display:flex;gap:8px;justify-content:flex-end;margin-top:14px}
  .good{color:var(--ok)} .bad{color:var(--err)} .mid{color:var(--warn)}
  @media (max-width:640px){
    #hud{flex-direction:column}
    #status{max-width:100%}
    .btn{padding:9px 11px;font-size:13px}
  }
  @media (prefers-reduced-motion:reduce){*{transition:none!important}}
</style>
</head>
<body>

<div id="hud">
  <div id="title">3상 유도전동기 결선 실습<small id="sub"></small></div>
  <div id="status" class="info" role="status" aria-live="polite"><b></b><span></span></div>
</div>

<div id="bar">
  <button class="btn" id="bHelp">📖 실습 설명</button>
  <button class="btn primary" id="bCheck">결선 검사</button>
  <button class="btn" id="bNew">새 실습</button>
  <button class="btn" id="bName">명판 변경</button>
  <button class="btn" id="bVolt">전원 380V</button>
  <button class="btn" id="bLog">📊 기록</button>
  <button class="btn" id="vr" disabled>VR 확인 중…</button>
</div>

<div id="modal" role="dialog" aria-modal="true"><div id="sheet"></div></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
(() => {
'use strict';
const FONT = '"Pretendard","Noto Sans KR","Malgun Gothic",sans-serif';
const $ = s => document.querySelector(s);
const clamp = (v,a,b) => Math.min(b, Math.max(a, v));
const SQ3 = Math.sqrt(3);

/* ---------- 데이터 ---------- */
// d: Δ결선 정격 선간전압, y: Y결선 정격 선간전압
const NAMEPLATES = [
  {label:'220/380V', d:220, y:380},
  {label:'380/660V', d:380, y:660},
  {label:'127/220V', d:127, y:220},
];
const VOLTS = [380, 220];
const st = {
  np:0, volt:380, wires:[], sel:null, hoverKey:null,
  log:[], burnt:false,
  sim:{mode:'idle', t:0, omega:0, target:0, dir:1, burning:false, burnT:0},
  status:{title:'', detail:'', level:'info'},
};
try { st.log = JSON.parse(localStorage.getItem('motor3_log')||'[]'); } catch(e){ st.log = []; }

/* ---------- 렌더러 / 씬 ---------- */
const renderer = new THREE.WebGLRenderer({antialias:true});
renderer.setPixelRatio(Math.min(window.devicePixelRatio||1, 2));
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
renderer.xr.enabled = true;
document.body.insertBefore(renderer.domElement, document.body.firstChild);
const el = renderer.domElement;

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x141a22);
scene.fog = new THREE.Fog(0x141a22, 6, 15);
const camera = new THREE.PerspectiveCamera(50, 1, 0.05, 60);
scene.add(camera);

scene.add(new THREE.HemisphereLight(0xd6e6ff, 0x1c232b, 0.85));
const sun = new THREE.DirectionalLight(0xffffff, 0.9);
sun.position.set(1.2, 3.5, 1.6);
sun.castShadow = true;
sun.shadow.mapSize.set(1024,1024);
Object.assign(sun.shadow.camera,{left:-2,right:2,top:2,bottom:-2,near:0.5,far:9});
scene.add(sun);
const heatLight = new THREE.PointLight(0xff5a1f, 0, 1.4);
scene.add(heatLight);

const stage = new THREE.Group();
stage.position.set(0, 0.85, -1.1);
scene.add(stage);

const M = (color, rough=0.6, metal=0.1, extra={}) => new THREE.MeshStandardMaterial(Object.assign({color, roughness:rough, metalness:metal}, extra));
function box(w,h,d,mat,x,y,z,parent=stage,shadow=true){
  const m = new THREE.Mesh(new THREE.BoxGeometry(w,h,d), mat);
  m.position.set(x,y,z); m.castShadow = shadow; m.receiveShadow = true; parent.add(m); return m;
}

/* 바닥 + 작업대 */
const floor = new THREE.Mesh(new THREE.PlaneGeometry(16,16), M(0x10151b,0.95,0));
floor.rotation.x = -Math.PI/2; floor.position.y = -0.85; floor.receiveShadow = true; stage.add(floor);
const grid = new THREE.GridHelper(16,32,0x26323f,0x1b242d); grid.position.y = -0.849; stage.add(grid);
box(2.4,0.06,1.0, M(0x56626f,0.5,0.2), 0,-0.03,0);
box(2.3,0.05,0.9, M(0x3a444f,0.7,0.1), 0,-0.085,0);
[[-1.12,-0.45],[1.12,-0.45],[-1.12,0.45],[1.12,0.45]].forEach(([x,z]) => box(0.06,0.76,0.06,M(0x2c343d,0.6,0.3),x,-0.47,z));
box(2.4,1.05,0.04, M(0x1d2833,0.8,0.1), 0,0.525,-0.5);          // 뒷판

/* ---------- 캔버스 텍스처 도우미 ---------- */
function canvasTex(w,h){
  const c = document.createElement('canvas'); c.width=w; c.height=h;
  const tex = new THREE.CanvasTexture(c); tex.anisotropy = 4;
  return {c, g:c.getContext('2d'), tex};
}
function rr(g,x,y,w,h,r){ g.beginPath(); g.moveTo(x+r,y); g.arcTo(x+w,y,x+w,y+h,r); g.arcTo(x+w,y+h,x,y+h,r); g.arcTo(x,y+h,x,y,r); g.arcTo(x,y,x+w,y,r); g.closePath(); }
function wrapLines(g,text,maxW){
  const out=[]; let line='';
  for(const ch of text){
    if(ch==='\n'){ out.push(line); line=''; continue; }
    const t=line+ch;
    if(g.measureText(t).width>maxW && line){ out.push(line); line=ch; } else line=t;
  }
  if(line) out.push(line); return out;
}

/* ---------- 전원 단자대 + 전동기 ---------- */
const TERM = {};        // id -> {id, base:Vector3, top:Vector3, mat, hit}
const hitMeshes = [];
const studMat = () => M(0xd6b264,0.3,0.85,{emissive:0x000000});

function makeLabel(text,w,h,color='#f2f6f9',parent=stage,x,y,z){
  const t = canvasTex(128,64);
  t.g.font = `700 44px ${FONT}`; t.g.textAlign='center'; t.g.textBaseline='middle'; t.g.fillStyle=color;
  t.g.fillText(text,64,34); t.tex.needsUpdate = true;
  const m = new THREE.Mesh(new THREE.PlaneGeometry(w,h), new THREE.MeshBasicMaterial({map:t.tex,transparent:true,depthWrite:false}));
  m.rotation.x = -Math.PI/2; m.position.set(x,y,z); parent.add(m); return m;
}
function makeTerminal(id,x,y,z,labelOffsetZ,labelColor){
  const mat = studMat();
  const stud = new THREE.Mesh(new THREE.CylinderGeometry(0.017,0.017,0.05,20), mat);
  stud.position.set(x,y+0.025,z); stud.castShadow = true; stage.add(stud);
  const nut = new THREE.Mesh(new THREE.CylinderGeometry(0.03,0.03,0.014,6), mat);
  nut.position.set(x,y+0.007,z); nut.castShadow = true; stage.add(nut);
  const washer = new THREE.Mesh(new THREE.CylinderGeometry(0.026,0.026,0.004,20), M(0x9aa5b0,0.4,0.8));
  washer.position.set(x,y+0.016,z); stage.add(washer);
  const hit = new THREE.Mesh(new THREE.CylinderGeometry(0.04,0.04,0.1,12), new THREE.MeshBasicMaterial({transparent:true,opacity:0,depthWrite:false}));
  hit.position.set(x,y+0.04,z); hit.userData.term = id; stage.add(hit); hitMeshes.push(hit);
  makeLabel(id,0.07,0.035,labelColor,stage,x,y+0.002,z+labelOffsetZ);
  TERM[id] = {id, base:new THREE.Vector3(x,y,z), top:new THREE.Vector3(x,y+0.05,z), mat};
}
// 전원 단자대 (바닥 위)
box(0.42,0.05,0.15, M(0x2a3139,0.6,0.2), -0.85,0.025,0.12);
['R','S','T'].forEach((id,i) => makeTerminal(id,-0.97+i*0.12,0.05,0.12,0.052,'#ffd27f'));
makeLabel('전원 3상',0.16,0.04,'#9fb4c4',stage,-0.85,0.052,0.175 + 0.0);

// 전동기
const motor = new THREE.Group(); motor.position.set(0.3,0,-0.02); stage.add(motor);
const bodyMat = M(0x2f7a64,0.45,0.35,{emissive:0x000000});
box(0.62,0.04,0.34, M(0x23272d,0.5,0.5), 0,0.02,0, motor);             // 베이스
const body = new THREE.Mesh(new THREE.CylinderGeometry(0.17,0.17,0.5,40), bodyMat);
body.rotation.z = Math.PI/2; body.position.set(0,0.21,0); body.castShadow = true; body.receiveShadow = true; motor.add(body);
for(let i=0;i<9;i++){
  const f = new THREE.Mesh(new THREE.CylinderGeometry(0.178,0.178,0.012,40), bodyMat);
  f.rotation.z = Math.PI/2; f.position.set(-0.2+i*0.05,0.21,0); f.castShadow = true; motor.add(f);
}
const capMat = M(0x2a3138,0.5,0.5);
[[-0.27],[0.27]].forEach(([x]) => { const c = new THREE.Mesh(new THREE.CylinderGeometry(0.16,0.17,0.05,32), capMat); c.rotation.z=Math.PI/2; c.position.set(x,0.21,0); c.castShadow = true; motor.add(c); });
const tbox = box(0.34,0.06,0.24, M(0x1c2127,0.7,0.1), 0,0.41,-0.02, motor);  // 단자함 (상면 y=0.44)
box(0.36,0.012,0.26, M(0x3b4651,0.5,0.4), 0,0.447,-0.02, motor);
// 회전체 (축 + 원판)
const rotor = new THREE.Group(); rotor.position.set(0.3+0.31,0.21,-0.02); stage.add(rotor);
const shaft = new THREE.Mesh(new THREE.CylinderGeometry(0.03,0.03,0.17,24), M(0xb9c2cb,0.25,0.9));
shaft.rotation.z = Math.PI/2; shaft.position.x = 0.07; shaft.castShadow = true; rotor.add(shaft);
const disc = new THREE.Mesh(new THREE.CylinderGeometry(0.085,0.085,0.02,32), M(0x3a444e,0.4,0.7));
disc.rotation.z = Math.PI/2; disc.position.x = 0.11; rotor.add(disc);
[0xff5a3c,0xf2f2f2,0x4aa3ff].forEach((c,i) => {
  const bar = new THREE.Mesh(new THREE.BoxGeometry(0.022,0.075,0.016), new THREE.MeshBasicMaterial({color:c}));
  bar.position.set(0.124,0,0);
  const holder = new THREE.Group(); holder.rotation.x = i*Math.PI*2/3; bar.position.y = 0.045; holder.add(bar); rotor.add(holder);
});
const arrowMat = new THREE.MeshBasicMaterial({color:0x7fd6c2});
// 모터 단자 (뒤줄 W2 U2 V2 / 앞줄 U1 V1 W1)
const MX = 0.3, MZ = -0.02;
[['W2',-0.09],['U2',0],['V2',0.09]].forEach(([id,dx]) => makeTerminal(id, MX+dx, 0.453, MZ-0.05, -0.055,'#ffffff'));
[['U1',-0.09],['V1',0],['W1',0.09]].forEach(([id,dx]) => makeTerminal(id, MX+dx, 0.453, MZ+0.05, 0.055,'#ffffff'));

// 명판
const nameTex = canvasTex(512,300);
box(0.25,0.15,0.01, M(0x70767d,0.4,0.7), 0.0,0.21,0.172, motor);
const nameMesh = new THREE.Mesh(new THREE.PlaneGeometry(0.23,0.135), new THREE.MeshStandardMaterial({map:nameTex.tex,roughness:0.4,metalness:0.3}));
nameMesh.position.set(0,0.21,0.1775); nameMesh.userData.nameplate = true; motor.add(nameMesh);
function drawNameplate(){
  const g = nameTex.g, np = NAMEPLATES[st.np];
  const grd = g.createLinearGradient(0,0,512,300); grd.addColorStop(0,'#dfe3e8'); grd.addColorStop(.5,'#b8bec6'); grd.addColorStop(1,'#d3d8de');
  g.fillStyle = grd; g.fillRect(0,0,512,300);
  g.strokeStyle='#58606a'; g.lineWidth=4; g.strokeRect(8,8,496,284);
  g.fillStyle='#1d252d'; g.textAlign='center'; g.textBaseline='alphabetic';
  g.font=`700 30px ${FONT}`; g.fillText('3상 유도전동기',256,52);
  g.font=`500 20px ${FONT}`; g.fillText('3 PHASE INDUCTION MOTOR',256,80);
  g.font=`800 64px ${FONT}`; g.fillText(`Δ/Y  ${np.label}`,256,164);
  g.font=`600 24px ${FONT}`; g.fillText('0.75kW   4P   60Hz   1730rpm',256,214);
  g.font=`500 20px ${FONT}`; g.fillText('INS.CLASS F    IP44    S1',256,254);
  nameTex.tex.needsUpdate = true;
}

/* ---------- 차단기, 램프, 전압 표시 ---------- */
const brk = new THREE.Group(); brk.position.set(-0.85,0.5,-0.455); stage.add(brk);
box(0.17,0.24,0.05, M(0x14181d,0.6,0.2), 0,0,0, brk);
const lever = new THREE.Mesh(new THREE.BoxGeometry(0.05,0.1,0.045), M(0xe9edf1,0.4,0.2)); lever.castShadow = true; brk.add(lever);
lever.position.set(0,0,0.04);
const brkHit = new THREE.Mesh(new THREE.BoxGeometry(0.2,0.28,0.12), new THREE.MeshBasicMaterial({transparent:true,opacity:0,depthWrite:false}));
brkHit.position.set(0,0,0.04); brkHit.userData.breaker = true; brk.add(brkHit);
const lampMat = new THREE.MeshStandardMaterial({color:0x222a31,emissive:0x000000,roughness:0.3});
const lamp = new THREE.Mesh(new THREE.SphereGeometry(0.03,20,16), lampMat); lamp.position.set(-0.85,0.74,-0.47); stage.add(lamp);
function setBreaker(on){ lever.rotation.x = on ? -0.5 : 0.5; lever.position.y = on ? 0.03 : -0.03; }
setBreaker(false);

const voltTex = canvasTex(512,200);
const voltMesh = new THREE.Mesh(new THREE.PlaneGeometry(0.4,0.156), new THREE.MeshBasicMaterial({map:voltTex.tex}));
voltMesh.position.set(-0.85,0.9,-0.477); stage.add(voltMesh);
function drawVolt(){
  const g=voltTex.g; g.fillStyle='#0b1218'; g.fillRect(0,0,512,200); g.strokeStyle='#2c3f50'; g.lineWidth=6; g.strokeRect(3,3,506,194);
  g.textAlign='center'; g.fillStyle='#7f98ab'; g.font=`600 28px ${FONT}`; g.fillText('공급 전원 (3상 60Hz)',256,50);
  g.fillStyle='#ffd27f'; g.font=`800 94px ${FONT}`; g.fillText(`${st.volt} V`,256,150);
  voltTex.tex.needsUpdate=true;
}

/* ---------- 상태 화면 + 3D 버튼 ---------- */
const scrTex = canvasTex(1024,320);
box(1.14,0.36,0.03, M(0x0d1217,0.5,0.3), 0.3,0.86,-0.47);
const scrMesh = new THREE.Mesh(new THREE.PlaneGeometry(1.1,0.344), new THREE.MeshBasicMaterial({map:scrTex.tex}));
scrMesh.position.set(0.3,0.86,-0.453); stage.add(scrMesh);
const LV = {info:'#7fd6c2', ok:'#4be08f', warn:'#ffb454', err:'#ff6b57'};
function drawScreen(){
  const g=scrTex.g, np=NAMEPLATES[st.np];
  g.fillStyle='#0b1218'; g.fillRect(0,0,1024,320);
  g.strokeStyle='#2c3f50'; g.lineWidth=6; g.strokeRect(3,3,1018,314);
  g.textAlign='left'; g.textBaseline='alphabetic';
  g.font=`600 26px ${FONT}`; g.fillStyle='#7f98ab';
  g.fillText(`명판 Δ/Y ${np.label}   ·   전원 3상 ${st.volt}V`,28,46);
  g.fillStyle = LV[st.status.level]||'#fff'; g.font=`700 36px ${FONT}`;
  const lines = wrapLines(g, st.status.title, 968).slice(0,4);
  lines.forEach((l,i) => g.fillText(l,28,106+i*50));
  const rpm = Math.round(Math.abs(st.sim.omega)/9*1730);
  g.textAlign='right'; g.font=`700 30px ${FONT}`; g.fillStyle='#e8eef3';
  g.fillText(`${rpm} rpm`,996,300);
  scrTex.tex.needsUpdate=true;
}

const buttons3d = [];
function make3dButton(x,labelFn,action){
  const t = canvasTex(256,80);
  const mesh = new THREE.Mesh(new THREE.PlaneGeometry(0.3,0.094), new THREE.MeshBasicMaterial({map:t.tex}));
  mesh.rotation.x = -1.0; mesh.position.set(x,0.045,0.38);
  const b = {mesh,t,labelFn,action,hover:false};
  mesh.userData.btn = b; stage.add(mesh); buttons3d.push(b);
  box(0.32,0.035,0.1, M(0x1a2027,0.6,0.2), x,0.0175,0.36);
  return b;
}
function draw3dButton(b){
  const g=b.t.g; g.clearRect(0,0,256,80);
  rr(g,3,3,250,74,14); g.fillStyle = b.hover ? '#35566a' : '#26333f'; g.fill();
  g.lineWidth=3; g.strokeStyle='#7fd6c2'; g.stroke();
  g.fillStyle='#e8eef3'; g.font=`700 28px ${FONT}`; g.textAlign='center'; g.textBaseline='middle';
  g.fillText(b.labelFn(),128,42); b.t.tex.needsUpdate=true;
}
make3dButton(-0.52,()=>'결선 검사',()=>checkWiring());
make3dButton(-0.17,()=>'새 실습',()=>newPractice());
make3dButton(0.18,()=>'명판 변경',()=>changeNameplate());
make3dButton(0.53,()=>`전원 ${st.volt}V`,()=>changeVolt());

/* ---------- 전선 ---------- */
const wireGroup = new THREE.Group(); stage.add(wireGroup);
const wireMeshes = [];
const WCOL = {R:0xd0453a, S:0x3a3f45, T:0x9aa4ad};
function wireColor(a,b){
  for(const id of [a,b]) if(WCOL[id]!==undefined) return WCOL[id];
  return 0xe8c547;
}
function rebuildWires(){
  wireMeshes.splice(0).forEach(m => { wireGroup.remove(m); m.traverse && m.traverse(o => { if(o.geometry) o.geometry.dispose(); }); });
  while(wireGroup.children.length){ const c = wireGroup.children[0]; wireGroup.remove(c); c.traverse(o => { if(o.geometry) o.geometry.dispose(); }); }
  st.wires.forEach((w,i) => {
    const A = TERM[w.a].top.clone(), B = TERM[w.b].top.clone();
    const dist = A.distanceTo(B);
    const lift = 0.08 + dist*0.2 + (i%4)*0.018;
    const up = new THREE.Vector3(0,1,0);
    const mid = A.clone().add(B).multiplyScalar(0.5).addScaledVector(up,lift);
    const curve = new THREE.CatmullRomCurve3([A, A.clone().addScaledVector(up,0.045), mid, B.clone().addScaledVector(up,0.045), B], false, 'catmullrom', 0.4);
    const mat = M(wireColor(w.a,w.b),0.5,0.1,{emissive:0x000000});
    const tube = new THREE.Mesh(new THREE.TubeGeometry(curve,40,0.0085,8,false), mat);
    tube.castShadow = true; tube.userData.wire = w.key;
    wireGroup.add(tube); wireMeshes.push(tube);
    [A,B].forEach(p => {
      const lug = new THREE.Mesh(new THREE.TorusGeometry(0.02,0.006,8,18), M(0xc2c8cf,0.35,0.8));
      lug.rotation.x = Math.PI/2; lug.position.copy(p).add(new THREE.Vector3(0,0.002,0)); wireGroup.add(lug);
    });
  });
}

/* ---------- 상태 / UI ---------- */
function setStatus(title, level='info', detail=''){
  st.status = {title, detail, level};
  const s = $('#status'); s.className = level;
  s.querySelector('b').textContent = title;
  s.querySelector('span').textContent = detail;
  drawScreen();
}
function refreshInfo(){
  const np = NAMEPLATES[st.np];
  $('#sub').textContent = `명판 Δ/Y ${np.label} · 전원 3상 ${st.volt}V · 전선 ${st.wires.length}개`;
  $('#bVolt').textContent = `전원 ${st.volt}V`;
  drawNameplate(); drawVolt(); buttons3d.forEach(draw3dButton); drawScreen();
}
function highlightSel(){
  Object.values(TERM).forEach(t => t.mat.emissive.setHex(t.id===st.sel ? 0xb88a00 : (t.id===st.hoverTerm ? 0x1b6f8f : 0)));
}

/* ---------- 결선 편집 ---------- */
function onEdit(msg){
  stopRun(); st.burnt = false;
  refreshInfo();
  setStatus(msg||'결선이 바뀌었습니다.','info','“결선 검사”를 눌러 확인하세요.');
}
function onTerm(id){
  if(!st.sel){ st.sel=id; highlightSel(); setStatus(`${id} 단자 선택`,'info','연결할 두 번째 단자를 누르세요. 같은 단자를 다시 누르면 취소됩니다.'); return; }
  if(st.sel===id){ st.sel=null; highlightSel(); setStatus('선택을 취소했습니다.','info'); return; }
  const key = [st.sel,id].sort().join('-');
  if(st.wires.some(w=>w.key===key)){ st.sel=null; highlightSel(); setStatus(`${key.replace('-',' ↔ ')} 은(는) 이미 연결되어 있습니다.`,'warn'); return; }
  st.wires.push({a:st.sel,b:id,key});
  const a=st.sel; st.sel=null; highlightSel(); rebuildWires();
  onEdit(`${a} ↔ ${id} 연결`);
}
function removeWire(key){
  st.wires = st.wires.filter(w=>w.key!==key); st.hoverKey=null; rebuildWires();
  onEdit(`${key.replace('-',' ↔ ')} 전선을 제거했습니다.`);
}

/* ---------- 결선 분석 ---------- */
const ALL = ['R','S','T','U1','V1','W1','U2','V2','W2'];
function analyze(){
  if(!st.wires.length) return {kind:'empty'};
  const p={}; ALL.forEach(i=>p[i]=i);
  const f = x => { while(p[x]!==x){ p[x]=p[p[x]]; x=p[x]; } return x; };
  st.wires.forEach(w => { p[f(w.a)] = f(w.b); });
  const g = x => f(x), same=(a,b)=>g(a)===g(b);
  const src=['R','S','T'], feed=['U1','V1','W1'];
  if(same('R','S')||same('S','T')||same('R','T')) return {kind:'short'};
  if(same('U1','V1')||same('V1','W1')||same('U1','W1')) return {kind:'phaseshort'};
  const map={}, missing=[];
  src.forEach(s => { const m = feed.filter(x=>same(x,s)); if(m.length===1) map[s]=m[0]; else missing.push(s); });
  if(missing.length) return {kind:'nofeed',missing};
  let topo=null;
  const gy = g('U2');
  if(same('U2','V2') && same('V2','W2') && ![...feed,...src].some(x=>g(x)===gy)) topo='Y';
  else if(same('U1','W2') && same('V1','U2') && same('W1','V2')) topo='Δ';
  if(!topo){
    const open = ['U2','V2','W2'].filter(id => !st.wires.some(w=>w.a===id||w.b===id));
    return {kind:'badjumper',open};
  }
  const perm = src.map(s => feed.indexOf(map[s]));
  let inv=0; for(let i=0;i<3;i++) for(let j=i+1;j<3;j++) if(perm[i]>perm[j]) inv++;
  return {kind:'ok', topo, dir: inv%2===0 ? 1 : -1, map};
}
function solutions(){
  const np=NAMEPLATES[st.np], s=[];
  if(Math.abs(np.d-st.volt)/np.d<0.1) s.push('Δ');
  if(Math.abs(np.y-st.volt)/np.y<0.1) s.push('Y');
  return s;
}

/* ---------- 검사 / 구동 ---------- */
function addLog(title, ok, topo){
  const np = NAMEPLATES[st.np];
  st.log.unshift({t:new Date().toLocaleTimeString('ko-KR',{hour12:false}), np:np.label, volt:st.volt, topo:topo||'-', res:title, ok});
  st.log = st.log.slice(0,100);
  try{ localStorage.setItem('motor3_log', JSON.stringify(st.log)); }catch(e){}
}
function checkWiring(){
  ensureAudio();
  stopRun(); st.burnt=false;
  const a = analyze(), np = NAMEPLATES[st.np];
  let title, level='err', detail='', ok=false, topo=null;
  switch(a.kind){
    case 'empty':
      title='결선이 없습니다.'; level='warn'; detail='전원 단자(R·S·T)와 전동기 단자(U1·V1·W1, U2·V2·W2)를 전선으로 연결하세요.'; break;
    case 'short':
      title='전원 단자끼리 연결되어 단락(합선)!'; detail='R·S·T 중 두 개 이상이 서로 연결되었습니다. 차단기가 트립됩니다.'; startRun('trip'); addLog('단락',false); break;
    case 'phaseshort':
      title='U1·V1·W1 단자끼리 연결되었습니다.'; detail='서로 다른 상이 직접 연결되어 단락 위험이 있습니다.'; startRun('trip'); addLog('단락',false); break;
    case 'nofeed':
      title=`전원선 ${a.missing.join('·')} 이(가) 전동기 U1·V1·W1에 연결되지 않았습니다.`; detail='3상 모두 서로 다른 입력 단자(U1·V1·W1)에 하나씩 연결해야 합니다. (결상 운전 위험)'; level='err'; addLog('전원 미결선',false); break;
    case 'badjumper':
      title='권선 결선이 Y도 Δ도 아닙니다.';
      detail = (a.open.length ? `연결되지 않은 단자: ${a.open.join(', ')}. ` : '') + 'Y: U2·V2·W2를 한 점에 묶음 / Δ: U1–W2, V1–U2, W1–V2 연결.';
      addLog('결선 오류',false); break;
    case 'ok': {
      topo = a.topo;
      const vph = topo==='Y' ? st.volt/SQ3 : st.volt;
      const ratio = vph/np.d;
      const dirTxt = a.dir===1 ? '정회전(축 방향에서 볼 때 시계)' : '역회전(반시계) — 상 2개가 바뀌어 있습니다';
      const pct = Math.round(ratio*100);
      if(ratio>=0.9 && ratio<=1.1){
        ok=true; level='ok';
        title=`정상 ${topo} 결선! ${dirTxt}`;
        detail=`상전압 ${Math.round(vph)}V = 정격의 ${pct}%. 전동기가 정격 전압으로 운전합니다.`;
        startRun('ok', a.dir); addLog('정상 운전',true,topo);
      } else if(ratio>1.1){
        level='err';
        title=`${topo} 결선: 과전압으로 권선 소손!`;
        detail=`권선에 정격의 ${pct}% 전압(${Math.round(vph)}V)이 걸렸습니다. 과전류·과열로 코일이 탑니다. `+ (solutions().length?'명판 전압과 전원 전압을 다시 비교해 보세요.':`이 명판(${np.label})은 전원 ${st.volt}V에 맞는 결선이 없습니다.`);
        startRun('burn', a.dir); addLog('과전압(소손)',false,topo);
      } else {
        level='warn';
        title=`${topo} 결선: 저전압으로 토크 부족`;
        detail=`상전압 ${Math.round(vph)}V = 정격의 ${pct}%. 기동이 안 되거나 매우 느리게 돌며 과열됩니다. `+ (solutions().length?'명판 전압과 전원 전압을 다시 비교해 보세요.':`이 명판(${np.label})은 전원 ${st.volt}V에 맞는 결선이 없습니다.`);
        startRun('under', a.dir); addLog('저전압',false,topo);
      }
      break; }
  }
  setStatus(title, level, detail);
}

/* 시뮬레이션 상태 */
function startRun(mode, dir=1){
  const s=st.sim; s.mode=mode; s.t=0; s.dir=dir; s.burning=false; s.burnT=0;
  s.target = mode==='ok'||mode==='burn' ? 9 : (mode==='under' ? 1.3 : 0);
  if(mode==='trip'){ setBreaker(true); s.tripT=0; setLamp(0xff3b2e); }
  else { setBreaker(true); setLamp(mode==='ok'?0x3cff9a: mode==='under'?0xffb454:0x3cff9a); }
}
function stopRun(){
  const s=st.sim; s.mode='idle'; s.target=0; s.burning=false; setBreaker(false); setLamp(0);
}
function setLamp(hex){ lampMat.emissive.setHex(hex); lampMat.color.setHex(hex||0x222a31); }
function toggleBreaker(){
  if(st.sim.mode!=='idle') { stopRun(); setStatus('차단기를 내렸습니다.','info','전동기가 서서히 멈춥니다.'); }
  else checkWiring();
}

/* 상태 변경 */
function changeNameplate(){ st.np=(st.np+1)%NAMEPLATES.length; onEdit(`명판 변경: Δ/Y ${NAMEPLATES[st.np].label}`); }
function changeVolt(){ st.volt = VOLTS[(VOLTS.indexOf(st.volt)+1)%VOLTS.length]; onEdit(`공급 전원 ${st.volt}V (3상)`); }
function newPractice(){
  const combos=[[0,220],[0,380],[1,380],[2,220]];
  let c; do{ c = combos[Math.floor(Math.random()*combos.length)]; }while(c[0]===st.np && c[1]===st.volt && Math.random()<0.7);
  st.np=c[0]; st.volt=c[1]; st.wires=[]; st.sel=null; st.hoverKey=null; rebuildWires(); highlightSel();
  stopRun(); st.burnt=false; refreshInfo();
  setStatus('새 실습: 명판과 전원 전압을 확인하세요.','info','명판의 전압과 전원 전압을 비교해 Y 또는 Δ 결선을 정하고 전선을 연결한 뒤 “결선 검사”를 누르세요.');
}

/* ---------- 연기 / 효과 ---------- */
const smoke=[]; const smokeGeo=new THREE.SphereGeometry(0.05,8,6);
function emitSmoke(){
  if(smoke.length>90) return;
  const m=new THREE.Mesh(smokeGeo,new THREE.MeshBasicMaterial({color:0x6b6b6b,transparent:true,opacity:0.5,depthWrite:false}));
  m.position.set(MX+(Math.random()-0.5)*0.3,0.5,MZ+(Math.random()-0.5)*0.15); stage.add(m);
  smoke.push({m,vy:0.12+Math.random()*0.08,vx:(Math.random()-0.5)*0.03,life:0,max:2.2+Math.random()});
}
function updateSmoke(dt){
  for(let i=smoke.length-1;i>=0;i--){
    const s=smoke[i]; s.life+=dt; s.m.position.y+=s.vy*dt; s.m.position.x+=s.vx*dt;
    const k=s.life/s.max; s.m.scale.setScalar(1+k*2.6); s.m.material.opacity=0.5*(1-k);
    if(k>=1){ stage.remove(s.m); s.m.material.dispose(); smoke.splice(i,1); }
  }
}

/* ---------- 소리 ---------- */
let actx=null, osc=null, gainN=null;
function ensureAudio(){
  try{
    if(!actx){
      actx=new (window.AudioContext||window.webkitAudioContext)();
      osc=actx.createOscillator(); osc.type='sawtooth'; osc.frequency.value=60;
      const lp=actx.createBiquadFilter(); lp.type='lowpass'; lp.frequency.value=380;
      gainN=actx.createGain(); gainN.gain.value=0;
      osc.connect(lp); lp.connect(gainN); gainN.connect(actx.destination); osc.start();
    }
    if(actx.state==='suspended') actx.resume();
  }catch(e){ actx=null; }
}
function updateAudio(){
  if(!actx||!gainN) return;
  const w=Math.abs(st.sim.omega);
  gainN.gain.setTargetAtTime(w>0.2?Math.min(0.05,0.012+w*0.004):0, actx.currentTime, 0.1);
  osc.frequency.setTargetAtTime(38+w*9, actx.currentTime, 0.1);
}

/* ---------- 매 프레임 갱신 ---------- */
let lastRpmDraw=0;
function updateSim(dt, now){
  const s=st.sim;
  s.t+=dt;
  if(s.mode==='trip'){
    s.tripT=(s.tripT||0)+dt;
    if(s.tripT>0.7){ stopRun(); setLamp(0); }
    else setLamp(Math.floor(s.tripT*10)%2?0xff3b2e:0);
  }
  if(s.mode==='ok'||s.mode==='under'||s.mode==='burn'){
    const tau = s.mode==='under'?1.6:0.8;
    s.omega += (s.target*s.dir - s.omega)*(1-Math.exp(-dt/tau));
    if(s.mode==='under'){ motor.position.x = 0.3 + Math.sin(s.t*55)*0.0015; motor.position.y = Math.sin(s.t*47)*0.0012; }
    if(s.mode==='burn' && s.t>1.2){
      if(!s.burning){ s.burning=true; setLamp(0xff3b2e); }
      s.target = Math.max(0,s.target-dt*2.4);
      if(s.t>4.5){ stopRun(); st.burnt=true; setStatus('전동기 소손','err','차단기가 동작했습니다. “새 실습” 또는 결선·명판을 바꿔 다시 시도하세요.'); }
    }
  } else {
    s.omega *= Math.exp(-dt*1.6); if(Math.abs(s.omega)<0.02) s.omega=0;
    motor.position.x=0.3; motor.position.y=0;
  }
  rotor.rotation.x -= s.omega*dt;
  // 발열
  const heatT = s.burning ? 1 : (st.burnt ? 0.25 : 0);
  const cur = bodyMat.emissive.r*1;
  const next = cur + (heatT*0.7 - cur)*Math.min(1,dt*1.5);
  bodyMat.emissive.setRGB(next, next*0.22, next*0.05);
  heatLight.position.set(stage.position.x+MX, stage.position.y+0.45, stage.position.z+MZ);
  heatLight.intensity = next*2.2;
  if(s.burning && Math.random()<0.7) emitSmoke();
  if(st.burnt && Math.random()<0.05) emitSmoke();
  updateSmoke(dt);
  if(now-lastRpmDraw>150 && (Math.abs(s.omega)>0.02 || lastRpmDraw!==-1)){ lastRpmDraw = Math.abs(s.omega)>0.02 ? now : -1; drawScreen(); }
  updateAudio();
}

/* ---------- 카메라 (PC) ---------- */
const cam = {theta:0, phi:1.0, r:2.1, target:new THREE.Vector3(0,1.15,-1.05)};
function updateCam(){
  const sp=Math.sin(cam.phi);
  camera.position.set(cam.target.x+cam.r*sp*Math.sin(cam.theta), cam.target.y+cam.r*Math.cos(cam.phi), cam.target.z+cam.r*sp*Math.cos(cam.theta));
  camera.lookAt(cam.target);
}
function resize(){
  const w=window.innerWidth,h=window.innerHeight;
  renderer.setSize(w,h,false); camera.aspect=w/h; camera.updateProjectionMatrix();
}
window.addEventListener('resize',resize);
resize();
if(window.innerWidth/window.innerHeight<1) cam.r=3.3;

/* ---------- 입력 ---------- */
const raycaster=new THREE.Raycaster(), mouse=new THREE.Vector2();
function pickables(){ return [...hitMeshes, ...wireMeshes, ...buttons3d.map(b=>b.mesh), brkHit, nameMesh]; }
function pickRay(){ const h=raycaster.intersectObjects(pickables(),false); return h.length?h[0].object:null; }
function pickMouse(e){
  const r=el.getBoundingClientRect();
  mouse.set(((e.clientX-r.left)/r.width)*2-1, -((e.clientY-r.top)/r.height)*2+1);
  raycaster.setFromCamera(mouse,camera); return pickRay();
}
function activate(o){
  const u=o.userData;
  if(u.term) onTerm(u.term);
  else if(u.wire) removeWire(u.wire);
  else if(u.btn) { ensureAudio(); u.btn.action(); }
  else if(u.breaker) toggleBreaker();
  else if(u.nameplate) changeNameplate();
}
function setHover(o){
  const u=o?o.userData:{};
  const term=u.term||null, wire=u.wire||null, btn=u.btn||null;
  if(term!==st.hoverTerm){ st.hoverTerm=term; highlightSel(); }
  if(wire!==st.hoverKey){
    st.hoverKey=wire;
    wireMeshes.forEach(m=>m.material.emissive.setHex(m.userData.wire===wire?0x7a1d14:0));
  }
  buttons3d.forEach(b=>{ const h=b===btn; if(b.hover!==h){ b.hover=h; draw3dButton(b); } });
  el.style.cursor = o ? 'pointer' : 'grab';
}
const ptrs=new Map(); let down=null;
el.addEventListener('pointerdown',e=>{
  el.setPointerCapture(e.pointerId);
  ptrs.set(e.pointerId,{x:e.clientX,y:e.clientY});
  if(ptrs.size===1) down={x:e.clientX,y:e.clientY,moved:false}; else if(down) down.moved=true;
});
el.addEventListener('pointermove',e=>{
  const p=ptrs.get(e.pointerId);
  if(!p){ setHover(pickMouse(e)); return; }
  const dx=e.clientX-p.x, dy=e.clientY-p.y;
  if(ptrs.size===1){
    if(down && Math.hypot(e.clientX-down.x,e.clientY-down.y)>6) down.moved=true;
    if(down && down.moved){ cam.theta-=dx*0.006; cam.phi=clamp(cam.phi-dy*0.005,0.25,1.45); }
  } else if(ptrs.size===2){
    const other=[...ptrs.entries()].find(([id])=>id!==e.pointerId)[1];
    const before=Math.hypot(p.x-other.x,p.y-other.y), after=Math.hypot(e.clientX-other.x,e.clientY-other.y);
    if(after>0) cam.r=clamp(cam.r*before/after,0.8,5);
  }
  p.x=e.clientX; p.y=e.clientY;
});
const endPtr=e=>{
  const wasSingle=ptrs.size===1;
  ptrs.delete(e.pointerId);
  if(wasSingle && down && !down.moved && e.type==='pointerup' && e.button===0){ const o=pickMouse(e); if(o) activate(o); }
  if(ptrs.size===0) down=null;
};
el.addEventListener('pointerup',endPtr);
el.addEventListener('pointercancel',endPtr);
el.addEventListener('wheel',e=>{ e.preventDefault(); cam.r=clamp(cam.r*(1+e.deltaY*0.001),0.8,5); },{passive:false});
el.style.cursor='grab';

/* ---------- 모달 (설명 / 기록) ---------- */
const modal=$('#modal'), sheet=$('#sheet');
function openModal(html){ sheet.innerHTML=html; modal.classList.add('show'); const b=sheet.querySelector('button'); if(b) b.focus(); }
function closeModal(){ modal.classList.remove('show'); }
modal.addEventListener('click',e=>{ if(e.target===modal) closeModal(); });
window.addEventListener('keydown',e=>{ if(e.key==='Escape') closeModal(); });
function showHelp(){
  openModal(`
  <h2>3상 유도전동기 결선 실습</h2>
  <p>전동기 <b>명판</b>의 전압과 <b>공급 전원</b> 전압을 비교해 알맞은 결선(Y 또는 Δ)을 만들고, 전동기가 정상으로 도는지 확인하는 실습입니다.</p>
  <h3>조작 방법</h3>
  <ul>
    <li>단자를 차례로 눌러 두 단자 사이에 전선을 연결합니다. 전선을 누르면 제거됩니다.</li>
    <li>화면 드래그로 회전, 휠·핀치로 확대/축소, 명판이나 3D 버튼도 눌러 쓸 수 있습니다.</li>
    <li>VR: 컨트롤러 트리거로 단자·전선·버튼·차단기를 선택합니다.</li>
  </ul>
  <h3>전동기 단자 배열</h3>
  <table><tr><th>뒷줄</th><td>W2</td><td>U2</td><td>V2</td></tr><tr><th>앞줄</th><td>U1</td><td>V1</td><td>W1</td></tr></table>
  <h3>결선 규칙</h3>
  <table>
    <tr><th>전원 연결</th><td><code>R–U1</code>, <code>S–V1</code>, <code>T–W1</code> (두 상을 바꾸면 역회전)</td></tr>
    <tr><th>Y 결선</th><td><code>U2–V2</code>, <code>V2–W2</code> (U2·V2·W2를 한 점에 묶음)</td></tr>
    <tr><th>Δ 결선</th><td><code>U1–W2</code>, <code>V1–U2</code>, <code>W1–V2</code></td></tr>
  </table>
  <h3>판정</h3>
  <p>권선 한 상에 걸리는 전압(Y는 선간전압÷√3, Δ는 선간전압)을 명판의 정격(Δ 결선 전압)과 비교합니다.</p>
  <ul>
    <li><span class="good">정격의 90~110%</span>: 정상 운전</li>
    <li><span class="mid">90% 미만</span>: 토크 부족, 느리게 회전·과열</li>
    <li><span class="bad">110% 초과</span>: 과전압으로 소손</li>
    <li><span class="bad">전원 단자끼리 연결</span>: 단락, 차단기 트립</li>
  </ul>
  <p>예) 명판 220/380V에 전원 380V → <b>Y</b>, 전원 220V → <b>Δ</b>.</p>
  <div class="row"><button class="btn primary" onclick="document.getElementById('modal').classList.remove('show')">닫기</button></div>`);
}
function showLog(){
  const total=st.log.length, okN=st.log.filter(l=>l.ok).length;
  const rows=st.log.map(l=>`<tr><td>${l.t}</td><td>${l.np}</td><td>${l.volt}V</td><td>${l.topo}</td><td class="${l.ok?'good':'bad'}">${l.res}</td></tr>`).join('');
  openModal(`<h2>📊 기록</h2>
  <p>검사 ${total}회 · 정상 운전 ${okN}회${total?` (${Math.round(okN/total*100)}%)`:''}</p>
  ${total?`<table><tr><th>시각</th><th>명판</th><th>전원</th><th>결선</th><th>결과</th></tr>${rows}</table>`:'<p>아직 기록이 없습니다. 결선 검사를 실행하면 결과가 쌓입니다.</p>'}
  <div class="row"><button class="btn" id="clr">기록 지우기</button><button class="btn primary" id="cls">닫기</button></div>`);
  $('#cls').onclick=closeModal;
  $('#clr').onclick=()=>{ st.log=[]; try{localStorage.removeItem('motor3_log');}catch(e){} showLog(); };
}
$('#bHelp').onclick=showHelp; $('#bLog').onclick=showLog;
$('#bCheck').onclick=()=>checkWiring(); $('#bNew').onclick=()=>{ensureAudio(); newPractice();};
$('#bName').onclick=changeNameplate; $('#bVolt').onclick=changeVolt;

/* ---------- VR ---------- */
const vrBtn=$('#vr');
(async()=>{
  if(!navigator.xr){ vrBtn.textContent='VR 미지원'; return; }
  try{
    const ok=await navigator.xr.isSessionSupported('immersive-vr');
    if(ok){ vrBtn.textContent='🥽 VR 입장'; vrBtn.disabled=false; } else vrBtn.textContent='VR 미지원';
  }catch(e){ vrBtn.textContent='VR 미지원'; }
})();
vrBtn.onclick=async()=>{
  if(renderer.xr.isPresenting){ renderer.xr.getSession().end(); return; }
  try{
    const s=await navigator.xr.requestSession('immersive-vr',{optionalFeatures:['local-floor','bounded-floor']});
    renderer.xr.setReferenceSpaceType('local-floor');
    await renderer.xr.setSession(s);
  }catch(e){ setStatus('VR 세션을 시작하지 못했습니다.','warn',String(e.message||e)); }
};
renderer.xr.addEventListener('sessionstart',()=>{ vrBtn.textContent='VR 종료'; });
renderer.xr.addEventListener('sessionend',()=>{ vrBtn.textContent='🥽 VR 입장'; resize(); });
const tmpM=new THREE.Matrix4();
for(let i=0;i<2;i++){
  const c=renderer.xr.getController(i);
  c.addEventListener('selectstart',()=>{
    ensureAudio();
    tmpM.identity().extractRotation(c.matrixWorld);
    raycaster.ray.origin.setFromMatrixPosition(c.matrixWorld);
    raycaster.ray.direction.set(0,0,-1).applyMatrix4(tmpM);
    const o=pickRay(); if(o) activate(o);
  });
  const line=new THREE.Line(new THREE.BufferGeometry().setFromPoints([new THREE.Vector3(),new THREE.Vector3(0,0,-1)]),new THREE.LineBasicMaterial({color:0x7fd6c2}));
  line.scale.z=3; c.add(line); scene.add(c);
}

/* ---------- 시작 ---------- */
refreshInfo();
setStatus('명판과 전원 전압을 확인하고 결선하세요.','info','단자를 차례로 눌러 전선을 연결한 뒤 “결선 검사”를 누르세요. 자세한 방법은 “실습 설명”을 참고하세요.');
const clock=new THREE.Clock();
renderer.setAnimationLoop(()=>{
  const dt=Math.min(clock.getDelta(),0.05), now=performance.now();
  if(!renderer.xr.isPresenting) updateCam();
  updateSim(dt,now);
  renderer.render(scene,camera);
});
})();
</script>
</body>
</html>
