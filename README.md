<!DOCTYPE html>
<html lang="it">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Widget Meteo Realtime</title>
  <style>
    :root{
      --bg: #0f1115;
      --card: rgba(255,255,255,0.06);
      --text: #e6e8eb;
      --muted:#a6adbb;
      --accent:#7dd3fc;
      --ok:#86efac;
      --warn:#fbbf24;
    }
    *{box-sizing:border-box}
    html,body{height:100%}
    body{
      margin:0; font: 14px/1.4 system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, Noto Sans, Arial, "Apple Color Emoji","Segoe UI Emoji";
      background: radial-gradient(1200px 800px at 80% -10%, rgba(125,211,252,.15), transparent 60%),
                  radial-gradient(1000px 800px at -20% 120%, rgba(134,239,172,.08), transparent 50%),
                  var(--bg);
      color:var(--text);
      display:flex; align-items:center; justify-content:center; padding:12px;
    }
    .card{
      width:min(680px, 100%);
      background:var(--card);
      border:1px solid rgba(255,255,255,0.08);
      box-shadow: 0 10px 30px rgba(0,0,0,.35), inset 0 1px 0 rgba(255,255,255,.04);
      border-radius:18px; padding:16px; backdrop-filter: blur(12px);
    }
    .row{display:flex; gap:14px; align-items:center; justify-content:space-between;}
    .row.wrap{flex-wrap:wrap}
    .city{font-weight:600; letter-spacing:.3px}
    .updated{color:var(--muted); font-size:12px}
    .current{
      display:flex; align-items:center; gap:14px; margin:8px 0 14px 0;
    }
    .big{
      font-size:40px; font-weight:700; letter-spacing:-.5px;
    }
    .chip{padding:6px 10px; border:1px solid rgba(255,255,255,.12); border-radius:999px; color:var(--muted)}
    .grid{
      display:grid; grid-template-columns: repeat(3,1fr); gap:10px; margin:10px 0 6px;
    }
    .cell{background:rgba(255,255,255,.04); border:1px solid rgba(255,255,255,.08); border-radius:12px; padding:10px}
    .cell b{display:block; font-size:13px; color:var(--muted); font-weight:600; margin-bottom:4px}
    .hrs{display:flex; gap:10px; overflow:auto; padding-bottom:8px;}
    .hr{min-width:70px; text-align:center; border:1px solid rgba(255,255,255,.08); background:rgba(255,255,255,.04); border-radius:12px; padding:8px}
    .hr small{display:block; color:var(--muted)}
    .flex{display:flex; align-items:center; gap:6px}
    .ok{color:var(--ok)} .warn{color:var(--warn)}
    .wicon{font-size:26px}
    .small{font-size:12px; color:var(--muted)}
    .spacer{flex:1}
    .btn{appearance:none; background:transparent; color:var(--muted); border:1px solid rgba(255,255,255,.15); border-radius:10px; padding:6px 10px; cursor:pointer}
  </style>
</head>
<body>
  <div class="card" id="card">
    <div class="row">
      <div class="city" id="city">Località…</div>
      <div class="spacer"></div>
      <button class="btn" id="refresh">Aggiorna</button>
    </div>

    <div class="current">
      <div class="wicon" id="icon">⛅</div>
      <div>
        <div class="big"><span id="temp">--</span>°C</div>
        <div class="small" id="desc">—</div>
      </div>
      <div class="spacer"></div>
      <div class="chip" id="feels">Percepita: --°C</div>
    </div>

    <div class="grid">
      <div class="cell"><b>Vento</b><div class="flex"><span id="wind">-- km/h</span><span id="winddir" class="small"></span></div></div>
      <div class="cell"><b>Umidità</b><div id="hum">--%</div></div>
      <div class="cell"><b>Pioggia (1h)</b><div id="rain">-- mm</div></div>
    </div>

    <div class="row wrap" style="margin-top:8px;">
      <div class="updated" id="updated">Aggiornamento…</div>
      <div class="spacer"></div>
      <div class="small">Fonte: <a href="https://open-meteo.com/" target="_blank" rel="noreferrer" style="color:var(--accent); text-decoration:none;">Open‑Meteo</a> • realtime</div>
    </div>

    <div class="small" style="margin-top:10px; margin-bottom:6px">Prossime ore</div>
    <div class="hrs" id="hours"></div>
  </div>

  <script>
    // ——— utils
    const $ = (id)=>document.getElementById(id);
    const WMAP = {
      0:["Sereno","☀️"], 1:["Perlopiù sereno","🌤️"], 2:["Parzialmente nuvoloso","⛅"], 3:["Nuvoloso","☁️"],
      45:["Nebbia","🌫️"], 48:["Brina","🌫️"], 51:["Pioviggine leggera","🌦️"], 53:["Pioviggine","🌦️"], 55:["Pioviggine intensa","🌧️"],
      56:["Pioggia gelata leggera","🌧️"], 57:["Pioggia gelata","🌧️"], 61:["Pioggia debole","🌧️"], 63:["Pioggia","🌧️"], 65:["Pioggia forte","🌧️"],
      66:["Rovesci gelati","🌧️"], 67:["Rovesci gelati forti","🌧️"], 71:["Neve debole","🌨️"], 73:["Neve","🌨️"], 75:["Neve forte","❄️"],
      77:["Granuli di neve","❄️"], 80:["Rovesci","🌧️"], 81:["Rovesci forti","🌧️"], 82:["Rovesci violenti","🌧️"], 95:["Temporali","⛈️"], 96:["Temporali con grandine","⛈️"], 99:["Temporali con grandine forte","⛈️"]
    };

    function degToArrow(d){
      if(d==null) return ""; const dirs = ["N","NE","E","SE","S","SO","O","NO"]; return dirs[Math.round(d/45)%8] + ` (${Math.round(d)}°)`;
    }
    function fmtTime(iso){ const d=new Date(iso); return d.toLocaleTimeString([], {hour:'2-digit', minute:'2-digit'}); }

    async function getCityFromCoords(lat, lon){
      try{
        const r = await fetch(`https://nominatim.openstreetmap.org/reverse?format=jsonv2&lat=${lat}&lon=${lon}`);
        const j = await r.json();
        return j.address?.city || j.address?.town || j.address?.village || j.display_name?.split(',')[0] || 'Località';
      }catch(e){ return 'Località'; }
    }

    async function fetchWeather(lat, lon){
      const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,relative_humidity_2m,apparent_temperature,precipitation,rain,cloud_cover,wind_speed_10m,wind_direction_10m,weather_code&hourly=temperature_2m,weather_code,precipitation&timezone=auto`;
      const res = await fetch(url);
      if(!res.ok) throw new Error('Errore dati meteo');
      return res.json();
    }

    function paint(data){
      const c = data.current;
      const [desc, icon] = WMAP[c.weather_code] || ["Condizioni", "⛅"];
      $('icon').textContent = icon; $('desc').textContent = desc;
      $('temp').textContent = Math.round(c.temperature_2m);
      $('feels').textContent = `Percepita: ${Math.round(c.apparent_temperature)}°C`;
      $('wind').textContent = `${Math.round(c.wind_speed_10m)} km/h`;
      $('winddir').textContent = degToArrow(c.wind_direction_10m);
      $('hum').textContent = `${Math.round(c.relative_humidity_2m)}%`;
      $('rain').textContent = `${(c.rain ?? 0).toFixed(1)} mm`;
      $('updated').textContent = `Aggiornato alle ${fmtTime(new Date())}`;

      // hours (next 8)
      const hrs = $('hours');
      hrs.innerHTML = '';
      const now = new Date();
      const times = data.hourly.time;
      const idx = times.findIndex(t => new Date(t) > now);
      for(let i=idx; i<Math.min(idx+8, times.length); i++){
        const t = times[i];
        const code = data.hourly.weather_code[i];
        const [dsc, ic] = WMAP[code] || ["", "⛅"];
        const temp = Math.round(data.hourly.temperature_2m[i]);
        const pr = data.hourly.precipitation?.[i] ?? 0;
        const el = document.createElement('div');
        el.className = 'hr';
        el.innerHTML = `<small>${fmtTime(t)}</small><div style="font-size:20px">${ic}</div><div>${temp}°C</div><small>${pr} mm</small>`;
        hrs.appendChild(el);
      }
    }

    async function load(lat, lon){
      try{
        const [data, city] = await Promise.all([fetchWeather(lat,lon), getCityFromCoords(lat,lon)]);
        $('city').textContent = city;
        paint(data);
      }catch(e){
        $('city').textContent = 'Errore meteo';
        $('desc').textContent = e.message || String(e);
      }
    }

    function fallback(){ // Trieste centro
      load(45.6495, 13.7768);
    }

    // init
    (function(){
      const qp = new URLSearchParams(location.search);
      const lat = parseFloat(qp.get('lat')); const lon = parseFloat(qp.get('lon'));
      if(!Number.isNaN(lat) && !Number.isNaN(lon)){
        load(lat,lon);
      } else if (navigator.geolocation){
        navigator.geolocation.getCurrentPosition(pos => {
          load(pos.coords.latitude, pos.coords.longitude);
        }, fallback, {enableHighAccuracy:true, maximumAge:60000, timeout:6000});
      } else { fallback(); }

      // manual refresh + auto refresh each 5 min
      $('refresh').addEventListener('click', () => {
        if(window.__lastLat && window.__lastLon) load(window.__lastLat, window.__lastLon); else fallback();
      });
      setInterval(()=>{
        if(window.__lastLat && window.__lastLon) load(window.__lastLat, window.__lastLon); else fallback();
      }, 5*60*1000);

      // store last coords when load is called
      const _load = load; load = function(a,b){ window.__lastLat=a; window.__lastLon=b; return _load(a,b); }
    })();
  </script>
</body>
</html>
