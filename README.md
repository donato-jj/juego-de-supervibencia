# juego-de-supervibencia
hola
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Cyberpunk 2D Vertical Hacker Game</title>
<style>
/* ============================= */
/* ESTILOS GENERALES Y RESET */
/* ============================= */
* { margin:0; padding:0; box-sizing:border-box; font-family: 'Courier New', monospace; }
body, html { width:100%; height:100%; overflow:hidden; background: #000; color:#0ff; }

/* ============================= */
/* FONDO CYBERPUNK DINÁMICO */
/* ============================= */
#game-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: linear-gradient(to bottom, #001, #003);
}

.background-layer {
  position:absolute;
  width:100%;
  height:200%;
  top:-100%;
  background-repeat:repeat-y;
  background-size:cover;
  animation: scrollBG 20s linear infinite;
}

.background-layer.layer1 { background: radial-gradient(circle at 50% 0%, #0ff2, #0000); animation-duration:25s; }
.background-layer.layer2 { background: radial-gradient(circle at 70% 0%, #f0f2, #0000); animation-duration:20s; }
.background-layer.layer3 { background: radial-gradient(circle at 30% 0%, #ff02, #0000); animation-duration:15s; }

@keyframes scrollBG { 0% { top:-100%; } 100% { top:0; } }

/* ============================= */
/* PERSONAJE PRINCIPAL */
/* ============================= */
#player {
  position:absolute;
  bottom:50px;
  left:50%;
  transform: translateX(-50%);
  width:40px;
  height:60px;
  background: linear-gradient(to right, #0ff, #0f0);
  border-radius:8px;
  z-index:10;
}

/* DISPAROS */
.bullet {
  position:absolute;
  width:6px;
  height:12px;
  background:#0ff;
  border-radius:3px;
}

/* ENEMIGOS */
.enemy {
  position:absolute;
  width:40px;
  height:40px;
  background: linear-gradient(to right, #f00, #ff0);
  border-radius:50%;
}

/* ============================= */
/* HUD */
/* ============================= */
#hud {
  position:absolute;
  top:10px;
  left:10px;
  color:#0ff;
  z-index:20;
}

#hud div { margin-bottom:5px; }

/* ============================= */
/* MENSAJE IA CON VOZ */
/* ============================= */
#ai-message {
  position:absolute;
  bottom:120px;
  left:50%;
  transform:translateX(-50%);
  padding:10px 20px;
  background: rgba(0,0,0,0.7);
  border:1px solid #0ff;
  border-radius:5px;
  font-size:14px;
  display:none;
  z-index:20;
}
</style>
</head>
<body>
<div id="game-container">
  <div class="background-layer layer1"></div>
  <div class="background-layer layer2"></div>
  <div class="background-layer layer3"></div>
  <div id="player"></div>
  <div id="hud">
    <div id="score">Puntuación: 0</div>
    <div id="level">Nivel: 1</div>
    <div id="health">Salud: 100</div>
  </div>
  <div id="ai-message"></div>
</div>

<script>
/* ============================= */
/* VARIABLES DEL JUEGO */
/* ============================= */
const game = document.getElementById('game-container');
const player = document.getElementById('player');
const aiMessage = document.getElementById('ai-message');
let score = 0;
let level = 1;
let health = 100;
let bullets = [];
let enemies = [];
let keys = {};
let playerSpeed = 5;
let bulletSpeed = 8;
let enemySpeed = 2;
let lastEnemySpawn = 0;
let enemyInterval = 2000;
let aiResponses = [
  "¡Cuidado! Se acerca un enemigo.",
  "Nivel completado. ¡Sigue así!",
  "Tu salud está baja, busca un bonus.",
  "Dispara rápido, no dejes que te rodeen.",
  "Un nuevo obstáculo ha aparecido.",
  "¡Bien hecho! Avanzas en la ciudad digital.",
  "Mantente alerta, los drones son rápidos.",
  "La energía del sistema está aumentando.",
  "Coopera con tu compañero para sobrevivir.",
  "Bonus detectado, acércate para recogerlo."
];

/* ============================= */
/* FUNCIONES DE IA CON VOZ */
/* ============================= */
function aiSpeak(message){
  aiMessage.innerText = message;
  aiMessage.style.display = 'block';
  let utter = new SpeechSynthesisUtterance(message);
  utter.lang = 'es-ES';
  utter.pitch = 1.2;
  utter.rate = 1;
  utter.volume = 1;
  speechSynthesis.speak(utter);
  setTimeout(()=>{aiMessage.style.display='none';},3000);
}

function randomAIMessage(){
  const msg = aiResponses[Math.floor(Math.random()*aiResponses.length)];
  aiSpeak(msg);
}

/* ============================= */
/* CONTROL DEL JUGADOR */
/* ============================= */
document.addEventListener('keydown', e=>{ keys[e.key.toLowerCase()] = true; });
document.addEventListener('keyup', e=>{ keys[e.key.toLowerCase()] = false; });

/* ============================= */
/* DISPAROS DEL JUGADOR */
/* ============================= */
function shoot(){
  const bullet = document.createElement('div');
  bullet.classList.add('bullet');
  bullet.style.left = (player.offsetLeft + player.offsetWidth/2 - 3)+'px';
  bullet.style.top = (player.offsetTop - 12)+'px';
  game.appendChild(bullet);
  bullets.push(bullet);
}

/* ============================= */
/* CREAR ENEMIGOS */
/* ============================= */
function spawnEnemy(){
  const enemy = document.createElement('div');
  enemy.classList.add('enemy');
  enemy.style.left = Math.random()*(game.offsetWidth-40)+'px';
  enemy.style.top = '-50px';
  game.appendChild(enemy);
  enemies.push(enemy);
  randomAIMessage();
}

/* ============================= */
/* COLISIONES */
/* ============================= */
function isColliding(a,b){
  const rectA = a.getBoundingClientRect();
  const rectB = b.getBoundingClientRect();
  return !(
    rectA.top > rectB.bottom ||
    rectA.bottom < rectB.top ||
    rectA.left > rectB.right ||
    rectA.right < rectB.left
  );
}

/* ============================= */
/* ACTUALIZAR JUEGO */
/* ============================= */
function update(){
  // Movimiento jugador
  if(keys['arrowleft'] && player.offsetLeft>0) player.style.left = (player.offsetLeft - playerSpeed)+'px';
  if(keys['arrowright'] && player.offsetLeft<game.offsetWidth-player.offsetWidth) player.style.left = (player.offsetLeft + playerSpeed)+'px';
  if(keys['arrowup'] && player.offsetTop>0) player.style.top = (player.offsetTop - playerSpeed)+'px';
  if(keys['arrowdown'] && player.offsetTop<game.offsetHeight-player.offsetHeight) player.style.top = (player.offsetTop + playerSpeed)+'px';
  if(keys[' ']){ shoot(); keys[' ']=false; }

  // Mover balas
  bullets.forEach((b,i)=>{
    b.style.top = (b.offsetTop - bulletSpeed)+'px';
    if(b.offsetTop < -20){ game.removeChild(b); bullets.splice(i,1);}
  });

  // Mover enemigos
  enemies.forEach((e,i)=>{
    e.style.top = (e.offsetTop + enemySpeed)+'px';
    // Colisión con jugador
    if(isColliding(e,player)){
      health -=10;
      document.getElementById('health').innerText='Salud: '+health;
      game.removeChild(e);
      enemies.splice(i,1);
      randomAIMessage();
      if(health<=0) { alert("¡GAME OVER!"); location.reload(); }
    }
    // Colisión con balas
    bullets.forEach((b,j)=>{
      if(isColliding(b,e)){
        score +=10;
        document.getElementById('score').innerText='Puntuación: '+score;
        game.removeChild(e); enemies.splice(i,1);
        game.removeChild(b); bullets.splice(j,1);
      }
    });
    // Salir de pantalla
    if(e.offsetTop > game.offsetHeight+50){
      game.removeChild(e);
      enemies.splice(i,1);
    }
  });

  // Subir nivel cada 500 puntos
  if(score>=level*500){
    level++;
    enemySpeed +=0.5;
    enemyInterval = Math.max(500,enemyInterval-200);
    document.getElementById('level').innerText='Nivel: '+level;
    aiSpeak('¡Nivel '+level+' alcanzado!');
  }

  // Spawn enemigos
  if(Date.now() - lastEnemySpawn > enemyInterval){
    spawnEnemy();
    lastEnemySpawn = Date.now();
  }

  requestAnimationFrame(update);
}

/* ============================= */
/* INICIO DEL JUEGO */
/* ============================= */
aiSpeak("Había una vez un chico hacker que quiso salvar a la humanidad ante el total desastre que se avecinaba.");
update();
</script>
</body>
</html>
