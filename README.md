<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Burgur Burgur Simple</title>
<style>
body { font-family: Arial; background:#f2f4f8; padding:20px; }
.container { max-width:1000px; margin:auto; background:white; padding:20px; border-radius:12px; }
input, button, textarea { padding:8px; margin:5px; border-radius:6px; border:1px solid #ccc; }
button { background:#007bff; color:white; cursor:pointer; border:none; }
button:hover { background:#0056b3; }
.flex { display:flex; gap:20px; }
.sidebar { width:250px; background:#f0f0f0; padding:15px; border-radius:10px; }
.game-area { flex:1; }
.item-row { display:flex; justify-content:space-between; padding:4px 0; }
.scoreboard-player { padding:5px; border-radius:6px; cursor:pointer; }
.scoreboard-player.active { background:#d1ffd6; }
#turnContainer { margin-top:20px; }
.notepad { margin-top:20px; padding:10px; background:#f8f8f8; border-radius:8px; }
.notepad textarea { width:100%; padding:8px; border-radius:6px; border:1px solid #ccc; }
</style>
</head>
<body>

<div class="container">
<h2>Bger </h2>

<!-- Login Area -->
<div id="loginArea">
<input id="roomCode" placeholder="Room Code">
<input id="playerName" placeholder="Your Name">
<button onclick="joinRoom()">Join Game</button>
</div>

<!-- Game Area -->
<div id="gameArea" class="flex" style="display:none;">
  <div class="sidebar">
    <h3>Players</h3>
    <div id="scoreboard"></div>
  </div>

  <div class="game-area">
    <h3>Room: <span id="roomDisplay"></span></h3>
    <h3>Current Turn: <span id="currentTurnPlayer"></span></h3>

    <h3>Items</h3>
    <div id="items"></div>

    <button onclick="endTurn()">End Turn</button>

    <div id="turnContainer"></div>
    <div>
      Total Money: $<span id="money">0</span> | 
      Total Points: <span id="points">0</span> | 
      Pocket: $<span id="pocket">0</span>
    </div>

    <!-- Pocket Controls -->
    <div>
      <input id="adjustMoney" type="number" placeholder="Change Total Money">
      <button onclick="adjustMoney()">Update Money</button>
      <input id="adjustPoints" type="number" placeholder="Change Total Points">
      <button onclick="adjustPoints()">Update Points</button>
      <input id="adjustPocket" type="number" placeholder="Change Pocket">
      <button onclick="adjustPocket()">Update Pocket</button>
    </div>

    <!-- Notepad -->
    <div class="notepad">
      <h3>Notepad</h3>
      <textarea id="notepad" rows="6" placeholder="Write notes here..."></textarea>
      <br>
      <button onclick="saveNotes()">Save Notes</button>
    </div>
  </div>
</div>

<script type="module">
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-app.js";
import { getFirestore, doc, setDoc, getDoc, collection, getDocs, onSnapshot } from "https://www.gstatic.com/firebasejs/10.7.1/firebase-firestore.js";

// Firebase config
const firebaseConfig = {
  apiKey: "AIzaSyDxu-Wn-34k8xI00hIi6sRuJusLc_rtAcg",
  authDomain: "burgur-game.firebaseapp.com",
  projectId: "burgur-game",
  storageBucket: "burgur-game.firebasestorage.app",
  messagingSenderId: "257283076694",
  appId: "1:257283076694:web:3476d11959fbfe9092c1f7",
  measurementId: "G-71RJPN5TSV"
};

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

let room="", player="", currentTurn=1, currentTurnPlayer="", viewingPlayer="";
let totalMoneyBase = 0, totalPointsBase = 0, pocketMoney = 0;

// ============================
// Items by Category
// ============================
const categories = {
  "Common ": [
    {name:"Patty", price:2, points:1},
    {name:"Cheese", price:2, points:1},
    {name:"Tomato", price:2, points:1},
    {name:"lettuce", price:2, points:1},
    {name:"Onion", price:2, points:1},
    {name:"Patty From Shop", price:1, points:1},
    {name:"Cheese From Shop", price:1, points:1},
    {name:"Tomato From Shop", price:1, points:1},
    {name:"lettuce From Shop", price:1, points:1},
    {name:"Onion From Shop", price:1, points:1}
  ],
  "Uncommon": [
    {name:"Pickle", price:4, points:2},
    {name:"Pesto", price:4, points:2},
    {name:"Kimchi", price:4, points:2},
    {name:"Pineapple", price:4, points:2},
    {name:"Shrimp", price:4, points:2},
    {name:"Pickle From Shop", price:2, points:2},
    {name:"Pesto From Shop", price:2, points:2},
    {name:"Kimchi From Shop", price:2, points:2},
    {name:"Pineapple From Shop", price:2, points:2},
    {name:"Shrimp From Shop", price:2, points:2}
  ],
  "Rare": [
    {name:"Rice Noodle", price:6, points:3},
    {name:"Macha", price:6, points:3},
    {name:"Chicken something", price:6, points:3},
    {name:"Chiese Sausage", price:6, points:3},
    {name:"things", price:6, points:3},
    {name:"Rice Noodle From Shop", price:3, points:3},
    {name:"Macha From Shop", price:3, points:1},
    {name:"Chicken something From Shop", price:3, points:3},
    {name:"Chiese Sausage From Shop", price:3, points:3},
    {name:"things From Shop", price:3, points:3}
  ],
  "Godly ": [
    {name:"Caviar", price:8, points:4},
    {name:"Truffle", price:8, points:4},
    {name:"Wagyu", price:8, points:4},
    {name:"things", price:8, points:4},
    {name:"things", price:8, points:4},
    {name:"Caviar From Shop", price:4, points:4},
    {name:"Truffle From Shop", price:4, points:4},
    {name:"Wagyu From Shop", price:4, points:4},
    {name:"things From Shop", price:4, points:4},
    {name:"things From Shop", price:4, points:4}
  ],
  "Poo ": [
    {name:"cardboard", price:0, points:0},
    {name:"Too Much Salt", price:0, points:0},
    {name:"things", price:0, points:0},
    {name:"things", price:0, points:0},
    {name:"Toy", price:0, points:0}
  ]
};

// Populate items buttons
const itemsDiv = document.getElementById("items");
for(const cat in categories){
  const catHeader = document.createElement("h4");
  catHeader.textContent = cat;
  itemsDiv.appendChild(catHeader);
  categories[cat].forEach(item=>{
    const div = document.createElement("div");
    div.className="item-row";
    div.innerHTML = `<span>${item.name} ($${item.price}, ${item.points}pts)</span> <button onclick="addItem('${item.name}',${item.price},${item.points})">Add</button>`;
    itemsDiv.appendChild(div);
  });
}

// ============================
// Join Room
// ============================
window.joinRoom = async function(){
  room=document.getElementById("roomCode").value;
  player=document.getElementById("playerName").value;
  if(!room||!player){ alert("Enter Room & Name"); return; }

  document.getElementById("loginArea").style.display="none";
  document.getElementById("gameArea").style.display="flex";
  document.getElementById("roomDisplay").innerText=room;
  viewingPlayer = player;

  const playerRef = doc(db,"rooms",room,"players",player);
  await setDoc(playerRef,{isOnline:true},{merge:true});

  const roomRef = doc(db,"rooms",room);
  const roomSnap = await getDoc(roomRef);
  if(!roomSnap.exists() || !roomSnap.data().currentTurnPlayer){
    await setDoc(roomRef,{currentTurnPlayer:player},{merge:true});
  }

  listenToRoom();
}

// ============================
// Add Item
// ============================
window.addItem = async function(name,price,points){
  if(player!==currentTurnPlayer){ alert("Wait your turn!"); return; }

  const playerRef = doc(db,"rooms",room,"players",player);
  const docSnap = await getDoc(playerRef);
  const turnKey="Turn"+currentTurn;

  let prev={};
  if(docSnap.exists() && docSnap.data()[turnKey]) prev=docSnap.data()[turnKey];

  const newTurn = {...prev};
  if(newTurn[name]){
    newTurn[name].quantity+=1;
    newTurn[name].money+=price;
    newTurn[name].points+=points;
  } else {
    newTurn[name]={quantity:1,money:price,points:points};
  }

  await setDoc(playerRef,{[turnKey]:newTurn},{merge:true});
  updateUI(viewingPlayer);
}

// ============================
// End Turn
// ============================
window.endTurn = async function(){
  const playersSnapshot = await getDocs(collection(db,"rooms",room,"players"));
  const playerNames = playersSnapshot.docs.map(d=>d.id);
  if(playerNames.length<1) return;

  const currentIndex = playerNames.indexOf(player);
  const nextIndex = (currentIndex+1)%playerNames.length;
  const nextPlayer = playerNames[nextIndex];
  currentTurn++;

  // Every 7 turns, add total money from items to pocket
  if(currentTurn % 7 === 0){
    playersSnapshot.docs.forEach(docSnap=>{
      const data = docSnap.data();
      let total = 0;
      for(const key in data){
        if(key.startsWith("Turn")){
          for(const i in data[key]) total += data[key][i].money;
        }
      }
      if(docSnap.id === player) pocketMoney += total;
    });
    alert("Cycle complete! Money added to pocket.");
  }

  await setDoc(doc(db,"rooms",room),{currentTurnPlayer:nextPlayer},{merge:true});
  updateUI(viewingPlayer);
}

// ============================
// Adjust Money / Points / Pocket
// ============================
window.adjustMoney = function(){
  const val = parseInt(document.getElementById("adjustMoney").value);
  if(!isNaN(val)) totalMoneyBase = val;
  updateUI(viewingPlayer);
}
window.adjustPoints = function(){
  const val = parseInt(document.getElementById("adjustPoints").value);
  if(!isNaN(val)) totalPointsBase = val;
  updateUI(viewingPlayer);
}
window.adjustPocket = function(){
  const val = parseInt(document.getElementById("adjustPocket").value);
  if(!isNaN(val)) pocketMoney = val;
  updateUI(viewingPlayer);
}

// ============================
// Listen Room Updates
// ============================
function listenToRoom(){
  const playersRef = collection(db,"rooms",room,"players");
  onSnapshot(playersRef,snapshot=>{
    let html="";
    snapshot.docs.forEach(docSnap=>{
      const p=docSnap.id;
      html+=`<div class="scoreboard-player ${p===currentTurnPlayer?'active':''}" onclick="switchView('${p}')">${p}</div>`;
    });
    document.getElementById("scoreboard").innerHTML=html;
    updateUI(viewingPlayer);
  });

  const roomRef = doc(db,"rooms",room);
  onSnapshot(roomRef,snap=>{
    if(snap.exists()){
      currentTurnPlayer = snap.data().currentTurnPlayer;
      document.getElementById("currentTurnPlayer").innerText=currentTurnPlayer;
    }
  });
}

// ============================
// Switch view to another player
// ============================
window.switchView = function(name){
  viewingPlayer = name;
  updateUI(viewingPlayer);
}

// ============================
// Update UI
// ============================
function updateUI(viewPlayer){
  const playerRef = doc(db,"rooms",room,"players",viewPlayer);
  getDoc(playerRef).then(docSnap=>{
    let html="", totalMoney=totalMoneyBase, totalPoints=totalPointsBase;
    if(docSnap.exists()){
      const data=docSnap.data();
      for(const turn in data){
        if(turn==="isOnline") continue;
        html+="<div><strong>"+turn+"</strong><br>";
        for(const item in data[turn]){
          const i=data[turn][item];
          html+=item+" x"+i.quantity+" $"+i.money+" | "+i.points+" pts<br>";
          totalMoney += i.money;
          totalPoints += i.points;
        }
        html+="</div>";
      }
    }
    document.getElementById("turnContainer").innerHTML=html;
    document.getElementById("money").innerText=totalMoney;
    document.getElementById("points").innerText=totalPoints;
    document.getElementById("pocket").innerText = pocketMoney;
  });
}

// ============================
// Notepad
// ============================
const noteArea = document.getElementById("notepad");
if(localStorage.getItem("playerNotes")) noteArea.value = localStorage.getItem("playerNotes");
window.saveNotes = function(){
  localStorage.setItem("playerNotes", noteArea.value);
  alert("Notes saved!");
}
</script>
</body>
</html>
