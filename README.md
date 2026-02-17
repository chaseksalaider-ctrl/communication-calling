<!doctype html>
<html lang="th">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Voice Room — โจเซฟ</title>
<style>
  body{font-family:sans-serif;margin:0;padding:18px;background:#BEBEBE;color:#111}
  .card{max-width:920px;margin:0 auto;background:#fff;padding:16px;border-radius:8px;box-shadow:0 6px 18px rgba(0,0,0,0.06)}
  h1{font-size:18px;margin:0 0 12px}
  .row{display:flex;gap:12px}
  input,select,button{padding:8px;border-radius:6px;border:1px solid #ccc}
  #participants{margin-top:12px}
  .person{padding:6px 8px;border-radius:6px;background:#f7f7f7;margin:6px 0;display:flex;justify-content:space-between;align-items:center}
  #localIndicator{font-size:12px;color:#666;margin-left:8px}
  .controls{display:flex;gap:8px;flex-direction:row;align-items:center}
  .btn{cursor:pointer}
  audio{display:block;margin-top:6px;width:100%}
  .muted{opacity:0.6;font-size:13px}
</style>
</head>
<body>

<div class="card">
  <h1>ห้องเสียงลับ (ทดลอง) — โมดุล3</h1>

  <div id="joinPane">
    <div style="display:flex;gap:8px;align-items:center">
      <input id="nameInput" placeholder="ใส่ชื่อ (ห้ามซ้ำในห้อง)" />
      <input id="roomInput" placeholder="รหัสห้อง (เช่น 73921)" />
      <button id="joinBtn" class="btn">เข้าห้อง</button>
    </div>
    <div class="muted" style="margin-top:8px">หมายเหตุ: ชื่อต้องไม่ซ้ำ ถ้าซ้ำจะเข้าห้องไม่ได้</div>
  </div>

  <div id="roomPane" style="display:none;margin-top:12px">
    <div style="display:flex;justify-content:space-between;align-items:center">
      <div>
        <strong>ห้อง:</strong> <span id="roomLabel"></span>
        <span id="localIndicator" class="muted"></span>
      </div>
      <div class="controls">
        <button id="muteBtn" class="btn">ปิดไมค์</button>
        <button id="volMuteBtn" class="btn">ปิดเสียงเข้า</button>
        <button id="leaveBtn" class="btn">ออกห้อง</button>
      </div>
    </div>

    <div id="participants">
      <h3 style="margin:12px 0 6px 0">ผู้ร่วมในห้อง</h3>
      <div id="list"></div>
    </div>

    <div id="audios"></div>
  </div>
</div>

<!-- Firebase lib -->
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/9.23.0/firebase-database-compat.js"></script>

<script>
/* ====== ใส่ config ของมึงตรงนี้ ====== */
const firebaseConfig = {
  apiKey: "AIzaSyAhnZF4kmaem05yZeCicqTbtJn7tUOEOsU",
  authDomain: "voice-room-firebase-38891.firebaseapp.com",
  databaseURL: "https://voice-room-firebase-38891-default-rtdb.firebaseio.com",
  projectId: "voice-room-firebase-38891",
  storageBucket: "voice-room-firebase-38891.firebasestorage.app",
  messagingSenderId: "673188196122",
  appId: "1:673188196122:web:de333d2acde11bf213c6a3",
};
firebase.initializeApp(firebaseConfig);
const db = firebase.database();

/* ======= การออกแบบ signaling / presence (ง่าย) =======
 Data structure:
 rooms/{roomId}/participants/{name} = { joinedAt: ts }
 rooms/{roomId}/signals/{autoId} = {from, to, type, sdp?, candidate?}
 ถ้า to === 'all' จะส่งหาทุกคน (ใช้ตอน announce join) แต่เรแนะนำ targeted
*/

let localStream = null;
let pcs = {}; // map remoteName -> RTCPeerConnection
let remoteAudioEls = {}; // remoteName -> audio element
let myName = '';
let roomId = '';
let participantsRef = null;
let signalsRef = null;
let joined = false;
let micMuted = false;
let audioMuted = false;

const joinBtn = document.getElementById('joinBtn');
const nameInput = document.getElementById('nameInput');
const roomInput = document.getElementById('roomInput');
const joinPane = document.getElementById('joinPane');
const roomPane = document.getElementById('roomPane');
const roomLabel = document.getElementById('roomLabel');
const listEl = document.getElementById('list');
const audiosEl = document.getElementById('audios');
const muteBtn = document.getElementById('muteBtn');
const volMuteBtn = document.getElementById('volMuteBtn');
const leaveBtn = document.getElementById('leaveBtn');
const localIndicator = document.getElementById('localIndicator');

const servers = { iceServers: [{ urls: 'stun:stun.l.google.com:19302' }] };

/* helper */
function setStatus(txt){ localIndicator.textContent = txt; }

async function getLocalAudio(){
  try {
    const s = await navigator.mediaDevices.getUserMedia({ audio: true, video:false });
    localStream = s;
    micMuted = false;
    updateMuteButton();
    return s;
  } catch (e) {
    alert('ไม่สามารถใช้ไมโครโฟนได้: ' + e.message);
    throw e;
  }
}

function updateMuteButton(){
  muteBtn.textContent = micMuted ? 'เปิดไมค์' : 'ปิดไมค์';
  volMuteBtn.textContent = audioMuted ? 'เปิดเสียงเข้า' : 'ปิดเสียงเข้า';
}

/* ตรวจชื่อซ้ำก่อน join */
async function nameExists(name, roomId){
  const snap = await db.ref(`rooms/${roomId}/participants/${name}`).once('value');
  return snap.exists();
}

function renderParticipants(listObj){
  listEl.innerHTML = '';
  if(!listObj) return;
  Object.keys(listObj).forEach(name => {
    const d = document.createElement('div');
    d.className = 'person';
    d.innerHTML = `<div>${name}</div><div class="muted">${new Date(listObj[name].joinedAt).toLocaleString()}</div>`;
    listEl.appendChild(d);
  });
}

/* join room */
joinBtn.onclick = async () => {
  myName = nameInput.value.trim();
  roomId = roomInput.value.trim();
  if(!myName || !roomId){ alert('ใส่ชื่อและรหัสห้องก่อน'); return; }

  if(await nameExists(myName, roomId)){
    alert('ชื่อซ้ำในห้องนี้ — เปลี่ยนชื่อก่อนเข้า');
    return;
  }

  // prepare local media
  await getLocalAudio();

  participantsRef = db.ref(`rooms/${roomId}/participants`);
  signalsRef = db.ref(`rooms/${roomId}/signals`);

  // add self to participants
  await participantsRef.child(myName).set({ joinedAt: Date.now() });
  // listen participants & signals
  participantsRef.on('value', snap => {
    const v = snap.val() || {};
    renderParticipants(v);
  });

  signalsRef.on('child_added', async snap => {
    const msg = snap.val();
    if(!msg) return;
    // ignore messages we created
    if(msg.to !== myName && msg.to !== 'all') return;
    // handle types
    if(msg.type === 'offer' && msg.to === myName){
      await handleOffer(msg);
    } else if(msg.type === 'answer' && msg.to === myName){
      await handleAnswer(msg);
    } else if(msg.type === 'candidate' && msg.to === myName){
      await handleCandidate(msg);
    } else if(msg.type === 'announce' && msg.to === 'all'){
      // other peer announces; if we are the older one, create offer to them
      // but simplest: when someone new joins they will create offers to existing ones
    }
    // remove processed signal to keep DB tidy
    // (optional) - avoid immediate remove if want history; here remove
    snap.ref.remove().catch(()=>{});
  });

  // announce to room: create offers to existing participants
  const current = (await participantsRef.once('value')).val() || {};
  const others = Object.keys(current).filter(n => n !== myName);
  // for each existing participant, create p2p (we will create offer)
  for(const other of others){
    await createPeerAndOffer(other);
  }

  // also watch when a new participant comes => that participant will create offers to us
  participantsRef.on('child_added', async snap => {
    const name = snap.key;
    if(name === myName) return;
    // wait a tiny bit to let them be ready, then (optionally) create offer if desired:
    // We'll let the newbie initiate offers to existing participants (safer)
  });

  joined = true;
  joinPane.style.display = 'none';
  roomPane.style.display = 'block';
  roomLabel.textContent = roomId;
  setStatus('ออนไลน์');
};

/* leave */
leaveBtn.onclick = async () => {
  if(!joined) return;
  // remove participant
  await participantsRef.child(myName).remove();
  // close peer connections
  Object.values(pcs).forEach(pc => pc.close());
  pcs = {};
  // remove audios
  audiosEl.innerHTML = '';
  remoteAudioEls = {};
  // stop local stream
  if(localStream) localStream.getTracks().forEach(t=>t.stop());
  localStream = null;
  joined = false;
  joinPane.style.display = 'block';
  roomPane.style.display = 'none';
  setStatus('');
};

/* create peer connection and offer to targetName */
async function createPeerAndOffer(targetName){
  if(pcs[targetName]) { console.warn('pc exists for', targetName); return; }
  const pc = new RTCPeerConnection(servers);
  pcs[targetName] = pc;

  // add local tracks
  if(localStream) localStream.getTracks().forEach(t => pc.addTrack(t, localStream));

  // remote audio
  const remoteStream = new MediaStream();
  const audioEl = document.createElement('audio');
  audioEl.autoplay = true;
  audioEl.controls = false;
  audioEl.dataset.peer = targetName;
  audiosEl.appendChild(document.createElement('div')).appendChild(audioEl);
  remoteAudioEls[targetName] = audioEl;

  pc.ontrack = ev => {
    ev.streams[0].getTracks().forEach(t => remoteStream.addTrack(t));
    audioEl.srcObject = remoteStream;
    audioEl.muted = audioMuted; // apply current incoming mute setting
  };

  pc.onicecandidate = ev => {
    if(ev.candidate){
      signalsRef.push({
        from: myName,
        to: targetName,
        type: 'candidate',
        candidate: ev.candidate.toJSON(),
        ts: Date.now()
      });
    }
  };

  // create offer
  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);

  // send offer via signaling
  signalsRef.push({
    from: myName,
    to: targetName,
    type: 'offer',
    sdp: offer.sdp,
    ts: Date.now()
  });
}

/* handle incoming offer -> create answer */
async function handleOffer(msg){
  const remote = msg.from;
  if(pcs[remote]) {
    console.warn('already have pc for', remote);
  } else {
    const pc = new RTCPeerConnection(servers);
    pcs[remote] = pc;

    if(localStream) localStream.getTracks().forEach(t => pc.addTrack(t, localStream));

    const remoteStream = new MediaStream();
    const audioEl = document.createElement('audio');
    audioEl.autoplay = true;
    audioEl.controls = false;
    audioEl.dataset.peer = remote;
    audiosEl.appendChild(document.createElement('div')).appendChild(audioEl);
    remoteAudioEls[remote] = audioEl;

    pc.ontrack = ev => {
      ev.streams[0].getTracks().forEach(t => remoteStream.addTrack(t));
      audioEl.srcObject = remoteStream;
      audioEl.muted = audioMuted;
    };

    pc.onicecandidate = ev => {
      if(ev.candidate){
        signalsRef.push({
          from: myName,
          to: remote,
          type: 'candidate',
          candidate: ev.candidate.toJSON(),
          ts: Date.now()
        });
      }
    };
  }

  const pc = pcs[remote];
  // set remote desc
  await pc.setRemoteDescription({ type: 'offer', sdp: msg.sdp });
  const answer = await pc.createAnswer();
  await pc.setLocalDescription(answer);

  // send answer
  signalsRef.push({
    from: myName,
    to: remote,
    type: 'answer',
    sdp: answer.sdp,
    ts: Date.now()
  });
}

/* handle answer */
async function handleAnswer(msg){
  const remote = msg.from;
  const pc = pcs[remote];
  if(!pc) return console.warn('no pc for answer from', remote);
  await pc.setRemoteDescription({ type: 'answer', sdp: msg.sdp });
}

/* handle candidate */
async function handleCandidate(msg){
  const remote = msg.from;
  const pc = pcs[remote];
  if(!pc) return;
  try {
    await pc.addIceCandidate(msg.candidate);
  } catch(e){
    console.warn('addIce failed', e);
  }
}

/* when new participant exists, the newbie will create offers to existing ones.
   But we should also react to departed peers: listen participants child_removed */
function listenRemovals(){
  participantsRef.on('child_removed', snap => {
    const name = snap.key;
    if(pcs[name]){
      pcs[name].close();
      delete pcs[name];
    }
    if(remoteAudioEls[name]){
      const el = remoteAudioEls[name];
      if(el && el.parentNode) el.parentNode.remove();
      delete remoteAudioEls[name];
    }
    // re-render participants will hide them anyway
  });
}

/* Mute / unmute local mic */
muteBtn.onclick = () => {
  if(!localStream) return;
  micMuted = !micMuted;
  localStream.getAudioTracks().forEach(t => t.enabled = !micMuted);
  updateMuteButton();
};

/* mute incoming audio (local control) */
volMuteBtn.onclick = () => {
  audioMuted = !audioMuted;
  Object.values(remoteAudioEls).forEach(a => a.muted = audioMuted);
  updateMuteButton();
};

/* helper to show current participants once */
async function refreshParticipantsOnce(){
  if(!participantsRef) return;
  const snap = await participantsRef.once('value');
  renderParticipants(snap.val() || {});
}

/* make sure to clean up when user closes tab */
window.addEventListener('beforeunload', async () => {
  if(joined){
    try{ await participantsRef.child(myName).remove(); }catch(e){}
  }
});

/* init */
setStatus('');
</script>
</body>
</html>
