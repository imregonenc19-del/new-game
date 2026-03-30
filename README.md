<!DOCTYPE html>
<html lang="nl">
<head>
<meta charset="UTF-8">
<title>Zombie Shooter FPS - Waves & Mini-Map</title>
<style>
body { margin:0; overflow:hidden; }
#ui { position:absolute; top:10px; left:10px; color:white; font-family:Arial; font-size:20px; }
#waveInfo { position:absolute; top:40px; left:10px; color:yellow; font-family:Arial; font-size:18px; }
#menu {
  position:absolute; top:0; left:0; width:100%; height:100%;
  display:flex; align-items:center; justify-content:center;
  background:black; color:white; font-size:30px; cursor:pointer;
}
#crosshair { position:absolute; top:50%; left:50%; width:10px; height:10px; margin-left:-5px; margin-top:-5px; border:2px solid white; border-radius:50%; }
#restart {
  display:none; position:absolute; top:50%; left:50%;
  transform:translate(-50%,-50%);
  background:black; color:white; padding:20px; font-size:25px; cursor:pointer;
}
#minimap {
  position:absolute; bottom:10px; right:10px; width:200px; height:200px;
  background:rgba(0,0,0,0.5); border:1px solid white;
}
</style>
</head>
<body>
<div id="menu">KLIK OM TE STARTEN</div>
<div id="ui"></div>
<div id="waveInfo"></div>
<div id="crosshair"></div>
<div id="restart">GAME OVER<br>KLIK OM TE HERSTARTEN</div>
<canvas id="minimap"></canvas>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r134/three.min.js"></script>
<script>
// --- Scene Setup ---
let scene=new THREE.Scene(); scene.fog=new THREE.FogExp2(0x000000,0.02);
let camera=new THREE.PerspectiveCamera(75,window.innerWidth/window.innerHeight,0.1,1000);
let renderer=new THREE.WebGLRenderer({antialias:true}); renderer.setSize(window.innerWidth,window.innerHeight);
document.body.appendChild(renderer.domElement);

// Lights
scene.add(new THREE.AmbientLight(0x404040));
let dirLight=new THREE.DirectionalLight(0xffffff,1); dirLight.position.set(5,10,5); scene.add(dirLight);

// Floor & walls
let texture=new THREE.TextureLoader().load('https://threejsfundamentals.org/threejs/resources/images/checker.png');
texture.wrapS=texture.wrapT=THREE.RepeatWrapping; texture.repeat.set(20,20);
let floor=new THREE.Mesh(new THREE.PlaneGeometry(200,200),new THREE.MeshStandardMaterial({map:texture})); floor.rotation.x=-Math.PI/2; scene.add(floor);

let walls=[]; for(let i=0;i<30;i++){
  let wall=new THREE.Mesh(new THREE.BoxGeometry(5,3,1),new THREE.MeshStandardMaterial({color:0x777777}));
  wall.position.set((Math.random()-0.5)*100,1.5,(Math.random()-0.5)*100); scene.add(wall); walls.push(wall);
}

// --- Player ---
let player=new THREE.Object3D(); player.position.y=1.8; scene.add(player); player.add(camera);

// Controls
let keys={}, bullets=[], zombies=[], powerups=[];
let hp=100, score=0, velocityY=0, canJump=true, fireRate=200, speedMultiplier=1;

// Gun
let gun=new THREE.Mesh(new THREE.BoxGeometry(0.3,0.2,1),new THREE.MeshStandardMaterial({color:0x111111}));
gun.position.set(0.5,-0.5,-1); camera.add(gun);

// Menu & restart
let menu=document.getElementById("menu"); menu.onclick=()=>{ menu.style.display="none"; document.body.requestPointerLock(); ambientMusic.play(); };
let restartDiv=document.getElementById("restart"); restartDiv.onclick=resetGame;

// Mini-map
let minimap=document.getElementById("minimap"); let minimapCtx=minimap.getContext("2d");

// Mouse look
document.addEventListener("mousemove",e=>{
  if(document.pointerLockElement===document.body){
    player.rotation.y-=e.movementX*0.002; camera.rotation.x-=e.movementY*0.002;
    camera.rotation.x=Math.max(-Math.PI/2,Math.min(Math.PI/2,camera.rotation.x));
  }
});

// Keyboard
window.addEventListener("keydown",e=>{ keys[e.key.toLowerCase()]=true; if(e.code==="Space"&&canJump){ velocityY=0.2; canJump=false; }});
window.addEventListener("keyup",e=>keys[e.key.toLowerCase()]=false);

// Sounds
let shootSound=new Audio("https://actions.google.com/sounds/v1/impacts/wood_plank_flicks.ogg");
let zombieSounds=["https://actions.google.com/sounds/v1/ambiences/zombie_groan.ogg","https://actions.google.com/sounds/v1/ambiences/zombie_moan.ogg"].map(url=>{ let a=new Audio(url); a.volume=0.3; return a;});
let ambientMusic=new Audio("https://actions.google.com/sounds/v1/ambiences/creepy_background.ogg"); ambientMusic.loop=true; ambientMusic.volume=0.1;

// --- Shooting ---
let lastShot=0;
window.addEventListener("click",()=>{
  if(document.pointerLockElement!==document.body) return;
  let now=performance.now(); if(now-lastShot<fireRate) return; lastShot=now;
  shootSound.currentTime=0; shootSound.play();
  let bullet=new THREE.Mesh(new THREE.SphereGeometry(0.1),new THREE.MeshBasicMaterial({color:0xffff00}));
  bullet.position.copy(camera.position); let dir=new THREE.Vector3(); camera.getWorldDirection(dir);
  bullet.velocity=dir.clone().multiplyScalar(2); scene.add(bullet); bullets.push(bullet);
  let flash=new THREE.Mesh(new THREE.SphereGeometry(0.2),new THREE.MeshBasicMaterial({color:0xffaa00}));
  flash.position.copy(camera.position.clone().add(dir.clone().multiplyScalar(0.5))); scene.add(flash); setTimeout(()=>scene.remove(flash),50);
  gun.position.z+=0.1; setTimeout(()=>gun.position.z-=0.1,50);
});

// --- Wave System ---
let currentWave=1; let zombiesRemaining=0; let waveCooldown=5*60; // 5 seconds countdown
function startWave(){
  zombiesRemaining=currentWave*5;
  document.getElementById("waveInfo").innerText=`Wave: ${currentWave}`;
  for(let i=0;i<zombiesRemaining;i++) spawnZombie();
}
setInterval(()=>{
  if(zombiesRemaining<=0 && hp>0){ waveCooldown--; if(waveCooldown<=0){ currentWave++; waveCooldown=5*60; startWave(); } }
},1000/60);

// --- Spawn zombie ---
function spawnZombie(){
  let type=Math.random(); let speed=0.03,hpZ=3,color=0x00aa00,scoreVal=1;
  if(type<0.6){ speed=0.04; hpZ=3; color=0x00aa00; scoreVal=1; } 
  else if(type<0.85){ speed=0.08; hpZ=2; color=0xaa0000; scoreVal=2; } 
  else{ speed=0.02; hpZ=6; color=0x333333; scoreVal=5; }
  let z=new THREE.Mesh(new THREE.CylinderGeometry(0.5,0.5,2,6),new THREE.MeshStandardMaterial({color:color}));
  let safe=false; while(!safe){ z.position.set((Math.random()-0.5)*80,1,(Math.random()-0.5)*80); safe=z.position.distanceTo(player.position)>10; walls.forEach(w=>{if(z.position.distanceTo(w.position)<3)safe=false;}); }
  z.hp=hpZ; z.speed=speed; z.scoreVal=scoreVal; scene.add(z); zombies.push(z);
  if(Math.random()<0.5){ let s=zombieSounds[Math.floor(Math.random()*zombieSounds.length)]; s.currentTime=0; s.play(); }
}

// --- Spawn powerups ---
function spawnPowerup(){
  let type=Math.random(); let color=0xffff00; if(type<0.33) color=0xff0000; else if(type<0.66) color=0x00ff00; else color=0x0000ff;
  let p=new THREE.Mesh(new THREE.BoxGeometry(1,1,1),new THREE.MeshStandardMaterial({color:color,emissive:color,emissiveIntensity:0.5}));
  p.position.set((Math.random()-0.5)*80,0.5,(Math.random()-0.5)*80); p.type=type; scene.add(p); powerups.push(p);
}
setInterval(spawnPowerup,8000);

// --- Collision ---
function checkCollision(pos){ for(let w of walls){ if(Math.abs(pos.x-w.position.x)<3 && Math.abs(pos.z-w.position.z)<1.5) return true; } return false; }

// Blood
function spawnBlood(pos){ for(let i=0;i<5;i++){ let p=new THREE.Mesh(new THREE.SphereGeometry(0.1), new THREE.MeshBasicMaterial({color:0xff0000})); p.position.copy(pos); p.velocity=new THREE.Vector3((Math.random()-0.5)*0.1,Math.random()*0.1,(Math.random()-0.5)*0.1); scene.add(p); setTimeout(()=>scene.remove(p),500); bullets.push(p); }}

// Reset
function resetGame(){ bullets.forEach(b=>scene.remove(b)); bullets=[]; zombies.forEach(z=>scene.remove(z)); zombies=[]; powerups.forEach(p=>scene.remove(p)); powerups=[]; player.position.set(0,1.8,0); hp=100; score=0; fireRate=200; speedMultiplier=1; restartDiv.style.display="none"; menu.style.display="none"; document.body.requestPointerLock(); ambientMusic.currentTime=0; ambientMusic.play(); currentWave=1; startWave(); }

// --- Update ---
function update(){
  if(hp<=0) return;

  let speed=keys.shift?0.2:0.1; speed*=speedMultiplier;
  let move=new THREE.Vector3(); if(keys.w) move.z-=speed; if(keys.s) move.z+=speed; if(keys.a) move.x-=speed; if(keys.d) move.x+=speed;
  let angle=player.rotation.y; let dx=move.x*Math.cos(angle)-move.z*Math.sin(angle); let dz=move.x*Math.sin(angle)+move.z*Math.cos(angle);
  let newPos=player.position.clone(); newPos.x+=dx; newPos.z+=dz; if(!checkCollision(newPos)){ player.position.x=newPos.x; player.position.z=newPos.z; }

  velocityY-=0.01; player.position.y+=velocityY; if(player.position.y<=1.8){ player.position.y=1.8; velocityY=0; canJump=true; }

  bullets=bullets.filter(b=>{ if(b.velocity) b.position.add(b.velocity); if(b.position.length()>100){scene.remove(b); return false;} return true; });

  zombies=zombies.filter(z=>{
    let dir=player.position.clone().sub(z.position).normalize(); z.position.add(dir.clone().multiplyScalar(z.speed));
    if(z.position.distanceTo(player.position)<1.5) hp-=0.3;
    bullets.forEach((b,bi)=>{ if(b.velocity && z.position.distanceTo(b.position)<1){ z.hp--; scene.remove(b); bullets.splice(bi,1); } });
    if(z.hp<=0){ spawnBlood(z.position.clone().add(new THREE.Vector3(0,1,0))); scene.remove(z); score+=z.scoreVal; zombiesRemaining--; return false; } return true;
  });

  // Power-ups
  powerups=powerups.filter(p=>{
    if(player.position.distanceTo(p.position)<2){
      if(p.type<0.33){ hp=Math.min(hp+30,100); } 
      else if(p.type<0.66){ fireRate=100; setTimeout(()=>fireRate=200,5000); } 
      else{ speedMultiplier=2; setTimeout(()=>speedMultiplier=1,5000); }
      scene.remove(p); return false;
    } return true;
  });

  // --- Mini-map ---
  minimapCtx.clearRect(0,0,minimap.width,minimap.height);
  minimapCtx.fillStyle="white"; let px=(player.position.x+100)/200*minimap.width; let pz=(player.position.z+100)/200*minimap.height;
  minimapCtx.fillRect(px-3,pz-3,6,6);
  minimapCtx.fillStyle="red"; zombies.forEach(z=>{ let zx=(z.position.x+100)/200*minimap.width; let zz=(z.position.z+100)/200*minimap.height; minimapCtx.fillRect(zx-2,zz-2,4,4); });
}

// Animate
function animate(){
  requestAnimationFrame(animate); update(); renderer.render(scene,camera);
  if(hp>0){ document.getElementById("ui").innerText=`HP:${Math.floor(hp)} Score:${score}`; document.getElementById("waveInfo").innerText=`Wave: ${currentWave}`; } 
  else{ document.getElementById("ui").innerText="GAME OVER"; restartDiv.style.display="block"; ambientMusic.pause(); }
}

// Start
ambientMusic.play(); startWave(); animate();
</script>
</body>
</html>
