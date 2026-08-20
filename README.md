<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>DESCENT — a maze you shouldn't be in</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Special+Elite&family=Courier+Prime:wght@400;700&display=swap');

  * { margin:0; padding:0; box-sizing:border-box; }
  html, body { width:100%; height:100%; overflow:hidden; background:#000; font-family:'Courier Prime', monospace; }
  canvas { display:block; }

  #blocker {
    position:fixed; inset:0; z-index:50;
    background:radial-gradient(ellipse at center, #1a0505 0%, #000 75%);
    display:flex; align-items:center; justify-content:center; flex-direction:column;
    color:#8a1f1f; text-align:center; cursor:pointer;
    animation: flicker 6s infinite;
  }
  @keyframes flicker {
    0%,19%,21%,23%,80%,100% { opacity:1; }
    20%,22% { opacity:0.82; }
    81% { opacity:0.9; }
  }
  #blocker h1 {
    font-family:'Special Elite', cursive;
    font-size:4.2rem; letter-spacing:0.35em; color:#b32020;
    text-shadow: 0 0 25px rgba(179,32,32,0.55), 0 0 4px #400;
    margin-bottom:0.4rem;
  }
  #blocker .sub { color:#6b6560; letter-spacing:0.25em; font-size:0.8rem; margin-bottom:2.5rem; text-transform:uppercase;}
  #blocker .rules { color:#7a7570; font-size:0.85rem; line-height:1.9; max-width:420px; }
  #blocker .rules b { color:#c98686; }
  #blocker .start { margin-top:2.2rem; padding:0.8rem 2.4rem; border:1px solid #5c1414; color:#c94040;
    letter-spacing:0.3em; font-size:0.8rem; text-transform:uppercase; }
  #blocker .start:hover { background:#2a0808; }

  #hud { position:fixed; inset:0; z-index:20; pointer-events:none; display:none; }
  #crosshair {
    position:absolute; top:50%; left:50%; width:6px; height:6px; margin:-3px 0 0 -3px;
    border-radius:50%; background:rgba(200,180,170,0.55);
  }
  #battery-wrap {
    position:absolute; bottom:28px; left:28px; width:160px;
    color:#8a8078; font-size:0.65rem; letter-spacing:0.15em; text-transform:uppercase;
  }
  #battery-bar-bg { width:100%; height:6px; border:1px solid #5a5048; margin-top:6px; }
  #battery-bar { height:100%; background:#c98a2f; width:100%; transition:width 0.15s linear; }
  #promptline { position:absolute; bottom:28px; right:28px; color:#645c56; font-size:0.65rem; letter-spacing:0.1em; text-align:right; }

  #vignette {
    position:fixed; inset:0; z-index:15; pointer-events:none;
    box-shadow: inset 0 0 180px 60px rgba(0,0,0,0.85);
    background: radial-gradient(ellipse at center, rgba(120,0,0,0) 40%, rgba(120,0,0,0) 100%);
    transition: background 0.3s ease;
  }

  #overlay-msg {
    position:fixed; inset:0; z-index:60; display:none; align-items:center; justify-content:center;
    flex-direction:column; text-align:center; color:#c94040; background:rgba(0,0,0,0.0);
  }
  #overlay-msg.dead { background:#3a0000; animation:deathflash 0.4s ease; }
  @keyframes deathflash { 0%{background:#ff0000;} 100%{background:#3a0000;} }
  #overlay-msg h1 { font-family:'Special Elite', cursive; font-size:5rem; letter-spacing:0.2em; }
  #overlay-msg.win h1 { color:#5fae6a; text-shadow:0 0 30px rgba(95,174,106,0.6); }
  #overlay-msg.dead h1 { color:#e23c3c; text-shadow:0 0 30px rgba(226,60,60,0.8); }
  #overlay-msg p { margin-top:1rem; color:#948a83; letter-spacing:0.15em; font-size:0.85rem; text-transform:uppercase; }
  #overlay-msg .again { margin-top:2.2rem; padding:0.7rem 2rem; border:1px solid #5c1414; color:#c94040;
    letter-spacing:0.25em; font-size:0.75rem; text-transform:uppercase; cursor:pointer; pointer-events:auto; }
  #overlay-msg .again:hover { background:#2a0808; }
</style>
</head>
<body>

<div id="blocker">
  <h1>DESCENT</h1>
  <div class="sub">something is already inside</div>
  <div class="rules">
    <b>WASD</b> to move &nbsp;·&nbsp; <b>mouse</b> to look &nbsp;·&nbsp; <b>F</b> flashlight &nbsp;·&nbsp; <b>shift</b> to run<br><br>
    Find the green door. Your light will not last. <br>Neither, perhaps, will you.
  </div>
  <div class="start">click to descend</div>
</div>

<div id="hud">
  <div id="crosshair"></div>
  <div id="battery-wrap">battery
    <div id="battery-bar-bg"><div id="battery-bar"></div></div>
  </div>
  <div id="promptline">F — flashlight</div>
</div>

<div id="vignette"></div>

<div id="overlay-msg">
  <h1 id="overlay-title"></h1>
  <p id="overlay-sub"></p>
  <div class="again" id="again-btn">try again</div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
(function(){

  // ---------- CONSTANTS ----------
  const N = 12;                 // maze grid size
  const CELL = 4;                // cell size in world units
  const WALL_H = 3.4;
  const WALL_T = 0.28;
  const PLAYER_R = 0.32;
  const PLAYER_H = 1.65;
  const BASE_SPEED = 3.6;
  const RUN_MULT = 1.7;
  const MONSTER_SPEED = 2.55;
  const CATCH_DIST = 1.05;

  let scene, camera, renderer, clock;
  let grid = [];
  let colliders = [];
  let player = { x: CELL/2, z: CELL/2, running:false };
  let velocityBobTime = 0;
  let flashOn = false;
  let battery = 100;
  let exitPos = null;
  let exitLight, exitMesh;
  let monster = { x:0, z:0, path:[], repathTimer:0, mesh:null, light:null };
  let keys = {};
  let gameActive = false;
  let audioCtx, droneGain, tensionGain, tensionOsc;
  let flashSpot, flashTarget;

  // ---------- MAZE GENERATION (recursive backtracker) ----------
  function generateMaze(){
    for (let i=0;i<N;i++){
      grid[i] = [];
      for (let j=0;j<N;j++){
        grid[i][j] = { N:true,S:true,E:true,W:true,visited:false };
      }
    }
    const stack = [];
    let ci=0, cj=0;
    grid[ci][cj].visited = true;
    stack.push([ci,cj]);
    const dirs = [
      {d:'N', dx:0, dy:-1, o:'S'},
      {d:'S', dx:0, dy:1,  o:'N'},
      {d:'E', dx:1, dy:0,  o:'W'},
      {d:'W', dx:-1,dy:0,  o:'E'}
    ];
    while (stack.length){
      const [i,j] = stack[stack.length-1];
      const options = [];
      for (const dr of dirs){
        const ni = i+dr.dx, nj = j+dr.dy;
        if (ni>=0 && ni<N && nj>=0 && nj<N && !grid[ni][nj].visited) options.push({dr, ni, nj});
      }
      if (options.length === 0){ stack.pop(); continue; }
      const pick = options[Math.floor(Math.random()*options.length)];
      grid[i][j][pick.dr.d] = false;
      grid[pick.ni][pick.nj][pick.dr.o] = false;
      grid[pick.ni][pick.nj].visited = true;
      stack.push([pick.ni, pick.nj]);
    }
  }

  function connected(i,j,ni,nj){
    if (ni<0||ni>=N||nj<0||nj>=N) return false;
    if (ni===i-1 && nj===j) return !grid[i][j].W;
    if (ni===i+1 && nj===j) return !grid[i][j].E;
    if (ni===i && nj===j-1) return !grid[i][j].N;
    if (ni===i && nj===j+1) return !grid[i][j].S;
    return false;
  }

  function bfsPath(si,sj,ti,tj){
    const q = [[si,sj]];
    const visited = new Set([si+','+sj]);
    const prev = {};
    while (q.length){
      const [i,j] = q.shift();
      if (i===ti && j===tj) break;
      const neighbors = [[i-1,j],[i+1,j],[i,j-1],[i,j+1]];
      for (const [ni,nj] of neighbors){
        const key = ni+','+nj;
        if (ni>=0&&ni<N&&nj>=0&&nj<N && !visited.has(key) && connected(i,j,ni,nj)){
          visited.add(key);
          prev[key] = [i,j];
          q.push([ni,nj]);
        }
      }
    }
    const path = [];
    let cur = [ti,tj];
    let key = ti+','+tj;
    if (!visited.has(key)) return path;
    while (!(cur[0]===si && cur[1]===sj)){
      path.unshift(cur);
      key = cur[0]+','+cur[1];
      cur = prev[key];
      if (!cur) break;
    }
    return path;
  }

  // ---------- SCENE BUILD ----------
  function buildScene(){
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x030202);
    scene.fog = new THREE.FogExp2(0x050302, 0.075);

    camera = new THREE.PerspectiveCamera(72, window.innerWidth/window.innerHeight, 0.1, 100);
    camera.rotation.order = 'YXZ';
    camera.position.set(player.x, PLAYER_H, player.z);

    renderer = new THREE.WebGLRenderer({ antialias:true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(Math.min(window.devicePixelRatio,2));
    document.body.appendChild(renderer.domElement);

    // dim ambient
    scene.add(new THREE.AmbientLight(0x1a1410, 0.5));
    const hemi = new THREE.HemisphereLight(0x201810, 0x000000, 0.25);
    scene.add(hemi);

    // floor
    const floorGeo = new THREE.PlaneGeometry(N*CELL+4, N*CELL+4);
    const floorMat = new THREE.MeshStandardMaterial({ color:0x2a1c15, roughness:1 });
    const floor = new THREE.Mesh(floorGeo, floorMat);
    floor.rotation.x = -Math.PI/2;
    floor.position.set(N*CELL/2, 0, N*CELL/2);
    scene.add(floor);

    const wallMat = new THREE.MeshStandardMaterial({ color:0x14100d, roughness:0.95 });

    function addWall(cx, cz, sx, sz){
      const geo = new THREE.BoxGeometry(sx, WALL_H, sz);
      const mesh = new THREE.Mesh(geo, wallMat);
      mesh.position.set(cx, WALL_H/2, cz);
      scene.add(mesh);
      colliders.push({
        minX: cx - sx/2 - PLAYER_R, maxX: cx + sx/2 + PLAYER_R,
        minZ: cz - sz/2 - PLAYER_R, maxZ: cz + sz/2 + PLAYER_R
      });
    }

    for (let i=0;i<N;i++){
      for (let j=0;j<N;j++){
        const x0 = i*CELL, z0 = j*CELL;
        if (grid[i][j].N) addWall(x0+CELL/2, z0, CELL+WALL_T, WALL_T);
        if (grid[i][j].W) addWall(x0, z0+CELL/2, WALL_T, CELL+WALL_T);
        if (i===N-1 && grid[i][j].E) addWall(x0+CELL, z0+CELL/2, WALL_T, CELL+WALL_T);
      }
      if (grid[i][N-1].S) addWall(i*CELL+CELL/2, N*CELL, CELL+WALL_T, WALL_T);
    }

    // exit
    exitPos = { x:(N-1)*CELL+CELL/2, z:(N-1)*CELL+CELL/2 };
    const doorGeo = new THREE.PlaneGeometry(CELL*0.7, WALL_H*0.85);
    const doorMat = new THREE.MeshStandardMaterial({ color:0x0a3a12, emissive:0x1fae3f, emissiveIntensity:0.9, side:THREE.DoubleSide });
    exitMesh = new THREE.Mesh(doorGeo, doorMat);
    exitMesh.position.set(exitPos.x, WALL_H*0.42, (N-1)*CELL + CELL - WALL_T);
    scene.add(exitMesh);
    exitLight = new THREE.PointLight(0x33ff66, 1.4, 8);
    exitLight.position.set(exitPos.x, 2, exitPos.z);
    scene.add(exitLight);

    // flashlight rig
    flashSpot = new THREE.SpotLight(0xffdca8, 2.4, 16, Math.PI/6.2, 0.55, 1.4);
    flashSpot.visible = false;
    camera.add(flashSpot);
    flashTarget = new THREE.Object3D();
    flashTarget.position.set(0,0,-1);
    camera.add(flashTarget);
    flashSpot.target = flashTarget;
    flashSpot.position.set(0,0,0.2);
    scene.add(camera);

    // monster mesh: a gaunt dark figure
    const mGroup = new THREE.Group();
    const bodyMat = new THREE.MeshStandardMaterial({ color:0x050505, roughness:1 });
    const body = new THREE.Mesh(new THREE.CylinderGeometry(0.28, 0.4, 1.7, 8), bodyMat);
    body.position.y = 0.95;
    mGroup.add(body);
    const head = new THREE.Mesh(new THREE.SphereGeometry(0.26, 8, 8), bodyMat);
    head.position.y = 1.9;
    mGroup.add(head);
    const eyeMat = new THREE.MeshStandardMaterial({ color:0x220000, emissive:0xff1010, emissiveIntensity:2 });
    const eyeGeo = new THREE.SphereGeometry(0.035, 6, 6);
    const eyeL = new THREE.Mesh(eyeGeo, eyeMat); eyeL.position.set(-0.09,1.92,0.22);
    const eyeR = new THREE.Mesh(eyeGeo, eyeMat); eyeR.position.set(0.09,1.92,0.22);
    mGroup.add(eyeL); mGroup.add(eyeR);
    monster.light = new THREE.PointLight(0xff2200, 0.6, 4);
    monster.light.position.y = 1.9;
    mGroup.add(monster.light);
    scene.add(mGroup);
    monster.mesh = mGroup;

    // spawn monster near center, away from player start
    const startCell = { i: Math.floor(N/2), j: Math.floor(N/2) };
    monster.x = startCell.i*CELL + CELL/2;
    monster.z = startCell.j*CELL + CELL/2;
    monster.mesh.position.set(monster.x, 0, monster.z);

    window.addEventListener('resize', onResize);
  }

  function onResize(){
    camera.aspect = window.innerWidth/window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }

  // ---------- COLLISION ----------
  function resolvePosition(px, pz){
    let x = px, z = pz;
    for (const c of colliders){
      if (x > c.minX && x < c.maxX && z > c.minZ && z < c.maxZ){
        const overlapLeft = x - c.minX;
        const overlapRight = c.maxX - x;
        const overlapTop = z - c.minZ;
        const overlapBottom = c.maxZ - z;
        const minOverlap = Math.min(overlapLeft, overlapRight, overlapTop, overlapBottom);
        if (minOverlap === overlapLeft) x = c.minX;
        else if (minOverlap === overlapRight) x = c.maxX;
        else if (minOverlap === overlapTop) z = c.minZ;
        else z = c.maxZ;
      }
    }
    return { x, z };
  }

  // ---------- INPUT ----------
  function initInput(){
    document.addEventListener('keydown', e => {
      keys[e.code] = true;
      if (e.code === 'KeyF') toggleFlash();
    });
    document.addEventListener('keyup', e => { keys[e.code] = false; });

    document.addEventListener('mousemove', e => {
      if (document.pointerLockElement !== renderer.domElement) return;
      const sens = 0.0022;
      camera.rotation.y -= e.movementX * sens;
      camera.rotation.x -= e.movementY * sens;
      camera.rotation.x = Math.max(-Math.PI/2+0.05, Math.min(Math.PI/2-0.05, camera.rotation.x));
    });
  }

  function toggleFlash(){
    if (!gameActive) return;
    if (!flashOn && battery <= 0) return;
    flashOn = !flashOn;
    flashSpot.visible = flashOn;
  }

  // ---------- AUDIO ----------
  function initAudio(){
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    const drone = audioCtx.createOscillator();
    drone.type = 'sine'; drone.frequency.value = 52;
    droneGain = audioCtx.createGain(); droneGain.gain.value = 0.05;
    drone.connect(droneGain); droneGain.connect(audioCtx.destination);
    drone.start();

    const drone2 = audioCtx.createOscillator();
    drone2.type = 'sine'; drone2.frequency.value = 57;
    const g2 = audioCtx.createGain(); g2.gain.value = 0.035;
    drone2.connect(g2); g2.connect(audioCtx.destination);
    drone2.start();

    tensionOsc = audioCtx.createOscillator();
    tensionOsc.type = 'sine'; tensionOsc.frequency.value = 110;
    tensionGain = audioCtx.createGain(); tensionGain.gain.value = 0;
    tensionOsc.connect(tensionGain); tensionGain.connect(audioCtx.destination);
    tensionOsc.start();
  }

  // ---------- GAME LOOP ----------
  function update(dt){
    // movement
    const moveF = (keys['KeyW']?1:0) - (keys['KeyS']?1:0);
    const moveR = (keys['KeyD']?1:0) - (keys['KeyA']?1:0);
    player.running = !!keys['ShiftLeft'] || !!keys['ShiftRight'];
    const speed = BASE_SPEED * (player.running ? RUN_MULT : 1);

    if (moveF !== 0 || moveR !== 0){
      const yaw = camera.rotation.y;
      const fx = -Math.sin(yaw), fz = -Math.cos(yaw);
      const rx = Math.cos(yaw), rz = -Math.sin(yaw);
      let dx = fx*moveF + rx*moveR;
      let dz = fz*moveF + rz*moveR;
      const len = Math.hypot(dx,dz) || 1;
      dx/=len; dz/=len;
      let nx = player.x + dx*speed*dt;
      let nz = player.z + dz*speed*dt;
      const resolved = resolvePosition(nx, nz);
      player.x = resolved.x; player.z = resolved.z;
      velocityBobTime += dt * (player.running ? 14 : 9);
    }
    player.x = Math.max(0.2, Math.min(N*CELL-0.2, player.x));
    player.z = Math.max(0.2, Math.min(N*CELL-0.2, player.z));

    const bob = Math.sin(velocityBobTime) * ((moveF||moveR) ? 0.045 : 0);
    camera.position.set(player.x, PLAYER_H + bob, player.z);

    // battery
    if (flashOn){
      battery = Math.max(0, battery - 7.5*dt);
      if (battery <= 0){ flashOn = false; flashSpot.visible = false; }
      flashSpot.intensity = 2.1 + Math.random()*0.6;
    } else {
      battery = Math.min(100, battery + 1.1*dt);
    }
    document.getElementById('battery-bar').style.width = battery + '%';

    // monster pathing
    monster.repathTimer -= dt;
    if (monster.repathTimer <= 0){
      monster.repathTimer = 0.45;
      const mi = Math.max(0,Math.min(N-1, Math.floor(monster.x/CELL)));
      const mj = Math.max(0,Math.min(N-1, Math.floor(monster.z/CELL)));
      const pi = Math.max(0,Math.min(N-1, Math.floor(player.x/CELL)));
      const pj = Math.max(0,Math.min(N-1, Math.floor(player.z/CELL)));
      const path = bfsPath(mi,mj,pi,pj);
      monster.path = path.map(c => ({ x: c[0]*CELL+CELL/2, z: c[1]*CELL+CELL/2 }));
    }
    if (monster.path.length){
      const wp = monster.path[0];
      const dx = wp.x - monster.x, dz = wp.z - monster.z;
      const d = Math.hypot(dx,dz);
      if (d < 0.15){ monster.path.shift(); }
      else {
        monster.x += (dx/d)*MONSTER_SPEED*dt;
        monster.z += (dz/d)*MONSTER_SPEED*dt;
      }
    }
    monster.mesh.position.set(monster.x, 0, monster.z);
    monster.mesh.lookAt(player.x, 0, player.z);

    // tension / vignette / audio
    const distToMonster = Math.hypot(player.x-monster.x, player.z-monster.z);
    const tension = Math.max(0, 1 - distToMonster/11);
    const vg = document.getElementById('vignette');
    vg.style.background = `radial-gradient(ellipse at center, rgba(140,0,0,${(tension*0.35).toFixed(2)}) 0%, rgba(0,0,0,0) 65%)`;
    if (audioCtx){
      tensionGain.gain.value = tension*0.09;
      droneGain.gain.value = 0.05 + tension*0.05;
    }

    if (distToMonster < CATCH_DIST){ endGame(false); return; }

    const distToExit = Math.hypot(player.x-exitPos.x, player.z-exitPos.z);
    exitLight.intensity = 1.1 + Math.sin(performance.now()*0.004)*0.3;
    if (distToExit < 1.4){ endGame(true); return; }
  }

  function animate(){
    if (!gameActive) return;
    requestAnimationFrame(animate);
    const dt = Math.min(0.05, clock.getDelta());
    update(dt);
    renderer.render(scene, camera);
  }

  // ---------- GAME STATE ----------
  function startGame(){
    document.getElementById('blocker').style.display = 'none';
    document.getElementById('hud').style.display = 'block';
    renderer.domElement.requestPointerLock();
    gameActive = true;
    clock = new THREE.Clock();
    if (!audioCtx) initAudio();
    animate();
  }

  function endGame(won){
    gameActive = false;
    document.exitPointerLock();
    const overlay = document.getElementById('overlay-msg');
    const title = document.getElementById('overlay-title');
    const sub = document.getElementById('overlay-sub');
    overlay.className = won ? 'win' : 'dead';
    overlay.style.display = 'flex';
    title.textContent = won ? 'YOU ESCAPED' : 'YOU DIED';
    sub.textContent = won ? 'the door closes behind you' : 'it was faster than you';
    if (audioCtx){ droneGain.gain.value = 0; tensionGain.gain.value = 0; }
  }

  document.getElementById('blocker').addEventListener('click', startGame);
  document.getElementById('again-btn').addEventListener('click', () => window.location.reload());

  // ---------- INIT ----------
  generateMaze();
  buildScene();
  initInput();

})();
</script>
</body>
</html>
