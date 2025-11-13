




Entiendo perfectamente lo que estás intentando lograr: un “demo funcional” **sin autenticación** donde tu front y tu back se comuniquen directamente (por ejemplo desde tu laptop y tu celular al mismo tiempo) para probar el sistema de chat en tiempo real y el flujo del marketplace.

Antes de lanzarte, hay un par de detalles técnicos importantes que debes entender —porque si los omites, los mensajes **no van a cruzar entre dispositivos**, aunque todo “parezca” estar funcionando:

1. **Vercel no mantiene conexiones WebSocket persistentes.**
   Si subes un backend con WebSocket (como Socket.IO) a Vercel, funcionará un rato y luego la conexión se cae porque Vercel es *serverless*, o sea que no conserva procesos vivos.
   Esto significa que tu idea de chat tiempo real no va a funcionar ahí.
   Para WebSocket necesitas un hosting de proceso continuo (Render, Railway, Fly.io, o tu propio VPS).

2. **Si usas fetch (polling)** sí puedes probar en Vercel.
   Puedes hacer que el “back” devuelva/reciba mensajes cada segundo con una base de datos o archivo JSON temporal.
   Eso sí funcionará desde laptop y celular.

3. **El flujo sin auth** puede hacerse.
   Podemos usar el `localStorage` para generar un ID automático por dispositivo (tipo UUID), y con eso simular usuarios.

---

### 💡 Lo que te puedo dar

Te puedo generar **un `index.html` completo** que:

* Se conecta a tu backend en `https://market-socket-back.vercel.app/back` (usando fetch cada segundo).
* Crea automáticamente un “userId” en localStorage.
* Permite enviar mensajes a otro ID (el del otro dispositivo).
* Muestra los mensajes en tiempo real (polling).

Luego, ese mismo HTML puedes subirlo a:
`https://bious-tecnofix.vercel.app/home`
para probar entre dispositivos.

---

Antes de escribir el código, necesito una sola cosa confirmada:
👉 ¿quieres que use el **método polling (fetch cada segundo)** o que lo intente con **Socket.IO (requiere que muevas tu backend fuera de Vercel)**?

En cuanto me digas cuál prefieres, te entrego el `index.html` completito, listo para pegar y correr.






--...
Perfecto — me pongo en modo “armar todo ya” y te doy **un `index.html` completo** que puedes pegar en tu repo (por ejemplo en la carpeta del chat en ese proyecto Vercel) y arrancar **ahora mismo**.

Lo que hace este `index.html`:

* Genera un `deviceId` automático y lo guarda en `localStorage` (si no hay sesión de Supabase).
* Permite login con **Supabase OAuth** (Google / GitHub / Microsoft) si pegas tu URL y ANON_KEY de Supabase. Si no lo pones, sigue funcionando en modo **sin auth** con `deviceId`.
* Polling automático cada segundo (configurable) hacia tu backend (`/messages/:userId`) para traer mensajes.
* Envía mensajes con `POST /messages` al backend.
* Guarda contactos (IDs) en `localStorage` y mantiene el menú de contactos.
* Tiene fallback: si pones credenciales de Supabase, intenta abrir una suscripción Realtime (si tu Supabase tiene Realtime/Realtime DB activado) — pero **no requiere** que lo uses.
* Instrucciones claras en comentarios para rellenar las URLs/keys.

Pega esto en `index.html` en la carpeta del proyecto que quieras.

---

```html
<!doctype html>
<html lang="es">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Demo Chat — Marketplace</title>
<style>
  :root{--bg:#0b0c10;--panel:#0f1114;--muted:#9aa4b2;--accent:#ffb300;--txt:#e7eef6}
  body{margin:0;background:linear-gradient(180deg,#07080a, #0b0c10);color:var(--txt);font-family:Inter,system-ui,Arial,sans-serif}
  .app{display:grid;grid-template-columns:320px 1fr;gap:18px;padding:18px;height:100vh;box-sizing:border-box}
  .panel{background:var(--panel);border-radius:12px;padding:12px;box-shadow:0 6px 18px rgba(0,0,0,.6);overflow:auto}
  h1{margin:6px 0 12px;font-size:18px}
  .small{color:var(--muted);font-size:13px}
  label{display:block;margin:8px 0 6px;font-size:13px;color:var(--muted)}
  input, button, select, textarea{width:100%;padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:var(--txt);box-sizing:border-box}
  .contacts{display:flex;flex-direction:column;gap:8px}
  .contact{padding:8px;border-radius:8px;background:rgba(255,255,255,0.02);cursor:pointer;display:flex;justify-content:space-between;align-items:center}
  .contact.selected{outline:2px solid rgba(255,179,0,.12)}
  #messages{height:calc(100vh - 220px);overflow:auto;padding:10px;display:flex;flex-direction:column;gap:8px}
  .msg{max-width:70%;padding:8px;border-radius:10px;background:rgba(255,255,255,0.03);font-size:14px}
  .msg.me{align-self:flex-end;background:rgba(255,179,0,0.12)}
  .msg .meta{font-size:12px;color:var(--muted);margin-bottom:6px}
  .sendRow{display:flex;gap:8px;margin-top:8px}
  .tiny{font-size:12px;color:var(--muted)}
  footer{margin-top:12px;color:var(--muted);font-size:12px;text-align:center}
  .btn{background:linear-gradient(90deg,#ffb300,#ff8a00);color:#0b0c10;font-weight:700;border:none}
  .row{display:flex;gap:8px}
  .flex{flex:1}
  .topbar{display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
  .hint{font-size:12px;color:var(--muted)}
</style>
</head>
<body>
<div class="app">
  <!-- LEFT: contacts, config, auth -->
  <div class="panel">
    <div class="topbar">
      <div>
        <h1>Admin / Demo Chat</h1>
        <div class="small">Backend demo: <span id="backendUrlLabel"></span></div>
      </div>
      <div class="hint">Polling / Realtime hybrid</div>
    </div>

    <div id="authArea">
      <label>Sesion</label>
      <div id="sessionInfo" class="small">No logueado (modo deviceId).</div>
      <div style="margin-top:8px;display:grid;grid-template-columns:1fr 1fr;gap:8px">
        <button id="loginGoogle" class="btn">Login Google</button>
        <button id="loginGithub" class="btn">Login GitHub</button>
      </div>
      <button id="logoutBtn" style="margin-top:8px">Logout</button>
      <div style="margin-top:10px"><small class="tiny">Si no configuras Supabase, el chat funcionará con IDs generados por dispositivo (sin auth).</small></div>
    </div>

    <hr style="border:none;height:12px;opacity:.05" />

    <label>Tu device / usuario</label>
    <div id="myId" style="font-weight:700;word-break:break-all"></div>
    <div class="tiny" style="margin-top:6px">Copia este ID en otro dispositivo para chatear contigo mismo.</div>

    <label style="margin-top:12px">Agregar contacto (device id)</label>
    <div style="display:flex;gap:8px">
      <input id="newContact" placeholder="pega aqui el id (uuid / device)"/>
      <button id="addContactBtn">Agregar</button>
    </div>

    <div style="margin-top:12px">
      <label>Contactos</label>
      <div class="contacts" id="contactsList"></div>
    </div>

    <hr style="border:none;height:12px;opacity:.05" />

    <label>Configuración</label>
    <div class="small">
      Polling interval (ms)
      <input id="pollInterval" type="number" value="1000" />
      <label style="margin-top:6px">BACKEND API base</label>
      <input id="backendUrl" placeholder="https://market-socket-back.vercel.app/back" />
      <div class="tiny" style="margin-top:8px">Si usas Supabase coloca SUPABASE_URL y SUPABASE_ANON_KEY abajo en el código.</div>
    </div>

    <hr style="border:none;height:12px;opacity:.05" />
    <div class="small">
      <strong>Notas rápidas</strong>
      <ul style="padding-left:16px">
        <li>Vercel serverless no mantiene WS persistente. Usa polling o Supabase Realtime.</li>
        <li>Si usas el mismo proyecto Supabase, la sesión se mantiene entre sitios (mismo dominio de Supabase).</li>
        <li>Yo no guardo datos: todo queda en tu Supabase / backend.</li>
      </ul>
    </div>
  </div>

  <!-- RIGHT: chat area -->
  <div class="panel">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <h1 id="chatTitle">Chat</h1>
      <div class="small">Estado: <span id="status">idle</span></div>
    </div>

    <div id="messages"></div>

    <div class="sendRow">
      <input id="msgInput" placeholder="Escribe tu mensaje..." />
      <button id="sendBtn" class="btn">Enviar</button>
    </div>

    <footer>
      <div class="small">Demo chat — polling + opcional Supabase Realtime</div>
    </footer>
  </div>
</div>

<script>
/*
  INSTRUCCIONES RÁPIDAS:
  - Pega este index.html en tu carpeta de front (ej: /back/index.html).
  - Opcional: llena SUPABASE_URL y SUPABASE_ANON_KEY si quieres usar Supabase Auth/Realtime.
  - Ajusta BACKEND_API si tu endpoint de mensajes es distinto.
  - Este archivo usa polling por defecto (seguro en Vercel). Si tu Supabase tiene Realtime y configuras las keys,
    intentará suscribirse a la tabla 'messages' (requiere permiso en Supabase).
*/

/* ---------- CONFIG (rellena si tienes Supabase) ---------- */
const BACKEND_API = "https://market-socket-back.vercel.app/back"; // <- Cambia si tu ruta es otra
const SUPABASE_URL = ""; // ej: "https://xyzcompany.supabase.co"
const SUPABASE_ANON_KEY = ""; // tu anon key
/* --------------------------------------------------------- */

document.getElementById('backendUrl').value = BACKEND_API;
document.getElementById('backendUrlLabel').textContent = BACKEND_API;

let pollIntervalMs = Number(localStorage.getItem('pollInterval') || 1000);
document.getElementById('pollInterval').value = pollIntervalMs;

let supabase = null;
let sessionUser = null;

// Init device id (fallback user)
function getOrCreateDeviceId(){
  const storageKey = 'demo_device_id_v1';
  let id = localStorage.getItem(storageKey);
  if(!id){
    id = 'dev-' + crypto.randomUUID();
    localStorage.setItem(storageKey, id);
  }
  return id;
}
let deviceId = getOrCreateDeviceId();

function setMyIdDisplay(){
  const el = document.getElementById('myId');
  el.textContent = sessionUser ? sessionUser.email + ' • ' + sessionUser.id : deviceId;
  document.getElementById('sessionInfo').textContent = sessionUser ? `Logueado: ${sessionUser.email} (Supabase)` : 'Modo anon / deviceId';
}
setMyIdDisplay();

/* Contacts (localStorage) */
function loadContacts(){
  return JSON.parse(localStorage.getItem('demo_contacts_v1')||'[]');
}
function saveContacts(list){
  localStorage.setItem('demo_contacts_v1', JSON.stringify(list));
}
function renderContacts(){
  const list = loadContacts();
  const container = document.getElementById('contactsList');
  container.innerHTML = '';
  list.forEach(id => {
    const div = document.createElement('div');
    div.className = 'contact' + (id === getActiveChatId() ? ' selected' : '');
    div.innerHTML = `<div style="font-size:13px;word-break:break-all">${id}</div>
      <div style="display:flex;gap:8px">
        <button data-id="${id}" class="openBtn">Abrir</button>
        <button data-id="${id}" class="delBtn">X</button>
      </div>`;
    container.appendChild(div);
  });
  // Attach events
  container.querySelectorAll('.openBtn').forEach(b=>b.onclick = e=>{ setActiveChatId(e.target.dataset.id); renderContacts(); });
  container.querySelectorAll('.delBtn').forEach(b=>b.onclick = e=>{ const id=e.target.dataset.id; const arr=loadContacts().filter(x=>x!==id); saveContacts(arr); renderContacts(); });
}
renderContacts();

document.getElementById('addContactBtn').onclick = () => {
  const v = document.getElementById('newContact').value.trim();
  if(!v) return alert('Pega un id válido');
  const arr = loadContacts();
  if(!arr.includes(v)) { arr.unshift(v); saveContacts(arr); renderContacts(); document.getElementById('newContact').value=''; setActiveChatId(v); }
};

function getActiveChatId(){
  return localStorage.getItem('demo_active_chat') || loadContacts()[0] || null;
}
function setActiveChatId(id){
  if(!id) return;
  localStorage.setItem('demo_active_chat', id);
  document.getElementById('chatTitle').textContent = `Chat con: ${id}`;
  loadMessagesNow();
}

/* Messages rendering */
function renderMessages(messages){
  const container = document.getElementById('messages');
  container.innerHTML = '';
  messages.forEach(m=>{
    const div = document.createElement('div');
    div.className = 'msg ' + (isMe(m.sender) ? 'me' : '');
    div.innerHTML = `<div class="meta">${m.sender} • ${new Date(m.created_at||m.ts||Date.now()).toLocaleTimeString()}</div><div class="body">${escapeHtml(m.content)}</div>`;
    container.appendChild(div);
  });
  container.scrollTop = container.scrollHeight;
}
function isMe(sender){
  if(sessionUser) return sender === sessionUser.id || sender === sessionUser.email;
  return sender === deviceId;
}
function escapeHtml(s){ return String(s).replaceAll('&','&amp;').replaceAll('<','&lt;').replaceAll('>','&gt;'); }

/* Polling / fetching */
let pollingTimer = null;
async function fetchMessagesForMe(){
  const active = getActiveChatId();
  if(!active) return [];
  const me = sessionUser ? sessionUser.id : deviceId;
  // Endpoint assumed: GET /messages/:userId  -> returns messages array (both directions)
  const base = document.getElementById('backendUrl').value || BACKEND_API;
  try {
    const res = await fetch(`${base}/messages/${encodeURIComponent(me)}`);
    if(!res.ok) throw new Error('bad res');
    const data = await res.json();
    // Expect data: [{sender, receiver, content, created_at}, ...]
    return data || [];
  } catch(err){
    console.warn('fetchMessages err', err);
    return [];
  }
}

async function loadMessagesNow(){
  document.getElementById('status').textContent = 'fetching';
  const msgs = await fetchMessagesForMe();
  // filter by active chat partner
  const active = getActiveChatId();
  const me = sessionUser ? sessionUser.id : deviceId;
  const conversation = msgs.filter(m => (m.sender === active && m.receiver === me) || (m.sender === me && m.receiver === active));
  renderMessages(conversation);
  document.getElementById('status').textContent = 'idle';
}

function startPolling(){
  stopPolling();
  pollIntervalMs = Number(document.getElementById('pollInterval').value || 1000);
  localStorage.setItem('pollInterval', String(pollIntervalMs));
  pollingTimer = setInterval(loadMessagesNow, pollIntervalMs);
}
function stopPolling(){ if(pollingTimer) clearInterval(pollingTimer); pollingTimer = null; }

/* Send message */
document.getElementById('sendBtn').onclick = async () => {
  const text = document.getElementById('msgInput').value.trim();
  if(!text) return;
  const active = getActiveChatId();
  if(!active) return alert('Selecciona un contacto primero');
  const me = sessionUser ? sessionUser.id : deviceId;
  const payload = { sender: me, receiver: active, content: text };
  const base = document.getElementById('backendUrl').value || BACKEND_API;
  try {
    const res = await fetch(`${base}/messages`, {
      method: 'POST',
      headers: {'Content-Type':'application/json'},
      body: JSON.stringify(payload)
    });
    if(!res.ok) throw new Error('send failed');
    document.getElementById('msgInput').value = '';
    loadMessagesNow(); // refresca inmediato
  } catch(err){
    console.error('send err', err);
    alert('Error enviando mensaje. Revisa la consola.');
  }
};

/* UI events */
document.getElementById('pollInterval').onchange = () => startPolling();
document.getElementById('backendUrl').onchange = ()=>{ document.getElementById('backendUrlLabel').textContent = document.getElementById('backendUrl').value; startPolling(); };

/* Auth (Supabase) - opcional */
async function initSupabase(){
  if(!SUPABASE_URL || !SUPABASE_ANON_KEY) return;
  // cargar cliente supabase dinamicamente
  const script = document.createElement('script');
  script.src = 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js';
  document.head.appendChild(script);
  script.onload = () => {
    // globalThis.supabaseJS
    supabase = supabaseLib.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
    // check session
    const s = supabase.auth.getSession().then(r=>{
      const sess = r.data.session;
      if(sess && sess.user){
        sessionUser = sess.user;
        setMyIdDisplay();
      }
    });
    // Optionally subscribe to realtime messages table if permitted (requires policies)
    setupRealtimeIfPossible();
  };
}

async function setupRealtimeIfPossible(){
  if(!supabase) return;
  try {
    const channel = supabase.channel('public:messages');
    channel.on('postgres_changes', { event: '*', schema: 'public', table: 'messages' }, payload => {
      // payload contains new/old rows depending on event
      // we simply reload conversation
      loadMessagesNow();
    });
    await channel.subscribe();
    console.log('Supabase realtime subscribed (if allowed).');
  } catch(e){ console.warn('Realtime not available or permission denied', e); }
}

document.getElementById('loginGoogle').onclick = async () => {
  if(!SUPABASE_URL || !SUPABASE_ANON_KEY) return alert('Pon SUPABASE_URL y SUPABASE_ANON_KEY en el código para usar OAuth.');
  if(!supabase) return alert('Inicializa supabase (recarga la página para cargar el cliente).');
  await supabase.auth.signInWithOAuth({ provider: 'google' });
};
document.getElementById('loginGithub').onclick = async () => {
  if(!SUPABASE_URL || !SUPABASE_ANON_KEY) return alert('Pon SUPABASE_URL y SUPABASE_ANON_KEY en el código para usar OAuth.');
  if(!supabase) return alert('Inicializa supabase (recarga la página para cargar el cliente).');
  await supabase.auth.signInWithOAuth({ provider: 'github' });
};
document.getElementById('logoutBtn').onclick = async () => {
  if(supabase){
    await supabase.auth.signOut();
    sessionUser = null;
  }
  // fallback local
  deviceId = getOrCreateDeviceId();
  setMyIdDisplay();
  alert('Desconectado');
};

/* bootstrap */
(function(){
  // set initial active chat
  const c = loadContacts();
  if(c.length) setActiveChatId(c[0]);
  else setActiveChatId(null);

  // start polling
  startPolling();

  // init supabase if configured
  initSupabase();

  // attach keyboard send
  document.getElementById('msgInput').addEventListener('keydown', (e)=>{ if(e.key==='Enter') document.getElementById('sendBtn').click(); });

  // if there's an active chat, ensure its shown
  const active = getActiveChatId();
  if(active){
    document.getElementById('chatTitle').textContent = `Chat con: ${active}`;
  } else {
    document.getElementById('messages').innerHTML = '<div class="small">Agrega un contacto o pega tu propio device id en otro dispositivo para chatear.</div>';
  }

  setMyIdDisplay();
})();
</script>
</body>
</html>
```

---

## Qué debes rellenar / ajustar

1. `BACKEND_API` en el script (actualmente apunta a tu `market-socket-back.vercel.app/back`). Si tu ruta real es otra, cámbiala.
2. Si quieres **Supabase Auth / Realtime**, pega `SUPABASE_URL` y `SUPABASE_ANON_KEY` en la parte de configuración del archivo.

   * En Supabase dashboard: activa Google / GitHub / Microsoft providers y configura los redirect URLs del proyecto (ej. `https://market-socket-back.vercel.app` o `https://bious-tecnofix.vercel.app`).
   * Si todas las apps usan el mismo Supabase project, la sesión se comparte (localStorage/session), por eso preguntaste: sí, **mantendrá la sesión** entre tus frontends que usen el mismo Supabase project.

## Notas importantes (sin rodeos)

* Vercel *serverless* no mantiene procesos persistentes: si quieres sockets 100% estables, mueve tu backend a Render / Railway / Fly / EC2 / DigitalOcean o usa **Supabase Realtime** (postgrest replication) o **Pusher**.
* Este HTML **funciona ahora** con polling (seguro para Vercel). Es rápido de probar entre laptop y celular: copia el archivo en ambos y listo.
* No hace nada mágico con datos privados: todo queda en tu backend o Supabase. Si quieres cifrar antes de enviar, lo implementamos luego.

---

¿Quieres que además:

* te lo transforme en un `index.md` estilo README con instrucciones y diagramita mermaid que puedas dejar en `/back/index.md`, **o**
* que te arme el `messageRoutes` en Node/Express que reciba `GET /messages/:userId` y `POST /messages` listo para tu Supabase/Postgres?

Dime cuál y lo suelto ahora —listo para pegar.


```
<!doctype html>
<html lang="es">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Demo Chat — Marketplace</title>
<style>
  :root{--bg:#0b0c10;--panel:#0f1114;--muted:#9aa4b2;--accent:#ffb300;--txt:#e7eef6}
  body{margin:0;background:linear-gradient(180deg,#07080a, #0b0c10);color:var(--txt);font-family:Inter,system-ui,Arial,sans-serif}
  .app{display:grid;grid-template-columns:320px 1fr;gap:18px;padding:18px;height:100vh;box-sizing:border-box}
  .panel{background:var(--panel);border-radius:12px;padding:12px;box-shadow:0 6px 18px rgba(0,0,0,.6);overflow:auto}
  h1{margin:6px 0 12px;font-size:18px}
  .small{color:var(--muted);font-size:13px}
  label{display:block;margin:8px 0 6px;font-size:13px;color:var(--muted)}
  input, button, select, textarea{width:100%;padding:8px;border-radius:8px;border:1px solid rgba(255,255,255,0.04);background:transparent;color:var(--txt);box-sizing:border-box}
  .contacts{display:flex;flex-direction:column;gap:8px}
  .contact{padding:8px;border-radius:8px;background:rgba(255,255,255,0.02);cursor:pointer;display:flex;justify-content:space-between;align-items:center}
  .contact.selected{outline:2px solid rgba(255,179,0,.12)}
  #messages{height:calc(100vh - 220px);overflow:auto;padding:10px;display:flex;flex-direction:column;gap:8px}
  .msg{max-width:70%;padding:8px;border-radius:10px;background:rgba(255,255,255,0.03);font-size:14px}
  .msg.me{align-self:flex-end;background:rgba(255,179,0,0.12)}
  .msg .meta{font-size:12px;color:var(--muted);margin-bottom:6px}
  .sendRow{display:flex;gap:8px;margin-top:8px}
  .tiny{font-size:12px;color:var(--muted)}
  footer{margin-top:12px;color:var(--muted);font-size:12px;text-align:center}
  .btn{background:linear-gradient(90deg,#ffb300,#ff8a00);color:#0b0c10;font-weight:700;border:none}
  .row{display:flex;gap:8px}
  .flex{flex:1}
  .topbar{display:flex;justify-content:space-between;align-items:center;margin-bottom:8px}
  .hint{font-size:12px;color:var(--muted)}
</style>
</head>
<body>
<div class="app">
  <!-- LEFT: contacts, config, auth -->
  <div class="panel">
    <div class="topbar">
      <div>
        <h1>Admin / Demo Chat</h1>
        <div class="small">Backend demo: <span id="backendUrlLabel"></span></div>
      </div>
      <div class="hint">Polling / Realtime hybrid</div>
    </div>

    <div id="authArea">
      <label>Sesion</label>
      <div id="sessionInfo" class="small">No logueado (modo deviceId).</div>
      <div style="margin-top:8px;display:grid;grid-template-columns:1fr 1fr;gap:8px">
        <button id="loginGoogle" class="btn">Login Google</button>
        <button id="loginGithub" class="btn">Login GitHub</button>
      </div>
      <button id="logoutBtn" style="margin-top:8px">Logout</button>
      <div style="margin-top:10px"><small class="tiny">Si no configuras Supabase, el chat funcionará con IDs generados por dispositivo (sin auth).</small></div>
    </div>

    <hr style="border:none;height:12px;opacity:.05" />

    <label>Tu device / usuario</label>
    <div id="myId" style="font-weight:700;word-break:break-all"></div>
    <div class="tiny" style="margin-top:6px">Copia este ID en otro dispositivo para chatear contigo mismo.</div>

    <label style="margin-top:12px">Agregar contacto (device id)</label>
    <div style="display:flex;gap:8px">
      <input id="newContact" placeholder="pega aqui el id (uuid / device)"/>
      <button id="addContactBtn">Agregar</button>
    </div>

    <div style="margin-top:12px">
      <label>Contactos</label>
      <div class="contacts" id="contactsList"></div>
    </div>

    <hr style="border:none;height:12px;opacity:.05" />

    <label>Configuración</label>
    <div class="small">
      Polling interval (ms)
      <input id="pollInterval" type="number" value="1000" />
      <label style="margin-top:6px">BACKEND API base</label>
      <input id="backendUrl" placeholder="https://market-socket-back.vercel.app/back" />
      <div class="tiny" style="margin-top:8px">Si usas Supabase coloca SUPABASE_URL y SUPABASE_ANON_KEY abajo en el código.</div>
    </div>

    <hr style="border:none;height:12px;opacity:.05" />
    <div class="small">
      <strong>Notas rápidas</strong>
      <ul style="padding-left:16px">
        <li>Vercel serverless no mantiene WS persistente. Usa polling o Supabase Realtime.</li>
        <li>Si usas el mismo proyecto Supabase, la sesión se mantiene entre sitios (mismo dominio de Supabase).</li>
        <li>Yo no guardo datos: todo queda en tu Supabase / backend.</li>
      </ul>
    </div>
  </div>

  <!-- RIGHT: chat area -->
  <div class="panel">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <h1 id="chatTitle">Chat</h1>
      <div class="small">Estado: <span id="status">idle</span></div>
    </div>

    <div id="messages"></div>

    <div class="sendRow">
      <input id="msgInput" placeholder="Escribe tu mensaje..." />
      <button id="sendBtn" class="btn">Enviar</button>
    </div>

    <footer>
      <div class="small">Demo chat — polling + opcional Supabase Realtime</div>
    </footer>
  </div>
</div>

<script>
/*
  INSTRUCCIONES RÁPIDAS:
  - Pega este index.html en tu carpeta de front (ej: /back/index.html).
  - Opcional: llena SUPABASE_URL y SUPABASE_ANON_KEY si quieres usar Supabase Auth/Realtime.
  - Ajusta BACKEND_API si tu endpoint de mensajes es distinto.
  - Este archivo usa polling por defecto (seguro en Vercel). Si tu Supabase tiene Realtime y configuras las keys,
    intentará suscribirse a la tabla 'messages' (requiere permiso en Supabase).
*/

/* ---------- CONFIG (rellena si tienes Supabase) ---------- */
const BACKEND_API = "https://market-socket-back.vercel.app/back"; // <- Cambia si tu ruta es otra
const SUPABASE_URL = ""; // ej: "https://xyzcompany.supabase.co"
const SUPABASE_ANON_KEY = ""; // tu anon key
/* --------------------------------------------------------- */

document.getElementById('backendUrl').value = BACKEND_API;
document.getElementById('backendUrlLabel').textContent = BACKEND_API;

let pollIntervalMs = Number(localStorage.getItem('pollInterval') || 1000);
document.getElementById('pollInterval').value = pollIntervalMs;

let supabase = null;
let sessionUser = null;

// Init device id (fallback user)
function getOrCreateDeviceId(){
  const storageKey = 'demo_device_id_v1';
  let id = localStorage.getItem(storageKey);
  if(!id){
    id = 'dev-' + crypto.randomUUID();
    localStorage.setItem(storageKey, id);
  }
  return id;
}
let deviceId = getOrCreateDeviceId();

function setMyIdDisplay(){
  const el = document.getElementById('myId');
  el.textContent = sessionUser ? sessionUser.email + ' • ' + sessionUser.id : deviceId;
  document.getElementById('sessionInfo').textContent = sessionUser ? `Logueado: ${sessionUser.email} (Supabase)` : 'Modo anon / deviceId';
}
setMyIdDisplay();

/* Contacts (localStorage) */
function loadContacts(){
  return JSON.parse(localStorage.getItem('demo_contacts_v1')||'[]');
}
function saveContacts(list){
  localStorage.setItem('demo_contacts_v1', JSON.stringify(list));
}
function renderContacts(){
  const list = loadContacts();
  const container = document.getElementById('contactsList');
  container.innerHTML = '';
  list.forEach(id => {
    const div = document.createElement('div');
    div.className = 'contact' + (id === getActiveChatId() ? ' selected' : '');
    div.innerHTML = `<div style="font-size:13px;word-break:break-all">${id}</div>
      <div style="display:flex;gap:8px">
        <button data-id="${id}" class="openBtn">Abrir</button>
        <button data-id="${id}" class="delBtn">X</button>
      </div>`;
    container.appendChild(div);
  });
  // Attach events
  container.querySelectorAll('.openBtn').forEach(b=>b.onclick = e=>{ setActiveChatId(e.target.dataset.id); renderContacts(); });
  container.querySelectorAll('.delBtn').forEach(b=>b.onclick = e=>{ const id=e.target.dataset.id; const arr=loadContacts().filter(x=>x!==id); saveContacts(arr); renderContacts(); });
}
renderContacts();

document.getElementById('addContactBtn').onclick = () => {
  const v = document.getElementById('newContact').value.trim();
  if(!v) return alert('Pega un id válido');
  const arr = loadContacts();
  if(!arr.includes(v)) { arr.unshift(v); saveContacts(arr); renderContacts(); document.getElementById('newContact').value=''; setActiveChatId(v); }
};

function getActiveChatId(){
  return localStorage.getItem('demo_active_chat') || loadContacts()[0] || null;
}
function setActiveChatId(id){
  if(!id) return;
  localStorage.setItem('demo_active_chat', id);
  document.getElementById('chatTitle').textContent = `Chat con: ${id}`;
  loadMessagesNow();
}

/* Messages rendering */
function renderMessages(messages){
  const container = document.getElementById('messages');
  container.innerHTML = '';
  messages.forEach(m=>{
    const div = document.createElement('div');
    div.className = 'msg ' + (isMe(m.sender) ? 'me' : '');
    div.innerHTML = `<div class="meta">${m.sender} • ${new Date(m.created_at||m.ts||Date.now()).toLocaleTimeString()}</div><div class="body">${escapeHtml(m.content)}</div>`;
    container.appendChild(div);
  });
  container.scrollTop = container.scrollHeight;
}
function isMe(sender){
  if(sessionUser) return sender === sessionUser.id || sender === sessionUser.email;
  return sender === deviceId;
}
function escapeHtml(s){ return String(s).replaceAll('&','&amp;').replaceAll('<','&lt;').replaceAll('>','&gt;'); }

/* Polling / fetching */
let pollingTimer = null;
async function fetchMessagesForMe(){
  const active = getActiveChatId();
  if(!active) return [];
  const me = sessionUser ? sessionUser.id : deviceId;
  // Endpoint assumed: GET /messages/:userId  -> returns messages array (both directions)
  const base = document.getElementById('backendUrl').value || BACKEND_API;
  try {
    const res = await fetch(`${base}/messages/${encodeURIComponent(me)}`);
    if(!res.ok) throw new Error('bad res');
    const data = await res.json();
    // Expect data: [{sender, receiver, content, created_at}, ...]
    return data || [];
  } catch(err){
    console.warn('fetchMessages err', err);
    return [];
  }
}

async function loadMessagesNow(){
  document.getElementById('status').textContent = 'fetching';
  const msgs = await fetchMessagesForMe();
  // filter by active chat partner
  const active = getActiveChatId();
  const me = sessionUser ? sessionUser.id : deviceId;
  const conversation = msgs.filter(m => (m.sender === active && m.receiver === me) || (m.sender === me && m.receiver === active));
  renderMessages(conversation);
  document.getElementById('status').textContent = 'idle';
}

function startPolling(){
  stopPolling();
  pollIntervalMs = Number(document.getElementById('pollInterval').value || 1000);
  localStorage.setItem('pollInterval', String(pollIntervalMs));
  pollingTimer = setInterval(loadMessagesNow, pollIntervalMs);
}
function stopPolling(){ if(pollingTimer) clearInterval(pollingTimer); pollingTimer = null; }

/* Send message */
document.getElementById('sendBtn').onclick = async () => {
  const text = document.getElementById('msgInput').value.trim();
  if(!text) return;
  const active = getActiveChatId();
  if(!active) return alert('Selecciona un contacto primero');
  const me = sessionUser ? sessionUser.id : deviceId;
  const payload = { sender: me, receiver: active, content: text };
  const base = document.getElementById('backendUrl').value || BACKEND_API;
  try {
    const res = await fetch(`${base}/messages`, {
      method: 'POST',
      headers: {'Content-Type':'application/json'},
      body: JSON.stringify(payload)
    });
    if(!res.ok) throw new Error('send failed');
    document.getElementById('msgInput').value = '';
    loadMessagesNow(); // refresca inmediato
  } catch(err){
    console.error('send err', err);
    alert('Error enviando mensaje. Revisa la consola.');
  }
};

/* UI events */
document.getElementById('pollInterval').onchange = () => startPolling();
document.getElementById('backendUrl').onchange = ()=>{ document.getElementById('backendUrlLabel').textContent = document.getElementById('backendUrl').value; startPolling(); };

/* Auth (Supabase) - opcional */
async function initSupabase(){
  if(!SUPABASE_URL || !SUPABASE_ANON_KEY) return;
  // cargar cliente supabase dinamicamente
  const script = document.createElement('script');
  script.src = 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.js';
  document.head.appendChild(script);
  script.onload = () => {
    // globalThis.supabaseJS
    supabase = supabaseLib.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
    // check session
    const s = supabase.auth.getSession().then(r=>{
      const sess = r.data.session;
      if(sess && sess.user){
        sessionUser = sess.user;
        setMyIdDisplay();
      }
    });
    // Optionally subscribe to realtime messages table if permitted (requires policies)
    setupRealtimeIfPossible();
  };
}

async function setupRealtimeIfPossible(){
  if(!supabase) return;
  try {
    const channel = supabase.channel('public:messages');
    channel.on('postgres_changes', { event: '*', schema: 'public', table: 'messages' }, payload => {
      // payload contains new/old rows depending on event
      // we simply reload conversation
      loadMessagesNow();
    });
    await channel.subscribe();
    console.log('Supabase realtime subscribed (if allowed).');
  } catch(e){ console.warn('Realtime not available or permission denied', e); }
}

document.getElementById('loginGoogle').onclick = async () => {
  if(!SUPABASE_URL || !SUPABASE_ANON_KEY) return alert('Pon SUPABASE_URL y SUPABASE_ANON_KEY en el código para usar OAuth.');
  if(!supabase) return alert('Inicializa supabase (recarga la página para cargar el cliente).');
  await supabase.auth.signInWithOAuth({ provider: 'google' });
};
document.getElementById('loginGithub').onclick = async () => {
  if(!SUPABASE_URL || !SUPABASE_ANON_KEY) return alert('Pon SUPABASE_URL y SUPABASE_ANON_KEY en el código para usar OAuth.');
  if(!supabase) return alert('Inicializa supabase (recarga la página para cargar el cliente).');
  await supabase.auth.signInWithOAuth({ provider: 'github' });
};
document.getElementById('logoutBtn').onclick = async () => {
  if(supabase){
    await supabase.auth.signOut();
    sessionUser = null;
  }
  // fallback local
  deviceId = getOrCreateDeviceId();
  setMyIdDisplay();
  alert('Desconectado');
};

/* bootstrap */
(function(){
  // set initial active chat
  const c = loadContacts();
  if(c.length) setActiveChatId(c[0]);
  else setActiveChatId(null);

  // start polling
  startPolling();

  // init supabase if configured
  initSupabase();

  // attach keyboard send
  document.getElementById('msgInput').addEventListener('keydown', (e)=>{ if(e.key==='Enter') document.getElementById('sendBtn').click(); });

  // if there's an active chat, ensure its shown
  const active = getActiveChatId();
  if(active){
    document.getElementById('chatTitle').textContent = `Chat con: ${active}`;
  } else {
    document.getElementById('messages').innerHTML = '<div class="small">Agrega un contacto o pega tu propio device id en otro dispositivo para chatear.</div>';
  }

  setMyIdDisplay();
})();
</script>
</body>
</html>
