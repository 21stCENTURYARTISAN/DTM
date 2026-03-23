<!DOCTYPE html>  
<html>  
<head>  
<meta name="viewport" content="width=device-width, initial-scale=1.0">  
<title>MiniMoog Web v3</title>  
  
<style>  
body {  
  margin:0;  
  background:#111;  
  color:#eee;  
  font-family:sans-serif;  
  overflow:hidden;  
}  
  
.container {  
  display:flex;  
  flex-direction:column;  
  height:100vh;  
}  
  
.controls {  
  display:flex;  
  justify-content:space-around;  
  padding:8px;  
  font-size:11px;  
}  
  
.controls div {  
  text-align:center;  
}  
  
input {  
  width:70px;  
}  
  
.keyboard {  
  display:flex;  
  flex:1;  
}  
  
.key {  
  flex:1;  
  border:1px solid black;  
  background:white;  
}  
  
.key:active {  
  background:#ccc;  
}  
</style>  
</head>  
  
<body>  
  
<div class="container">  
  
<div class="controls">  
  <div>Cutoff<br><input id="cutoff" type="range" min="100" max="8000" value="1200"></div>  
  <div>Res<br><input id="res" type="range" min="0" max="4" step="0.01" value="1.2"></div>  
  <div>Env<br><input id="env" type="range" min="0" max="3000" value="1200"></div>  
  <div>Attack<br><input id="attack" type="range" min="0.001" max="0.5" step="0.001" value="0.01"></div>  
  <div>Decay<br><input id="decay" type="range" min="0.05" max="2" step="0.01" value="0.6"></div>  
  <div>Glide<br><input id="glide" type="range" min="0" max="0.2" step="0.001" value="0.05"></div>  
</div>  
  
<div class="keyboard" id="keyboard"></div>  
  
</div>  
  
<script>  
const ctx = new (window.AudioContext || window.webkitAudioContext)();  
  
// --- NONLINEAR LADDER FILTER ---  
function createLadder() {  
  let input = ctx.createGain();  
  let output = ctx.createGain();  
  
  let stages = [];  
  for (let i = 0; i < 4; i++) {  
    let shaper = ctx.createWaveShaper();  
    let curve = new Float32Array(1024);  
    for (let j = 0; j < 1024; j++) {  
      let x = (j / 512) - 1;  
      curve[j] = Math.tanh(x); // saturation  
    }  
    shaper.curve = curve;  
  
    let filter = ctx.createBiquadFilter();  
    filter.type = "lowpass";  
  
    input.connect(shaper);  
    shaper.connect(filter);  
  
    input = filter;  
    stages.push(filter);  
  }  
  
  input.connect(output);  
  
  let feedback = ctx.createGain();  
  feedback.gain.value = 1;  
  
  output.connect(feedback);  
  feedback.connect(stages[0]);  
  
  return {  
    input: stages[0],  
    output,  
    setCutoff(v) {  
      stages.forEach(s => s.frequency.value = v);  
    },  
    setRes(v) {  
      feedback.gain.value = v;  
    }  
  };  
}  
  
// --- OSCILLATORS ---  
let osc1 = ctx.createOscillator();  
let osc2 = ctx.createOscillator();  
let osc3 = ctx.createOscillator();  
  
osc1.type = "sawtooth";  
osc2.type = "sawtooth";  
osc3.type = "square";  
  
let mix = ctx.createGain();  
mix.gain.value = 1.2; // overdrive  
  
osc1.connect(mix);  
osc2.connect(mix);  
osc3.connect(mix);  
  
let ladder = createLadder();  
let amp = ctx.createGain();  
  
amp.gain.value = 0;  
  
mix.connect(ladder.input);  
ladder.output.connect(amp);  
amp.connect(ctx.destination);  
  
osc1.start();  
osc2.start();  
osc3.start();  
  
// --- DRIFT ---  
setInterval(() => {  
  osc1.detune.value = (Math.random() - 0.5) * 5;  
  osc2.detune.value = (Math.random() - 0.5) * 7;  
}, 200);  
  
// --- STATE ---  
let currentFreq = 0;  
  
// --- PLAY ---  
function noteOn(freq) {  
  let now = ctx.currentTime;  
  
  let glide = parseFloat(glideSlider.value);  
  
  osc1.frequency.setTargetAtTime(freq, now, glide);  
  osc2.frequency.setTargetAtTime(freq * 0.995, now, glide);  
  osc3.frequency.setTargetAtTime(freq * 2, now, glide);  
  
  let cutoff = parseFloat(cutoffSlider.value);  
  let env = parseFloat(envSlider.value);  
  
  ladder.setRes(parseFloat(resSlider.value));  
  ladder.setCutoff(cutoff);  
  
  // AMP ENV  
  amp.gain.cancelScheduledValues(now);  
  amp.gain.setValueAtTime(0, now);  
  amp.gain.linearRampToValueAtTime(1, now + parseFloat(attackSlider.value));  
  amp.gain.exponentialRampToValueAtTime(0.001, now + parseFloat(decaySlider.value));  
  
  // FILTER ENV  
  ladder.setCutoff(cutoff);  
  ladder.setCutoff(cutoff + env);  
  setTimeout(() => ladder.setCutoff(cutoff), decaySlider.value * 1000);  
}  
  
// --- UI refs ---  
const cutoffSlider = document.getElementById("cutoff");  
const resSlider = document.getElementById("res");  
const envSlider = document.getElementById("env");  
const attackSlider = document.getElementById("attack");  
const decaySlider = document.getElementById("decay");  
const glideSlider = document.getElementById("glide");  
  
// --- 12 notes ---  
const notes = [  
  261.63,277.18,293.66,311.13,  
  329.63,349.23,369.99,392.00,  
  415.30,440.00,466.16,493.88  
];  
  
const keyboard = document.getElementById("keyboard");  
  
notes.forEach(freq => {  
  let key = document.createElement("div");  
  key.className = "key";  
  
  key.ontouchstart = () => {  
    ctx.resume();  
    noteOn(freq);  
  };  
  
  key.onmousedown = () => {  
    ctx.resume();  
    noteOn(freq);  
  };  
  
  keyboard.appendChild(key);  
});  
</script>  
  
</body>  
</html>  
