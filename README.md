<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Crypto Dashboard — Juan Campos</title>
<link href="https://fonts.googleapis.com/css2?family=DM+Serif+Display:ital@0;1&family=DM+Sans:wght@300;400;500;600;700&display=swap" rel="stylesheet">

<!-- Firebase -->
<script type="module">
  import { initializeApp }   from "https://www.gstatic.com/firebasejs/10.12.2/firebase-app.js";
  import { getDatabase, ref, set, onValue } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-database.js";
  import { getAuth, signInAnonymously, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/10.12.2/firebase-auth.js";

  /* ▼▼▼  PEGA AQUÍ TU CONFIGURACIÓN DE FIREBASE  ▼▼▼ */
  const firebaseConfig = {
    apiKey:            "AIzaSyDdUBB4yMHhUL_TMuc0qx_WweHON3TCK8I",
    authDomain:        "dashboard-cripto-53ea6.firebaseapp.com",
    databaseURL:       "https://dashboard-cripto-53ea6-default-rtdb.europe-west1.firebasedatabase.app",
    projectId:         "dashboard-cripto-53ea6",
    storageBucket:     "dashboard-cripto-53ea6.firebasestorage.app",
    messagingSenderId: "310927364843",
    appId:             "1:310927364843:web:291c2c24212acc323da4ae"
  };
  /* ▲▲▲  FIN  ▲▲▲ */

  let db, uid;
  window._DB = {};

  try {
    const app  = initializeApp(firebaseConfig);
    db         = getDatabase(app);
    const auth = getAuth(app);
    signInAnonymously(auth).catch(e => console.warn(e));

    onAuthStateChanged(auth, user => {
      if (!user) return;
      uid = user.uid;
      window._fbOK = true;

      set(ref(db, 'sessions/' + Date.now()), { uid, ts: new Date().toISOString() });

      onValue(ref(db, 'sessions'), s => {
        const d  = s.val();
        const el = document.getElementById('sc');
        if (el && d) el.textContent = Object.keys(d).length + ' sesiones';
      });

      // Notas de análisis por gráfico
      onValue(ref(db, 'tv_layouts'), s => {
        const d = s.val();
        if (!d) return;
        window._DB = d;
        const dates = Object.values(d).map(v => v.savedAt).filter(Boolean).sort();
        const last  = dates[dates.length - 1];
        const el    = document.getElementById('ls');
        if (el && last) el.textContent = 'Últ. guardado: ' + new Date(last).toLocaleString('es-ES');
        Object.keys(d).forEach(w => { if (d[w] && d[w].savedAt) _applyState(w, d[w]); });
      });

      // Datos ETF guardados manualmente
      onValue(ref(db, 'etf_data'), s => {
        const d = s.val();
        if (d) {
          window._etfData = d;
          renderETFFromFirebase(d);
        }
      });
    });
  } catch(e) {
    console.warn('Firebase no configurado:', e.message);
    window._fbOK = false;
  }

  window.FB_save = async (wid, payload) => {
    if (!db) return false;
    try {
      await set(ref(db, 'tv_layouts/' + wid), { ...payload, savedBy: uid, savedAt: new Date().toISOString() });
      return true;
    } catch(e) { return false; }
  };

  window.FB_saveETF = async (payload) => {
    if (!db) return false;
    try {
      await set(ref(db, 'etf_data'), { ...payload, updatedBy: uid, updatedAt: new Date().toISOString() });
      return true;
    } catch(e) { return false; }
  };

  window.FB_get = wid => window._DB[wid] || null;
</script>

<style>
*,*::before,*::after { box-sizing: border-box; margin: 0; padding: 0; }
:root {
  --bg:#050c1a; --bg-card:#0f1f35; --bg-el:#162a42; --bg-side:#080f1e;
  --b0:rgba(56,139,253,.1); --b1:rgba(56,139,253,.2); --b2:rgba(56,139,253,.42);
  --cyan:#00d4ff;   --cD:rgba(0,212,255,.1);
  --gold:#f0c040;   --gD:rgba(240,192,64,.1);
  --green:#00e5a0;  --grD:rgba(0,229,160,.1);
  --red:#ff4d6d;    --rD:rgba(255,77,109,.08);
  --purple:#a78bfa;
  --t1:#e8f0fe; --t2:#8ba4c8; --t3:#4a6080;
  --fd:'DM Serif Display',Georgia,serif;
  --fb:'DM Sans',system-ui,sans-serif;
  --r:16px; --rs:6px; --rm:10px;
}
html { scroll-behavior: smooth; }
body { background:var(--bg); color:var(--t1); font-family:var(--fb); font-size:14px; line-height:1.6; min-height:100vh; overflow-x:hidden; display:flex; flex-direction:column; }
body::before { content:''; position:fixed; inset:0; background-image:linear-gradient(rgba(56,139,253,.03) 1px,transparent 1px),linear-gradient(90deg,rgba(56,139,253,.03) 1px,transparent 1px); background-size:48px 48px; pointer-events:none; z-index:0; }

/* APP LAYOUT */
.app { display:flex; flex:1; position:relative; z-index:1; }

/* SIDEBAR */
.sidebar { width:62px; flex-shrink:0; background:var(--bg-side); border-right:1px solid var(--b0); display:flex; flex-direction:column; align-items:center; padding:14px 0; gap:4px; position:sticky; top:0; height:100vh; transition:width .25s ease; z-index:100; overflow:hidden; }
.sidebar.expanded { width:176px; }
.slogo { width:34px; height:34px; background:linear-gradient(135deg,var(--cyan),#0070e0); border-radius:var(--rm); display:flex; align-items:center; justify-content:center; font-size:17px; font-weight:700; color:#000; box-shadow:0 0 16px rgba(0,212,255,.3); margin-bottom:10px; cursor:pointer; flex-shrink:0; }
.ni { width:calc(100% - 10px); display:flex; align-items:center; gap:9px; padding:9px 11px; border-radius:var(--rm); cursor:pointer; border:1px solid transparent; white-space:nowrap; overflow:hidden; user-select:none; transition:background .2s,border-color .2s; }
.ni:hover { background:rgba(0,212,255,.06); border-color:var(--b0); }
.ni.active { background:rgba(0,212,255,.1); border-color:var(--b1); }
.ni.active .ni-icon { color:var(--cyan); }
.ni.active .ni-lbl { color:var(--t1); }
.ni-icon { font-size:17px; flex-shrink:0; width:20px; text-align:center; }
.ni-lbl { font-size:13px; font-weight:600; color:var(--t2); opacity:0; transition:opacity .2s; }
.sidebar.expanded .ni-lbl { opacity:1; }
.ndiv { width:calc(100% - 20px); height:1px; background:var(--b0); margin:4px 0; }
.stoggle { margin-top:auto; width:calc(100% - 10px); display:flex; align-items:center; justify-content:center; gap:7px; padding:7px; border-radius:var(--rm); cursor:pointer; border:1px solid var(--b0); color:var(--t3); font-size:11px; transition:background .2s,color .2s; white-space:nowrap; overflow:hidden; }
.stoggle:hover { background:rgba(255,255,255,.04); color:var(--t2); }
.sidebar.expanded .stoggle { justify-content:flex-start; padding:7px 11px; }
.st-icon { flex-shrink:0; font-size:13px; }
.st-lbl { opacity:0; transition:opacity .2s; }
.sidebar.expanded .st-lbl { opacity:1; }

/* MAIN */
.main { flex:1; min-width:0; display:flex; flex-direction:column; }

/* HEADER */
.header { padding:16px 22px 0; display:flex; align-items:center; justify-content:space-between; flex-wrap:wrap; gap:10px; animation:fadeUp .5s ease both; }
.htitle h1 { font-family:var(--fd); font-size:19px; letter-spacing:-.3px; line-height:1.2; }
.htitle p { font-size:11px; color:var(--t3); letter-spacing:.07em; text-transform:uppercase; font-weight:500; margin-top:1px; }
.hmeta { display:flex; align-items:center; gap:10px; flex-wrap:wrap; }
.spill { display:flex; align-items:center; gap:5px; background:var(--bg-card); border:1px solid var(--b0); padding:5px 10px; border-radius:100px; font-size:12px; color:var(--t2); }
.sdot { width:6px; height:6px; border-radius:50%; background:var(--green); box-shadow:0 0 7px var(--green); animation:pdot 2s ease-in-out infinite; }
@keyframes pdot { 0%,100%{opacity:1} 50%{opacity:.3} }
#sc,#ls { font-size:11px; color:var(--t3); }

/* TAB PANELS */
.tab-panel { display:none; flex-direction:column; flex:1; padding:0 22px 22px; animation:fadeUp .4s ease both; }
.tab-panel.active { display:flex; }

/* METRICS */
.metrics { display:grid; grid-template-columns:repeat(auto-fit,minmax(128px,1fr)); gap:10px; padding:14px 22px 0; animation:fadeUp .5s ease both; }
.mc { background:var(--bg-card); border:1px solid var(--b0); border-radius:var(--r); padding:12px 14px; transition:border-color .2s,transform .2s; cursor:default; }
.mc:hover { border-color:var(--b2); transform:translateY(-2px); }
.ml { font-size:10px; font-weight:600; letter-spacing:.1em; text-transform:uppercase; color:var(--t3); margin-bottom:4px; }
.mv { font-family:var(--fd); font-size:20px; line-height:1; margin-bottom:3px; }
.mv.c{color:var(--cyan)}.mv.g{color:var(--gold)}.mv.gr{color:var(--green)}.mv.p{color:var(--purple)}.mv.r{color:var(--red)}
.mb { display:inline-flex; align-items:center; font-size:10px; font-weight:600; padding:1px 6px; border-radius:100px; background:rgba(0,229,160,.1); color:var(--green); }

/* SECTION */
.section { padding:0 0 14px; }
.stitle { display:flex; align-items:center; gap:10px; margin-bottom:11px; padding-top:16px; }
.stitle h2 { font-family:var(--fd); font-size:14px; color:var(--t2); font-weight:400; font-style:italic; white-space:nowrap; }
.sline { flex:1; height:1px; background:var(--b0); }

/* GRIDS */
.g2 { display:grid; grid-template-columns:1fr 1fr; gap:12px; }
.g3 { display:grid; grid-template-columns:repeat(3,1fr); gap:12px; }
@media(max-width:1100px){.g3{grid-template-columns:1fr 1fr}}
@media(max-width:700px){.g2,.g3{grid-template-columns:1fr}}

/* CHART CARD */
.cc { background:var(--bg-card); border:1px solid var(--b0); border-radius:var(--r); overflow:hidden; display:flex; flex-direction:column; transition:border-color .2s,box-shadow .2s; box-shadow:0 4px 18px rgba(0,0,0,.35); animation:fadeUp .45s ease both; }
.cc:hover { border-color:var(--b1); box-shadow:0 0 26px rgba(0,212,255,.06); }
.cch { display:flex; align-items:center; justify-content:space-between; padding:10px 13px 9px; border-bottom:1px solid var(--b0); flex-shrink:0; gap:8px; }
.cct { display:flex; align-items:center; gap:7px; min-width:0; flex:1; }
.tick { font-size:13px; font-weight:700; letter-spacing:.03em; white-space:nowrap; }
.tdesc { font-size:11px; color:var(--t3); white-space:nowrap; overflow:hidden; text-overflow:ellipsis; }
.tag { font-size:10px; font-weight:600; letter-spacing:.06em; text-transform:uppercase; padding:2px 8px; border-radius:100px; border:1px solid; white-space:nowrap; flex-shrink:0; }
.tag.c {color:var(--cyan); border-color:rgba(0,212,255,.3); background:var(--cD)}
.tag.g {color:var(--gold); border-color:rgba(240,192,64,.3); background:var(--gD)}
.tag.gr{color:var(--green);border-color:rgba(0,229,160,.3); background:var(--grD)}
.tag.p {color:var(--purple);border-color:rgba(167,139,250,.3);background:rgba(167,139,250,.1)}
.tag.r {color:var(--red);  border-color:rgba(255,77,109,.3); background:var(--rD)}

/* CHART AREA */
.ca { position:relative; flex:1; background:#080f1e; overflow:hidden; }
.ca-ov { position:absolute; inset:0; display:flex; flex-direction:column; align-items:center; justify-content:center; gap:9px; background:rgba(5,12,26,0); transition:background .25s; pointer-events:none; }
.ca:hover .ca-ov { background:rgba(5,12,26,.6); pointer-events:auto; }
.ca:hover .otv { opacity:1; transform:translateY(0); }
.ca:hover .ca-hint { opacity:1; }
.otv { display:flex; align-items:center; gap:7px; background:linear-gradient(135deg,var(--cyan),#0060d0); color:#000; font-family:var(--fb); font-size:12px; font-weight:700; padding:8px 18px; border-radius:100px; border:none; text-decoration:none; opacity:0; transform:translateY(5px); transition:opacity .2s,transform .2s; box-shadow:0 4px 16px rgba(0,212,255,.4); letter-spacing:.02em; }
.ca-hint { font-size:11px; color:rgba(255,255,255,.4); opacity:0; transition:opacity .2s; pointer-events:none; }

/* NOTES PANEL */
.np { border-top:1px solid var(--b0); background:var(--bg-el); flex-shrink:0; }
.nph { display:flex; align-items:center; justify-content:space-between; padding:7px 13px; cursor:pointer; user-select:none; transition:background .15s; }
.nph:hover { background:rgba(255,255,255,.02); }
.nhl { display:flex; align-items:center; gap:6px; font-size:12px; color:var(--t2); font-weight:500; min-width:0; }
.ndot { width:6px; height:6px; border-radius:50%; background:var(--t3); flex-shrink:0; transition:background .3s,box-shadow .3s; }
.ndot.on { background:var(--green); box-shadow:0 0 6px var(--green); }
.ndate { font-size:10px; color:var(--t3); font-style:italic; margin-left:2px; white-space:nowrap; }
.nacts { display:flex; gap:5px; flex-shrink:0; }
.nb { display:flex; align-items:center; gap:3px; border:1px solid var(--b1); background:var(--bg-card); font-family:var(--fb); font-size:11px; font-weight:600; padding:3px 9px; border-radius:var(--rs); cursor:pointer; transition:background .2s; white-space:nowrap; }
.nb.s { color:var(--green); } .nb.s:hover { background:var(--grD); border-color:rgba(0,229,160,.4); }
.nb.d { color:var(--red); }   .nb.d:hover { background:var(--rD);  border-color:rgba(255,77,109,.35); }
.nbody { padding:0 13px 10px; display:none; }
.nbody.open { display:block; }
.nflds { display:grid; grid-template-columns:1fr 1fr; gap:7px; margin:8px 0 6px; }
.nf label { display:block; font-size:10px; font-weight:600; letter-spacing:.07em; text-transform:uppercase; color:var(--t3); margin-bottom:3px; }
.nf input,.nf select,.ntxt { width:100%; background:var(--bg); border:1px solid var(--b1); border-radius:var(--rs); color:var(--t1); font-family:var(--fb); font-size:12px; padding:5px 8px; outline:none; }
.nf input:focus,.nf select:focus,.ntxt:focus { border-color:var(--b2); }
.nf select option { background:var(--bg-card); }
.ntxt { height:60px; resize:vertical; line-height:1.5; display:block; margin-top:2px; }
.ntxt-lbl { font-size:10px; font-weight:600; letter-spacing:.07em; text-transform:uppercase; color:var(--t3); margin-bottom:3px; margin-top:6px; display:block; }
.nprev { padding:6px 13px 8px; font-size:11px; color:var(--t2); line-height:1.5; display:none; white-space:pre-wrap; word-break:break-word; border-top:1px dashed rgba(56,139,253,.1); background:rgba(0,0,0,.12); }
.nprev.open { display:block; }
.bbadge { display:inline-flex; align-items:center; font-size:10px; font-weight:700; padding:1px 7px; border-radius:100px; margin-left:4px; vertical-align:middle; }
.bb-bull    { background:rgba(0,229,160,.12); color:var(--green); border:1px solid rgba(0,229,160,.3); }
.bb-bear    { background:rgba(255,77,109,.12); color:var(--red);   border:1px solid rgba(255,77,109,.3); }
.bb-neutral { background:rgba(240,192,64,.1);  color:var(--gold);  border:1px solid rgba(240,192,64,.25); }
.bb-wait    { background:rgba(167,139,250,.1); color:var(--purple);border:1px solid rgba(167,139,250,.25); }

/* ═══ ETF PANEL ═══════════════════════════════════ */
.etf-source-bar {
  display:flex; align-items:center; justify-content:space-between;
  flex-wrap:wrap; gap:10px;
  background:rgba(0,212,255,.04); border:1px solid rgba(0,212,255,.18);
  border-radius:var(--rm); padding:11px 16px; margin-bottom:14px;
}
.etf-source-bar-left { display:flex; align-items:center; gap:10px; }
.etf-source-icon { font-size:20px; }
.etf-source-text strong { font-size:13px; color:var(--cyan); font-weight:600; display:block; }
.etf-source-text p { font-size:11px; color:var(--t3); margin-top:1px; }
.etf-source-links { display:flex; gap:8px; flex-wrap:wrap; }
.ext-link {
  display:inline-flex; align-items:center; gap:5px;
  font-size:11px; font-weight:600; padding:5px 11px; border-radius:var(--rs);
  border:1px solid; text-decoration:none; transition:background .2s;
}
.ext-link.c { color:var(--cyan); border-color:rgba(0,212,255,.3); background:var(--cD); }
.ext-link.c:hover { background:rgba(0,212,255,.18); }
.ext-link.g { color:var(--gold); border-color:rgba(240,192,64,.3); background:var(--gD); }
.ext-link.g:hover { background:rgba(240,192,64,.18); }

/* UPDATE FORM */
.etf-update-card {
  background:var(--bg-card); border:1px solid var(--b1);
  border-radius:var(--r); padding:16px; margin-bottom:14px;
}
.etf-update-title {
  display:flex; align-items:center; gap:8px;
  font-size:13px; font-weight:600; color:var(--t1); margin-bottom:14px;
}
.etf-update-title span { font-size:16px; }
.etf-form-grid { display:grid; grid-template-columns:repeat(auto-fit,minmax(160px,1fr)); gap:10px; margin-bottom:12px; }
.ef { }
.ef label { display:block; font-size:10px; font-weight:600; letter-spacing:.08em; text-transform:uppercase; color:var(--t3); margin-bottom:4px; }
.ef input { width:100%; background:var(--bg); border:1px solid var(--b1); border-radius:var(--rs); color:var(--t1); font-family:var(--fb); font-size:13px; padding:7px 10px; outline:none; transition:border-color .2s; }
.ef input:focus { border-color:var(--b2); }
.ef input::placeholder { color:var(--t3); }
.etf-form-actions { display:flex; align-items:center; gap:10px; flex-wrap:wrap; }
.etf-save-btn {
  display:flex; align-items:center; gap:6px;
  background:linear-gradient(135deg,rgba(0,229,160,.2),rgba(0,229,160,.1));
  border:1px solid rgba(0,229,160,.4); color:var(--green);
  font-family:var(--fb); font-size:12px; font-weight:700;
  padding:8px 18px; border-radius:var(--rs); cursor:pointer;
  transition:background .2s,border-color .2s;
}
.etf-save-btn:hover { background:rgba(0,229,160,.25); border-color:rgba(0,229,160,.6); }
.etf-form-note { font-size:11px; color:var(--t3); font-style:italic; }

/* ETF DATA DISPLAY */
.etf-data-grid { display:grid; grid-template-columns:1fr 1fr; gap:12px; margin-bottom:12px; }
@media(max-width:700px){ .etf-data-grid { grid-template-columns:1fr; } }

.etf-card {
  background:var(--bg-card); border:1px solid var(--b0);
  border-radius:var(--r); overflow:hidden;
}
.etf-card:hover { border-color:var(--b1); }
.etf-card-hdr {
  display:flex; align-items:center; justify-content:space-between;
  padding:10px 14px; border-bottom:1px solid var(--b0);
}
.etf-card-name { font-size:13px; font-weight:700; color:var(--t1); }
.etf-card-sub  { font-size:10px; color:var(--t3); margin-top:1px; }
.etf-kpis { display:grid; grid-template-columns:repeat(3,1fr); gap:0; }
.etf-kpi {
  padding:12px 14px; text-align:center;
  border-right:1px solid var(--b0);
}
.etf-kpi:last-child { border-right:none; }
.etf-kpi-lbl { font-size:10px; font-weight:600; letter-spacing:.08em; text-transform:uppercase; color:var(--t3); margin-bottom:4px; }
.etf-kpi-val { font-family:var(--fd); font-size:22px; line-height:1; }
.etf-kpi-val.pos { color:var(--green); }
.etf-kpi-val.neg { color:var(--red); }
.etf-kpi-val.neu { color:var(--gold); }
.etf-kpi-val.dim { color:var(--t3); }
.etf-kpi-sub { font-size:10px; color:var(--t3); margin-top:2px; }

/* FLOW BARS */
.flow-section { padding:12px 14px; border-top:1px solid var(--b0); }
.flow-section-title { font-size:10px; font-weight:600; letter-spacing:.08em; text-transform:uppercase; color:var(--t3); margin-bottom:8px; }
.flow-row { display:flex; align-items:center; gap:8px; margin-bottom:4px; }
.flow-date { font-size:10px; color:var(--t3); width:40px; flex-shrink:0; text-align:right; }
.flow-bar-wrap { flex:1; height:14px; background:rgba(255,255,255,.04); border-radius:3px; overflow:hidden; }
.flow-bar { height:100%; border-radius:3px; transition:width .5s ease; min-width:2px; }
.flow-bar.pos { background:linear-gradient(90deg,rgba(0,229,160,.5),var(--green)); }
.flow-bar.neg { background:linear-gradient(90deg,rgba(255,77,109,.5),var(--red)); }
.flow-val { font-size:10px; font-weight:600; width:54px; flex-shrink:0; text-align:right; }
.flow-val.pos { color:var(--green); } .flow-val.neg { color:var(--red); }

/* COMBINED TOTALS */
.etf-combined { background:var(--bg-card); border:1px solid var(--b0); border-radius:var(--r); padding:14px 16px; }
.etf-combined-title { font-size:12px; font-weight:600; color:var(--t2); margin-bottom:12px; display:flex; align-items:center; gap:6px; }
.etf-combined-kpis { display:grid; grid-template-columns:repeat(auto-fit,minmax(120px,1fr)); gap:8px; }
.eck { background:var(--bg-el); border-radius:var(--rm); padding:10px 12px; text-align:center; }
.eck-lbl { font-size:10px; font-weight:600; letter-spacing:.08em; text-transform:uppercase; color:var(--t3); margin-bottom:3px; }
.eck-val { font-family:var(--fd); font-size:19px; line-height:1; }
.eck-val.pos{color:var(--green)}.eck-val.neg{color:var(--red)}.eck-val.neu{color:var(--gold)}.eck-val.dim{color:var(--t3)}
.eck-sub { font-size:10px; color:var(--t3); margin-top:2px; }

/* LAST UPDATED badge */
.updated-badge {
  display:inline-flex; align-items:center; gap:5px;
  font-size:11px; color:var(--t3); background:var(--bg-el);
  border:1px solid var(--b0); border-radius:100px; padding:3px 10px;
}
.updated-badge.fresh { color:var(--green); border-color:rgba(0,229,160,.3); }

/* TOAST */
#toast { position:fixed; bottom:16px; right:18px; z-index:9999; background:var(--bg-el); border:1px solid var(--b2); color:var(--t1); font-family:var(--fb); font-size:13px; font-weight:500; padding:9px 16px; border-radius:var(--rm); pointer-events:none; opacity:0; transform:translateY(6px); transition:opacity .25s,transform .25s; box-shadow:0 4px 16px rgba(0,0,0,.5); max-width:320px; }
#toast.show { opacity:1; transform:translateY(0); }
#toast.ok  { border-color:rgba(0,229,160,.5); color:var(--green); }
#toast.err { border-color:rgba(255,77,109,.5); color:var(--red); }

/* FOOTER */
.footer { position:relative; z-index:1; border-top:1px solid var(--b0); padding:11px 22px; display:flex; align-items:center; justify-content:space-between; flex-wrap:wrap; gap:8px; }
.footer-note { font-size:11px; color:var(--t3); font-style:italic; }
.footer-dev  { font-family:var(--fd); font-size:13px; color:var(--t2); }
.footer-dev span { color:var(--cyan); font-style:italic; }


/* ETF LAUNCH CARD */
.etf-launch-card {
  background:var(--bg-card); border:1px solid var(--b1);
  border-radius:var(--r); padding:28px 24px; margin-bottom:14px;
}
.etf-launch-inner {
  display:flex; align-items:flex-start; gap:18px; margin-bottom:22px;
}
.etf-launch-logo { flex-shrink:0; margin-top:2px; }
.etf-launch-title {
  font-family:var(--fd); font-size:18px; color:var(--t1);
  margin-bottom:8px; line-height:1.3;
}
.etf-launch-sub {
  font-size:13px; color:var(--t2); line-height:1.6; max-width:600px;
}
.etf-launch-actions {
  display:flex; align-items:center; gap:10px; flex-wrap:wrap; margin-bottom:20px;
}
.etf-main-btn {
  display:inline-flex; align-items:center; gap:8px;
  background:linear-gradient(135deg,var(--cyan),#0060d0);
  color:#000; font-family:var(--fb); font-size:13px; font-weight:700;
  padding:11px 24px; border-radius:100px; text-decoration:none;
  box-shadow:0 4px 20px rgba(0,212,255,.35); letter-spacing:.02em;
  transition:box-shadow .2s,transform .2s;
}
.etf-main-btn:hover { box-shadow:0 6px 28px rgba(0,212,255,.5); transform:translateY(-1px); }
.etf-sec-btn {
  display:inline-flex; align-items:center; gap:6px;
  border:1px solid var(--b1); background:var(--bg-el);
  color:var(--t2); font-family:var(--fb); font-size:12px; font-weight:600;
  padding:9px 16px; border-radius:100px; text-decoration:none;
  transition:border-color .2s,color .2s,background .2s;
}
.etf-sec-btn:hover { border-color:var(--b2); color:var(--t1); background:rgba(56,139,253,.08); }
.etf-info-row {
  display:flex; flex-direction:column; gap:8px;
  padding:14px 16px; background:var(--bg-el);
  border-radius:var(--rm); border:1px solid var(--b0);
}
.etf-info-item {
  display:flex; align-items:center; gap:10px;
  font-size:12px; color:var(--t2); line-height:1.4;
}
.etf-info-dot {
  width:8px; height:8px; border-radius:50%; flex-shrink:0;
}
.etf-info-dot.c { background:var(--cyan); box-shadow:0 0 6px var(--cyan); }
.etf-info-dot.g { background:var(--gold); box-shadow:0 0 6px var(--gold); }


/* ═══ MARKET TABLE ═══════════════════════════════════════════ */
.mtable-wrap { overflow-x:auto; margin-bottom:14px; }
.mtable {
  width:100%; border-collapse:collapse; font-size:12px;
  table-layout:auto;
}
.mtable th {
  font-size:10px; font-weight:700; letter-spacing:.08em; text-transform:uppercase;
  color:var(--t3); padding:8px 10px; text-align:right; border-bottom:1px solid var(--b0);
  white-space:nowrap; background:var(--bg-el);
}
.mtable th:first-child { text-align:left; min-width:130px; }
.mtable th.th-chart { text-align:center; min-width:80px; }
.mtable td {
  padding:8px 10px; border-bottom:1px solid rgba(56,139,253,.06);
  text-align:right; vertical-align:middle; white-space:nowrap;
}
.mtable td:first-child { text-align:left; }
.mtable tr:hover td { background:rgba(56,139,253,.04); }
.mtable tr.loading td { color:var(--t3); font-style:italic; text-align:center; }

/* Asset name cell */
.asset-cell { display:flex; align-items:center; gap:8px; }
.asset-icon {
  width:24px; height:24px; border-radius:50%; background:var(--bg-el);
  border:1px solid var(--b0); display:flex; align-items:center;
  justify-content:center; font-size:11px; font-weight:700; flex-shrink:0;
  overflow:hidden;
}
.asset-icon img { width:100%; height:100%; object-fit:cover; border-radius:50%; }
.asset-name { font-weight:600; color:var(--t1); font-size:12px; }
.asset-sub  { font-size:10px; color:var(--t3); }

/* Sparkline */
.spark-cell { text-align:center; }
canvas.spark { display:block; margin:0 auto; }

/* Price cells */
.price-val { font-family:var(--fd); font-size:13px; color:var(--t1); }
.mcap-val  { color:var(--t2); }
.fdv-val   { color:var(--t3); }
.ratio-val { color:var(--gold); font-weight:600; }

/* Change cells */
.chg { font-weight:700; font-size:11px; padding:2px 6px; border-radius:4px; display:inline-block; }
.chg.pos { color:var(--green); background:rgba(0,229,160,.08); }
.chg.neg { color:var(--red);   background:rgba(255,77,109,.08); }
.chg.neu { color:var(--t3);    background:rgba(255,255,255,.04); }

/* Section divider row */
.mtable tr.section-row td {
  background:var(--bg-el); color:var(--cyan); font-size:10px; font-weight:700;
  letter-spacing:.1em; text-transform:uppercase; padding:5px 10px;
  border-bottom:1px solid var(--b1); text-align:left;
}

/* Refresh bar */
.table-toolbar {
  display:flex; align-items:center; justify-content:space-between;
  flex-wrap:wrap; gap:8px; margin-bottom:10px;
}
.table-toolbar-title {
  font-family:var(--fd); font-size:15px; color:var(--t2); font-style:italic;
}
.refresh-btn {
  display:flex; align-items:center; gap:5px;
  background:var(--bg-el); border:1px solid var(--b1);
  color:var(--cyan); font-family:var(--fb); font-size:11px; font-weight:600;
  padding:5px 12px; border-radius:var(--rs); cursor:pointer;
  transition:background .2s; white-space:nowrap;
}
.refresh-btn:hover { background:var(--cD); border-color:var(--b2); }
.refresh-btn.spinning svg { animation:spin .7s linear infinite; }
.last-update { font-size:10px; color:var(--t3); }

/* TOP WINNERS TABLE */
.winners-toolbar {
  display:flex; align-items:center; gap:8px; margin-bottom:10px; flex-wrap:wrap;
}
.winners-toolbar-title {
  font-family:var(--fd); font-size:15px; color:var(--t2); font-style:italic; margin-right:auto;
}
.tf-btn {
  font-size:11px; font-weight:700; padding:5px 12px; border-radius:var(--rs);
  border:1px solid var(--b1); background:var(--bg-el); color:var(--t3);
  cursor:pointer; transition:all .2s; letter-spacing:.04em;
}
.tf-btn.active { background:rgba(0,212,255,.1); border-color:var(--b2); color:var(--cyan); }
.tf-btn:hover:not(.active) { background:rgba(255,255,255,.04); color:var(--t2); }

.rank-cell { color:var(--t3); font-size:11px; width:28px; }
.winner-chg { font-family:var(--fd); font-size:16px; color:var(--green); }
.winner-mcap { font-size:11px; color:var(--t3); }

/* Spin */
.spin { display:inline-block; width:14px; height:14px; border:2px solid var(--b0); border-top-color:var(--cyan); border-radius:50%; animation:spin .7s linear infinite; }
@keyframes spin { to { transform:rotate(360deg); } }

@keyframes fadeUp { from{opacity:0;transform:translateY(12px)} to{opacity:1;transform:translateY(0)} }
.cc:nth-child(1){animation-delay:.03s}.cc:nth-child(2){animation-delay:.07s}.cc:nth-child(3){animation-delay:.11s}
::-webkit-scrollbar{width:5px;height:5px}::-webkit-scrollbar-track{background:var(--bg)}::-webkit-scrollbar-thumb{background:var(--b1);border-radius:3px}
</style>
</head>
<body>

<div class="app">

  <!-- SIDEBAR -->
  <nav class="sidebar" id="sidebar">
    <div class="slogo" onclick="toggleSidebar()">₿</div>
    <div class="ni active" id="nav-macro" onclick="switchTab('macro')">
      <span class="ni-icon">🌐</span><span class="ni-lbl">MACRO</span>
    </div>
    <div class="ni" id="nav-hype" onclick="switchTab('hype')">
      <span class="ni-icon" style="width:22px;display:flex;align-items:center;justify-content:center"><img src="data:image/webp;base64,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" style="width:22px;height:22px;border-radius:50%;object-fit:cover;flex-shrink:0"></span><span class="ni-lbl">HYPE</span>
    </div>
    <div class="ndiv"></div>
    <div class="ni" id="nav-etf" onclick="switchTab('etf')">
      <span class="ni-icon">📊</span><span class="ni-lbl">ETF Flows</span>
    </div>
    <div class="stoggle" onclick="toggleSidebar()">
      <span class="st-icon" id="st-icon">→</span>
      <span class="st-lbl">Contraer</span>
    </div>
  </nav>

  <!-- MAIN -->
  <div class="main">

    <!-- HEADER -->
    <div class="header">
      <div class="htitle">
        <h1 id="tab-title">Dashboard · Mercado Cripto — MACRO</h1>
        <p>Análisis Global · 1W · Firebase Sync</p>
      </div>
      <div class="hmeta">
        <div class="spill"><div class="sdot"></div><span>En vivo</span></div>
        <div id="sc"></div><div id="ls"></div>
      </div>
    </div>

    <!-- ══ TAB: MACRO ══════════════════════════════ -->
    <div class="tab-panel active" id="tab-macro">
      <div class="metrics" style="padding-left:0;padding-right:0">
        <div class="mc"><div class="ml">BTC/USDT</div><div class="mv c">BTCUSDT</div><div class="mb">1W</div></div>
        <div class="mc"><div class="ml">BTC Dominance</div><div class="mv g">BTC.D</div><div class="mb">1W</div></div>
        <div class="mc"><div class="ml">Total Mkt Cap</div><div class="mv gr">TOTAL</div><div class="mb">1W</div></div>
        <div class="mc"><div class="ml">TOTAL2</div><div class="mv p">ex-BTC</div><div class="mb">1W</div></div>
        <div class="mc"><div class="ml">TOTAL3</div><div class="mv r">ex-BTC/ETH</div><div class="mb">1W</div></div>
      </div>

      <div class="section">
        <div class="stitle"><h2>Bitcoin · BTC/USDT con Indicadores</h2><div class="sline"></div></div>
        <div class="cc">
          <div class="cch">
            <div class="cct"><span class="tick">BTCUSDT</span><span class="tdesc">· Binance · 1W · EMA20w · SMA21w · MACD · RSI</span></div>
            <span class="tag c">BTC Principal</span>
          </div>
          <div class="ca" style="height:500px">
            <div id="tv-btcusdt" style="height:500px;width:100%"></div>
            <div class="ca-ov">
              <a class="otv" href="https://www.tradingview.com/chart/?symbol=BINANCE:BTCUSDT&interval=W" target="_blank">
                <svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>
                Abrir en TradingView
              </a>
              <span class="ca-hint">Los dibujos se guardan en tu cuenta de TradingView</span>
            </div>
          </div>
          <div id="notes-btcusdt" class="np"></div>
        </div>
      </div>

      <div class="section">
        <div class="stitle"><h2>Dominancia &amp; Capitalización Total</h2><div class="sline"></div></div>
        <div class="g2">
          <div class="cc">
            <div class="cch"><div class="cct"><span class="tick">BTC.D</span><span class="tdesc">· Dominancia BTC</span></div><span class="tag c">Dominancia</span></div>
            <div class="ca" style="height:340px">
              <div id="tv-btcd" style="height:340px;width:100%"></div>
              <div class="ca-ov"><a class="otv" href="https://www.tradingview.com/chart/?symbol=CRYPTOCAP:BTC.D&interval=W" target="_blank"><svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>Abrir en TradingView</a></div>
            </div>
            <div id="notes-btcd" class="np"></div>
          </div>
          <div class="cc">
            <div class="cch"><div class="cct"><span class="tick">TOTAL</span><span class="tdesc">· Cap. Total del Mercado</span></div><span class="tag g">Market Cap</span></div>
            <div class="ca" style="height:340px">
              <div id="tv-total" style="height:340px;width:100%"></div>
              <div class="ca-ov"><a class="otv" href="https://www.tradingview.com/chart/?symbol=CRYPTOCAP:TOTAL&interval=W" target="_blank"><svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>Abrir en TradingView</a></div>
            </div>
            <div id="notes-total" class="np"></div>
          </div>
        </div>
      </div>

      <div class="section">
        <div class="stitle"><h2>Altcoins · TOTAL2, TOTAL3 &amp; OTHERS</h2><div class="sline"></div></div>
        <div class="g3">
          <div class="cc">
            <div class="cch"><div class="cct"><span class="tick">TOTAL2</span><span class="tdesc">· ex-BTC</span></div><span class="tag gr">Alts+ETH</span></div>
            <div class="ca" style="height:280px">
              <div id="tv-total2" style="height:280px;width:100%"></div>
              <div class="ca-ov"><a class="otv" href="https://www.tradingview.com/chart/?symbol=CRYPTOCAP:TOTAL2&interval=W" target="_blank"><svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>Abrir en TradingView</a></div>
            </div>
            <div id="notes-total2" class="np"></div>
          </div>
          <div class="cc">
            <div class="cch"><div class="cct"><span class="tick">TOTAL3</span><span class="tdesc">· ex-BTC &amp; ETH</span></div><span class="tag p">Alts Puras</span></div>
            <div class="ca" style="height:280px">
              <div id="tv-total3" style="height:280px;width:100%"></div>
              <div class="ca-ov"><a class="otv" href="https://www.tradingview.com/chart/?symbol=CRYPTOCAP:TOTAL3&interval=W" target="_blank"><svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>Abrir en TradingView</a></div>
            </div>
            <div id="notes-total3" class="np"></div>
          </div>
          <div class="cc">
            <div class="cch"><div class="cct"><span class="tick">OTHERS</span><span class="tdesc">· ex-Top 10</span></div><span class="tag r">Small Caps</span></div>
            <div class="ca" style="height:280px">
              <div id="tv-others" style="height:280px;width:100%"></div>
              <div class="ca-ov"><a class="otv" href="https://www.tradingview.com/chart/?symbol=CRYPTOCAP:OTHERS&interval=W" target="_blank"><svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>Abrir en TradingView</a></div>
            </div>
            <div id="notes-others" class="np"></div>
          </div>
        </div>
      </div>
    </div>

    <!-- ══ TAB: HYPE ════════════════════════════════ -->
    <div class="tab-panel" id="tab-hype">
      <div class="metrics" style="padding-left:0;padding-right:0">
        <div class="mc"><div class="ml">HYPE Price</div><div class="mv c">HYPEUSDT</div><div class="mb">1W</div></div>
        <div class="mc"><div class="ml">HYPE/BTC</div><div class="mv g">Ratio</div><div class="mb">1W</div></div>
        <div class="mc"><div class="ml">THYP Acumulado</div><div class="mv gr" id="hm-thyp-acc">—</div><div class="mb">21Shares</div></div>
        <div class="mc"><div class="ml">BHYP Acumulado</div><div class="mv p" id="hm-bhyp-acc">—</div><div class="mb">Bitwise</div></div>
        <div class="mc"><div class="ml">Total ETF Flows</div><div class="mv r" id="hm-total-acc">—</div><div class="mb">Combinado</div></div>
      </div>
      <div class="section">
        <div class="stitle"><h2>HYPE · Hyperliquid</h2><div class="sline"></div></div>
        <div class="g2">
          <div class="cc">
            <div class="cch"><div class="cct"><span class="tick">HYPE/USDT</span><span class="tdesc">· KuCoin · Hyperliquid</span></div><span class="tag c">HYPE</span></div>
            <div class="ca" style="height:400px">
              <div id="tv-hype" style="height:400px;width:100%"></div>
              <div class="ca-ov"><a class="otv" href="https://www.tradingview.com/chart/?symbol=KUCOIN:HYPEUSDT&interval=W" target="_blank"><svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>Abrir en TradingView</a><span class="ca-hint">Los dibujos se guardan en tu cuenta</span></div>
            </div>
            <div id="notes-hype" class="np"></div>
          </div>
          <div class="cc">
            <div class="cch"><div class="cct"><span class="tick">HYPE/BTC</span><span class="tdesc">· KuCoin ÷ Binance · Ratio</span></div><span class="tag g">HYPE vs BTC</span></div>
            <div class="ca" style="height:400px">
              <div id="tv-hypebtc" style="height:400px;width:100%"></div>
              <div class="ca-ov"><a class="otv" href="https://www.tradingview.com/chart/?symbol=KUCOIN:HYPEUSDT/BINANCE:BTCUSDT&interval=W" target="_blank"><svg width="13" height="13" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>Abrir en TradingView</a><span class="ca-hint">Los dibujos se guardan en tu cuenta</span></div>
            </div>
            <div id="notes-hypebtc" class="np"></div>
          </div>
        </div>
      </div>
    </div>

    <!-- ══ TAB: ETF FLOWS ═══════════════════════════ -->
    <div class="tab-panel" id="tab-etf">
      <div class="section">
        <div class="stitle"><h2>ETF Flows · HYPE Spot ETFs — THYP &amp; BHYP</h2><div class="sline"></div></div>


        <!-- BOTÓN SOSOVALUE -->
        <div class="etf-launch-card">
          <div class="etf-launch-inner">
            <div class="etf-launch-logo">
              <svg width="40" height="40" viewBox="0 0 40 40" fill="none">
                <rect width="40" height="40" rx="10" fill="rgba(0,212,255,0.1)"/>
                <path d="M8 28 L14 16 L20 24 L26 12 L32 22" stroke="#00d4ff" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" fill="none"/>
                <circle cx="32" cy="22" r="2.5" fill="#00d4ff"/>
              </svg>
            </div>
            <div class="etf-launch-text">
              <div class="etf-launch-title">HYPE Spot ETF Flows — THYP &amp; BHYP</div>
              <div class="etf-launch-sub">Datos de netflows en tiempo real: flujo diario, acumulado, AUM y histórico de THYP (21Shares) y BHYP (Bitwise). Disponibles en SoSoValue y CoinGlass — ambas fuentes ofrecen tablas completas con datos actualizados al cierre de cada sesión.</div>
            </div>
          </div>
          <div class="etf-launch-actions">
            <a href="https://sosovalue.com/assets/etf/us-hype-spot" target="_blank" class="etf-main-btn">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>
              SoSoValue · HYPE ETF
            </a>
            <a href="https://www.coinglass.com/etf/hype" target="_blank" class="etf-main-btn" style="background:linear-gradient(135deg,#f0c040,#e07b00);box-shadow:0 4px 18px rgba(240,192,64,.35)">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>
              CoinGlass · HYPE ETF
            </a>
            <a href="https://hypurrintel.xyz/etf-overview" target="_blank" class="etf-sec-btn">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>
              HypurrIntel
            </a>
            <a href="https://farside.co.uk/hyperliquid/" target="_blank" class="etf-sec-btn">
              <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><path d="M18 13v6a2 2 0 01-2 2H5a2 2 0 01-2-2V8a2 2 0 012-2h6"/><polyline points="15 3 21 3 21 9"/><line x1="10" y1="14" x2="21" y2="3"/></svg>
              Farside
            </a>
          </div>
          <div class="etf-info-row">
            <div class="etf-info-item">
              <span class="etf-info-dot c"></span>
              <span><strong>THYP</strong> — 21Shares Hyperliquid ETP · NYSE Arca · Lanzado 12 May 2025</span>
            </div>
            <div class="etf-info-item">
              <span class="etf-info-dot g"></span>
              <span><strong>BHYP</strong> — Bitwise Hyperliquid ETF · NYSE Arca · Lanzado 14 May 2025</span>
            </div>
            <div class="etf-info-item" style="margin-top:4px;border-top:1px solid var(--b0);padding-top:8px">
              <span style="font-size:11px;color:var(--t3)">
                ⚠️ CoinGlass y SoSoValue no permiten incrustar sus datos en webs externas (política CORS/iframe). Los botones abren la fuente original en una pestaña nueva con los datos completos y actualizados en tiempo real.
              </span>
            </div>
          </div>
        </div>


      </div><!-- /etf section -->

      <!-- ═══ TABLA DE ACTIVOS ═══════════════════════════════════════ -->
      <div class="section">
        <div class="table-toolbar">
          <span class="table-toolbar-title">Visión Global de Mercado</span>
          <div style="display:flex;align-items:center;gap:10px;flex-wrap:wrap">
            <span class="last-update" id="market-last-update"></span>
            <button class="refresh-btn" id="market-refresh-btn" onclick="loadMarketTable()">
              <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><polyline points="23 4 23 10 17 10"/><polyline points="1 20 1 14 7 14"/><path d="M3.51 9a9 9 0 0114.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0020.49 15"/></svg>
              Actualizar
            </button>
          </div>
        </div>
        <div class="mtable-wrap">
          <table class="mtable" id="market-table">
            <thead>
              <tr>
                <th>Activo</th>
                <th class="th-chart">30D</th>
                <th>Precio</th>
                <th>Mkt Cap</th>
                <th>FDV</th>
                <th>FDV:MC</th>
                <th>1D</th>
                <th>1S</th>
                <th>1M</th>
                <th>3M</th>
                <th>YTD</th>
              </tr>
            </thead>
            <tbody id="market-tbody">
              <tr class="loading"><td colspan="11">⏳ Cargando datos de mercado...</td></tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- ═══ TOP WINNERS ═══════════════════════════════════════════ -->
      <div class="section">
        <div class="winners-toolbar">
          <span class="winners-toolbar-title">🏆 Top Winners Cripto</span>
          <button class="tf-btn active" id="wbtn-1d" onclick="setWinnersTF('1d')">1D</button>
          <button class="tf-btn" id="wbtn-7d"  onclick="setWinnersTF('7d')">1S</button>
          <button class="tf-btn" id="wbtn-30d" onclick="setWinnersTF('30d')">1M</button>
          <button class="refresh-btn" onclick="loadWinners()" style="margin-left:4px">
            <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5"><polyline points="23 4 23 10 17 10"/><polyline points="1 20 1 14 7 14"/><path d="M3.51 9a9 9 0 0114.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0020.49 15"/></svg>
            Actualizar
          </button>
        </div>
        <div class="mtable-wrap">
          <table class="mtable" id="winners-table">
            <thead>
              <tr>
                <th style="text-align:left;width:28px">#</th>
                <th style="text-align:left">Activo</th>
                <th class="th-chart">30D</th>
                <th>Precio</th>
                <th>Mkt Cap</th>
                <th id="winners-chg-header">Ganancia 1D</th>
              </tr>
            </thead>
            <tbody id="winners-tbody">
              <tr class="loading"><td colspan="6">⏳ Cargando top winners...</td></tr>
            </tbody>
          </table>
        </div>
      </div>

    </div><!-- /tab-etf -->

    <!-- FOOTER -->
    <footer class="footer">
      <div class="footer-note">Dashboard de análisis cripto · TradingView · Firebase · Datos ETF: SoSoValue (actualización manual)</div>
      <div class="footer-dev">Desarrollado por <span>Juan Campos</span></div>
    </footer>

  </div><!-- /main -->
</div><!-- /app -->

<div id="toast"></div>

<!-- TRADINGVIEW -->
<script type="text/javascript" src="https://s3.tradingview.com/tv.js"></script>
<script>
var WIDGETS = [
  { id:'btcusdt', symbol:'BINANCE:BTCUSDT', h:500, tab:'macro',
    studies:[
      {id:'MASimple@tv-basicstudies',inputs:{length:147}},
      {id:'MAExp@tv-basicstudies',   inputs:{length:140}},
      {id:'MACD@tv-basicstudies'},{id:'RSI@tv-basicstudies'}
    ]},
  { id:'btcd',    symbol:'CRYPTOCAP:BTC.D',                 h:340, tab:'macro' },
  { id:'total',   symbol:'CRYPTOCAP:TOTAL',                 h:340, tab:'macro' },
  { id:'total2',  symbol:'CRYPTOCAP:TOTAL2',                h:280, tab:'macro' },
  { id:'total3',  symbol:'CRYPTOCAP:TOTAL3',                h:280, tab:'macro' },
  { id:'others',  symbol:'CRYPTOCAP:OTHERS',                h:280, tab:'macro' },
  { id:'hype',    symbol:'KUCOIN:HYPEUSDT',                 h:400, tab:'hype'  },
  { id:'hypebtc', symbol:'KUCOIN:HYPEUSDT/BINANCE:BTCUSDT', h:400, tab:'hype'  }
];

var _openNotes  = {};
var _tvInited   = { macro:false, hype:false };
var _currentTab = 'macro';

/* TOAST */
function toast(msg, type) {
  var t = document.getElementById('toast');
  t.textContent = msg; t.className = 'show '+(type||'');
  clearTimeout(t._t); t._t = setTimeout(function(){ t.className=''; }, 3500);
}

/* FMT */
function fmtM(v) {
  if (v === null || v === undefined || v === '' || isNaN(parseFloat(v))) return '—';
  var n   = parseFloat(v);
  var abs = Math.abs(n);
  var sgn = n >= 0 ? '+' : '-';
  if (abs >= 1000) return sgn + (abs/1000).toFixed(2) + 'B';
  return sgn + abs.toFixed(1) + 'M';
}
function fmtAUM(v) {
  if (!v || isNaN(parseFloat(v))) return '—';
  return '$' + parseFloat(v).toFixed(1) + 'M';
}

/* SIDEBAR */
function toggleSidebar() {
  var sb = document.getElementById('sidebar');
  sb.classList.toggle('expanded');
  document.getElementById('st-icon').textContent = sb.classList.contains('expanded') ? '←' : '→';
}

/* TABS */
function switchTab(tab) {
  _currentTab = tab;
  ['macro','hype','etf'].forEach(function(t) {
    document.getElementById('tab-'+t).classList.toggle('active', t===tab);
    document.getElementById('nav-'+t).classList.toggle('active', t===tab);
  });
  var titles = {
    macro: 'Dashboard · Mercado Cripto — MACRO',
    hype:  'Dashboard · Hyperliquid — HYPE',
    etf:   'Dashboard · ETF Flows — HYPE Spot'
  };
  document.getElementById('tab-title').textContent = titles[tab] || '';
  if ((tab==='macro' && !_tvInited.macro) || (tab==='hype' && !_tvInited.hype)) {
    setTimeout(function(){ initTV(tab); }, 80);
  }
}

/* TV WIDGETS */
function initTV(tab) {
  if (_tvInited[tab]) return;
  _tvInited[tab] = true;
  WIDGETS.filter(function(w){ return w.tab===tab; }).forEach(function(def) {
    try {
      new TradingView.widget({
        autosize:false, width:'100%', height:def.h,
        symbol:def.symbol, interval:'W',
        timezone:'Europe/Madrid', theme:'dark', style:'1', locale:'es',
        toolbar_bg:'#0a1628', enable_publishing:false,
        hide_top_toolbar:false, hide_side_toolbar:false, hide_legend:false,
        save_image:true, allow_symbol_change:true, withdateranges:true,
        show_popup_button:true, popup_width:'1400', popup_height:'900',
        container_id:'tv-'+def.id, studies:def.studies||[],
        drawings_access:{type:'all'}, backgroundColor:'#050c1a',
        gridColor:'rgba(56,139,253,0.04)',
        overrides:{
          'paneProperties.background':'#050c1a','paneProperties.backgroundType':'solid',
          'paneProperties.vertGridProperties.color':'rgba(56,139,253,0.04)',
          'paneProperties.horzGridProperties.color':'rgba(56,139,253,0.04)',
          'mainSeriesProperties.candleStyle.upColor':'#00e5a0',
          'mainSeriesProperties.candleStyle.downColor':'#ff4d6d',
          'mainSeriesProperties.candleStyle.borderUpColor':'#00e5a0',
          'mainSeriesProperties.candleStyle.borderDownColor':'#ff4d6d',
          'mainSeriesProperties.candleStyle.wickUpColor':'#00e5a0',
          'mainSeriesProperties.candleStyle.wickDownColor':'#ff4d6d'
        }
      });
    } catch(e) { console.warn('TV error', def.id, e); }
  });
}

/* ══════════════════════════════════════════════════
   ETF DATA — GUARDAR Y RENDERIZAR
══════════════════════════════════════════════════ */

function renderETFFromFirebase(d) {
  if (!d) return;

  var display = document.getElementById('etf-display');
  if (display) display.style.display = 'block';

  // THYP
  setKPI('d-thyp-day', fmtM(d.thyp && d.thyp.day), parseFloat(d.thyp && d.thyp.day) >= 0 ? 'pos' : 'neg');
  setKPI('d-thyp-acc', fmtM(d.thyp && d.thyp.acc), 'pos');
  setKPI('d-thyp-aum', fmtAUM(d.thyp && d.thyp.aum), 'neu');

  // BHYP
  setKPI('d-bhyp-day', fmtM(d.bhyp && d.bhyp.day), parseFloat(d.bhyp && d.bhyp.day) >= 0 ? 'pos' : 'neg');
  setKPI('d-bhyp-acc', fmtM(d.bhyp && d.bhyp.acc), 'pos');
  setKPI('d-bhyp-aum', fmtAUM(d.bhyp && d.bhyp.aum), 'neu');

  // Combinados
  var tDay = parseFloat((d.thyp && d.thyp.day) || 0);
  var bDay = parseFloat((d.bhyp && d.bhyp.day) || 0);
  var tAcc = parseFloat((d.thyp && d.thyp.acc) || 0);
  var bAcc = parseFloat((d.bhyp && d.bhyp.acc) || 0);
  var tAum = parseFloat((d.thyp && d.thyp.aum) || 0);
  var bAum = parseFloat((d.bhyp && d.bhyp.aum) || 0);

  setKPI('d-comb-day', fmtM(tDay + bDay), (tDay+bDay) >= 0 ? 'pos' : 'neg');
  setKPI('d-comb-acc', fmtM(tAcc + bAcc), 'pos');
  setKPI('d-comb-aum', fmtAUM(tAum + bAum), 'neu');

  // Timestamp
  if (d.updatedAt) {
    var dt = new Date(d.updatedAt).toLocaleString('es-ES');
    setKPI('d-updated', dt, 'dim');
    var badge = document.getElementById('etf-updated-badge');
    if (badge) {
      badge.textContent = 'Actualizado: ' + dt;
      badge.className = 'updated-badge fresh';
    }
  }

  // Métricas en HYPE tab
  var setH = function(id, val) { var el=document.getElementById(id); if(el) el.textContent=val; };
  setH('hm-thyp-acc', tAcc ? '$'+tAcc.toFixed(0)+'M' : '—');
  setH('hm-bhyp-acc', bAcc ? '$'+bAcc.toFixed(0)+'M' : '—');
  setH('hm-total-acc', (tAcc+bAcc) ? '$'+(tAcc+bAcc).toFixed(0)+'M' : '—');

  // Rellenar formulario con últimos valores
  if (d.thyp) {
    if (d.thyp.day) document.getElementById('f-thyp-day').value = d.thyp.day;
    if (d.thyp.acc) document.getElementById('f-thyp-acc').value = d.thyp.acc;
    if (d.thyp.aum) document.getElementById('f-thyp-aum').value = d.thyp.aum;
  }
  if (d.bhyp) {
    if (d.bhyp.day) document.getElementById('f-bhyp-day').value = d.bhyp.day;
    if (d.bhyp.acc) document.getElementById('f-bhyp-acc').value = d.bhyp.acc;
    if (d.bhyp.aum) document.getElementById('f-bhyp-aum').value = d.bhyp.aum;
  }
  if (d.history) document.getElementById('f-history').value = d.history;

  // Barras de historial
  renderFlowBars(d.history);
}

function setKPI(id, val, cls) {
  var el = document.getElementById(id); if (!el) return;
  el.textContent = val;
  el.className = el.className.replace(/\bpos\b|\bneg\b|\bneu\b|\bdim\b/g,'').trim();
  if (cls) el.classList.add(cls);
}

function renderFlowBars(histStr) {
  if (!histStr || !histStr.trim()) return;

  var rows = histStr.split(',').map(function(s){ return s.trim(); }).filter(Boolean);
  var parsed = rows.map(function(r) {
    var parts = r.split('|');
    return {
      date:  (parts[0]||'').trim(),
      thyp:  parseFloat(parts[1]) || 0,
      bhyp:  parseFloat(parts[2]) || 0
    };
  });

  var maxT = Math.max.apply(null, parsed.map(function(p){ return Math.abs(p.thyp); }));
  var maxB = Math.max.apply(null, parsed.map(function(p){ return Math.abs(p.bhyp); }));

  function makeBar(val, max) {
    var pct = max > 0 ? (Math.abs(val)/max*100).toFixed(1) : 0;
    var cls = val >= 0 ? 'pos' : 'neg';
    var lbl = (val >= 0 ? '+' : '') + val.toFixed(1) + 'M';
    return '<div class="flow-row">' +
      '<div class="flow-bar-wrap"><div class="flow-bar '+cls+'" style="width:'+pct+'%"></div></div>' +
      '<span class="flow-val '+cls+'">'+lbl+'</span>' +
    '</div>';
  }

  var thypHtml = '<div class="flow-section-title">Flujo diario THYP</div>' +
    parsed.map(function(p) {
      var pct = maxT > 0 ? (Math.abs(p.thyp)/maxT*100).toFixed(1) : 0;
      var cls = p.thyp >= 0 ? 'pos' : 'neg';
      var lbl = (p.thyp >= 0 ? '+' : '') + p.thyp.toFixed(1) + 'M';
      return '<div class="flow-row">' +
        '<span class="flow-date">'+p.date+'</span>' +
        '<div class="flow-bar-wrap"><div class="flow-bar '+cls+'" style="width:'+pct+'%"></div></div>' +
        '<span class="flow-val '+cls+'">'+lbl+'</span>' +
      '</div>';
    }).join('');

  var bhypHtml = '<div class="flow-section-title">Flujo diario BHYP</div>' +
    parsed.map(function(p) {
      var pct = maxB > 0 ? (Math.abs(p.bhyp)/maxB*100).toFixed(1) : 0;
      var cls = p.bhyp >= 0 ? 'pos' : 'neg';
      var lbl = (p.bhyp >= 0 ? '+' : '') + p.bhyp.toFixed(1) + 'M';
      return '<div class="flow-row">' +
        '<span class="flow-date">'+p.date+'</span>' +
        '<div class="flow-bar-wrap"><div class="flow-bar '+cls+'" style="width:'+pct+'%"></div></div>' +
        '<span class="flow-val '+cls+'">'+lbl+'</span>' +
      '</div>';
    }).join('');

  var te = document.getElementById('thyp-flows'); if(te) te.innerHTML = thypHtml;
  var be = document.getElementById('bhyp-flows'); if(be) be.innerHTML = bhypHtml;
}

/* ══ NOTAS ══════════════════════════════════════ */
var biasMap = {
  bull:    ['bb-bull',    '🟢 Alcista'],
  bear:    ['bb-bear',    '🔴 Bajista'],
  neutral: ['bb-neutral', '🟡 Neutral'],
  wait:    ['bb-wait',    '🟣 En espera']
};

function buildNotes(wid) {
  var el = document.getElementById('notes-'+wid); if (!el) return;
  el.innerHTML =
    '<div class="nph" onclick="toggleNotes(\''+wid+'\')">'+
      '<div class="nhl"><div class="ndot" id="nd-'+wid+'"></div>'+
        '<span id="nl-'+wid+'">📝 Notas de análisis</span>'+
        '<span class="ndate" id="ndt-'+wid+'"></span></div>'+
      '<div class="nacts" onclick="event.stopPropagation()">'+
        '<button class="nb s" onclick="saveN(\''+wid+'\')">💾 Guardar</button>'+
        '<button class="nb d" onclick="delN(\''+wid+'\')">✕</button>'+
      '</div></div>'+
    '<div class="nbody" id="nb-'+wid+'">'+
      '<div class="nflds">'+
        '<div class="nf"><label>Bias</label><select id="nbi-'+wid+'">'+
          '<option value="">— sin definir —</option>'+
          '<option value="bull">🟢 Alcista</option>'+
          '<option value="bear">🔴 Bajista</option>'+
          '<option value="neutral">🟡 Neutral</option>'+
          '<option value="wait">🟣 En espera</option>'+
        '</select></div>'+
        '<div class="nf"><label>Niveles clave</label>'+
          '<input id="nle-'+wid+'" type="text" placeholder="S: 58k / R: 72k"></div>'+
      '</div>'+
      '<span class="ntxt-lbl">Análisis</span>'+
      '<textarea class="ntxt" id="nte-'+wid+'" placeholder="Confluencias, zonas, plan..."></textarea>'+
    '</div>'+
    '<div class="nprev" id="npv-'+wid+'"></div>';
}

function toggleNotes(wid) {
  _openNotes[wid] = !_openNotes[wid];
  var b = document.getElementById('nb-'+wid);
  var p = document.getElementById('npv-'+wid);
  if (b) b.className = 'nbody' + (_openNotes[wid] ? ' open' : '');
  if (p) p.className = 'nprev' + (!_openNotes[wid] && p.textContent.trim() ? ' open' : '');
}

function saveN(wid) {
  var bias   = (document.getElementById('nbi-'+wid)||{}).value||'';
  var levels = (document.getElementById('nle-'+wid)||{}).value||'';
  var notes  = (document.getElementById('nte-'+wid)||{}).value||'';
  if (!bias && !levels && !notes.trim()) { toast('Escribe algo antes de guardar','err'); return; }
  if (!window.FB_save) { toast('Firebase no configurado','err'); return; }
  window.FB_save(wid, {bias, levels, notes}).then(function(ok) {
    if (ok) { toast('Notas guardadas ✓','ok'); _applyState(wid, {bias,levels,notes,savedAt:new Date().toISOString()}); }
    else toast('Error al guardar','err');
  });
}

function delN(wid) {
  if (!confirm('¿Borrar notas de este gráfico?')) return;
  ['nbi-','nle-','nte-'].forEach(function(p){ var e=document.getElementById(p+wid); if(e) e.value=''; });
  if (window.FB_save) window.FB_save(wid,{bias:'',levels:'',notes:''}).then(function(ok){
    if (ok) { toast('Notas borradas',''); _applyEmpty(wid); }
  });
}

function _applyState(wid, d) {
  var dot=document.getElementById('nd-'+wid);
  var lbl=document.getElementById('nl-'+wid);
  var dt =document.getElementById('ndt-'+wid);
  var pv =document.getElementById('npv-'+wid);
  if (dot) dot.className='ndot on';
  if (dt)  dt.textContent = d.savedAt ? '· '+new Date(d.savedAt).toLocaleString('es-ES') : '';
  var bm = d.bias && biasMap[d.bias];
  if (lbl) lbl.innerHTML='📝 Notas'+(bm?'  <span class="bbadge '+bm[0]+'">'+bm[1]+'</span>':'');
  if (pv) {
    var ls=[]; if(d.levels) ls.push('📍 '+d.levels); if(d.notes&&d.notes.trim()) ls.push(d.notes.trim());
    pv.textContent=ls.join('\n'); if(!_openNotes[wid]&&ls.length) pv.className='nprev open';
  }
  var eb=document.getElementById('nbi-'+wid); var el=document.getElementById('nle-'+wid); var et=document.getElementById('nte-'+wid);
  if(eb&&d.bias)   eb.value=d.bias;
  if(el&&d.levels) el.value=d.levels;
  if(et&&d.notes)  et.value=d.notes;
}

function _applyEmpty(wid) {
  var dot=document.getElementById('nd-'+wid); var lbl=document.getElementById('nl-'+wid);
  var dt=document.getElementById('ndt-'+wid); var pv=document.getElementById('npv-'+wid);
  if(dot) dot.className='ndot'; if(lbl) lbl.textContent='📝 Notas de análisis';
  if(dt) dt.textContent=''; if(pv){ pv.textContent=''; pv.className='nprev'; }
}

/* ARRANQUE */
WIDGETS.forEach(function(def){ buildNotes(def.id); });

function loadFirebaseNotes() {
  WIDGETS.forEach(function(def){
    var d = window.FB_get ? window.FB_get(def.id) : null;
    if (d && (d.notes||d.bias||d.levels)) _applyState(def.id, d);
  });
  if (window._etfData) renderETFFromFirebase(window._etfData);
}
setTimeout(loadFirebaseNotes, 1500);
setTimeout(loadFirebaseNotes, 4000);

if (typeof TradingView !== 'undefined') {
  initTV('macro');
} else {
  var tvs = document.querySelector('script[src*="tv.js"]');
  if (tvs) tvs.addEventListener('load', function(){ initTV('macro'); });
}

/* ═══════════════════════════════════════════════════════════════
   MARKET TABLE + TOP WINNERS
   Crypto:  CoinGecko en dos batches (evita rate limit)
   Índices: Financial Modeling Prep (FMP) — free key, 250/día
            Registro gratuito en: https://site.financialmodelingprep.com
   Auto-refresh cada 1 hora cuando el tab ETF está activo
═══════════════════════════════════════════════════════════════ */

var _winnersTF    = '1d';
var _sparkCache   = {};
var _marketLoaded = false;
var SPARK_COLOR   = '#00d4ff';
var SPARK_FILL    = 'rgba(0,212,255,0.12)';

/* ─────────────────────────────────────────────────────────────
   ▼▼▼  PEGA AQUÍ TU API KEY DE FMP (registro gratis en fmp)  ▼▼▼
   https://site.financialmodelingprep.com/login → Dashboard
──────────────────────────────────────────────────────────────*/
var FMP_KEY = '7oNShANDOh78VcdLdGMtTePYjTZzJuPB';
/* ▲▲▲  FIN CONFIGURACIÓN FMP  ▲▲▲ */

var TH_ROW = '<tr style="background:var(--bg-el)">' +
  '<th style="text-align:left;min-width:130px">Activo</th>' +
  '<th style="text-align:center;min-width:80px">30D</th>' +
  '<th>Precio</th><th>Mkt Cap</th><th>FDV</th><th>FDV:MC</th>' +
  '<th>1D</th><th>1S</th><th>1M</th><th>3M</th><th>YTD</th></tr>';

/* ── ACTIVOS ─────────────────────────────────────────────────── */
// Índices: FMP symbols
var TRAD_ASSETS = [
  { id:'^GSPC', sym:'SP500',   name:'S&P 500',          sub:'Índice USA'       },
  { id:'^NDX',  sym:'NQ100',   name:'Nasdaq 100',        sub:'Tech USA'         },
  { id:'^RUT',  sym:'RUT2000', name:'Russell 2000',      sub:'Small Caps USA'   },
  { id:'RSP',   sym:'RSP',     name:'S&P 500 Eq.Weight', sub:'Equal Weight ETF' },
];

// Crypto batch A: top MC (IDs más seguros, sin conflictos)
var CRYPTO_BATCH_A = [
  { id:'bitcoin',        sym:'BTC',  name:'Bitcoin',      section:'top' },
  { id:'ethereum',       sym:'ETH',  name:'Ethereum',     section:'top' },
  { id:'ripple',         sym:'XRP',  name:'XRP',          section:'top' },
  { id:'binancecoin',    sym:'BNB',  name:'BNB',          section:'top' },
  { id:'solana',         sym:'SOL',  name:'Solana',       section:'top' },
  { id:'tron',           sym:'TRX',  name:'TRON',         section:'top' },
  { id:'dogecoin',       sym:'DOGE', name:'Dogecoin',     section:'top' },
  { id:'hyperliquid',    sym:'HYPE', name:'Hyperliquid',  section:'top' },
  { id:'cardano',        sym:'ADA',  name:'Cardano',      section:'top' },
  { id:'bitcoin-cash',   sym:'BCH',  name:'Bitcoin Cash', section:'top' },
  { id:'monero',         sym:'XMR',  name:'Monero',       section:'top' },
];

// Crypto batch B: altcoins (fetch separado con delay para evitar 429)
var CRYPTO_BATCH_B = [
  { id:'morpho',             sym:'MORPHO', name:'Morpho',         section:'alt' },
  { id:'centrifuge',         sym:'CFG',    name:'Centrifuge',     section:'alt' },
  { id:'maple-finance',      sym:'SYRUP',  name:'Maple Finance',  section:'alt' },
  { id:'meta-2',             sym:'META',   name:'Meta (token)',   section:'alt' },
  { id:'zcash',              sym:'ZEC',    name:'Zcash',          section:'alt' },
  { id:'collector-crypt',    sym:'CARDS',  name:'Collector Crypt',section:'alt' },
  { id:'aerodrome-finance',  sym:'AERO',   name:'Aerodrome',      section:'alt' },
];

var ALL_ASSETS = CRYPTO_BATCH_A.concat(CRYPTO_BATCH_B);

/* ── FORMATTERS ──────────────────────────────────────────────── */
function fmtPrice(v) {
  if (v == null) return '—';
  if (v >= 1000) return '$' + v.toLocaleString('en-US',{maximumFractionDigits:0});
  if (v >= 1)    return '$' + v.toFixed(2);
  if (v >= 0.001)return '$' + v.toFixed(4);
  return '$' + v.toFixed(6);
}
function fmtBig(v) {
  if (!v) return '—';
  if (v >= 1e12) return '$'+(v/1e12).toFixed(2)+'T';
  if (v >= 1e9)  return '$'+(v/1e9).toFixed(2)+'B';
  if (v >= 1e6)  return '$'+(v/1e6).toFixed(1)+'M';
  return '$'+v.toFixed(0);
}
function fmtPct(v) {
  if (v == null) return '<span class="chg neu">—</span>';
  var cls = v > 0 ? 'pos' : v < 0 ? 'neg' : 'neu';
  return '<span class="chg '+cls+'">'+(v>0?'+':'')+v.toFixed(2)+'%</span>';
}

/* ── SPARKLINE ───────────────────────────────────────────────── */
function drawSpark(id, prices) {
  var c = document.getElementById(id);
  if (!c || !prices || prices.length < 2) return;
  var ctx = c.getContext('2d'), W = c.width, H = c.height;
  ctx.clearRect(0,0,W,H);
  var p = prices.filter(function(x){ return x != null && !isNaN(x); });
  if (p.length < 2) return;
  var mn=Math.min.apply(null,p), mx=Math.max.apply(null,p), rng=mx-mn||1;
  var pts = p.map(function(v,i){ return {x:i/(p.length-1)*W, y:H-((v-mn)/rng)*(H-4)-2}; });
  var g = ctx.createLinearGradient(0,0,0,H);
  g.addColorStop(0,SPARK_FILL); g.addColorStop(1,'rgba(0,0,0,0)');
  ctx.beginPath(); ctx.moveTo(pts[0].x,H);
  pts.forEach(function(pt){ ctx.lineTo(pt.x,pt.y); });
  ctx.lineTo(pts[pts.length-1].x,H); ctx.closePath();
  ctx.fillStyle=g; ctx.fill();
  ctx.beginPath();
  pts.forEach(function(pt,i){ i?ctx.lineTo(pt.x,pt.y):ctx.moveTo(pt.x,pt.y); });
  ctx.strokeStyle=SPARK_COLOR; ctx.lineWidth=1.5; ctx.stroke();
}
function redrawAll(){
  Object.keys(_sparkCache).forEach(function(id){ drawSpark(id,_sparkCache[id]); });
}

/* ═══════════════════════════════════════════════════════════════
   FETCH ÍNDICES — FMP (Financial Modeling Prep)
   Free key: 250 peticiones/día. Registro gratis en 30s en:
   https://site.financialmodelingprep.com/login
═══════════════════════════════════════════════════════════════ */
var _tradCache = {};

function fetchIndicesFMP() {
  if (FMP_KEY === 'TU_FMP_API_KEY') {
    // Sin key configurada: mostrar aviso en tabla
    TRAD_ASSETS.forEach(function(a){ _tradCache[a.id] = null; });
    renderMarketTable();
    showFMPNotice();
    return Promise.resolve();
  }

  // FMP batch quote: devuelve precio actual + % cambio del día
  var symbols = TRAD_ASSETS.map(function(a){ return encodeURIComponent(a.id); }).join(',');
  var quoteUrl = 'https://financialmodelingprep.com/api/v3/quote/' + symbols + '?apikey=' + FMP_KEY;

  return fetch(quoteUrl)
    .then(function(r){ if(!r.ok) throw new Error('FMP '+r.status); return r.json(); })
    .then(function(quotes){
      var quoteMap = {};
      quotes.forEach(function(q){ quoteMap[q.symbol] = q; });

      // Fetch historical para sparkline + variaciones 1W/1M/3M/YTD
      var histPromises = TRAD_ASSETS.map(function(a){
        var histUrl = 'https://financialmodelingprep.com/api/v3/historical-price-full/' +
          encodeURIComponent(a.id) + '?apikey=' + FMP_KEY + '&timeseries=260';
        return fetch(histUrl)
          .then(function(r){ return r.ok ? r.json() : {historical:[]}; })
          .then(function(h){
            var hist = (h.historical || []).slice().reverse(); // oldest→newest
            var q    = quoteMap[a.id] || {};
            var price = q.price;
            var d1    = q.changesPercentage || null;

            function pctFromIdx(n) {
              if (hist.length <= n || !price) return null;
              var p = parseFloat(hist[hist.length-1-n].adjClose || hist[hist.length-1-n].close);
              return p ? ((price-p)/p*100) : null;
            }

            var yrStr = String(new Date().getFullYear());
            var ytdClose = null;
            for (var i=0; i<hist.length; i++){
              if ((hist[i].date||'').startsWith(yrStr)){ ytdClose=parseFloat(hist[i].adjClose||hist[i].close); break; }
            }

            _tradCache[a.id] = {
              price: price,
              d1:    d1,
              w1:    pctFromIdx(5),
              m1:    pctFromIdx(22),
              m3:    pctFromIdx(63),
              ytd:   ytdClose && price ? ((price-ytdClose)/ytdClose*100) : null,
              spark: hist.slice(-30).map(function(d){ return parseFloat(d.adjClose||d.close); })
            };
          })
          .catch(function(){ _tradCache[a.id] = quoteMap[a.id] ? { price: quoteMap[a.id].price, d1: quoteMap[a.id].changesPercentage, w1:null,m1:null,m3:null,ytd:null,spark:[] } : null; });
      });

      return Promise.all(histPromises);
    });
}

function showFMPNotice() {
  // Inserta aviso en las filas de índices
  TRAD_ASSETS.forEach(function(a){
    var sid = 'sp-trad-'+a.id.replace(/[^a-z0-9]/gi,'');
    // Marcado como "sin key" — se maneja en renderMarketTable
  });
}

/* ═══════════════════════════════════════════════════════════════
   FETCH CRYPTO — CoinGecko (dos batches con delay)
═══════════════════════════════════════════════════════════════ */
var _cgCache = {};

function fetchCGBatch(assets) {
  var ids = assets.map(function(a){ return a.id; }).join(',');
  var url = 'https://api.coingecko.com/api/v3/coins/markets' +
    '?vs_currency=usd&ids='+ids+
    '&order=market_cap_desc&per_page=25&page=1' +
    '&sparkline=true&price_change_percentage=24h,7d,30d,200d';
  return fetch(url)
    .then(function(r){ if(!r.ok) throw new Error('CG '+r.status); return r.json(); })
    .then(function(data){
      data.forEach(function(c){ _cgCache[c.id] = c; });
    });
}

function fetchAllCrypto() {
  // Batch A primero, luego batch B con 2s de delay para evitar 429
  return fetchCGBatch(CRYPTO_BATCH_A).then(function(){
    return new Promise(function(resolve){
      setTimeout(function(){
        fetchCGBatch(CRYPTO_BATCH_B).then(resolve).catch(function(e){
          console.warn('CG Batch B:', e.message);
          resolve();
        });
      }, 2000);
    });
  });
}

function getCG(id){ return _cgCache[id] || null; }

/* ── RENDER ──────────────────────────────────────────────────── */
function secHdr(label) {
  return '<tr class="section-row"><td colspan="11">'+label+'</td></tr>'+TH_ROW;
}

function mkRow(sid, icon, name, sub, price, mcap, fdv, d1, w1, m1, m3, ytd, spark) {
  _sparkCache[sid] = spark||[];
  var ratio = (fdv&&mcap) ? (fdv/mcap).toFixed(2)+'x' : '—';
  return '<tr>'+
    '<td><div class="asset-cell">'+
      '<div class="asset-icon">'+icon+'</div>'+
      '<div><div class="asset-name">'+name+'</div><div class="asset-sub">'+sub+'</div></div>'+
    '</div></td>'+
    '<td class="spark-cell"><canvas id="'+sid+'" class="spark" width="80" height="32"></canvas></td>'+
    '<td><span class="price-val">'+fmtPrice(price)+'</span></td>'+
    '<td><span class="mcap-val">'+fmtBig(mcap)+'</span></td>'+
    '<td><span class="fdv-val">'+(fdv?fmtBig(fdv):'—')+'</span></td>'+
    '<td><span class="ratio-val">'+ratio+'</span></td>'+
    '<td>'+fmtPct(d1)+'</td><td>'+fmtPct(w1)+'</td>'+
    '<td>'+fmtPct(m1)+'</td><td>'+fmtPct(m3)+'</td><td>'+fmtPct(ytd)+'</td>'+
  '</tr>';
}

function noKeyRow(name, sym) {
  return '<tr>'+
    '<td><div class="asset-cell">'+
      '<div class="asset-icon"><span style="font-size:9px;color:var(--gold)">'+sym.slice(0,2)+'</span></div>'+
      '<div><div class="asset-name">'+name+'</div><div class="asset-sub">'+sym+'</div></div>'+
    '</div></td>'+
    '<td colspan="10" style="font-size:11px;color:var(--gold);text-align:center">'+
      '⚠️ Configura FMP_KEY en el HTML para ver datos de índices'+
    '</td></tr>';
}

function skelRow(name, sym) {
  return '<tr>'+
    '<td><div class="asset-cell">'+
      '<div class="asset-icon"><span style="font-size:9px;color:var(--cyan)">'+sym[0]+'</span></div>'+
      '<div><div class="asset-name">'+name+'</div><div class="asset-sub">'+sym+'</div></div>'+
    '</div></td>'+
    '<td colspan="10" style="color:var(--t3);font-size:11px;text-align:center">⏳</td></tr>';
}

function renderMarketTable() {
  var tbody = document.getElementById('market-tbody');
  if (!tbody) return;
  var html = '';
  var noKey = FMP_KEY === 'TU_FMP_API_KEY';

  // Sección 1 — Índices
  html += secHdr('📈 Índices Tradicionales');
  TRAD_ASSETS.forEach(function(a) {
    if (noKey) { html += noKeyRow(a.name, a.sym); return; }
    var d = _tradCache[a.id];
    var sid = 'sp-trad-'+a.id.replace(/[^a-z0-9]/gi,'');
    if (!d) { html += skelRow(a.name, a.sym); return; }
    html += mkRow(sid,
      '<span style="font-size:9px;font-weight:700;color:var(--gold)">'+a.sym.slice(0,3)+'</span>',
      a.name, a.sub, d.price, null, null, d.d1, d.w1, d.m1, d.m3, d.ytd, d.spark||[]);
  });

  // Sección 2 — Top cripto
  html += secHdr('🟠 Top Cripto por Market Cap');
  CRYPTO_BATCH_A.forEach(function(a) {
    var d = getCG(a.id), sid = 'sp-cg-'+a.id.replace(/[^a-z0-9]/gi,'');
    if (!d) { html += skelRow(a.name, a.sym); return; }
    var icon = d.image?'<img src="'+d.image+'" alt="">':'<span style="font-size:9px;font-weight:700;color:var(--cyan)">'+a.sym[0]+'</span>';
    html += mkRow(sid, icon, a.name, a.sym,
      d.current_price, d.market_cap, d.fully_diluted_valuation,
      d.price_change_percentage_24h,
      d.price_change_percentage_7d_in_currency||d.price_change_percentage_7d,
      d.price_change_percentage_30d_in_currency||d.price_change_percentage_30d,
      d.price_change_percentage_200d||null, null,
      d.sparkline&&d.sparkline.price?d.sparkline.price.slice(-30):[]);
  });

  // Sección 3 — Altcoins
  html += secHdr('🔬 Altcoins Seleccionadas');
  CRYPTO_BATCH_B.forEach(function(a) {
    var d = getCG(a.id), sid = 'sp-cg-'+a.id.replace(/[^a-z0-9]/gi,'');
    if (!d) { html += skelRow(a.name, a.sym); return; }
    var icon = d.image?'<img src="'+d.image+'" alt="">':'<span style="font-size:9px;font-weight:700;color:var(--cyan)">'+a.sym[0]+'</span>';
    html += mkRow(sid, icon, a.name, a.sym,
      d.current_price, d.market_cap, d.fully_diluted_valuation,
      d.price_change_percentage_24h,
      d.price_change_percentage_7d_in_currency||d.price_change_percentage_7d,
      d.price_change_percentage_30d_in_currency||d.price_change_percentage_30d,
      d.price_change_percentage_200d||null, null,
      d.sparkline&&d.sparkline.price?d.sparkline.price.slice(-30):[]);
  });

  tbody.innerHTML = html;
  setTimeout(redrawAll, 80);
}

/* ── LOAD ────────────────────────────────────────────────────── */
function loadMarketTable() {
  var btn = document.getElementById('market-refresh-btn');
  if (btn) btn.classList.add('spinning');
  renderMarketTable();

  var cgDone=false, tradDone=false;
  function tryDone(){
    if (!cgDone||!tradDone) return;
    renderMarketTable();
    var el=document.getElementById('market-last-update');
    if (el) el.textContent='Act. '+new Date().toLocaleTimeString('es-ES');
    if (btn) btn.classList.remove('spinning');
    _marketLoaded=true;
  }

  fetchAllCrypto()
    .then(function(){ cgDone=true; renderMarketTable(); tryDone(); })
    .catch(function(e){
      console.warn('CoinGecko:', e.message);
      cgDone=true; tryDone();
      setTimeout(function(){ fetchAllCrypto().then(function(){renderMarketTable();}).catch(function(){}); }, 35000);
    });

  fetchIndicesFMP()
    .then(function(){ tradDone=true; tryDone(); })
    .catch(function(e){ console.warn('FMP:', e); tradDone=true; tryDone(); });
}

/* ── TOP WINNERS ─────────────────────────────────────────────── */
function setWinnersTF(tf){
  _winnersTF=tf;
  ['1d','7d','30d'].forEach(function(t){
    var b=document.getElementById('wbtn-'+t); if(b) b.classList.toggle('active',t===tf);
  });
  var h=document.getElementById('winners-chg-header');
  if(h) h.textContent={'1d':'Ganancia 1D','7d':'Ganancia 1S','30d':'Ganancia 1M'}[tf]||'Ganancia';
  loadWinners();
}

function loadWinners(){
  var tbody=document.getElementById('winners-tbody');
  if(tbody) tbody.innerHTML='<tr class="loading"><td colspan="6">⏳ Cargando top winners...</td></tr>';
  var tf=_winnersTF==='1d'?'24h':_winnersTF==='7d'?'7d':'30d';
  fetch('https://api.coingecko.com/api/v3/coins/markets?vs_currency=usd&order=market_cap_desc&per_page=250&page=1&sparkline=true&price_change_percentage='+tf)
    .then(function(r){ if(!r.ok) throw new Error(r.status); return r.json(); })
    .then(function(coins){
      var fld='price_change_percentage_'+tf+'_in_currency';
      var fbk='price_change_percentage_'+(_winnersTF==='1d'?'24h':tf);
      var out=coins
        .filter(function(c){ return (c.market_cap||0)>50e6&&(c[fld]||c[fbk]||0)>0; })
        .sort(function(a,b){ return (b[fld]||b[fbk]||0)-(a[fld]||a[fbk]||0); })
        .slice(0,15);
      var html='';
      out.forEach(function(c,i){
        var chg=c[fld]||c[fbk]||0;
        var sid='wsp-'+c.id.replace(/[^a-z0-9]/gi,'');
        _sparkCache[sid]=(c.sparkline&&c.sparkline.price)?c.sparkline.price:[];
        html+='<tr>'+
          '<td class="rank-cell">'+(i+1)+'</td>'+
          '<td><div class="asset-cell">'+
            '<div class="asset-icon">'+(c.image?'<img src="'+c.image+'" alt="">':c.symbol[0].toUpperCase())+'</div>'+
            '<div><div class="asset-name">'+c.name+'</div><div class="asset-sub">'+c.symbol.toUpperCase()+'</div></div>'+
          '</div></td>'+
          '<td class="spark-cell"><canvas id="'+sid+'" class="spark" width="80" height="32"></canvas></td>'+
          '<td><span class="price-val">'+fmtPrice(c.current_price)+'</span></td>'+
          '<td><span class="winner-mcap">'+fmtBig(c.market_cap)+'</span></td>'+
          '<td><span class="winner-chg">+'+chg.toFixed(2)+'%</span></td>'+
        '</tr>';
      });
      tbody.innerHTML=html;
      setTimeout(function(){ out.forEach(function(c){ var s='wsp-'+c.id.replace(/[^a-z0-9]/gi,''); drawSpark(s,_sparkCache[s]); }); },80);
    })
    .catch(function(e){
      console.error('Winners:',e);
      if(tbody) tbody.innerHTML='<tr class="loading"><td colspan="6">⚠️ CoinGecko rate limit — reintentando en 35s...</td></tr>';
      setTimeout(loadWinners,35000);
    });
}

/* ── TAB TRIGGER + AUTO REFRESH ─────────────────────────────── */
var _origSwitchTab=switchTab;
switchTab=function(tab){
  _origSwitchTab(tab);
  if(tab==='etf'&&!_marketLoaded){
    setTimeout(function(){ loadMarketTable(); loadWinners(); },200);
  }
};
setInterval(function(){ if(_currentTab==='etf'){ loadMarketTable(); loadWinners(); } },3600000);
</script>
</body>
</html>
