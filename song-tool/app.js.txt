let songData = [];
let searchHistory = JSON.parse(localStorage.getItem("searchHis") || "[]");
let countdownList = JSON.parse(localStorage.getItem("cdList") || "[]");
let theme = localStorage.getItem("theme") || "light";

// 主题初始化
if(theme === "dark") document.body.classList.add("dark-theme");
document.getElementById("themeBtn").onclick = ()=>{
  document.body.classList.toggle("dark-theme");
  localStorage.setItem("theme", document.body.classList.contains("dark-theme")?"dark":"light");
};

// 加载线上songList.json，失败兜底提示
async function loadSongs(){
  try{
    const res = await fetch("./songList.json");
    songData = await res.json();
    renderFilter();
    renderList();
  }catch(e){
    alert("歌单文件加载失败，请检查songList.json路径！");
    console.error(e);
  }
}

// 自动生成筛选下拉选项（重点！标签不用手写）
function renderFilter(){
  const getVal = key => [...new Set(songData.map(i=>i[key]))].sort();
  renderSelect("filterStyle", getVal("style"));
  renderSelect("filterMood", getVal("mood"));
  renderSelect("filterPk", getVal("pkTheme"));
  renderSelect("filterYear", getVal("year"));
  renderSelect("filterInitial", getVal("initial"));
}
function renderSelect(id, arr){
  const sel = document.getElementById(id);
  sel.innerHTML = `<option value="">全部</option>`;
  arr.forEach(v=>sel.innerHTML += `<option value="${v}">${v}</option>`);
}

// 筛选+搜索逻辑
function getFiltered(){
  const kw = document.getElementById("searchInput").value.trim().toLowerCase();
  const fStyle = document.getElementById("filterStyle").value;
  const fMood = document.getElementById("filterMood").value;
  const fPk = document.getElementById("filterPk").value;
  const fYear = document.getElementById("filterYear").value;
  const fIni = document.getElementById("filterInitial").value;

  return songData.filter(item=>{
    const matchKw = !kw || item.name.toLowerCase().includes(kw) || item.singer.toLowerCase().includes(kw) || item.lyric.toLowerCase().includes(kw);
    const s = !fStyle || item.style === fStyle;
    const m = !fMood || item.mood === fMood;
    const pk = !fPk || item.pkTheme === fPk;
    const y = !fYear || item.year === fYear;
    const i = !fIni || item.initial === fIni;
    return matchKw && s && m && pk && y && i;
  })
}

function renderList(){
  const wrap = document.getElementById("songList");
  const list = getFiltered();
  wrap.innerHTML = "";
  list.forEach(song=>{
    const div = document.createElement("div");
    div.className = "song-card";
    div.innerHTML = `
      <div><b>${song.name}</b> · ${song.singer}</div>
      <div style="font-size:12px;opacity:0.7;margin-top:4px">${song.style}｜${song.mood}｜${song.pkTheme}</div>
    `;
    div.onclick = (e)=>{
      createParticle(e.clientX,e.clientY);
    }
    wrap.appendChild(div);
  })
}

// 搜索
const searchInput = document.getElementById("searchInput");
searchInput.oninput = ()=>{
  renderList();
}
searchInput.onkeydown = (e)=>{
  if(e.key === "Enter"){
    const k = searchInput.value.trim();
    if(k && !searchHistory.includes(k)){
      searchHistory.unshift(k);
      searchHistory = searchHistory.slice(0,6);
      localStorage.setItem("searchHis",JSON.stringify(searchHistory));
    }
    renderHis();
  }
}
function renderHis(){
  document.getElementById("searchHistory").innerText = "最近搜索："+searchHistory.join(" / ");
}
renderHis();

// 随机抽歌 + 复制
document.getElementById("randBtn").onclick = async ()=>{
  const arr = getFiltered();
  if(!arr.length) return alert("当前筛选下没有歌曲");
  const pick = arr[Math.floor(Math.random()*arr.length)];
  await navigator.clipboard.writeText(pick.name);
  alert(`抽到：${pick.name}（已复制歌名）`);
}

// 猜歌名小游戏
document.getElementById("gameBtn").onclick = ()=>{
  const arr = getFiltered();
  if(!arr.length) return alert("没有歌曲");
  const pick = arr[Math.floor(Math.random()*arr.length)];
  const ans = prompt(`猜歌名！歌词：${pick.lyric}`);
  if(!ans) return;
  if(ans.trim() === pick.name) alert("答对啦！");
  else alert(`答错咯，答案是：${pick.name}`);
}

// 倒数日渲染
function renderCd(){
  const wrap = document.getElementById("countdownList");
  wrap.innerHTML = "";
  countdownList.forEach((item,idx)=>{
    const target = new Date(item.date);
    const now = new Date();
    const diff = Math.ceil((target - now)/(1000*3600*24));
    const text = diff >= 0 ? `还有 ${diff} 天` : `已过去 ${-diff} 天`;
    const div = document.createElement("div");
    div.style="display:flex;justify-content:space-between;padding:4px 0";
    div.innerHTML = `<span>${item.name}：${text}</span><button style="padding:2px 6px;font-size:12px" data-idx="${idx}">删除</button>`;
    wrap.appendChild(div);
  })
  wrap.querySelectorAll("button[data-idx]").forEach(btn=>{
    btn.onclick = ()=>{
      const i = Number(btn.dataset.idx);
      countdownList.splice(i,1);
      localStorage.setItem("cdList",JSON.stringify(countdownList));
      renderCd();
    }
  })
}
document.getElementById("addCd").onclick = ()=>{
  const name = document.getElementById("cdName").value.trim();
  const date = document.getElementById("cdDate").value;
  if(!name || !date) return alert("填写名称和日期");
  countdownList.push({name,date});
  localStorage.setItem("cdList",JSON.stringify(countdownList));
  document.getElementById("cdName").value="";
  document.getElementById("cdDate").value="";
  renderCd();
}
renderCd();

// 花瓣粒子特效
function createParticle(x,y){
  const canvas = document.getElementById("particleCanvas");
  for(let i=0;i<12;i++){
    const p = document.createElement("div");
    p.style=`
      position:absolute;left:${x}px;top:${y}px;width:10px;height:10px;
      background:#f2b8cb;border-radius:50%;opacity:0.8;
      transform:translate(-50%,-50%);
    `;
    canvas.appendChild(p);
    const angle = (Math.PI*2/12)*i;
    const vx = Math.cos(angle)*60;
    const vy = Math.sin(angle)*60;
    p.animate([
      {transform:`translate(-50%,-50%)`,opacity:0.8},
      {transform:`translate(calc(-50% + ${vx}px),calc(-50% + ${vy}px))`,opacity:0}
    ],{duration:700,easing:"ease-out"});
    setTimeout(()=>p.remove(),700);
  }
}

// 筛选框变更自动刷新列表
["filterStyle","filterMood","filterPk","filterYear","filterInitial"].forEach(id=>{
  document.getElementById(id).onchange = renderList;
})

// 启动
loadSongs();
