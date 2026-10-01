<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>3D Playground</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600&display=swap" rel="stylesheet">
<style>
:root{--bg:#e6ebf2;--panel:#fbfcfe;--ink:#1b2433;--muted:#5d6b82;--line:#c9d2df;--accent:#3556f5;--grid:#aab6c8;
box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#141a26;--panel:#1d2535;--ink:#e8edf6;--muted:#93a1ba;--line:#33405a;--accent:#7b93ff;--grid:#33405a}}
:root[data-theme="dark"]{--bg:#141a26;--panel:#1d2535;--ink:#e8edf6;--muted:#93a1ba;--line:#33405a;--accent:#7b93ff;--grid:#33405a}
html{scroll-padding-top:env(safe-area-inset-top,0px)}
*{box-sizing:border-box}
html,body{height:100%;margin:0}
body{background:var(--bg);color:var(--ink);font-family:"Space Grotesk",system-ui,sans-serif;overflow:hidden}
#stage{position:fixed;inset:0;touch-action:none;cursor:grab}
#stage:active{cursor:grabbing}
canvas{display:block;width:100%;height:100%}
header{position:fixed;left:20px;top:calc(16px + env(safe-area-inset-top,0px));pointer-events:none;max-width:60vw}
h1{font-size:clamp(1.4rem,4vw,2.2rem);margin:0 0 4px;font-weight:600;letter-spacing:-.02em}
header p{margin:0;color:var(--muted);font-size:.95rem}
aside{position:fixed;right:16px;bottom:calc(16px + env(safe-area-inset-bottom,0px));width:min(300px,calc(100vw - 32px));
background:var(--panel);border:1px solid var(--line);border-radius:14px;padding:16px;display:grid;gap:14px}
label,.row-title{font-size:.9rem;color:var(--muted)}
.shapes{display:grid;grid-template-columns:repeat(5,1fr);gap:6px}
button{font:inherit;font-size:.82rem;padding:8px 0;border-radius:8px;border:1px solid var(--line);background:transparent;color:var(--ink);cursor:pointer}
button[aria-pressed="true"]{background:var(--accent);border-color:var(--accent);color:#fff}
button:focus-visible,input:focus-visible{outline:2px solid var(--accent);outline-offset:2px}
.row{display:flex;align-items:center;justify-content:space-between;gap:12px}
input[type=range]{width:150px;accent-color:var(--accent)}
input[type=color]{width:44px;height:30px;border:1px solid var(--line);border-radius:6px;background:none;padding:2px}
input[type=checkbox]{width:18px;height:18px;accent-color:var(--accent)}
</style>
</head>
<body>
<div id="stage"></div>
<header><h1>3D Playground</h1><p>Drag to rotate. Scroll or pinch to zoom.</p></header>
<aside>
  <div><div class="row-title" style="margin-bottom:6px">Shape</div><div class="shapes" id="shapes"></div></div>
  <div class="row"><label for="color">Color</label><input type="color" id="color" value="#3556f5"></div>
  <div class="row"><label for="speed">Spin speed</label><input type="range" id="speed" min="0" max="3" step="0.1" value="1"></div>
  <div class="row"><label for="wire">Wireframe</label><input type="checkbox" id="wire"></div>
</aside>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
(function(){
  var stage=document.getElementById('stage');
  var renderer=new THREE.WebGLRenderer({antialias:true,alpha:true});
  renderer.setPixelRatio(Math.min(devicePixelRatio,2));
  stage.appendChild(renderer.domElement);
  var scene=new THREE.Scene();
  var camera=new THREE.PerspectiveCamera(50,1,0.1,100);
  var dist=5, yaw=0.6, pitch=0.35;

  scene.add(new THREE.AmbientLight(0xffffff,0.55));
  var key=new THREE.DirectionalLight(0xffffff,0.9); key.position.set(4,6,5); scene.add(key);
  var rim=new THREE.PointLight(0xffffff,0.5); rim.position.set(-5,-2,-4); scene.add(rim);

  var grid=new THREE.GridHelper(14,14,0x8896ad,0x8896ad);
  grid.position.y=-1.6; grid.material.transparent=true; grid.material.opacity=0.35; scene.add(grid);

  var geos={
    Cube:function(){return new THREE.BoxGeometry(1.8,1.8,1.8)},
    Sphere:function(){return new THREE.SphereGeometry(1.2,48,32)},
    Torus:function(){return new THREE.TorusGeometry(1.1,0.45,24,64)},
    Knot:function(){return new THREE.TorusKnotGeometry(0.95,0.32,140,20)},
    Cone:function(){return new THREE.ConeGeometry(1.2,2.2,40)}
  };
  var mat=new THREE.MeshStandardMaterial({color:0x3556f5,roughness:0.4,metalness:0.15});
  var mesh=new THREE.Mesh(geos.Cube(),mat); scene.add(mesh);

  var shapesEl=document.getElementById('shapes');
  Object.keys(geos).forEach(function(name,i){
    var b=document.createElement('button');
    b.textContent=name; b.setAttribute('aria-pressed',i===0);
    b.onclick=function(){
      mesh.geometry.dispose(); mesh.geometry=geos[name]();
      [].forEach.call(shapesEl.children,function(c){c.setAttribute('aria-pressed',c===b)});
    };
    shapesEl.appendChild(b);
  });
  var speed=1;
  document.getElementById('color').oninput=function(e){mat.color.set(e.target.value)};
  document.getElementById('speed').oninput=function(e){speed=parseFloat(e.target.value)};
  document.getElementById('wire').onchange=function(e){mat.wireframe=e.target.checked};

  var pts={}, last=0, dragging=false, px=0, py=0;
  stage.addEventListener('pointerdown',function(e){stage.setPointerCapture(e.pointerId);pts[e.pointerId]=e;dragging=true;px=e.clientX;py=e.clientY;last=0});
  stage.addEventListener('pointermove',function(e){
    if(!pts[e.pointerId])return;
    var ids=Object.keys(pts);
    if(ids.length===2){
      pts[e.pointerId]=e;
      var a=pts[ids[0]],b=pts[ids[1]];
      var d=Math.hypot(a.clientX-b.clientX,a.clientY-b.clientY);
      if(last)dist=clamp(dist*last/d,2.5,12); last=d; return;
    }
    pts[e.pointerId]=e;
    yaw-=(e.clientX-px)*0.008; pitch=clamp(pitch+(e.clientY-py)*0.008,-1.3,1.3);
    px=e.clientX;py=e.clientY;
  });
  function up(e){delete pts[e.pointerId];last=0;dragging=Object.keys(pts).length>0}
  stage.addEventListener('pointerup',up); stage.addEventListener('pointercancel',up);
  stage.addEventListener('wheel',function(e){e.preventDefault();dist=clamp(dist+e.deltaY*0.005,2.5,12)},{passive:false});
  function clamp(v,a,b){return Math.max(a,Math.min(b,v))}

  function resize(){
    var w=stage.clientWidth,h=stage.clientHeight;
    renderer.setSize(w,h,false); camera.aspect=w/h; camera.updateProjectionMatrix();
  }
  addEventListener('resize',resize); resize();

  var reduce=matchMedia('(prefers-reduced-motion: reduce)').matches;
  var clock=new THREE.Clock();
  function tick(){
    var dt=clock.getDelta();
    if(!reduce){mesh.rotation.y+=dt*0.6*speed; mesh.rotation.x+=dt*0.25*speed}
    camera.position.set(Math.sin(yaw)*Math.cos(pitch)*dist,Math.sin(pitch)*dist,Math.cos(yaw)*Math.cos(pitch)*dist);
    camera.lookAt(0,0,0);
    renderer.render(scene,camera);
    requestAnimationFrame(tick);
  }
  tick();
})();
</script>
</body>
</html>
