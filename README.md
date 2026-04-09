<!DOCTYPE html>
<html lang="zh-Hant">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>雲端心情溫度計</title>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.22.0/firebase-database-compat.js"></script>
    <style>
        :root { --primary: #4a90e2; --bg: #f4f7f9; }
        body { font-family: sans-serif; background: var(--bg); padding: 10px; margin: 0; touch-action: manipulation; -webkit-user-select: none; }
        .container { max-width: 900px; margin: 0 auto; }
        .card { background: #fff; padding: 15px; border-radius: 12px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); margin-bottom: 15px; }
        h2 { font-size: 1.1rem; color: #2c3e50; margin: 0 0 10px 0; border-left: 4px solid var(--primary); padding-left: 10px; }
        canvas { width: 100% !important; height: auto !important; background: #fff; border: 1px solid #ddd; border-radius: 8px; cursor: pointer; display: block; touch-action: none; }
        select { width: 100%; padding: 12px; font-size: 16px; border-radius: 8px; border: 1px solid #ccc; margin-bottom: 10px; background: white; }
        .notes-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 10px; }
        .note-box { background: #fff; padding: 10px; border: 1px solid #eee; border-radius: 8px; }
        textarea { width: 100%; height: 80px; padding: 8px; border: 1px solid #ddd; border-radius: 6px; box-sizing: border-box; resize: none; font-size: 14px; }
        .status { font-size: 12px; color: #27ae60; font-weight: bold; margin-bottom: 5px; }
        .hint { font-size: 11px; color: #999; margin-top: 5px; }
    </style>
</head>
<body>
<div class="container">
    <div class="card">
        <div id="syncStatus" class="status">● 連線中...</div>
        <select id="dateSelect"></select>
        <canvas id="dayChart" width="800" height="320"></canvas>
        <div class="hint">💡 點擊上方灰色橫線位置即可標記心情 (點擊垂直線最準)</div>
    </div>
    <div class="card">
        <h2>週版平均趨勢</h2>
        <canvas id="weekChart" width="1400" height="320"></canvas>
        <div id="notes" class="notes-grid" style="margin-top:20px;"></div>
    </div>
</div>

<script>
const firebaseConfig = {
  apiKey: "AIzaSyBj3pvp-UrqNuFFqWQPD5OVoieuJI5DeoE",
  authDomain: "mood-81873.firebaseapp.com",
  databaseURL: "https://mood-81873-default-rtdb.firebaseio.com",
  projectId: "mood-81873",
  storageBucket: "mood-81873.firebasestorage.app",
  messagingSenderId: "888330529357",
  appId: "1:888330529357:web:46058d2f5298126e4c0a04"
};

firebase.initializeApp(firebaseConfig);
const db = firebase.database();

const times = ["9","11","13","15","17","19","21","23"];
const dates = ["3/30 (一)","3/31 (二)","4/01 (三)","4/02 (四)","4/03 (五)","4/04 (六)","4/05 (日)",
               "4/06 (一)","4/07 (二)","4/08 (三)","4/09 (四)","4/10 (五)","4/11 (六)","4/12 (日)",
               "4/13 (一)","4/14 (二)","4/15 (三)","4/16 (四)","4/17 (五)","4/18 (六)","4/19 (日)"];

let allData = {}, noteData = {}, weekData = {};
const ds = document.getElementById("dateSelect");
const notesDiv = document.getElementById("notes");

// 建立筆記區域
dates.forEach(d => {
    let o = document.createElement("option"); o.text = d; ds.add(o);
    let box = document.createElement("div"); box.className = "note-box";
    box.innerHTML = `<strong>${d}</strong><textarea id="note-${d}" placeholder="今天的心情覺察..."></textarea>`;
    notesDiv.appendChild(box);
    box.querySelector('textarea').addEventListener('change', (e) => {
        db.ref('moodData/notes/' + d).set(e.target.value);
    });
});

// 即時同步雲端
db.ref('moodData').on('value', (snap) => {
    const data = snap.val() || {};
    allData = data.scores || {};
    noteData = data.notes || {};
    dates.forEach(d => {
        const el = document.getElementById(`note-${d}`);
        if(noteData[d] !== undefined && el !== document.activeElement) el.value = noteData[d];
    });
    document.getElementById("syncStatus").innerText = "● 雲端已同步 (隨時點擊更新)";
    refresh();
});

function refresh() {
    draw("dayChart", allData[ds.value] || {}, times, 90, false);
    dates.forEach(d => {
        let v = Object.values(allData[d] || {});
        if(v.length > 0) weekData[d] = v.reduce((a,b)=>a+b)/v.length;
    });
    draw("weekChart", weekData, dates, 60, true);
}

function draw(id, data, labels, xGap, isW) {
    const c = document.getElementById(id), ctx = c.getContext("2d");
    ctx.clearRect(0,0,c.width,c.height);
    // 背景參考線
    ctx.strokeStyle = "#f0f0f0"; ctx.lineWidth = 1;
    for(let i=-10; i<=10; i+=5) {
        let y = 160-i*10; ctx.beginPath(); ctx.moveTo(50,y); ctx.lineTo(c.width,y); ctx.stroke();
        ctx.fillStyle = "#bbb"; ctx.font = "12px Arial";
        ctx.fillText(i, 15, y+5);
    }
    // 時間標籤
    labels.forEach((l, i) => {
        let x = 70+i*xGap;
        ctx.fillStyle = "#888"; ctx.fillText(isW?l.split(' ')[0]:l+":00", x-15, 315);
    });
    // 折線與點
    let pts = [];
    labels.forEach((l, i) => {
        if(data[l] !== undefined) pts.push({x: 70+i*xGap, y: 160-data[l]*10});
    });
    if(pts.length > 0) {
        ctx.beginPath(); ctx.strokeStyle = isW?"#f39c12":"#3498db"; ctx.lineWidth = 4;
        pts.forEach((p,i) => i===0?ctx.moveTo(p.x,p.y):ctx.lineTo(p.x,p.y)); ctx.stroke();
        pts.forEach(p => {
            ctx.beginPath(); ctx.arc(p.x,p.y,6,0,Math.PI*2); ctx.fillStyle="#fff"; ctx.fill(); 
            ctx.strokeStyle = isW?"#f39c12":"#3498db"; ctx.stroke();
        });
    }
}

// 核心修正：點擊與觸摸邏輯
const handleEvent = (e) => {
    e.preventDefault();
    const c = document.getElementById("dayChart");
    const r = c.getBoundingClientRect();
    
    // 取得點擊在畫布上的相對比例
    const clientX = e.clientX || (e.touches && e.touches[0].clientX);
    const clientY = e.clientY || (e.touches && e.touches[0].clientY);
    
    const x = (clientX - r.left) * (800 / r.width);
    const y = (clientY - r.top) * (320 / r.height);
    
    // 計算點擊的是第幾個時間點
    const i = Math.round((x - 70) / 90);
    
    if (i >= 0 && i < times.length) {
        // 計算分數並限制在 -10 ~ 10
        let val = Math.round((160 - y) / 10);
        val = Math.max(-10, Math.min(10, val));
        
        // 寫入 Firebase
        db.ref('moodData/scores/' + ds.value + '/' + times[i]).set(val);
        
        // 手機震動回饋 (若支援)
        if(navigator.vibrate) navigator.vibrate(10);
    }
};

const canvas = document.getElementById("dayChart");
canvas.addEventListener("mousedown", handleEvent);
canvas.addEventListener("touchstart", handleEvent, {passive: false});

ds.addEventListener("change", refresh);
</script>
</body>
</html>
