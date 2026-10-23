
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>GuitarBro 4.0</title>

<style>
*{box-sizing:border-box}
body{
  margin:0;background:#101710;color:white;
  font-family:Arial,sans-serif;padding:16px;
}
main{max-width:600px;margin:auto}
h1,h2{color:#8df59b}
section{
  background:#203023;padding:18px;
  border-radius:16px;margin:16px 0;
}
button{
  padding:12px;border:0;border-radius:10px;
  background:#8df59b;color:#102012;
  font-weight:bold;margin:4px;cursor:pointer;
}
button:disabled{opacity:.5}
.wide{width:100%;margin:6px 0}
.small{color:#c5d2c5;font-size:14px}
.center{text-align:center}
.strings{display:grid;grid-template-columns:repeat(3,1fr);gap:6px}
.strings button{margin:0}
.selected{background:#f4c969}
#estado{font-size:24px;font-weight:bold}
#nota{font-size:38px;margin:8px}
.meter{
  height:12px;background:#475548;
  border-radius:10px;position:relative;margin:24px 0;
}
#aguja{
  position:absolute;left:50%;top:-6px;
  width:4px;height:24px;background:white;
  border-radius:4px;transform:translateX(-50%);
}
#medio{
  position:absolute;left:50%;height:100%;
  width:3px;background:#8df59b;
}
</style>
</head>

<body>
<main>
<h1>🎸 GuitarBro</h1>
<p>Aprende guitarra jugando.</p>

<section>
<h2>📚 Lección rápida</h2>
<p>Las cuerdas al aire, de grave a aguda:</p>
<h3>Mi · La · Re · Sol · Si · Mi</h3>
<p>E · A · D · G · B · E</p>
<p class="small">Pisa los trastes detrás de la barra metálica y practica despacio.</p>
</section>

<section>
<h2>🎵 Afinador real</h2>
<p class="small">Selecciona una cuerda, activa el micrófono y tócala sola.</p>

<div class="strings" id="botonesCuerda"></div>

<p id="objetivo" class="center">Objetivo: Mi grave · 82.41 Hz</p>

<button id="iniciar" class="wide" onclick="iniciar()">🎙️ Activar micrófono</button>
<button id="parar" class="wide" onclick="parar()" style="display:none;background:#f4c969">Detener</button>

<div class="center">
  <p id="estado">Esperando...</p>
  <div id="nota">--</div>
  <p id="frecuencia">Frecuencia: -- Hz</p>
  <p id="desviacion">Desviación: --</p>
</div>

<div class="meter">
  <div id="medio"></div>
  <div id="aguja"></div>
</div>
<div class="small" style="display:flex;justify-content:space-between">
<span>♭ Grave</span><span>Afinada</span><span>Aguda ♯</span>
</div>
<p id="consejo">Activa el micrófono para comenzar.</p>
<p class="small">Si la nota está baja, aprieta muy poco la clavija. Si está alta, aflójala muy poco. Detente si la cuerda está demasiado tensa.</p>
</section>

<section>
<h2>🎮 Quiz de guitarra</h2>
<p id="contador"></p>
<h3 id="pregunta"></h3>
<div id="respuestas"></div>
<p id="resultado"></p>
<button onclick="siguiente()">Siguiente ➜</button>
</section>

<section>
<h2>🏆 Mi progreso</h2>
<p>Puntos: <b id="puntos">0</b></p>
<p id="nivel">Nivel: Principiante</p>
</section>
</main>

<script>
// AFINADOR
const cuerdas=[
 {nombre:"Mi grave",nota:"E2",hz:82.41},
 {nombre:"La",nota:"A2",hz:110},
 {nombre:"Re",nota:"D3",hz:146.83},
 {nombre:"Sol",nota:"G3",hz:196},
 {nombre:"Si",nota:"B3",hz:246.94},
 {nombre:"Mi agudo",nota:"E4",hz:329.63}
];

let seleccion=0,contexto=null,stream=null,analizador=null;
let buffer=null,frame=null,activo=false,ultima=0;

const $=id=>document.getElementById(id);

cuerdas.forEach((c,i)=>{
 const b=document.createElement("button");
 b.textContent=(6-i)+"ª "+c.nombre;
 if(i===0)b.classList.add("selected");
 b.onclick=()=>{
  seleccion=i;
  document.querySelectorAll(".strings button").forEach(x=>x.classList.remove("selected"));
  b.classList.add("selected");
  $("objetivo").textContent="Objetivo: "+c.nombre+" · "+c.nota+" · "+c.hz.toFixed(2)+" Hz";
  $("estado").textContent="Toca la cuerda";
 };
 $("botonesCuerda").appendChild(b);
});

async function iniciar(){
 if(activo)return;
 try{
  if(!navigator.mediaDevices?.getUserMedia){
   throw Error("HTTPS");
  }
  stream=await navigator.mediaDevices.getUserMedia({
   audio:{
    echoCancellation:false,
    noiseSuppression:false,
    autoGainControl:false
   }
  });
  contexto=new(window.AudioContext||window.webkitAudioContext)();
  await contexto.resume();
  analizador=contexto.createAnalyser();
  analizador.fftSize=8192;
  contexto.createMediaStreamSource(stream).connect(analizador);
  buffer=new Float32Array(analizador.fftSize);
  activo=true;
  $("iniciar").style.display="none";
  $("parar").style.display="block";
  $("estado").textContent="Escuchando 🎙️";
  $("consejo").textContent="Toca una cuerda y deja que resuene.";
  escuchar();
 }catch(e){
  $("estado").textContent="No se pudo activar";
  $("consejo").textContent=
   "Permite el micrófono. Si no aparece el permiso, prueba en una página HTTPS fuera del editor.";
  parar();
 }
}

function parar(){
 activo=false;
 if(frame!==null)cancelAnimationFrame(frame);
 frame=null;
 if(stream)stream.getTracks().forEach(t=>t.stop());
 if(contexto&&contexto.state!=="closed")contexto.close();
 stream=null;contexto=null;analizador=null;buffer=null;
 $("iniciar").style.display="block";
 $("parar").style.display="none";
 $("estado").textContent="Micrófono apagado";
}

function frecuenciaOnda(a){
 let media=0;
 for(let x of a)media+=x;
 media/=a.length;

 let rms=0;
 for(let i=0;i<a.length;i++){
  a[i]-=media;
  rms+=a[i]*a[i];
 }
 if(Math.sqrt(rms/a.length)<0.009)return null;

 const sr=contexto.sampleRate;
 const min=Math.floor(sr/400);
 const max=Math.min(Math.floor(sr/65),a.length/2);
 let mejorLag=-1,mejor=-Infinity;

 // Autocorrelación de la señal de audio.
 for(let lag=min;lag<=max;lag++){
  let suma=0;
  for(let i=0;i<a.length-lag;i++)suma+=a[i]*a[i+lag];
  if(suma>mejor){mejor=suma;mejorLag=lag;}
 }
 if(mejorLag<1)return null;

 const corr=lag=>{
  let s=0;
  for(let i=0;i<a.length-lag;i++)s+=a[i]*a[i+lag];
  return s;
 };
 let y1=corr(mejorLag-1),y2=corr(mejorLag),y3=corr(mejorLag+1);
 let den=y1-2*y2+y3;
 let ajuste=den===0?0:0.5*(y1-y3)/den;
 ajuste=Math.max(-1,Math.min(1,ajuste));
 let hz=sr/(mejorLag+ajuste);
 return hz>=65&&hz<=400?hz:null;
}

function escuchar(){
 if(!activo)return;
 analizador.getFloatTimeDomainData(buffer);
 let hz=frecuenciaOnda(buffer);
 let ahora=performance.now();

 if(hz&&ahora-ultima>100){
  ultima=ahora;
  mostrar(hz);
 }
 frame=requestAnimationFrame(escuchar);
}

function mostrar(hz){
 const c=cuerdas[seleccion];
 const cents=1200*Math.log2(hz/c.hz);
 const n=Math.round(69+12*Math.log2(hz/440));
 const nombres=["Do","Do♯","Re","Re♯","Mi","Fa","Fa♯","Sol","Sol♯","La","La♯","Si"];
 const nombre=nombres[(n%12+12)%12]+(Math.floor(n/12)-1);

 $("nota").textContent=nombre;
 $("frecuencia").textContent="Frecuencia: "+hz.toFixed(2)+" Hz";
 $("desviacion").textContent="Desviación: "+Math.round(cents)+" cents";
 $("aguja").style.left=(50+Math.max(-45,Math.min(45,cents)))+"%";

 if(Math.abs(cents)<=5){
  $("estado").textContent="✅ AFINADA";
  $("estado").style.color="#8df59b";
  $("consejo").textContent="¡Perfecto! No muevas más la clavija.";
 }else if(cents<0){
  $("estado").textContent="♭ MUY GRAVE";
  $("estado").style.color="#f4c969";
  $("consejo").textContent="La nota está baja. Aprieta muy poquito y vuelve a tocar.";
 }else{
  $("estado").textContent="♯ MUY AGUDA";
  $("estado").style.color="#f4c969";
  $("consejo").textContent="La nota está alta. Afloja muy poquito y vuelve a tocar.";
 }
}

// QUIZ
const preguntas=[
 ["¿Cuántas cuerdas tiene la guitarra estándar?",["4","6","8"],1],
 ["¿Cuál es la cuerda más grave?",["Mi grave","Si","Mi agudo"],0],
 ["¿Qué significa Am?",["La menor","Do mayor","Sol mayor"],0],
 ["¿Qué acorde es C?",["Re","Do mayor","Mi menor"],1],
 ["¿Para qué sirve un afinador?",["Afinar","Pintar","Cambiar ritmo"],0],
 ["¿Cuántos semitonos hay en una octava?",["8","10","12"],2],
 ["¿Qué es tocar al aire?",["Sin pisar trastes","Muy fuerte","Rápido"],0],
 ["¿Qué ayuda a marcar el ritmo?",["Metrónomo","Afinador","Capotraste"],0],
 ["¿Qué acorde es Em?",["Mi menor","La mayor","Do menor"],0],
 ["¿Cómo conviene practicar?",["Despacio y con atención","Lo más rápido posible","Sin escuchar"],0]
];
let q=0,pts=0,respondida=false;

function cargar(){
 respondida=false;
 $("resultado").textContent="";
 if(q>=preguntas.length){
  $("pregunta").textContent="¡Terminaste! 🎉";
  $("contador").textContent="Quiz completado";
  $("respuestas").innerHTML="";
  return;
 }
 $("contador").textContent="Pregunta "+(q+1)+" de "+preguntas.length;
 $("pregunta").textContent=preguntas[q][0];
 $("respuestas").innerHTML="";
 preguntas[q][1].forEach((op,i)=>{
  let b=document.createElement("button");
  b.className="wide";
  b.textContent=op;
  b.onclick=()=>contestar(i);
  $("respuestas").appendChild(b);
 });
}
function contestar(i){
 if(respondida||q>=preguntas.length)return;
 respondida=true;
 if(i===preguntas[q][2]){
  pts+=10;$("resultado").textContent="¡Correcto! +10 puntos 🎉";
 }else{
  $("resultado").textContent="Correcto: "+preguntas[q][1][preguntas[q][2]];
 }
 $("puntos").textContent=pts;
 $("nivel").textContent="Nivel: "+(pts>=70?"Avanzado":pts>=30?"Aprendiz":"Principiante");
}
function siguiente(){if(q<preguntas.length){q++;cargar();}}
cargar();
</script>
</body>
</html>

