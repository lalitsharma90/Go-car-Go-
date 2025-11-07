<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>3D Car Game — Challenges (Final)</title>
<style>
  *{margin:0;padding:0;box-sizing:border-box}
  body{overflow:hidden;font-family:"Comic Sans MS",cursive,sans-serif;background:#87CEEB;}

  /* Start Menu */
  #menu{
    position:absolute;inset:0;z-index:20;
    display:flex;flex-direction:column;align-items:center;justify-content:center;
    background:radial-gradient(circle at 30% 30%,#d2f0ff 0%,#a3e0ff 40%,#87CEEB 100%);
  }

  .menu-btn{
    position:relative;z-index:2;padding:14px 28px;margin:10px;
    border-radius:14px;border:3px solid #000;
    background:linear-gradient(180deg,#ffeb3b,#ffb300);
    font-size:20px;font-weight:bold;
    box-shadow:0 10px 25px rgba(0,0,0,0.25);
  }
  .menu-btn:active{transform:scale(.96)}

  #scoreBoard,#coinBoard{
    position:absolute;top:12px;z-index:15;
    background:rgba(255,235,59,0.95);
    padding:5px 9px;border-radius:10px;border:2px solid #000;
    font-weight:bold;font-size:15px;
  }
  #scoreBoard{left:12px}
  #coinBoard{right:12px}

  #ui{
    position:absolute;bottom:18px;left:0;right:0;
    z-index:15;display:flex;justify-content:space-between;
    padding:0 30px;pointer-events:none;
  }
  .btn{
    pointer-events:auto;width:70px;height:70px;border-radius:50%;
    background:#ffeb3b;border:3px solid #000;
    box-shadow:0 8px 20px rgba(0,0,0,.25);
    font-size:28px;line-height:64px;text-align:center;
  }
  .btn:active{transform:scale(.95);background:#ffb300}

  .challenge-text {
    position:absolute;top:20%;left:50%;transform:translateX(-50%);
    background:rgba(0,0,0,0.75);color:#fff;
    padding:12px 20px;border-radius:10px;
    font-size:18px;text-align:center;z-index:25;
    animation:fadein 0.6s ease;
  }
  @keyframes fadein{from{opacity:0;transform:translate(-50%,-20px)}to{opacity:1;transform:translate(-50%,0)}}

  .confetti {
    position:absolute;border-radius:2px;z-index:100;
    animation:fall 2.6s linear forwards;opacity:0.95;
    will-change: transform, opacity;
  }
  @keyframes fall{ to{transform:translateY(100vh) rotate(720deg);opacity:0} }

  /* small top-left challenge helper (optional) */
  #challengeHint {
    position:absolute; top:56px; left:12px; z-index:16;
    background: rgba(255,255,255,0.85);
    padding:6px 10px; border-radius:8px; border:2px solid #000; display:none;
    font-weight:bold; font-size:13px;
  }
</style>
</head>
<body>

<div id="menu">
  <button id="freeMode" class="menu-btn">😻 Free Mode</button>
  <button id="challengeMode" class="menu-btn">⚠️ Challenge Mode</button>
</div>

<div id="scoreBoard">🏁 Score: 0</div>
<div id="coinBoard">💰 Coins: 0</div>
<div id="challengeHint"></div>
<canvas id="game"></canvas>

<div id="ui" style="display:none;">
  <div id="left" class="btn">⟵</div>
  <div id="right" class="btn">⟶</div>
</div>

<script src="https://cdn.jsdelivr.net/npm/three@0.152.2/build/three.min.js"></script>
<script>
/* -----------------------------
   Full game (final) with 7 challenges:
   1) score (1500)
   2) coins (10)
   3) time (30s)
   4) speed (reach speed threshold)
   5) rock dodging (avoid N rocks passing)
   6) coin streak (collect K coins consecutively)
   7) long drive (60s)
   ----------------------------- */

const canvas = document.getElementById("game");
const renderer = new THREE.WebGLRenderer({ canvas, antialias:true });
renderer.setSize(innerWidth, innerHeight);
renderer.setPixelRatio(Math.min(devicePixelRatio,2));

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x87CEEB);

const camera = new THREE.PerspectiveCamera(75, innerWidth/innerHeight, 0.1, 1000);
camera.position.set(0,6,14);
camera.rotation.x = -0.3;

/* lights */
const dirLight = new THREE.DirectionalLight(0xffffff, 1.2);
dirLight.position.set(10,20,10); scene.add(dirLight);
scene.add(new THREE.HemisphereLight(0xffffff, 0x444444, 0.6));

/* road & ground */
const road = new THREE.Mesh(new THREE.PlaneGeometry(12,300), new THREE.MeshPhongMaterial({color:0x303030}));
road.rotation.x = -Math.PI/2; road.position.z = -120; road.position.y = 0.1; scene.add(road);
const ground = new THREE.Mesh(new THREE.PlaneGeometry(200,300), new THREE.MeshPhongMaterial({color:0x228B22}));
ground.rotation.x = -Math.PI/2; ground.position.y = -0.1; scene.add(ground);

/* ----------------
   Lamborghini-like matte car (procedural)
   ---------------- */
const bodyMat = new THREE.MeshStandardMaterial({ color: 0xff1f1f, metalness: 0.2, roughness: 0.5 });
const trimMat = new THREE.MeshStandardMaterial({ color: 0x222222, metalness: 0.2, roughness: 0.4 });
const glassMat = new THREE.MeshStandardMaterial({ color: 0x0b0b0b, metalness: 0.05, roughness: 0.18 });
const tyreMat = new THREE.MeshStandardMaterial({ color: 0x0b0b0b, metalness: 0.0, roughness: 0.92 });
const rimMat = new THREE.MeshStandardMaterial({ color: 0x444444, metalness: 0.7, roughness: 0.25 });

const carGroup = new THREE.Group();
/* main long low body */
const main = new THREE.Mesh(new THREE.BoxGeometry(2.4, 0.46, 4.2), bodyMat);
main.position.y = 0.45; carGroup.add(main);
/* hood and cabin */
const hood = new THREE.Mesh(new THREE.BoxGeometry(2.4, 0.34, 1.6), bodyMat);
hood.position.set(0,0.6,1.15); hood.rotation.x = -0.06; carGroup.add(hood);
const cabin = new THREE.Mesh(new THREE.BoxGeometry(1.56, 0.46, 1.7), bodyMat);
cabin.position.set(0,0.98,-0.2); carGroup.add(cabin);
/* soften corners with little spheres */
const cornerGeo = new THREE.SphereGeometry(0.16, 10, 10);
[[-1.15,0.7,2.0],[1.15,0.7,2.0],[-1.15,0.7,-2.0],[1.15,0.7,-2.0]].forEach(p=>{
  const s = new THREE.Mesh(cornerGeo, bodyMat); s.position.set(...p); carGroup.add(s);
});
/* splitter */
const splitter = new THREE.Mesh(new THREE.BoxGeometry(1.9, 0.06, 0.28), trimMat);
splitter.position.set(0,0.32,2.12); carGroup.add(splitter);
/* side intakes */
const sideL = new THREE.Mesh(new THREE.BoxGeometry(0.5,0.18,0.95), trimMat); sideL.position.set(-1.18,0.45,0.2); sideL.rotation.y = 0.08; carGroup.add(sideL);
const sideR = sideL.clone(); sideR.position.x = 1.18; sideR.rotation.y = -0.08; carGroup.add(sideR);
/* windows */
const wind = new THREE.Mesh(new THREE.BoxGeometry(1.3,0.34,0.02), glassMat); wind.position.set(0,1.05,0.35); wind.rotation.x = -0.12; carGroup.add(wind);
const sideWL = new THREE.Mesh(new THREE.BoxGeometry(0.02,0.32,1.05), glassMat); sideWL.position.set(-0.66,1.05,-0.12); carGroup.add(sideWL);
const sideWR = sideWL.clone(); sideWR.position.x = 0.66; carGroup.add(sideWR);
/* rear deck & spoiler */
const rearDeck = new THREE.Mesh(new THREE.BoxGeometry(1.7,0.12,0.68), bodyMat); rearDeck.position.set(0,0.85,-2.05); carGroup.add(rearDeck);
const spoiler = new THREE.Mesh(new THREE.BoxGeometry(1.5,0.06,0.22), trimMat); spoiler.position.set(0,0.98,-2.3); carGroup.add(spoiler);
/* wheels */
const wheelPositions = [[-1.05,0.2,1.55],[1.05,0.2,1.55],[-1.05,0.2,-1.6],[1.05,0.2,-1.6]];
wheelPositions.forEach(pos=>{
  const arch = new THREE.Mesh(new THREE.BoxGeometry(0.52,0.26,0.98), bodyMat); arch.position.set(pos[0],0.46,pos[2]); carGroup.add(arch);
  const tyre = new THREE.Mesh(new THREE.TorusGeometry(0.39,0.12,16,48), tyreMat); tyre.rotation.x = Math.PI/2; tyre.position.set(pos[0],pos[1],pos[2]); carGroup.add(tyre);
  const rim = new THREE.Mesh(new THREE.CylinderGeometry(0.15,0.15,0.06,24), rimMat); rim.rotation.x = Math.PI/2; rim.position.set(pos[0],pos[1],pos[2]); carGroup.add(rim);
});
carGroup.scale.set(0.98,0.98,0.98); carGroup.position.set(0,0.4,4); scene.add(carGroup);

/* Env: trees, mountains, lane stripes (unchanged from before) */
const trees = [];
for(let i=0;i<18;i++){
  const trunk = new THREE.Mesh(new THREE.CylinderGeometry(.1,.1,1), new THREE.MeshPhongMaterial({color:0x8B4513}));
  const leaves = new THREE.Mesh(new THREE.SphereGeometry(.7), new THREE.MeshPhongMaterial({color:0x228B22}));
  const g = new THREE.Group(); trunk.position.y = .5; leaves.position.y = 1.3; g.add(trunk,leaves);
  g.position.set((i%2==0?-7:7),0,-i*18); scene.add(g); trees.push(g);
}
const leftM = [], rightM = [];
for(let i=0;i<12;i++){
  const m = new THREE.Mesh(new THREE.ConeGeometry(6+Math.random()*4,6+Math.random()*6,5),
    new THREE.MeshPhongMaterial({color:0x6b7a4f,flatShading:true}));
  m.position.set(-25,0,-i*30); scene.add(m); leftM.push(m);
  const m2 = m.clone(); m2.position.x = 25; scene.add(m2); rightM.push(m2);
}
const laneStrips = []; const posX = [-1.5,1.5];
for(let x of posX){ for(let z=-10; z>-300; z-=14){
  const p = new THREE.Mesh(new THREE.PlaneGeometry(.18,6), new THREE.MeshBasicMaterial({color:0xffffff}));
  p.rotation.x = -Math.PI/2; p.position.set(x,.12,z); scene.add(p); laneStrips.push(p);
}}

/* Rocks & Coins */
const rocks = [], coins = [];
function createRock(){
  const r = new THREE.Mesh(new THREE.DodecahedronGeometry(1.3), new THREE.MeshPhongMaterial({color:0x8B4513}));
  const lanes = [-3,0,3];
  r.position.set(lanes[Math.floor(Math.random()*3)], 0.8, -120);
  r.__countedPassed = false;
  scene.add(r); rocks.push(r);
}
function createCoin(){
  const c = new THREE.Mesh(new THREE.CylinderGeometry(0.6,0.6,0.15,32), new THREE.MeshPhongMaterial({color:0xFFD700}));
  c.rotation.x = Math.PI/2;
  const lanes = [-3,0,3];
  c.position.set(lanes[Math.floor(Math.random()*3)], 1.5, -120);
  scene.add(c); coins.push(c);
}

/* Gameplay vars */
let carSpeed = 0.08, moveL=false, moveR=false, score = 0, coinsCount = 0, started=false;
let challenge=false, challengeType=null, challengeTarget=0, challengeTime=0;
let timerStart = 0;

/* NEW challenge-specific trackers */
let speedTarget = 0.28;        // speed challenge target (tuneable)
let rocksAvoidedNeeded = 15;   // rock dodging challenge default
let avoidTracker = 0;          // rocks passed safely
let coinStreakTarget = 5;      // coin streak target (consecutive)
let coinStreakCurrent = 0;
let longDriveTarget = 60;      // seconds

/* UI & controls */
const leftBtn = document.getElementById("left"), rightBtn = document.getElementById("right");
leftBtn.addEventListener("touchstart", ()=>moveL=true); leftBtn.addEventListener("touchend", ()=>moveL=false);
rightBtn.addEventListener("touchstart", ()=>moveR=true); rightBtn.addEventListener("touchend", ()=>moveR=false);

function pickRandomChallenge(){
  // 7 types
  const types = ["score","coins","time","speed","avoid","streak","long"];
  return types[Math.floor(Math.random()*types.length)];
}

function startGame(isChallenge){
  document.getElementById("menu").style.display = "none";
  document.getElementById("ui").style.display = "flex";
  started = true;
  // reset base trackers
  score = 0; coinsCount = 0; document.getElementById("scoreBoard").innerText = "🏁 Score: 0"; document.getElementById("coinBoard").innerText = "💰 Coins: 0";
  avoidTracker = 0; coinStreakCurrent = 0; window.rocksPassed = 0;
  if(isChallenge){
    challenge = true;
    challengeType = pickRandomChallenge();
    // set parameters
    if(challengeType === "score"){ challengeTarget = 1500; }
    if(challengeType === "coins"){ challengeTarget = 10; }
    if(challengeType === "time"){ challengeTime = 30; timerStart = Date.now(); }
    if(challengeType === "speed"){ speedTarget = 0.28; } // reach carSpeed >= speedTarget
    if(challengeType === "avoid"){ rocksAvoidedNeeded = 15; window.rocksPassed = 0; }
    if(challengeType === "streak"){ coinStreakTarget = 5; coinStreakCurrent = 0; }
    if(challengeType === "long"){ longDriveTarget = 60; timerStart = Date.now(); }
    // show popup
    const hint = document.createElement("div");
    hint.className = "challenge-text";
    hint.id = "activeChallenge";
    const txt = (challengeType==="score"?"Reach 1500 Score":
                 challengeType==="coins"?"Collect 10 Coins":
                 challengeType==="time"?"Survive 30 Seconds":
                 challengeType==="speed"?"Reach top Speed":
                 challengeType==="avoid"?"Avoid 15 Rocks":
                 challengeType==="streak"?"Collect 5 Coins In A Row":
                 "Survive 60 Seconds");
    hint.innerText = "Challenge: " + txt;
    document.body.appendChild(hint);
    setTimeout(()=>{ if(hint) hint.remove(); }, 4500);
    // show small hint box
    const hbox = document.getElementById("challengeHint");
    hbox.style.display = "block";
    hbox.innerText = "Challenge: " + txt;
  } else {
    challenge = false;
    challengeType = null;
    document.getElementById("challengeHint").style.display = "none";
  }

  // start spawners (clear any old intervals by using named handles stored on window)
  if (!window._rockInterval) window._rockInterval = setInterval(createRock, 2000);
  if (!window._coinInterval) window._coinInterval = setInterval(createCoin, 2500);

  // reset speed to initial
  carSpeed = 0.08;
}

/* Start buttons */
document.getElementById("freeMode").onclick = ()=> startGame(false);
document.getElementById("challengeMode").onclick = ()=> startGame(true);

/* Confetti */
function showConfetti(){
  for(let i=0;i<150;i++){
    const div = document.createElement("div");
    div.className = "confetti";
    div.style.left = (Math.random()*100) + "vw";
    div.style.width = (6 + Math.random()*10) + "px";
    div.style.height = (8 + Math.random()*18) + "px";
    div.style.background = `hsl(${Math.random()*360}, 80%, ${50 + Math.random()*10}%)`;
    div.style.transform = `rotate(${Math.random()*360}deg)`;
    div.style.animationDelay = (Math.random()*0.6) + "s";
    document.body.appendChild(div);
    setTimeout(()=>div.remove(), 3600);
  }
}

/* goHome: celebrate then return menu */
function goHome(text){
  // stop spawn intervals
  if(window._rockInterval){ clearInterval(window._rockInterval); window._rockInterval = null; }
  if(window._coinInterval){ clearInterval(window._coinInterval); window._coinInterval = null; }

  showConfetti();
  started = false;
  challenge = false;
  document.getElementById("challengeHint").style.display = "none";
  setTimeout(()=>{
    document.getElementById("menu").innerHTML = `
      <div style="text-align:center;color:white;text-shadow:1px 1px 3px black;">
        <div style="font-size:28px;font-weight:bold;margin-bottom:12px;">${text}</div>
        <button id='restartBtn' class='menu-btn'>🔁 Home</button>
      </div>`;
    document.getElementById("menu").style.display = "flex";
    document.getElementById("ui").style.display = "none";
    document.getElementById("restartBtn").onclick = ()=> location.reload();
  }, 1200);
}

/* Helper: increment rocksPassed when they pass beyond z threshold */
setInterval(()=>{
  for(let i=rocks.length-1;i>=0;i--){
    const r = rocks[i];
    if(!r) continue;
    if(r.position.z > 9.5 && !r.__countedPassed){
      r.__countedPassed = true;
      window.rocksPassed = (window.rocksPassed || 0) + 1;
      // safely remove the rock from scene
      scene.remove(r);
      rocks.splice(i,1);
      // if in avoid challenge, count progress
      if(challenge && challengeType === "avoid"){
        if(window.rocksPassed >= rocksAvoidedNeeded){
          goHome("🏁 Avoid Challenge Complete!");
          window.rocksPassed = 0;
        }
      }
      // also use avoidTracker generically
      avoidTracker = window.rocksPassed || avoidTracker;
    }
  }
}, 200);

/* Main animation loop */
function animate(){
  requestAnimationFrame(animate);
  if(!started){ renderer.render(scene, camera); return; }

  // controls
  if(moveL && carGroup.position.x > -3.5) carGroup.position.x -= 0.15;
  if(moveR && carGroup.position.x < 3.5) carGroup.position.x += 0.15;

  // move world
  road.position.z += carSpeed; if(road.position.z > 0) road.position.z = -120;
  ground.position.z += carSpeed; if(ground.position.z > 0) ground.position.z = -120;
  laneStrips.forEach(s=>{ s.position.z += carSpeed * 10; if(s.position.z > 20) s.position.z -= 420; });
  trees.forEach(t=>{ t.position.z += carSpeed * 5; if(t.position.z > 10) t.position.z = -150; });
  leftM.forEach((m,i)=>{ m.position.z += carSpeed * 1.5; if(m.position.z > 40) m.position.z = -400 + i*10; });
  rightM.forEach((m,i)=>{ m.position.z += carSpeed * 1.5; if(m.position.z > 40) m.position.z = -400 + i*10; });

  // rocks movement & collision
  for(let i=rocks.length-1;i>=0;i--){
    const r = rocks[i];
    r.position.z += carSpeed * 10;
    if(r.position.z > 10){
      // removal handled by pass-checker; keep removal here too if not counted
      if(!r.__countedPassed){
        r.__countedPassed = true;
        window.rocksPassed = (window.rocksPassed || 0) + 1;
        if(challenge && challengeType === "avoid"){
          if(window.rocksPassed >= rocksAvoidedNeeded){
            goHome("🏁 Avoid Challenge Complete!"); window.rocksPassed = 0;
          }
        }
      }
      scene.remove(r); rocks.splice(i,1); continue;
    }
    // collision detection with car
    if(Math.abs(carGroup.position.x - r.position.x) < 1.3 && Math.abs(carGroup.position.z - r.position.z) < 1.5){
      // collision -> fail immediately (game over)
      goHome("💥 Game Over!");
      return;
    }
  }

  // coins movement & pickup
  for(let i=coins.length-1;i>=0;i--){
    const c = coins[i];
    c.position.z += carSpeed * 10;
    c.rotation.y += 0.12;
    if(c.position.z > 10){
      scene.remove(c); coins.splice(i,1); // missed coin resets streak
      coinStreakCurrent = 0;
      continue;
    }
    if(Math.abs(carGroup.position.x - c.position.x) < 1.2 && Math.abs(carGroup.position.z - c.position.z) < 1.5){
      scene.remove(c); coins.splice(i,1); coinsCount++;
      document.getElementById("coinBoard").innerText = "💰 Coins: " + coinsCount;
      // coin streak logic
      coinStreakCurrent++;
      if(challenge && challengeType === "streak"){
        if(coinStreakCurrent >= coinStreakTarget){
          goHome("💥 Coin Streak Complete!");
          coinStreakCurrent = 0;
          return;
        }
      }
      // if not streak challenge, just increase coins
    }
  }

  // score & speed growth
  score++;
  document.getElementById("scoreBoard").innerText = "🏁 Score: " + score;
  // small speed increase with score milestones
  if(score % 1000 === 0) carSpeed += 0.02;

  // Speed challenge check
  if(challenge && challengeType === "speed"){
    // condition: carSpeed reaches certain threshold
    if(carSpeed >= speedTarget){
      goHome("⚡ Speed Challenge Complete!");
      return;
    }
  }

  // Score & coins & time & long drive checks
  if(challenge){
    if(challengeType === "score" && score >= challengeTarget){ goHome("🏆 Score Challenge Complete!"); return; }
    if(challengeType === "coins" && coinsCount >= challengeTarget){ goHome("💰 Coins Challenge Complete!"); return; }
    if(challengeType === "time"){
      const elapsed = (Date.now() - timerStart) / 1000;
      if(elapsed >= challengeTime){ goHome("⏳ Time Challenge Complete!"); return; }
    }
    if(challengeType === "long"){
      const elapsed = (Date.now() - timerStart) / 1000;
      if(elapsed >= longDriveTarget){ goHome("🏁 Long Drive Complete!"); return; }
    }
  }

  renderer.render(scene, camera);
}
animate();

/* responsiveness */
addEventListener('resize', ()=>{
  renderer.setSize(innerWidth, innerHeight);
  camera.aspect = innerWidth / innerHeight;
  camera.updateProjectionMatrix();
});

/* ensure rock/coin intervals are cleared on page reload/navigation */
addEventListener('visibilitychange', ()=>{
  if(document.hidden){
    if(window._rockInterval){ clearInterval(window._rockInterval); window._rockInterval = null; }
    if(window._coinInterval){ clearInterval(window._coinInterval); window._coinInterval = null; }
  }
});

</script>
</body>
</html>
