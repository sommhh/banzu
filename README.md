

Meta AIに聞く...
<!DOCTYPE html>
<html lang="ja">
<head>
<!-- Playables SDK -->
<script>// Playables SDK v1.0.0
// Game lifecycle bridge: rAF-based game-ready detection + event communication
(function() {
  'use strict';

  // Idempotency: skip if already initialized (e.g., server-side injection
  // followed by client-side inject-javascript via the Bloks webview component).
  if (window.playablesSDK) return;

  var HANDLER_NAME = 'playablesGameEventHandler';
  var ANDROID_BRIDGE_NAME = '_MetaPlayablesBridge';
  var RAF_FRAME_THRESHOLD = 3;

  var gameReadySent = false;
  var firstInteractionSent = false;
  var errorSent = false;
  var frameCount = 0;
  var originalRAF = window.requestAnimationFrame;

  // --- Transport Layer ---

  function hasIOSBridge() {
    return !!(window.webkit &&
              window.webkit.messageHandlers &&
              window.webkit.messageHandlers[HANDLER_NAME]);
  }

  function hasAndroidBridge() {
    return !!(window[ANDROID_BRIDGE_NAME] &&
              typeof window[ANDROID_BRIDGE_NAME].postEvent === 'function');
  }

  function isInIframe() {
    return !!(window.parent && window.parent !== window);
  }

  function sendEvent(eventName, payload) {
    var message = {
      type: eventName,
      payload: payload || {},
      timestamp: Date.now()
    };

    if (hasIOSBridge()) {
      try {
        window.webkit.messageHandlers[HANDLER_NAME].postMessage(message);
      } catch (e) { /* ignore */ }
      return;
    }

    if (hasAndroidBridge()) {
    try {
      var p = payload || {};
      p.__secureToken = window.__fbAndroidBridgeAuthToken || '';
      window[ANDROID_BRIDGE_NAME].postEvent(
        eventName,
        JSON.stringify(p)
      );
    } catch (e) { /* ignore */ }
    return;
  }

    if (isInIframe()) {
      try {
        window.parent.postMessage(message, '*');
      } catch (e) { /* ignore */ }
      return;
    }
  }

  // --- rAF Game-Ready Detection ---

  function onFrame() {
    if (gameReadySent) return;

    frameCount++;
    if (frameCount >= RAF_FRAME_THRESHOLD) {
      gameReadySent = true;
      sendEvent('game_ready', {
        frame_count: frameCount,
        detected_at: Date.now()
      });
      return;
    }

    originalRAF.call(window, onFrame);
  }

  if (originalRAF) {
    window.requestAnimationFrame = function(callback) {
      if (!gameReadySent) {
        return originalRAF.call(window, function(timestamp) {
          frameCount++;
          if (frameCount >= RAF_FRAME_THRESHOLD && !gameReadySent) {
            gameReadySent = true;
            sendEvent('game_ready', {
              frame_count: frameCount,
              detected_at: Date.now()
            });
          }
          callback(timestamp);
        });
      }
      return originalRAF.call(window, callback);
    };
  }

  // --- First User Interaction Detection ---

  function setupFirstInteractionDetection() {
    var events = ['touchstart', 'mousedown', 'keydown'];

    function onFirstInteraction() {
      if (firstInteractionSent) return;
      firstInteractionSent = true;
      sendEvent('user_interaction_start', null);

      for (var i = 0; i < events.length; i++) {
        document.removeEventListener(events[i], onFirstInteraction, true);
      }
    }

    for (var i = 0; i < events.length; i++) {
      document.addEventListener(events[i], onFirstInteraction, true);
    }
  }

  if (document.readyState === 'loading') {
    document.addEventListener('DOMContentLoaded', setupFirstInteractionDetection);
  } else {
    setupFirstInteractionDetection();
  }

  // --- Auto Error Capture ---

  window.addEventListener('error', function(event) {
    if (errorSent) return;
    errorSent = true;
    sendEvent('error', {
      message: event.message || 'Unknown error',
      source: event.filename || '',
      lineno: event.lineno || 0,
      colno: event.colno || 0,
      auto_captured: true
    });
  });

  window.addEventListener('unhandledrejection', function(event) {
    if (errorSent) return;
    errorSent = true;
    var reason = event.reason;
    sendEvent('error', {
      message: (reason instanceof Error) ? reason.message : String(reason),
      type: 'unhandled_promise_rejection',
      auto_captured: true
    });
  });

  // --- Public API ---

  window.playablesSDK = {
    complete: function(score) {
      sendEvent('game_ended', {
        score: score,
        completed: true
      });
    },

    error: function(message) {
      if (errorSent) return;
      errorSent = true;
      sendEvent('error', {
        message: message || 'Unknown error',
        auto_captured: false
      });
    },

    sendEvent: function(eventName, payload) {
      if (!eventName || typeof eventName !== 'string') return;
      sendEvent(eventName, payload);
    }
  };

  // Kick off rAF detection in case no game code calls rAF immediately
  if (originalRAF) {
    originalRAF.call(window, onFrame);
  }
})();</script>
<script>window.Intl=window.Intl||{};Intl.t=function(s){return(Intl._locale&&Intl._locale[s])||s;};</script>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>分電盤エディタ</title>
<script src="https://cdn.tailwindcss.com"></script>
<link href="https://fonts.googleapis.com/css2?family=Noto+Sans+JP:wght@400;500;700&family=IBM+Plex+Mono:wght@500&display=swap" rel="stylesheet">
<style>
  html,body{font-family:"Noto Sans JP",system-ui,sans-serif;background:#f1f5f9}
  @media print{@page{size:B5 landscape;margin:2mm}body{background:#fff}.no-print{display:none!important}#paper{padding:0!important;box-shadow:none!important}.screen-sched{display:none!important}#printList{display:block!important}#diagram{width:250mm!important;height:auto!important;margin:0 auto}h1{margin-bottom:1mm!important}}
  .mono{font-family:"IBM Plex Mono",monospace}
  .sched-input{width:100%;background:transparent;outline:none;padding:2px 4px;border-radius:3px}
  .sched-input:focus{background:#fef3c7;box-shadow:inset 0 0 0 1px #f59e0b}
  .btn-tab{border:1px solid #cbd5e1;padding:2px 8px;border-radius:6px;font-size:12px;background:#fff;min-width:48px}
  .btn-tab.active{background:#0f172a;color:#fff;border-color:#0f172a}
  .btn-ghost{border:1px solid #e2e8f0;padding:4px 10px;border-radius:8px;background:#fff;font-size:13px}
</style>
</head>
<body class="text-slate-800">
<div class="no-print sticky top-0 z-40 backdrop-blur bg-white/80 border-b">
  <div class="max-w-[1400px] mx-auto px-4 py-3 flex flex-wrap gap-3">
    <div class="flex items-center gap-2"><span class="text-[11px]">引込</span>
      <select id="inlet" class="border rounded-lg px-2 py-1.5 text-sm">
        <option selected>CVT 8sq</option><option>CVT 14sq</option><option>CVT 22sq</option><option>CVT 38sq</option><option>CVT 60sq</option>
      </select></div>
    <div class="flex items-center gap-2"><span class="text-[11px]">開閉器</span><button id="tglSwitch" class="btn-tab">なし</button></div>
    <div class="flex items-center gap-2"><span class="text-[11px]">主幹</span>
      <select id="mainType" class="border rounded-lg px-2 py-1.5 text-sm"><option selected>ELB</option><option>MCB</option></select>
      <select id="mainAmp" class="border rounded-lg px-2 py-1.5 text-sm"><option selected>50A</option><option>60A</option><option>75A</option><option>100A</option><option>125A</option></select>
    </div>
    <div class="flex items-center gap-1"><input id="siteName" placeholder="現場名" class="border rounded px-2 py-1 text-sm w-32" value=""><span class="text-[11px] mr-1 ml-2">回路数</span><button id="minus2" class="btn-ghost">-2</button><span id="circCount" class="w-8 text-center">16</span><button id="plus2" class="btn-ghost">+2</button></div>
    <div class="ml-auto flex gap-2"><button id="reset" class="btn-ghost">初期化</button><button id="print" class="bg-slate-900 text-white px-4 py-1.5 rounded-lg text-sm">印刷</button></div>
  </div>
</div>
<div class="max-w-[1400px] mx-auto my-6">
<div id="paper" class="bg-white shadow-xl ring-1 rounded-xl p-6">
<h1 class="text-[22px] font-bold" id="docTitle"></h1>
<div class="border rounded-lg overflow-hidden mt-2"><svg id="diagram" viewBox="0 0 1220 460" class="w-full"></svg></div>
<div id="printList" class="hidden print:block mt-2"></div>
<div class="grid md:grid-cols-2 gap-6 mt-6 screen-sched">
<div><div class="text-[12px] font-semibold mb-1.5">回路表（左）</div><div id="schedLeft" class="border rounded-lg overflow-hidden"></div></div>
<div><div class="text-[12px] font-semibold mb-1.5">回路表（右）</div><div id="schedRight" class="border rounded-lg overflow-hidden"></div></div>
</div></div></div>
<script>
const NS="http://www.w3.org/2000/svg",$=s=>document.querySelector(s),svg=$("#diagram");
let state={inlet:"CVT 8sq",hasSwitch:false,mainType:"ELB",mainAmp:"50A",circuits:[],site:""};
function initCircuits(n=16){
  state.circuits=Array.from({length:n},(_,i)=>({no:i+1,name:["照明","コンセント","エアコン","冷蔵庫","電子レンジ","IH","洗面","トイレ","外灯","予備"][i]||`回路${i+1}`,voltage:100,amp:20}));
  $("#circCount").textContent=n;
}
function sLine(x1,y1,x2,y2,w=1.7){const l=document.createElementNS(NS,"line");l.setAttribute("x1",x1);l.setAttribute("y1",y1);l.setAttribute("x2",x2);l.setAttribute("y2",y2);l.setAttribute("stroke","#000");l.setAttribute("stroke-width",w);svg.appendChild(l)}
function sRect(x,y,w,h,r=0){const e=document.createElementNS(NS,"rect");e.setAttribute("x",x);e.setAttribute("y",y);e.setAttribute("width",w);e.setAttribute("height",h);e.setAttribute("rx",r);e.setAttribute("fill","#fff");e.setAttribute("stroke","#000");e.setAttribute("stroke-width",1.7);svg.appendChild(e)}
function sCircle(x,y,r){const e=document.createElementNS(NS,"circle");e.setAttribute("cx",x);e.setAttribute("cy",y);e.setAttribute("r",r);e.setAttribute("fill","#fff");e.setAttribute("stroke","#000");e.setAttribute("stroke-width",1.7);svg.appendChild(e)}
function sText(x,y,t,s=12,w=500,a="middle"){const e=document.createElementNS(NS,"text");e.setAttribute("x",x);e.setAttribute("y",y);e.setAttribute("text-anchor",a);e.setAttribute("font-size",s);e.setAttribute("font-weight",w);e.textContent=t;svg.appendChild(e)}
function renderDiagram(){
  svg.innerHTML="";const leftX=180,busY=300;
  sText(60,32,state.inlet,13,600,"start");sText(leftX-70,78,"1φ3W",14,600,"start");
  for(let i=-1;i<=1;i++)sLine(leftX-12+i*8,60,leftX-22+i*8,80);
  sLine(leftX-17,70,leftX-2,70,2);sLine(leftX,70,leftX,110);
  sCircle(leftX,130,22);sText(leftX,135,"WH",12,700);sLine(leftX,152,leftX,185);
  let curY=185;
  if(state.hasSwitch){sRect(leftX-20,curY,40,28,2);sLine(leftX-20,curY+28,leftX+20,curY);sText(leftX-28,curY-6,"開閉器",11,500,"end");sLine(leftX,curY+28,leftX,curY+55);curY+=55;}
  const mainY=curY+8;sRect(leftX-28,mainY,56,42,3);sText(leftX,mainY+18,state.mainType,13,700);sText(leftX,mainY+35,state.mainAmp,12,600);
  sLine(leftX,mainY+42,leftX,busY);sLine(leftX,busY,leftX+26,busY,2.2);
  const busEnd=1160;sLine(leftX+26,busY,busEnd,busY,3);
  const startX=260,pitch=64,topY=busY-58,botY=busY+58;
  state.circuits.forEach((c,i)=>{const col=Math.floor(i/2),isTop=i%2===0,x=startX+col*pitch,yB=isTop?topY:botY;
    sLine(x,busY,x,isTop?yB+14:yB-14);sRect(x-14,yB-14,28,28,2);sText(x,yB+4,c.amp+"A",11,700);
    const ny=isTop?yB-36:yB+36;if(c.voltage===100){sCircle(x,ny,10);sText(x,ny+4,c.no,11)}else{sCircle(x,ny,14);sCircle(x,ny,10);sText(x,ny+4,c.no,11)}
  });
  sLine(busEnd,busY-8,busEnd,busY+8,2);
}
function renderSchedule(){
  const make=list=>{const w=document.createElement("div");const t=document.createElement("table");t.className="w-full text-[12px]";t.innerHTML=`<thead class="bg-slate-50"><tr class="border-b"><th class="w-11 py-1">No</th><th class="text-left py-1">回路名称</th><th class="w-24 py-1">電圧</th><th class="w-20 py-1">容量</th></tr></thead><tbody></tbody>`;const b=t.querySelector("tbody");
    list.forEach(c=>{const r=document.createElement("tr");r.className="border-b";r.innerHTML=`<td class="text-center mono">${c.no}</td><td><input data-no="${c.no}" data-k="name" class="sched-input" value="${c.name}"></td><td class="text-center"><button data-no="${c.no}" data-k="v100" class="btn-tab ${c.voltage===100?'active':''} mr-1">100V</button><button data-no="${c.no}" data-k="v200" class="btn-tab ${c.voltage===200?'active':''}">200V</button></td><td class="text-center"><button data-no="${c.no}" data-k="a20" class="btn-tab ${c.amp===20?'active':''} mr-1">20A</button><button data-no="${c.no}" data-k="a30" class="btn-tab ${c.amp===30?'active':''}">30A</button></td>`;b.appendChild(r)});w.appendChild(t);return w};
  const h=Math.ceil(state.circuits.length/2);$("#schedLeft").innerHTML="";$("#schedRight").innerHTML="";$("#schedLeft").appendChild(make(state.circuits.slice(0,h)));$("#schedRight").appendChild(make(state.circuits.slice(h)));
  document.querySelectorAll(".sched-input").forEach(i=>i.addEventListener("input",e=>{const c=state.circuits.find(x=>x.no==e.target.dataset.no);c.name=e.target.value}));
  document.querySelectorAll("button[data-k]").forEach(b=>b.addEventListener("click",e=>{const c=state.circuits.find(x=>x.no==e.currentTarget.dataset.no),k=e.currentTarget.dataset.k;if(k==="v100")c.voltage=100;if(k==="v200")c.voltage=200;if(k==="a20")c.amp=20;if(k==="a30")c.amp=30;renderAll()}));
}
function renderAll(){
  renderDiagram();renderSchedule();document.getElementById("docTitle").textContent = state.site || "";
  // print list
  const pl=document.getElementById("printList");
  const half=Math.ceil(state.circuits.length/2);
  let html='<div style="display:grid;grid-template-columns:1fr 1fr;gap:8px;font-size:10px"><table style="width:100%;border-collapse:collapse"><thead><tr style="background:#f1f5f9"><th style="border:1px solid #000;padding:2px;width:24px">No</th><th style="border:1px solid #000;padding:2px">名称</th><th style="border:1px solid #000;padding:2px;width:36px">電圧</th><th style="border:1px solid #000;padding:2px;width:36px">容量</th></tr></thead><tbody>';
  state.circuits.slice(0,half).forEach(c=>{html+=`<tr><td style="border:1px solid #000;text-align:center">${c.no}</td><td style="border:1px solid #000;padding:0 4px">${c.name}</td><td style="border:1px solid #000;text-align:center">${c.voltage}V</td><td style="border:1px solid #000;text-align:center">${c.amp}A</td></tr>`});
  html+='</tbody></table><table style="width:100%;border-collapse:collapse"><thead><tr style="background:#f1f5f9"><th style="border:1px solid #000;padding:2px;width:24px">No</th><th style="border:1px solid #000;padding:2px">名称</th><th style="border:1px solid #000;padding:2px;width:36px">電圧</th><th style="border:1px solid #000;padding:2px;width:36px">容量</th></tr></thead><tbody>';
  state.circuits.slice(half).forEach(c=>{html+=`<tr><td style="border:1px solid #000;text-align:center">${c.no}</td><td style="border:1px solid #000;padding:0 4px">${c.name}</td><td style="border:1px solid #000;text-align:center">${c.voltage}V</td><td style="border:1px solid #000;text-align:center">${c.amp}A</td></tr>`});
  html+='</tbody></table></div>';
  pl.innerHTML=html;
}
$("#inlet").onchange=e=>{state.inlet=e.target.value;renderDiagram()};
$("#mainType").onchange=e=>{state.mainType=e.target.value;renderDiagram()};
$("#mainAmp").onchange=e=>{state.mainAmp=e.target.value;renderDiagram()};
$("#tglSwitch").onclick=e=>{state.hasSwitch=!state.hasSwitch;e.currentTarget.classList.toggle("active",state.hasSwitch);e.currentTarget.textContent=state.hasSwitch?"あり":"なし";renderDiagram()};
$("#plus2").onclick=()=>{if(state.circuits.length<30){initCircuits(state.circuits.length+2);renderAll()}};
$("#minus2").onclick=()=>{if(state.circuits.length>4){initCircuits(state.circuits.length-2);renderAll()}};
$("#reset").onclick=()=>{initCircuits(16);renderAll()};
$("#print").onclick=()=>window.print();
$("#siteName").oninput=e=>{state.site=e.target.value;renderAll()};
initCircuits(16);renderAll();
</script>
</body>
</html>
