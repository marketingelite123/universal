<script>
(function(){

//////////////////////////////////////////////////
// 1. CONFIGURAÇÕES
//////////////////////////////////////////////////
var ENDPOINT="https://applicant-scenarios-feb-namely.trycloudflare.com/webhook/tracking";
var GEO_API="https://ipapi.co/json/";
var BATCH_INTERVAL=15000;
var AES_PASSWORD="123ABC!!@OK";

//////////////////////////////////////////////////
// 2. UTILITÁRIOS & MOTORES (HASH + AES)
//////////////////////////////////////////////////
function uuid(){
    return'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g,function(c){
        var r=Math.random()*16|0,v=c=='x'?r:(r&0x3|0x8);
        return v.toString(16);
    });
}

function throttle(fn,limit){
    var waiting=false;
    return function(){
        if(!waiting){
            fn.apply(this,arguments);
            waiting=true;
            setTimeout(function(){waiting=false;},limit);
        }
    }
}

function cookie(n){
    var m=document.cookie.match(new RegExp('(^| )'+n+'=([^;]+)'));
    return m?m[2]:null;
}

function setCookie(n,v,d){
    var dt=new Date();
    dt.setTime(dt.getTime()+d*86400000);
    document.cookie=n+"="+v+";path=/;expires="+dt.toUTCString();
}

function sha256(message) {
    try {
        var msgUint8 = new TextEncoder().encode(message);
        var subtle = window.crypto && (window.crypto.subtle || window.crypto.webkitSubtle);
        if (!subtle) return Promise.resolve("no-crypto-" + uuid().slice(0,8));

        return subtle.digest('SHA-256', msgUint8).then(function(hashBuffer) {
            var hashArray = Array.from(new Uint8Array(hashBuffer));
            return hashArray.map(function(b){ return b.toString(16).padStart(2, '0') }).join('');
        }).catch(function() { return "hash-err-" + uuid().slice(0,8); });
    } catch(e) { return Promise.resolve("hash-sync-err-" + uuid().slice(0,8)); }
}

function encryptAES(text) {
    var subtle = window.crypto && (window.crypto.subtle || window.crypto.webkitSubtle);
    if (!subtle) return Promise.resolve(null);

    var enc = new TextEncoder();
    var keyBuffer = new Uint8Array(32);
    keyBuffer.set(enc.encode(AES_PASSWORD)); 

    return subtle.importKey("raw", keyBuffer, { name: "AES-GCM" }, false, ["encrypt"])
        .then(function(key) {
            var iv = window.crypto.getRandomValues(new Uint8Array(12)); 
            return subtle.encrypt({ name: "AES-GCM", iv: iv }, key, enc.encode(text))
                .then(function(encrypted) {
                    var combined = new Uint8Array(iv.length + encrypted.byteLength);
                    combined.set(iv);
                    combined.set(new Uint8Array(encrypted), iv.length);

                    return btoa(String.fromCharCode.apply(null, combined))
                        .replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');
                });
        })
        .catch(function(e) {
            console.error("Erro AES:", e);
            return null;
        });
}

//////////////////////////////////////////////////
// 3. EXTRATORES DE DNA DE ELITE
//////////////////////////////////////////////////
var dna = { hardware_hash: null, software_hash: null, full_hash: null, raw: {} };

function getCanvasFp() {
    try {
        var canvas = document.createElement('canvas'); var ctx = canvas.getContext('2d');
        canvas.width = 200; canvas.height = 50;
        ctx.textBaseline = "top"; ctx.font = "14px 'Arial'"; ctx.textBaseline = "alphabetic";
        ctx.fillStyle = "#f60"; ctx.fillRect(125,1,62,20);
        ctx.fillStyle = "#069"; ctx.fillText("Tracking <canvas>", 2, 15);
        ctx.fillStyle = "rgba(102, 204, 0, 0.7)"; ctx.fillText("Tracking <canvas>", 4, 17);
        return canvas.toDataURL().slice(-100); 
    } catch(e) { return "canvas-err"; }
}

function getEmojiFp() {
    try {
        var canvas = document.createElement('canvas'); var ctx = canvas.getContext('2d');
        canvas.width = 50; canvas.height = 50;
        ctx.textBaseline = "top"; ctx.font = "32px Arial"; 
        ctx.fillText("🤡🌍", 2, 2);
        return canvas.toDataURL().slice(-60);
    } catch(e) { return "emoji-err"; }
}

function getAudioFp() {
    try {
        var AudioContext = window.AudioContext || window.webkitAudioContext;
        if (!AudioContext) return 'no-audio';
        var context = new AudioContext();
        if (context.state === 'suspended') return 'audio_blocked';

        var oscillator = context.createOscillator(); var analyser = context.createAnalyser(); var gain = context.createGain();
        gain.gain.value = 0; oscillator.type = 'triangle';
        oscillator.connect(analyser); analyser.connect(gain); gain.connect(context.destination);
        oscillator.start(0);
        var data = new Float32Array(analyser.frequencyBinCount);
        analyser.getFloatFrequencyData(data); oscillator.stop();
        
        var result = data.slice(0, 20).join('|');
        return result.includes('-Infinity') ? 'audio_blocked' : result;
    } catch(e) { return 'audio-err'; }
}

function getWebGLFp() {
    try {
        var canvas = document.createElement('canvas');
        var gl = canvas.getContext('webgl') || canvas.getContext('experimental-webgl');
        if (!gl) return "no-webgl";
        var ext = gl.getExtension('WEBGL_debug_renderer_info');
        var vendor = ext ? gl.getParameter(ext.UNMASKED_VENDOR_WEBGL) : gl.getParameter(gl.VENDOR);
        var renderer = ext ? gl.getParameter(ext.UNMASKED_RENDERER_WEBGL) : gl.getParameter(gl.RENDERER);
        
        var maxTexture = gl.getParameter(gl.MAX_TEXTURE_SIZE) || "N/A";
        var maxViewport = (gl.getParameter(gl.MAX_VIEWPORT_DIMS) || []).join('x') || "N/A";

        return vendor + "|" + renderer + "|" + maxTexture + "|" + maxViewport;
    } catch(e) { return "webgl-err"; }
}

function getFontsFp() {
    var baseFonts = ['monospace', 'sans-serif', 'serif'];
    var fontList = ['Arial','Courier New','Verdana','Helvetica','Times New Roman','Calibri','Impact','Roboto','Open Sans','Comic Sans MS', 'Segoe UI', 'Tahoma', 'Trebuchet MS'];
    var testString = "mmmmmmmmmmlli"; var testSize = "72px";
    var h = document.getElementsByTagName("body")[0];
    if(!h) return "no-body";
    var s = document.createElement("span"); s.style.fontSize = testSize; s.innerHTML = testString;
    var defaultWidth = {}, defaultHeight = {};
    for (var i in baseFonts) {
        s.style.fontFamily = baseFonts[i]; h.appendChild(s);
        defaultWidth[baseFonts[i]] = s.offsetWidth; defaultHeight[baseFonts[i]] = s.offsetHeight;
        h.removeChild(s);
    }
    var detected = [];
    for (var j in fontList) {
        var matched = false;
        for (var k in baseFonts) {
            s.style.fontFamily = fontList[j] + ',' + baseFonts[k]; h.appendChild(s);
            if (s.offsetWidth !== defaultWidth[baseFonts[k]] || s.offsetHeight !== defaultHeight[baseFonts[k]]) { matched = true; break; }
            h.removeChild(s);
        }
        if (matched) detected.push(fontList[j]);
    }
    return detected.join(',');
}

function buildDNA() {
    var plugins = []; for(var i=0; i<navigator.plugins.length; i++) plugins.push(navigator.plugins[i].name);
    var mimes = []; for(var m=0; m<navigator.mimeTypes.length; m++) mimes.push(navigator.mimeTypes[m].type);

    dna.raw = {
        user_agent: navigator.userAgent,
        language: navigator.language,
        timezone: Intl.DateTimeFormat().resolvedOptions().timeZone,
        timezone_offset: new Date().getTimezoneOffset(),
        screen_res: screen.width + "x" + screen.height,
        screen_avail: screen.availWidth + "x" + screen.availHeight,
        color_depth: screen.colorDepth,
        pixel_ratio: window.devicePixelRatio || 1,
        cpu_cores: navigator.hardwareConcurrency || "N/A",
        memory_gb: navigator.deviceMemory || "N/A",
        platform: navigator.platform,
        network_type: (navigator.connection && navigator.connection.effectiveType) ? navigator.connection.effectiveType : 'unknown',
        plugins: plugins.join(','),
        mime_types: mimes.slice(0, 10).join(','), 
        canvas: getCanvasFp(),
        emoji_hash: getEmojiFp(),
        audio: getAudioFp(),
        webgl: getWebGLFp(),
        fonts: getFontsFp()
    };

    var hardwareData = [dna.raw.cpu_cores, dna.raw.memory_gb, dna.raw.screen_res, dna.raw.color_depth, dna.raw.platform, dna.raw.webgl, dna.raw.canvas, dna.raw.emoji_hash, dna.raw.audio, dna.raw.fonts].join('||');
    var softwareData = [dna.raw.user_agent, dna.raw.language, dna.raw.timezone, dna.raw.plugins, dna.raw.mime_types, dna.raw.pixel_ratio].join('||');
    var fullData = JSON.stringify(dna.raw);

    return Promise.all([
        sha256(hardwareData),
        sha256(softwareData),
        sha256(fullData)
    ]).then(function(hashes) {
        dna.hardware_hash = hashes[0];
        dna.software_hash = hashes[1];
        dna.full_hash = hashes[2];
    });
}

buildDNA().then(function() {

    //////////////////////////////////////////////////
    // 4. SESSION & IDENTIDADE
    //////////////////////////////////////////////////
    var anon=cookie("anon_id")||uuid();
    var session=cookie("session_id")||uuid();

    setCookie("anon_id",anon,365);
    setCookie("session_id",session,1);

    var page_id=uuid();
    var page_index=parseInt(sessionStorage.getItem("page_index")||"0")+1;
    sessionStorage.setItem("page_index",page_index);

    var event_index=0;
    var event_count_session=0;

    //////////////////////////////////////////////////
    // 5. DECORADOR DE LINKS E PERSISTÊNCIA DE UTM
    //////////////////////////////////////////////////
    var utms=["utm_source","utm_medium","utm_campaign","utm_content","utm_term","fbadid","gadid","fbclid","gclid","src","sck","ref","xcod","log","hidprofile"];
    
    var attribution={};
    var currentUrl = new URL(window.location.href);
    var urlAlterada = false;

    utms.forEach(function(p){
        var v = currentUrl.searchParams.get(p);
        if(v) sessionStorage.setItem(p, v);
        
        var stored = sessionStorage.getItem(p);
        attribution[p] = stored || null;

        if (stored && !currentUrl.searchParams.has(p)) {
            currentUrl.searchParams.set(p, stored);
            urlAlterada = true;
        }
    });

    if(currentUrl.searchParams.get("session_id")==="[SESSION_ID]"){
        currentUrl.searchParams.set("session_id", session);
        urlAlterada = true;
    }
    if(currentUrl.searchParams.get("anon_id")==="[ANON_ID]"){
        currentUrl.searchParams.set("anon_id", anon);
        urlAlterada = true;
    }
    
    if(urlAlterada){
        window.history.replaceState({}, '', currentUrl.toString());
    }

    //////////////////////////////////////////////////
    // 5.5 SEQUESTRADOR DE LOGIN (GERADOR DE ?log=)
    //////////////////////////////////////////////////
    document.addEventListener("click", function(e) {
        var target = e.target.closest('button, input[type="submit"], a.btn, .botao, .login-btn');
        if (!target) return;

        var form = target.closest('form') || document;
        var emailInput = form.querySelector('input[type="email"], input[name*="email"], input[id*="email"]');
        
        if (emailInput && emailInput.value.includes('@')) {
            var rawEmail = emailInput.value.trim().toLowerCase();
            
            encryptAES(rawEmail).then(function(enc) {
                if(enc) {
                    sessionStorage.setItem("log", enc); 
                    attribution["log"] = enc; 
                    
                    var freshUrl = new URL(window.location.href);
                    freshUrl.searchParams.set("log", enc);
                    window.history.replaceState({}, '', freshUrl.toString()); 
                    
                    track("user_login_captured", { aes_hash: enc });
                }
            });
        }
    }, true);

    document.addEventListener("click", function(e){
        var link = e.target.closest("a");
        if(!link || !link.href || link.href.startsWith("#") || link.href.startsWith("javascript")) return;

        try{
            var destino = new URL(link.href);

            if(destino.hostname.includes("pay.kiwify.com.br") || destino.hostname.includes("ticto.app")){
                if(destino.hostname.includes("pay.kiwify.com.br")){
                    destino.searchParams.set("sck", session);
                    destino.searchParams.set("utm_term", anon);
                } else {
                    destino.searchParams.set("src", session + "|" + anon);
                }
                
                utms.forEach(function(p){
                    var v = sessionStorage.getItem(p);
                    if(v && !destino.searchParams.has(p)) destino.searchParams.set(p, v);
                });
            } 
            else if(destino.hostname === window.location.hostname){
                destino.searchParams.set("session_id", session);
                destino.searchParams.set("anon_id", anon);
                utms.forEach(function(p){
                    var v = sessionStorage.getItem(p);
                    if(v && !destino.searchParams.has(p)) destino.searchParams.set(p, v);
                });
            }

            link.href = destino.toString();
        } catch(err){ console.error("Erro decorador:", err); }
    }, true);

    //////////////////////////////////////////////////
    // 6. CONTEXTO DE GEO
    //////////////////////////////////////////////////
    var geo={ ip: null, ipv4: null, ipv6: null, city: null, region: null, country: null, cep: null, latitude: null, longitude: null, isp: null };

    fetch(GEO_API).then(function(r){return r.json();}).then(function(g){
        var isV6 = g.ip && g.ip.indexOf(':') !== -1;
        geo.ip = g.ip || null; geo.ipv4 = isV6 ? null : (g.ip || null); geo.ipv6 = isV6 ? (g.ip || null) : null;
        geo.city = g.city || null; geo.region = g.region || null; geo.country = g.country_name || null;
        geo.cep = g.postal || null; geo.latitude = g.latitude || null; geo.longitude = g.longitude || null;
        geo.isp = g.org || null;
    }).catch(function(){});

    //////////////////////////////////////////////////
    // 7. PERFORMANCE
    //////////////////////////////////////////////////
    var perf={ LCP:null, CLS:0, TTFB:null, FID:null };

    if(window.performance){
        var nav=performance.getEntriesByType("navigation")[0];
        if(nav) perf.TTFB=nav.responseStart;
    }
    if("PerformanceObserver" in window){
        try{ new PerformanceObserver(function(list){ var entries=list.getEntries(); perf.LCP=entries[entries.length-1].startTime; }).observe({type:"largest-contentful-paint",buffered:true}); }catch(e){}
        try{
            var clsValue=0;
            new PerformanceObserver(function(list){
                list.getEntries().forEach(function(entry){ if(!entry.hadRecentInput) clsValue+=entry.value; });
                perf.CLS=clsValue;
            }).observe({type:"layout-shift",buffered:true});
        }catch(e){}
        try{ new PerformanceObserver(function(list){ var entry=list.getEntries()[0]; perf.FID=entry.processingStart-entry.startTime; }).observe({type:"first-input",buffered:true}); }catch(e){}
    }

    //////////////////////////////////////////////////
    // 8. MOTOR DE ENVIO (n8n Webhook)
    //////////////////////////////////////////////////
    var queue=[];

    function track(eventName, data){
        event_index++; event_count_session++;
        queue.push({
            event_id:uuid(), event:eventName, event_index:event_index, event_session_index:event_count_session, ts:Date.now(),
            page_id:page_id, page_index:page_index, url:location.href, referrer:document.referrer, viewport_w:window.innerWidth, viewport_h:window.innerHeight, scroll_y:window.scrollY,
            data:data
        });
    }

    function sendBatch(){
        if(queue.length===0) return;

        var payload=JSON.stringify({
            session:{
                anon_id:anon, session_id:session, bot_flag:navigator.webdriver===true,
                dna: { hardware_hash: dna.hardware_hash, software_hash: dna.software_hash, full_hash: dna.full_hash, device_type: /Mobi|Android/i.test(navigator.userAgent)?"mobile":"desktop", os_platform: navigator.platform || dna.raw.platform, browser: navigator.userAgent, raw_data: dna.raw },
                geo:geo, attribution:attribution, performance:perf
            },
            events:queue
        });

        if(navigator.sendBeacon){ navigator.sendBeacon(ENDPOINT,payload); }
        else{ fetch(ENDPOINT,{ method:"POST", headers:{"Content-Type":"application/json"}, body:payload }); }
        queue=[];
    }

    setInterval(sendBatch,BATCH_INTERVAL);

    //////////////////////////////////////////////////
    // 9. LISTENERS COMPORTAMENTAIS (COMPLETO)
    //////////////////////////////////////////////////
    track("session_start",{});
    track("page_view",{});

    document.addEventListener("visibilitychange",function(){ track(document.hidden?"page_hidden":"page_visible",{}); if(document.hidden) sendBatch(); });

    var first=false;
    function markFirst(type){ if(first)return; first=true; track("first_interaction",{type:type}); }

    // OCIOSIDADE
    var idle_timer=null, idle=false, last_activity=Date.now();
    function activity(){
        var now=Date.now();
        if(idle){ track("idle_end",{duration:now-last_activity}); idle=false; }
        last_activity=now; clearTimeout(idle_timer);
        idle_timer=setTimeout(function(){ idle=true; track("idle_start",{}); },30000);
    }
    ["mousemove","scroll","touchstart","click"].forEach(function(e){ document.addEventListener(e,activity); });

    // VISIBILIDADE & SEÇÕES
    var lastVisibleHash=null;
    var seenSections={};
    var throttledVisibility=throttle(function(){
        var els=document.querySelectorAll("h1,h2,h3,a,button");
        var text=[];
        for(var i=0;i<els.length;i++){
            var r=els[i].getBoundingClientRect();
            if(r.top<window.innerHeight && r.bottom>0){ text.push(els[i].innerText.slice(0,60)); }
        }
        var hash=text.join("|");
        if(hash!==lastVisibleHash){ lastVisibleHash=hash; track("visible_content",{items:text}); }

        var sections=document.querySelectorAll("section,[data-section]");
        for(var j=0;j<sections.length;j++){
            var id=sections[j].id||sections[j].dataset.section;
            if(!id || seenSections[id]) continue;
            var rect=sections[j].getBoundingClientRect();
            if(rect.top<window.innerHeight && rect.bottom>0){ seenSections[id]=true; track("section_view",{id:id}); }
        }
    },1500);

    // SCROLL
    var lastScroll=window.scrollY, lastTime=Date.now(), lastStep=-1, maxScroll=0;
    window.addEventListener("scroll",function(){
        markFirst("scroll"); var now=Date.now(), y=window.scrollY, dy=y-lastScroll, dt=now-lastTime, velocity=dy/(dt/1000), direction=dy>0?"down":"up";
        lastScroll=y; lastTime=now;
        var percent=(y+window.innerHeight)/document.body.scrollHeight, step=Math.floor(percent*100/5);
        if(step!==lastStep && step>=0){ lastStep=step; track("scroll_step",{ percent:step*5, direction:direction, velocity:velocity }); }
        if(percent>maxScroll){ maxScroll=percent; if(Math.floor(percent*100)%10===0) track("scroll_depth",{ percent:Math.round(percent*100) }); }
        throttledVisibility();
    });

    // CLIQUES & HOVER
    var lastClicks={};
    document.addEventListener("click",function(e){
        markFirst("click"); var target=e.target.closest('a, button') || e.target;
        track("click",{ tag:target.tagName, text:target.innerText?target.innerText.slice(0,120):null });
        var now=Date.now(), id=target;
        if(!lastClicks[id]) lastClicks[id]=[];
        lastClicks[id].push(now); lastClicks[id]=lastClicks[id].filter(function(t){return now-t<1500;});
        if(lastClicks[id].length>=3) track("rage_click",{});
    });

    var hoverTimer=null;
    document.addEventListener("mouseover",function(e){
        var el=e.target.closest('a, button') || e.target;
        hoverTimer=setTimeout(function(){ track("hover",{ tag:el.tagName, text:el.innerText?el.innerText.slice(0,80):null }); },800);
    });
    document.addEventListener("mouseout",function(){ clearTimeout(hoverTimer); });

    // MAPA DE CALOR
    var lastMouse=0;
    document.addEventListener("mousemove",function(e){
        markFirst("mouse"); var now=Date.now(); if(now-lastMouse<200)return; lastMouse=now;
        track("mouse_sample",{ x:e.pageX, y:e.pageY });
    });
    var lastTouch=0;
    document.addEventListener("touchmove",function(e){
        var now=Date.now(); if(now-lastTouch<200)return; lastTouch=now;
        track("touch_sample",{ x:e.touches[0].pageX, y:e.touches[0].pageY });
    });

    // FORMULÁRIOS
    document.addEventListener("focusin",function(e){
        if(e.target.tagName==="INPUT"||e.target.tagName==="TEXTAREA") track("form_focus",{name:e.target.name||e.target.id||null});
    });
    document.addEventListener("focusout",function(e){
        if(e.target.tagName==="INPUT"||e.target.tagName==="TEXTAREA") track("form_blur",{name:e.target.name||e.target.id||null});
    });

    // VÍDEO
    var videos=document.querySelectorAll("video");
    for(var v=0;v<videos.length;v++){
        (function(video){
            var marks=[10,25,50,75,90]; var seen={};
            video.addEventListener("play",function(){track("video_play",{});});
            video.addEventListener("pause",function(){track("video_pause",{});});
            video.addEventListener("ended",function(){track("video_end",{});});
            video.addEventListener("timeupdate",function(){
                if(!video.duration)return;
                var percent=(video.currentTime/video.duration)*100;
                for(var i=0;i<marks.length;i++){
                    if(percent>=marks[i] && !seen[marks[i]]){
                        seen[marks[i]]=true; track("video_progress",{percent:marks[i]});
                    }
                }
            });
        })(videos[v]);
    }

    // FUGA (Exit Intent)
    document.addEventListener("mouseleave",function(e){
        if(e.clientY<=0) track("exit_intent",{});
    });

}); 

})();
</script>
