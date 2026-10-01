<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>3D Studio</title>
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600&display=swap" rel="stylesheet">
<style>
:root{--bg:#e6ebf2;--panel:#fbfcfe;--ink:#1b2433;--muted:#5d6b82;--line:#c9d2df;--accent:#3556f5;
box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#141a26;--panel:#1d2535;--ink:#e8edf6;--muted:#93a1ba;--line:#33405a;--accent:#7b93ff}}
:root[data-theme="dark"]{--bg:#141a26;--panel:#1d2535;--ink:#e8edf6;--muted:#93a1ba;--line:#33405a;--accent:#7b93ff}
html{scroll-padding-top:env(safe-area-inset-top,0px)}
*{box-sizing:border-box}
html,body{height:100%;margin:0}
body{background:var(--bg);color:var(--ink);font-family:"Space Grotesk",system-ui,sans-serif;overflow:hidden;font-size:14px}
#stage{position:fixed;inset:0;touch-action:none;cursor:grab;background:var(--bg)}
canvas{display:block;width:100%;height:100%}
#top{position:fixed;left:12px;top:calc(12px + env(safe-area-inset-top,0px));right:12px;pointer-events:none;display:flex;flex-direction:column;gap:8px;align-items:flex-start}
#top>*{pointer-events:auto}
h1{font-size:1.4rem;margin:0;font-weight:600;letter-spacing:-.02em;pointer-events:none}
#bar{display:flex;flex-wrap:wrap;gap:6px;max-width:min(520px,calc(100vw - 340px))}
aside{position:fixed;right:12px;top:calc(12px + env(safe-area-inset-top,0px));bottom:calc(12px + env(safe-area-inset-bottom,0px));width:300px;background:var(--panel);border:1px solid var(--line);border-radius:14px;padding:14px;overflow-y:auto;display:flex;flex-direction:column;gap:12px}
.tabs{display:grid;grid-template-columns:1fr 1fr;gap:6px}
button{font:inherit;font-size:.85rem;padding:7px 10px;border-radius:8px;border:1px solid var(--line);background:var(--panel);color:var(--ink);cursor:pointer}
button:hover{border-color:var(--accent)}
button[aria-pressed="true"]{background:var(--accent);border-color:var(--accent);color:#fff}
button.danger{color:#c0392b}
button:focus-visible,input:focus-visible{outline:2px solid var(--accent);outline-offset:2px}
.group{display:grid;gap:9px}
.group h2{font-size:.85rem;margin:4px 0 0;font-weight:600;color:var(--muted)}
.row{display:flex;align-items:center;justify-content:space-between;gap:10px}
.row label{color:var(--muted);flex:0 0 78px}
.row input[type=range]{flex:1;min-width:0;accent-color:var(--accent)}
.row output{flex:0 0 40px;text-align:right;font-variant-numeric:tabular-nums}
input[type=color]{width:44px;height:28px;border:1px solid var(--line);border-radius:6px;background:none;padding:2px}
input[type=checkbox]{width:18px;height:18px;accent-color:var(--accent)}
#list{display:grid;gap:5px}
#list button{text-align:left}
.actions{display:flex;gap:6px;flex-wrap:wrap}
.empty{color:var(--muted);margin:0}
.hint{position:fixed;left:12px;bottom:calc(12px + env(safe-area-inset-bottom,0px));color:var(--muted);margin:0;pointer-events:none}
@media (max-width:720px){
  aside{top:auto;left:12px;right:12px;width:auto;max-height:46vh}
  #bar{max-width:100%}
  .hint{display:none}
}
</style>
</head>
<body>
<div id="stage"></div>
<div id="top"><h1>3D Studio</h1><div id="bar" role="toolbar" aria-label="Add shape"></div></div>
<p class="hint">Drag to orbit. Scroll or pinch to zoom. Click a shape to select it. Delete key removes it.</p>
<aside>
  <div class="tabs"><button id="tObj" aria-pressed="true">Object</button><button id="tScene" aria-pressed="false">Scene</button></div>
  <div id="objTab" class="group"></div>
  <div id="sceneTab" class="group" hidden></div>
</aside>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
(function(){
var $=function(id){return document.getElementById(id)};
var stage=$('stage');
var renderer=new THREE.WebGLRenderer({antialias:true,alpha:true});
renderer.setPixelRatio(Math.min(devicePixelRatio,2));
renderer.shadowMap.enabled=true;
stage.appendChild(renderer.domElement);
var scene=new THREE.Scene();
var camera=new THREE.PerspectiveCamera(50,1,0.1,100);
var cam={dist:9,yaw:0.7,pitch:0.4,auto:false};

var amb=new THREE.AmbientLight(0xffffff,0.55); scene.add(amb);
var key=new THREE.DirectionalLight(0xffffff,0.9); key.castShadow=true;
key.shadow.mapSize.set(1024,1024);
['left','bottom'].forEach(function(k){key.shadow.camera[k]=-9});
['right','top'].forEach(function(k){key.shadow.camera[k]=9});
key.shadow.camera.far=30; scene.add(key);
var rim=new THREE.PointLight(0xffffff,0.4); rim.position.set(-6,2,-5); scene.add(rim);
var lightAngle=40;
function setLight(){var a=lightAngle*Math.PI/180;key.position.set(Math.cos(a)*7,8,Math.sin(a)*7)}
setLight();

var FLOOR=-2;
var grid=new THREE.GridHelper(20,20,0x8896ad,0x8896ad);
grid.position.y=FLOOR; grid.material.transparent=true; grid.material.opacity=0.35; scene.add(grid);
var ground=new THREE.Mesh(new THREE.PlaneGeometry(40,40),new THREE.ShadowMaterial({opacity:0.22}));
ground.rotation.x=-Math.PI/2; ground.position.y=FLOOR+0.001; ground.receiveShadow=true; scene.add(ground);

var G={
 Cube:function(){return new THREE.BoxGeometry(1.6,1.6,1.6)},
 Sphere:function(){return new THREE.SphereGeometry(1,48,32)},
 Torus:function(){return new THREE.TorusGeometry(0.9,0.35,24,64)},
 Knot:function(){return new THREE.TorusKnotGeometry(0.8,0.27,140,20)},
 Cone:function(){return new THREE.ConeGeometry(1,2,40)},
 Cylinder:function(){return new THREE.CylinderGeometry(0.8,0.8,1.8,40)},
 Gem:function(){return new THREE.IcosahedronGeometry(1,0)},
 Ring:function(){return new THREE.TorusGeometry(1.2,0.08,16,100)}
};
var PALETTE=[0x3556f5,0xf5576c,0x2fbf8f,0xf5b835,0x9b59d6,0x16a6c9];
var objs=[],sel=null,count=0;
var box=new THREE.BoxHelper(new THREE.Mesh(new THREE.BoxGeometry(1,1,1)),0xf5b835); box.visible=false; scene.add(box);

function add(type,pos){
  var m=new THREE.Mesh(G[type](),new THREE.MeshStandardMaterial({color:PALETTE[count%PALETTE.length],roughness:0.4,metalness:0.15}));
  m.castShadow=true;
  var n=objs.length;
  m.position.set(pos?pos[0]:((n%5)-2)*1.8,pos?pos[1]:0,pos?pos[2]:Math.floor(n/5)*1.8);
  m.userData={name:type+' '+(++count),spin:0};
  scene.add(m); objs.push(m); select(m); return m;
}
function select(m){sel=m||null; refresh()}
function remove(){
  if(!sel)return; scene.remove(sel); sel.material.dispose();
  objs.splice(objs.indexOf(sel),1); select(null);
}
function duplicate(){
  if(!sel)return; var c=sel.clone(); c.material=sel.material.clone();
  c.userData={name:sel.userData.name+' copy',spin:sel.userData.spin};
  c.position.x+=0.6; c.position.z+=0.6; scene.add(c); objs.push(c); select(c);
}

/* ---------- UI ---------- */
var bar=$('bar');
Object.keys(G).forEach(function(t){var b=document.createElement('button');b.textContent='+ '+t;b.onclick=function(){add(t)};bar.appendChild(b)});

var syncs=[];
function slider(parent,label,min,max,step,get,set,needSel){
  var r=document.createElement('div');r.className='row';
  var id='s'+Math.random().toString(36).slice(2);
  var l=document.createElement('label');l.htmlFor=id;l.textContent=label;
  var i=document.createElement('input');i.type='range';i.id=id;i.min=min;i.max=max;i.step=step;
  var o=document.createElement('output');
  var fmt=function(v){return step<1?(+v).toFixed(2):Math.round(v)};
  i.oninput=function(){if(needSel&&!sel)return;set(parseFloat(i.value));o.textContent=fmt(i.value)};
  r.appendChild(l);r.appendChild(i);r.appendChild(o);parent.appendChild(r);
  syncs.push(function(){if(needSel&&!sel)return;var v=get();i.value=v;o.textContent=fmt(v)});
}
function color(parent,label,get,set,needSel){
  var r=document.createElement('div');r.className='row';
  var id='c'+Math.random().toString(36).slice(2);
  var l=document.createElement('label');l.htmlFor=id;l.textContent=label;
  var i=document.createElement('input');i.type='color';i.id=id;
  i.oninput=function(){if(needSel&&!sel)return;set(i.value)};
  r.appendChild(l);r.appendChild(i);parent.appendChild(r);
  syncs.push(function(){if(needSel&&!sel)return;i.value=get()});
}
function check(parent,label,get,set,needSel){
  var r=document.createElement('div');r.className='row';
  var id='k'+Math.random().toString(36).slice(2);
  var l=document.createElement('label');l.htmlFor=id;l.textContent=label;
  var i=document.createElement('input');i.type='checkbox';i.id=id;
  i.onchange=function(){if(needSel&&!sel)return;set(i.checked)};
  r.appendChild(l);r.appendChild(i);parent.appendChild(r);
  syncs.push(function(){if(needSel&&!sel)return;i.checked=get()});
}
function head(p,t){var h=document.createElement('h2');h.textContent=t;p.appendChild(h)}
function btn(p,t,fn,cls){var b=document.createElement('button');b.textContent=t;b.onclick=fn;if(cls)b.className=cls;p.appendChild(b);return b}
var D=180/Math.PI,R=Math.PI/180;

/* Object tab */
var objTab=$('objTab');
var list=document.createElement('div');list.id='list';objTab.appendChild(list);
var empty=document.createElement('p');empty.className='empty';empty.textContent='Nothing selected. Click a shape, or add one from the top bar.';objTab.appendChild(empty);
var props=document.createElement('div');props.className='group';objTab.appendChild(props);
var acts=document.createElement('div');acts.className='actions';
btn(acts,'Duplicate',duplicate);btn(acts,'Delete',remove,'danger');props.appendChild(acts);
head(props,'Position');
['x','y','z'].forEach(function(a){slider(props,a.toUpperCase(),-6,6,0.05,function(){return sel.position[a]},function(v){sel.position[a]=v;box.update()},true)});
head(props,'Rotation (degrees)');
['x','y','z'].forEach(function(a){slider(props,a.toUpperCase(),0,360,1,function(){return ((sel.rotation[a]*D)%360+360)%360},function(v){sel.rotation[a]=v*R},true)});
head(props,'Size');
slider(props,'Scale',0.2,3,0.05,function(){return sel.scale.x},function(v){sel.scale.set(v,v,v);box.update()},true);
head(props,'Material');
color(props,'Color',function(){return '#'+sel.material.color.getHexString()},function(v){sel.material.color.set(v)},true);
slider(props,'Roughness',0,1,0.01,function(){return sel.material.roughness},function(v){sel.material.roughness=v},true);
slider(props,'Metalness',0,1,0.01,function(){return sel.material.metalness},function(v){sel.material.metalness=v},true);
slider(props,'Opacity',0.1,1,0.01,function(){return sel.material.opacity},function(v){sel.material.opacity=v;sel.material.transparent=v<1},true);
check(props,'Wireframe',function(){return sel.material.wireframe},function(v){sel.material.wireframe=v},true);
head(props,'Animation');
slider(props,'Spin',0,4,0.1,function(){return sel.userData.spin},function(v){sel.userData.spin=v},true);

function refresh(){
  list.innerHTML='';
  objs.forEach(function(m){
    var b=document.createElement('button');b.textContent=m.userData.name;
    b.setAttribute('aria-pressed',m===sel);b.onclick=function(){select(m)};list.appendChild(b);
  });
  props.hidden=!sel;empty.hidden=!!sel;
  if(sel){box.visible=true;box.setFromObject(sel)}else box.visible=false;
  syncs.forEach(function(f){f()});
}

/* Scene tab */
var sc=$('sceneTab');
head(sc,'Lighting');
slider(sc,'Main light',0,2,0.05,function(){return key.intensity},function(v){key.intensity=v});
slider(sc,'Ambient',0,1.5,0.05,function(){return amb.intensity},function(v){amb.intensity=v});
slider(sc,'Light angle',0,360,1,function(){return lightAngle},function(v){lightAngle=v;setLight()});
color(sc,'Light color',function(){return '#'+key.color.getHexString()},function(v){key.color.set(v)});
check(sc,'Shadows',function(){return ground.visible},function(v){ground.visible=v;key.castShadow=v});
head(sc,'World');
color(sc,'Background',function(){return bgHex},function(v){bgHex=v;stage.style.background=v});
check(sc,'Grid',function(){return grid.visible},function(v){grid.visible=v});
check(sc,'Auto-orbit',function(){return cam.auto},function(v){cam.auto=v});
head(sc,'Scene');
var sa=document.createElement('div');sa.className='actions';
btn(sa,'Reset camera',function(){cam.dist=9;cam.yaw=0.7;cam.pitch=0.4});
btn(sa,'Random scene',function(){
  clearAll();var t=Object.keys(G);
  for(var i=0;i<8;i++){var m=add(t[Math.floor(Math.random()*t.length)],[(Math.random()-0.5)*8,(Math.random()-0.5)*3,(Math.random()-0.5)*8]);
    m.rotation.set(Math.random()*6,Math.random()*6,0);m.scale.setScalar(0.5+Math.random()*0.9);m.userData.spin=Math.random()*2}
  select(null)});
btn(sa,'Clear all',function(){clearAll();refresh()},'danger');
sc.appendChild(sa);
var bgHex='#e6ebf2';
function clearAll(){objs.forEach(function(m){scene.remove(m);m.material.dispose()});objs=[];sel=null;count=0}

$('tObj').onclick=$('tScene').onclick=function(e){
  var o=e.target===$('tObj');
  $('tObj').setAttribute('aria-pressed',o);$('tScene').setAttribute('aria-pressed',!o);
  objTab.hidden=!o;sc.hidden=o;
};
addEventListener('keydown',function(e){
  if((e.key==='Delete'||e.key==='Backspace')&&sel&&!/INPUT|TEXTAREA/.test(e.target.tagName)){e.preventDefault();remove()}
});

/* ---------- Camera input + picking ---------- */
var pts={},last=0,px=0,py=0,moved=0,ray=new THREE.Raycaster(),mouse=new THREE.Vector2();
function clamp(v,a,b){return Math.max(a,Math.min(b,v))}
stage.addEventListener('pointerdown',function(e){
  stage.setPointerCapture(e.pointerId);pts[e.pointerId]=e;px=e.clientX;py=e.clientY;last=0;moved=0;
});
stage.addEventListener('pointermove',function(e){
  if(!pts[e.pointerId])return;
  var ids=Object.keys(pts);pts[e.pointerId]=e;
  if(ids.length===2){
    var a=pts[ids[0]],b=pts[ids[1]],d=Math.hypot(a.clientX-b.clientX,a.clientY-b.clientY);
    if(last)cam.dist=clamp(cam.dist*last/d,3,22);last=d;moved=99;return;
  }
  moved+=Math.abs(e.clientX-px)+Math.abs(e.clientY-py);
  cam.yaw-=(e.clientX-px)*0.008;cam.pitch=clamp(cam.pitch+(e.clientY-py)*0.008,-0.2,1.45);
  px=e.clientX;py=e.clientY;
});
function up(e){
  var single=Object.keys(pts).length===1;
  if(e.type==='pointerup'&&single&&moved<6){
    var r=stage.getBoundingClientRect();
    mouse.set(((e.clientX-r.left)/r.width)*2-1,-((e.clientY-r.top)/r.height)*2+1);
    ray.setFromCamera(mouse,camera);
    var hit=ray.intersectObjects(objs)[0];select(hit?hit.object:null);
  }
  delete pts[e.pointerId];last=0;
}
stage.addEventListener('pointerup',up);stage.addEventListener('pointercancel',up);
stage.addEventListener('wheel',function(e){e.preventDefault();cam.dist=clamp(cam.dist+e.deltaY*0.008,3,22)},{passive:false});

function resize(){var w=stage.clientWidth,h=stage.clientHeight;renderer.setSize(w,h,false);camera.aspect=w/h;camera.updateProjectionMatrix()}
addEventListener('resize',resize);resize();

/* ---------- Loop ---------- */
var reduce=matchMedia('(prefers-reduced-motion: reduce)').matches,clock=new THREE.Clock();
function tick(){
  var dt=clock.getDelta();
  if(!reduce){
    objs.forEach(function(m){if(m.userData.spin)m.rotation.y+=dt*m.userData.spin});
    if(cam.auto&&!Object.keys(pts).length)cam.yaw+=dt*0.3;
  }
  camera.position.set(Math.sin(cam.yaw)*Math.cos(cam.pitch)*cam.dist,Math.sin(cam.pitch)*cam.dist,Math.cos(cam.yaw)*Math.cos(cam.pitch)*cam.dist);
  camera.lookAt(0,0,0);
  if(sel)box.setFromObject(sel);
  renderer.render(scene,camera);
  requestAnimationFrame(tick);
}
add('Cube',[-2.4,0,0]);add('Sphere',[0,0,0]);add('Knot',[2.4,0,0]).userData.spin=1;
select(null);
tick();
})();
</script>
</body>
</html>
