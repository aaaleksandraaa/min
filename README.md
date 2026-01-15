<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<title>P2P Групповой Wi-Fi чат</title>
<style>
body { font-family: sans-serif; margin: 20px; }
#chat { border:1px solid #ccc; height:300px; overflow-y:auto; padding:5px; margin-bottom:10px; }
input { width:70%; }
button { width:28%; }
</style>
</head>
<body>

<h2>P2P Групповой Wi-Fi чат</h2>

<div id="chat"></div>
<input id="msg" placeholder="Сообщение">
<button onclick="send()">Отправить</button>

<script>
const chat = document.getElementById('chat');
const peers = {}; // словарь всех подключений
let localId = Math.floor(Math.random()*1000000); // уникальный ID для браузера

// Сигналинговый сервер (бесплатный, публичный) для обмена офферами
const SIGNAL_SERVER = 'wss://signaling.simplewebrtc.com';
const ws = new WebSocket(SIGNAL_SERVER);

// Добавляем сообщение в чат
function addMessage(msg){
  const div = document.createElement('div');
  div.textContent = msg;
  chat.appendChild(div);
  chat.scrollTop = chat.scrollHeight;
}

// ==== WebSocket сигналинг ====
ws.onopen = () => {
  ws.send(JSON.stringify({type:'join', id: localId}));
};

ws.onmessage = async event => {
  const data = JSON.parse(event.data);

  // Игнорируем свои сообщения
  if(data.from === localId) return;

  if(data.type === 'offer'){
    // Новый оффер — создаём RTCPeerConnection
    const pc = new RTCPeerConnection();
    peers[data.from] = pc;

    // DataChannel для сообщений
    pc.ondatachannel = e => {
      const channel = e.channel;
      channel.onmessage = e => addMessage('👤 ' + e.data);
      pc.channel = channel;
    };

    // ICE кандидаты
    pc.onicecandidate = e => {
      if(e.candidate) ws.send(JSON.stringify({type:'candidate', candidate:e.candidate, to:data.from, from:localId}));
    };

    await pc.setRemoteDescription(new RTCSessionDescription(data.offer));
    const answer = await pc.createAnswer();
    await pc.setLocalDescription(answer);
    ws.send(JSON.stringify({type:'answer', answer: answer, to: data.from, from: localId}));

  } else if(data.type === 'answer'){
    const pc = peers[data.from];
    if(!pc) return;
    await pc.setRemoteDescription(new RTCSessionDescription(data.answer));

  } else if(data.type === 'candidate'){
    const pc = peers[data.from];
    if(pc) pc.addIceCandidate(new RTCIceCandidate(data.candidate));
  } else if(data.type === 'join'){
    // Новый участник — создаём RTCPeerConnection и оффер
    const pc = new RTCPeerConnection();
    peers[data.id] = pc;

    const channel = pc.createDataChannel('chat');
    channel.onmessage = e => addMessage('👤 ' + e.data);
    pc.channel = channel;

    pc.onicecandidate = e => {
      if(e.candidate) ws.send(JSON.stringify({type:'candidate', candidate:e.candidate, to:data.id, from:localId}));
    };

    const offer = await pc.createOffer();
    await pc.setLocalDescription(offer);
    ws.send(JSON.stringify({type:'offer', offer: offer, to:data.id, from: localId}));
  }
};

// ==== Отправка сообщений всем подключённым пирами ====
function send(){
  const msgInput = document.getElementById('msg');
  if(msgInput.value){
    addMessage('🧑‍💻 ' + msgInput.value);
    for(const id in peers){
      const pc = peers[id];
      if(pc.channel && pc.channel.readyState === 'open'){
        pc.channel.send(msgInput.value);
      }
    }
    msgInput.value = '';
  }
}
</script>

</body>
</html>
