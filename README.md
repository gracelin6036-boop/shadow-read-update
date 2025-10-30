<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8" />
  <title>ShadowReadPlayer — 本地/线上视频跟读播放器</title>
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <style>
    :root{--accent:#0ea5a3;--muted:#6b7280}
    body{font-family:system-ui,-apple-system,Segoe UI,Roboto,Helvetica,Arial;color:#111;background:#f3f4f6;padding:18px}
    .wrap{max-width:980px;margin:0 auto;background:#fff;border-radius:10px;padding:16px;box-shadow:0 6px 20px rgba(2,6,23,0.08)}
    h1{font-size:18px;margin:0 0 12px}
    video{width:100%;border-radius:8px;background:#000}
    .controls{display:flex;gap:8px;flex-wrap:wrap;align-items:center;margin-top:10px}
    button,select,input[type="file"]{padding:6px 10px;border-radius:6px;border:1px solid #e6e9ee;background:#fff;cursor:pointer}
    .subtitleBox{margin-top:12px;padding:12px;background:#f8fafc;border-radius:8px}
    .subtitleMain{font-size:18px;font-weight:600}
    .subtitleCn{color:var(--muted);margin-top:6px}
    .words button{background:none;border:0;padding:0;margin:0;cursor:pointer;text-decoration:underline;color:#0f172a}
    .listGrid{display:flex;gap:12px;margin-top:14px}
    .panel{flex:1;max-height:260px;overflow:auto;border:1px solid #f1f5f9;padding:8px;border-radius:8px;background:#fff}
    .panel .item{padding:8px;border-radius:6px;cursor:pointer}
    .item.active{background:#ecfeff;border-left:4px solid var(--accent)}
    .small{font-size:12px;color:var(--muted)}
    .row{display:flex;gap:8px;align-items:center}
    textarea{width:100%;min-height:76px;padding:8px;border-radius:8px;border:1px solid #e6e9ee}
    .wordbook .rowItem{display:flex;justify-content:space-between;padding:6px;border-bottom:1px solid #fafafa}
    .danger{color:#ef4444}
    footer{margin-top:12px;font-size:12px;color:var(--muted)}
  </style>
</head>
<body>
  <div class="wrap">
    <h1>ShadowReadPlayer — 跟读播放器（HTML 版）</h1>

    <!-- 主视频源（已替换为 Google Drive 直链） -->
    <div>
      <label class="small">当前视频来源（可选：选择本地文件或粘贴远程 URL）</label>
      <div style="display:flex;gap:8px;margin-top:8px;align-items:center;">
        <input id="fileInput" type="file" accept="video/*" />
        <input id="urlInput" placeholder="粘贴远程视频 URL 后回车（例如 CDN / 直链 mp4）" style="flex:1;padding:6px;border-radius:6px;border:1px solid #e6e9ee" />
        <button id="useDrive">使用 Drive 视频</button>
      </div>
    </div>

    <div style="margin-top:10px">
      <video id="player" controls crossorigin playsinline preload="metadata"></video>
    </div>

    <div class="controls">
      <button id="playBtn">播放 / 暂停</button>
      <button id="prevBtn">上一句</button>
      <button id="nextBtn">下一句</button>
      <button id="loopBtn">句子循环：关</button>
      <label>速率
        <select id="rateSel">
          <option>0.5</option><option>0.75</option><option selected>1</option><option>1.25</option><option>1.5</option><option>1.75</option><option>2</option>
        </select>
      </label>

      <button id="langBtn" style="margin-left:auto">切换字幕（英文）</button>
    </div>

    <div class="subtitleBox">
      <div class="subtitleMain" id="subtitleMain">——</div>
      <div class="subtitleCn" id="subtitleCn"></div>
      <div class="small" id="timeRange"></div>
      <div style="margin-top:8px" class="words" id="subtitleWords"></div>
    </div>

    <div class="row" style="margin-top:10px;align-items:center">
      <button id="playSentence">播放本句</button>
      <button id="gotoSentence">跳转本句（不播放）</button>
      <button id="copySentence">复制句子</button>

      <div style="margin-left:auto;display:flex;gap:8px;align-items:center">
        <button id="recordBtn">开始录音（跟读）</button>
        <div style="width:150px">
          <div class="small">实时音量</div>
          <div style="background:#eee;height:8px;border-radius:6px;overflow:hidden"><div id="volBar" style="height:8px;width:0%;background:#10b981"></div></div>
        </div>
        <div id="scoreDisplay" class="small" style="min-width:160px"></div>
      </div>
    </div>

    <div style="margin-top:14px">
      <h4 style="margin:6px 0">逐句听写（手动）</h4>
      <div class="small">听完当前句后，在下方输入你听到的句子并提交，系统会给出相似度评分（客户端计算）。</div>
      <textarea id="dictationInput" placeholder="在这里输入你听到的句子..."></textarea>
      <div style="margin-top:6px;display:flex;gap:8px;align-items:center">
        <button id="submitDict">提交听写</button>
        <button id="clearDict">清空</button>
        <div id="dictScore" class="small" style="margin-left:auto"></div>
      </div>
    </div>

    <div class="listGrid">
      <div class="panel">
        <div style="display:flex;justify-content:space-between;align-items:center">
          <strong>逐句列表</strong>
          <button id="exportTranscript">导出逐句 JSON</button>
        </div>
        <div id="listArea" style="margin-top:8px"></div>
      </div>

      <div class="panel" style="min-width:260px">
        <strong>单词本 / 生词</strong>
        <div class="wordbook" id="wordbookArea" style="margin-top:8px"></div>
        <div style="margin-top:10px" class="small">提示：点击字幕单词查词（无 API 时为本地提示），双击单词加入单词本。</div>
      </div>
    </div>

    <footer>
      <div class="small">注：Drive 直链可能在某些情况下返回下载（Content-Disposition）或有限速；如在 FlowUs iframe 中遇到问题，建议把视频上传到支持静态托管的 CDN（Vercel/Netlify/S3）。</div>
    </footer>
  </div>

<script>
/* ============ 配置 ============ */
const VIDEO_URL = 'https://drive.google.com/uc?export=download&id=1bIOYXSre9Ny_iZ7jk6RJwZu3K7h6ITsj'; // 已转换为 Drive 直链
const SCORE_API = ''; // 若有后端评分接口，填写 URL；否则留空（本地回放）
const DICT_API = '';  // 若有词典 API，可填写 GET 接口地址，例如 https://api.example/dict?q=WORD

/* ============ 逐句数据（你提供的中英对照） ============ */
const TRANSCRIPT = [
  { start: 0.0, end: 7.5, text: "We have sanded, we have cocked, we have gotten rid of the dust, the dirt, we've taped, and finally we are ready to paint.", text_cn: "我们已经把墙面打磨好了，也打了密封胶，清理掉灰尘和污垢，贴好了美纹纸。终于，我们准备好开始刷漆啦。" },
  { start: 7.6, end: 11.2, text: "Going to go in with a coat of primer first, and then I've got our paint here.", text_cn: "先上第一层底漆，然后我这儿已经准备好了正式的油漆。" },
  { start: 11.3, end: 16.5, text: "We did decide to go with sea salt, so I'm really excited to see what it looks like up on our walls.", text_cn: "我们最终决定选“Sea Salt”这个颜色，我真的很期待它刷在墙上的效果。" },
  { start: 16.6, end: 20.5, text: "We got the one coat coverage, but hopefully this should go fairly quickly.", text_cn: "我们买的是“一遍就能完全覆盖”的漆，希望这样可以刷得比较快。" },
  { start: 20.6, end: 27.4, text: "I feel like with projects like this, you always think it's going to be painting that's going to take the most time, but it's really all the prep work leading up to that.", text_cn: "我觉得像这种项目啊，人总以为最花时间的是刷漆，其实真正耗时的是之前的那些准备工作。" },
  { start: 27.5, end: 33.0, text: "We're done that now, though. So, let's start getting paint up on the walls. I can't wait to see what this color looks like in this room.", text_cn: "不过这些我们都已经搞定了！那就开始把漆刷上墙吧～我已经迫不及待想看这颜色在房间里的效果了。" },
  { start: 33.1, end: 36.2, text: "So, after priming, I mainly focused on the doors and all the trim pieces.", text_cn: "上完底漆后，我主要集中刷门和那些装饰线条部分。" },
  { start: 36.3, end: 40.8, text: "We waited a few hours for everything to dry and then went in with our sea salt paint.", text_cn: "我们等了几个小时让漆面完全干透，然后开始上“Sea Salt”的正式漆。" },
  { start: 40.9, end: 46.3, text: "Our strategy was to use a brush for the trim and any areas that were too narrow for our roller to really get in between,", text_cn: "我们的策略是：用刷子刷装饰线条和那些太窄、滚筒刷进不去的地方。" },
  { start: 46.4, end: 52.0, text: "but then for anything that was wide enough, I love the look of a space that's been brushed with roller, it does look just so much more clean.", text_cn: "但对于足够宽的区域，我真的很喜欢用滚筒刷出的效果——那样的墙面看起来更整洁、更干净。" }
];

/* ============ 状态 / DOM ============ */
const player = document.getElementById('player');
const fileInput = document.getElementById('fileInput');
const urlInput = document.getElementById('urlInput');
const useDriveBtn = document.getElementById('useDrive');
const playBtn = document.getElementById('playBtn');
const prevBtn = document.getElementById('prevBtn');
const nextBtn = document.getElementById('nextBtn');
const loopBtn = document.getElementById('loopBtn');
const rateSel = document.getElementById('rateSel');
const langBtn = document.getElementById('langBtn');

const subtitleMain = document.getElementById('subtitleMain');
const subtitleCn = document.getElementById('subtitleCn');
const timeRange = document.getElementById('timeRange');
const subtitleWords = document.getElementById('subtitleWords');

const playSentenceBtn = document.getElementById('playSentence');
const gotoSentenceBtn = document.getElementById('gotoSentence');
const copySentenceBtn = document.getElementById('copySentence');

const recordBtn = document.getElementById('recordBtn');
const volBar = document.getElementById('volBar');
const scoreDisplay = document.getElementById('scoreDisplay');

const dictInput = document.getElementById('dictationInput');
const submitDict = document.getElementById('submitDict');
const clearDict = document.getElementById('clearDict');
const dictScore = document.getElementById('dictScore');

const listArea = document.getElementById('listArea');
const exportBtn = document.getElementById('exportTranscript');

const wordbookArea = document.getElementById('wordbookArea');

/* ============ 应用逻辑 ============ */
let lang = 'en';
let cueIndex = 0;
let loopSentence = false;
let wordbook = JSON.parse(localStorage.getItem('sr_wordbook_v1')||'[]');
let newWords = JSON.parse(localStorage.getItem('sr_newwords_v1')||'[]');

function saveWords(){ localStorage.setItem('sr_wordbook_v1', JSON.stringify(wordbook)); localStorage.setItem('sr_newwords_v1', JSON.stringify(newWords)); }

function loadVideo(url){
  player.src = url;
  player.load();
  cueIndex = 0;
  renderCue();
}

/* 绑定文件 / URL 输入 */
fileInput.addEventListener('change', e=>{
  const f = e.target.files && e.target.files[0];
  if (!f) return;
  const u = URL.createObjectURL(f);
  loadVideo(u);
});
urlInput.addEventListener('keypress', e=>{
  if (e.key==='Enter'){
    const v = e.target.value.trim();
    if(v) loadVideo(v);
  }
});
useDriveBtn.addEventListener('click', ()=> loadVideo(VIDEO_URL));

/* 播放控制 */
playBtn.addEventListener('click', ()=> player.paused ? player.play() : player.pause() );
prevBtn.addEventListener('click', ()=> gotoCue(Math.max(0, cueIndex-1)) );
nextBtn.addEventListener('click', ()=> gotoCue(Math.min(TRANSCRIPT.length-1, cueIndex+1)) );
loopBtn.addEventListener('click', ()=>{
  loopSentence = !loopSentence;
  loopBtn.innerText = '句子循环：' + (loopSentence ? '开' : '关');
});
rateSel.addEventListener('change', ()=> player.playbackRate = Number(rateSel.value) );
langBtn.addEventListener('click', ()=> {
  lang = lang==='en' ? 'cn' : 'en';
  langBtn.innerText = `切换字幕（${lang==='en'?'英文':'中文'}）`;
  renderCue();
});

/* 时间更新 -> 自动定位句子 */
player.addEventListener('timeupdate', ()=>{
  const t = player.currentTime;
  const idx = TRANSCRIPT.findIndex(s => t >= s.start && t <= s.end);
  if (idx !== -1 && idx !== cueIndex){
    cueIndex = idx; renderCue();
  }
  if (loopSentence && cueIndex !== -1 && t >= (TRANSCRIPT[cueIndex].end - 0.05)){
    player.currentTime = Math.max(TRANSCRIPT[cueIndex].start + 0.01, 0);
    player.play();
  }
});

/* 渲染当前句子 & 单词按钮 & 列表 */
function renderCue(){
  const cur = TRANSCRIPT[cueIndex] || {text:'', text_cn:'', start:0, end:0};
  subtitleMain.innerText = lang==='en' ? cur.text : cur.text_cn;
  subtitleCn.innerText = lang==='en' ? cur.text_cn : cur.text;
  timeRange.innerText = `${cur.start.toFixed(2)}s — ${cur.end.toFixed(2)}s`;
  // 单词可点
  subtitleWords.innerHTML = '';
  const parts = (lang==='en' ? cur.text : cur.text_cn).split(/(\s+|[,.?!;:—–，。？！；：])/g).filter(Boolean);
  parts.forEach((p,i)=>{
    const btn = document.createElement('button');
    btn.innerText = p;
    btn.title = '单击查词 / 双击加入单词本';
    btn.addEventListener('click', ()=> lookupWord(p.replace(/[^\w\u4e00-\u9fff'-]/g,'')) );
    btn.addEventListener('dblclick', ()=> addToWordbook(p.replace(/[^\w\u4e00-\u9fff'-]/g,'')) );
    subtitleWords.appendChild(btn);
    subtitleWords.appendChild(document.createTextNode(' '));
  });
  renderList();
  renderWordbook();
}

/* 逐句列表 */
function renderList(){
  listArea.innerHTML = '';
  TRANSCRIPT.forEach((t,i)=>{
    const div = document.createElement('div');
    div.className = 'item' + (i===cueIndex ? ' active' : '');
    div.innerHTML = `<div style="font-weight:600">${t.text}</div><div class="small">${t.start.toFixed(2)}s — ${t.end.toFixed(2)}s</div>`;
    div.addEventListener('click', ()=> gotoCue(i) );
    listArea.appendChild(div);
  });
}

/* 跳转 */
function gotoCue(i, autoplay=true){
  if (i<0 || i>=TRANSCRIPT.length) return;
  cueIndex = i;
  player.currentTime = Math.max(TRANSCRIPT[i].start + 0.01, 0);
  if (autoplay) player.play();
  renderCue();
}

/* 播放 / 跳转当前句 */
playSentenceBtn.addEventListener('click', ()=> gotoCue(cueIndex, true) );
gotoSentenceBtn.addEventListener('click', ()=> gotoCue(cueIndex, false) );
copySentenceBtn.addEventListener('click', ()=> { navigator.clipboard.writeText(TRANSCRIPT[cueIndex].text||''); alert('已复制当前英文句子'); } );

/* 查词（如果无 DICT_API 则提示） */
async function lookupWord(word){
  const clean = (''+word).trim();
  if (!clean) return;
  if (!DICT_API){
    // 本地提示 + 快速 Google 搜索
    const ok = confirm(`未配置在线词典 API。\n是否在新标签页用 Google 搜索 "${clean}"？`);
    if (ok) window.open('https://www.google.com/search?q=' + encodeURIComponent(clean + ' meaning'), '_blank');
    return;
  }
  try {
    const res = await fetch(`${DICT_API}?q=${encodeURIComponent(clean)}`);
    const j = await res.json();
    alert(JSON.stringify(j,null,2));
  } catch (e){
    alert('查词失败：' + e.message);
  }
}

/* 生词本 */
function addToWordbook(word){
  const w = (''+word).toLowerCase().trim();
  if (!w) return;
  if (!wordbook.find(x=>x.word===w)) wordbook.unshift({word:w, added:Date.now()});
  if (!newWords.includes(w)) newWords.unshift(w);
  saveWords();
  renderWordbook();
}
function toggleNew(word){
  if (newWords.includes(word)) newWords = newWords.filter(x=>x!==word);
  else newWords.unshift(word);
  saveWords(); renderWordbook();
}
function renderWordbook(){
  wordbookArea.innerHTML = '';
  if (!wordbook.length) { wordbookArea.innerHTML = '<div class="small">暂无收藏，双击字幕单词可加入单词本。</div>'; return; }
  wordbook.forEach(w=>{
    const div = document.createElement('div');
    div.className = 'rowItem';
    div.innerHTML = `<div style="display:flex;justify-content:space-between;align-items:center"><div>${w.word}</div>
      <div style="display:flex;gap:6px"><button class="smallBtn">${ newWords.includes(w.word) ? '已标记' : '标记生词' }</button><button class="smallBtn danger">移除</button></div></div>`;
    const markBtn = div.querySelector('.smallBtn');
    const remBtn = div.querySelector('.danger');
    markBtn.addEventListener('click', ()=> { toggleNew(w.word); });
    remBtn.addEventListener('click', ()=> { wordbook = wordbook.filter(x=>x.word!==w.word); saveWords(); renderWordbook(); });
    wordbookArea.appendChild(div);
  });
}

/* 导出 JSON */
exportBtn.addEventListener('click', ()=>{
  const blob = new Blob([JSON.stringify({transcript:TRANSCRIPT},null,2)],{type:'application/json'});
  const a = document.createElement('a'); a.href = URL.createObjectURL(blob); a.download = 'transcript.json'; a.click();
});

/* 简单听写评分（Levenshtein）*/
function levenshtein(a,b){
  const m=a.length,n=b.length; const dp=Array.from({length:m+1},()=>Array(n+1).fill(0));
  for(let i=0;i<=m;i++) dp[i][0]=i; for(let j=0;j<=n;j++) dp[0][j]=j;
  for(let i=1;i<=m;i++) for(let j=1;j<=n;j++){
    dp[i][j]=Math.min(dp[i-1][j]+1, dp[i][j-1]+1, dp[i-1][j-1] + (a[i-1]===b[j-1]?0:1));
  }
  return dp[m][n];
}
submitDict.addEventListener('click', ()=>{
  const user = (dictInput.value||'').toLowerCase().replace(/[^a-z\u4e00-\u9fff\s]/g,'').trim();
  const ref = (TRANSCRIPT[cueIndex].text||'').toLowerCase().replace(/[^a-z\u4e00-\u9fff\s]/g,'').trim();
  if (!user) { alert('请输入听写内容'); return; }
  const dist = levenshtein(user,ref);
  const max = Math.max(ref.length, user.length,1);
  const score = Math.max(0, Math.round((1 - dist/max)*100));
  dictScore.innerText = '听写得分：' + score;
});
clearDict.addEventListener('click', ()=> { dictInput.value=''; dictScore.innerText=''; });

/* 录音（MediaRecorder + 简单可视化） */
let mediaRec = null, audioCtx=null, analyser=null, sourceNode=null;
recordBtn.addEventListener('click', async ()=>{
  if (!mediaRec || mediaRec.state === 'inactive'){
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      audioCtx = new (window.AudioContext || window.webkitAudioContext)();
      analyser = audioCtx.createAnalyser();
      sourceNode = audioCtx.createMediaStreamSource(stream);
      sourceNode.connect(analyser);
      analyser.fftSize = 256;
      const data = new Uint8Array(analyser.frequencyBinCount);
      const tick = ()=> {
        analyser.getByteFrequencyData(data);
        let sum = 0; for(let i=0;i<data.length;i++) sum += data[i];
        const avg = sum / data.length;
        volBar.style.width = Math.min(100, avg/2) + '%';
        if (mediaRec && mediaRec.state === 'recording') requestAnimationFrame(tick);
      };
      mediaRec = new MediaRecorder(stream);
      const chunks = [];
      mediaRec.ondataavailable = e => chunks.push(e.data);
      mediaRec.onstop = async ()=>{
        const blob = new Blob(chunks, { type:'audio/webm' });
        if (SCORE_API){
          // 上传并显示评分（SCORE_API 应接受 multipart/form-data，返回 JSON）
          try {
            scoreDisplay.innerText = '评分中...';
            const form = new FormData();
            form.append('audio', blob, 'rec.webm');
            form.append('referenceText', TRANSCRIPT[cueIndex].text || '');
            const res = await fetch(SCORE_API, { method:'POST', body: form });
            const j = await res.json();
            scoreDisplay.innerText = j.score ? `AI 评分：${j.score}` : JSON.stringify(j);
          } catch (e) {
            scoreDisplay.innerText = '评分错误：' + e.message;
          }
        } else {
          // 本地回放示例
          const au = document.createElement('audio'); au.src = URL.createObjectURL(blob); au.controls=true; au.play();
          scoreDisplay.innerText = '录音已生成（未配置评分接口）';
        }
      };
      mediaRec.start();
      tick();
      recordBtn.innerText = '停止录音';
    } catch (e){
      alert('无法访问麦克风：' + e.message);
    }
  } else {
    mediaRec.stop();
    recordBtn.innerText = '开始录音（跟读）';
    if (audioCtx) { audioCtx.close(); audioCtx = null; analyser=null; sourceNode=null; }
    mediaRec = null;
  }
});

/* 简单按键支持 */
window.addEventListener('keydown', e=>{
  if (e.code === 'Space'){ e.preventDefault(); player.paused ? player.play() : player.pause(); }
  else if (e.key === ',') gotoCue(Math.max(0, cueIndex-1));
  else if (e.key === '.') gotoCue(Math.min(TRANSCRIPT.length-1, cueIndex+1));
});

/* 初始化 */
(function init(){
  // 先默认加载 Drive 链接（也可不加载）
  loadVideo(VIDEO_URL);
  renderCue();
})();
</script>
</body>
</html>
