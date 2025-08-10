# writing-logger
<!doctype html>
<html lang="vi">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Writer Logger — Ghi lại quá trình viết</title>
<style>
  :root { font-family: system-ui, -apple-system, Segoe UI, Roboto, sans-serif; }
  body { margin: 0; background: #fafafa; color: #111; }
  header { padding: 16px 20px; background: #fff; border-bottom: 1px solid #eee; position: sticky; top: 0; z-index: 5;}
  header h1 { margin: 0; font-size: 18px; }
  main { display: grid; grid-template-columns: 1fr 320px; gap: 16px; padding: 16px; }
  #editorWrap { background: #fff; border: 1px solid #e5e5e5; border-radius: 10px; }
  #toolbar { padding: 10px 12px; border-bottom: 1px solid #eee; display:flex; gap:8px; align-items:center; flex-wrap: wrap;}
  #toolbar input, #toolbar button, #toolbar select { padding: 8px 10px; border-radius: 8px; border: 1px solid #ddd; background:#fff; }
  #editor { width: 100%; min-height: 60vh; border: 0; padding: 14px; font-size: 16px; line-height: 1.6; outline: none; resize: vertical; }
  aside { position: sticky; top: 68px; align-self: start; }
  .card { background:#fff; border:1px solid #e5e5e5; border-radius:10px; padding:12px 14px; margin-bottom: 12px; }
  .row { display:flex; justify-content: space-between; margin: 6px 0; font-size: 14px; }
  .muted { color:#666; font-size: 12px; }
  .btn-primary { background:#111; color:#fff; border-color:#111; }
  .danger { color:#b00020; }
  footer { padding: 12px 20px; color:#666; font-size: 12px; }
  code.small { background:#f4f4f4; padding:2px 6px; border-radius:6px; }
</style>
</head>
<body>
<header>
  <h1>Writer Logger — Ghi lại quá trình viết</h1>
</header>

<main>
  <section id="editorWrap">
    <div id="toolbar">
      <label> Mã người viết:
        <input id="participantId" placeholder="VD: HS-001" />
      </label>
      <label> Đề bài:
        <input id="prompt" placeholder="Nhập đề bài / câu hỏi" size="40"/>
      </label>
      <label> Ngưỡng tạm dừng (giây):
        <select id="idleThreshold">
          <option value="2">2</option>
          <option value="5" selected>5</option>
          <option value="10">10</option>
          <option value="30">30</option>
          <option value="60">60</option>
        </select>
      </label>
      <button id="btnStart" class="btn-primary">Bắt đầu ghi</button>
      <button id="btnStop">Dừng ghi</button>
      <button id="btnMark">Đánh dấu mốc</button>
    </div>
    <textarea id="editor" placeholder="Viết bài tại đây..."></textarea>
  </section>

  <aside>
    <div class="card">
      <div class="row"><strong>Trạng thái</strong><span id="status">Chưa ghi</span></div>
      <div class="row"><span>Bắt đầu</span><span id="startedAt">—</span></div>
      <div class="row"><span>Kết thúc</span><span id="endedAt">—</span></div>
      <div class="row"><span>Thời lượng phiên</span><span id="duration">0:00</span></div>
    </div>

    <div class="card">
      <div class="row"><span>Tổng ký tự</span><span id="chars">0</span></div>
      <div class="row"><span>Tổng từ (approx.)</span><span id="words">0</span></div>
      <div class="row"><span>Số lần <code class="small">paste</code></span><span id="pastes">0</span></div>
      <div class="row"><span>Số lần xóa</span><span id="deletes">0</span></div>
      <div class="row"><span>Burst gõ</span><span id="bursts">0</span></div>
      <div class="row"><span>Tạm dừng ≥ ngưỡng</span><span id="pauses">0</span></div>
    </div>

    <div class="card">
      <div class="row"><button id="btnExportJson" class="btn-primary">Xuất log (JSON)</button></div>
      <div class="row"><button id="btnExportCsv">Xuất log (CSV)</button></div>
      <div class="row"><button id="btnSaveLocal">Lưu tạm (Local)</button></div>
      <div class="row"><button id="btnLoadLocal">Khôi phục (Local)</button></div>
      <div class="row"><button id="btnClearLocal"><span class="danger">Xóa Local</span></button></div>
      <p class="muted">Dữ liệu chỉ ghi trong trang này (không gửi đi đâu) trừ khi bạn tự xuất file.</p>
    </div>
  </aside>
</main>

<footer>
  <p class="muted">
    Ghi lại: <strong>keydown/input/paste/cut/selection/focus/blur/visibility</strong> + timestamp. 
    Mẹo: dùng cùng file này cho tất cả HS, mỗi người điền <em>Mã người viết</em> trước khi bấm <em>Bắt đầu ghi</em>.
  </p>
</footer>

<script>
(() => {
  // === State ===
  let isRecording = false;
  let sessionStart = null;
  let sessionEnd = null;
  let lastEventTs = null;
  let bursts = 0;
  let pauses = 0;
  let pasteCount = 0;
  let deleteCount = 0;
  let idleThresholdMs = 5000;

  const logs = []; // mỗi phần tử là 1 event log

  // === DOM ===
  const el = (id) => document.getElementById(id);
  const editor = el('editor');
  const status = el('status');
  const startedAt = el('startedAt');
  const endedAt = el('endedAt');
  const duration = el('duration');
  const chars = el('chars');
  const words = el('words');
  const pastes = el('pastes');
  const deletes = el('deletes');
  const burstsEl = el('bursts');
  const pausesEl = el('pauses');
  const participantId = el('participantId');
  const prompt = el('prompt');
  const idleSelect = el('idleThreshold');

  // === Helpers ===
  const nowISO = () => new Date().toISOString();
  const fmtDuration = (ms) => {
    if (!ms || ms < 0) return '0:00';
    const s = Math.floor(ms/1000);
    const m = Math.floor(s/60);
    const sec = s % 60;
    return `${m}:${sec.toString().padStart(2,'0')}`;
  };
  const wordCount = (t) => (t.trim().match(/\S+/g) || []).length;

  function updateStats() {
    const txt = editor.value;
    chars.textContent = txt.length.toString();
    words.textContent = wordCount(txt).toString();
    pastes.textContent = pasteCount.toString();
    deletes.textContent = deleteCount.toString();
    burstsEl.textContent = bursts.toString();
    pausesEl.textContent = pauses.toString();

    const end = sessionEnd ?? new Date();
    if (sessionStart) duration.textContent = fmtDuration(end - sessionStart);
  }

  function bumpPauseAndBurst(ts) {
    if (lastEventTs) {
      const gap = ts - lastEventTs;
      if (gap >= idleThresholdMs) {
        pauses++;
        bursts++; // một đợt gõ mới sau khoảng dừng dài
      }
    } else {
      bursts = 1; // đợt đầu tiên
    }
    lastEventTs = ts;
  }

  function pushLog(evtType, payload = {}) {
    const ts = Date.now();
    bumpPauseAndBurst(ts);

    const entry = {
      t: new Date(ts).toISOString(),
      type: evtType,    // 'keydown','input','paste','cut','selection','focus','blur','visibility','mark'
      pid: participantId.value || null,
      prompt: prompt.value || null,
      caretStart: editor.selectionStart,
      caretEnd: editor.selectionEnd,
      textLen: editor.value.length,
      ...payload
    };
    logs.push(entry);
    updateStats();
  }

  function start() {
    if (isRecording) return;
    idleThresholdMs = parseInt(idleSelect.value,10) * 1000;
    isRecording = true;
    sessionStart = new Date();
    sessionEnd = null;
    lastEventTs = null;
    bursts = 0; pauses = 0; pasteCount = 0; deleteCount = 0;
    logs.length = 0; // reset
    status.textContent = 'Đang ghi…';
    startedAt.textContent = sessionStart.toLocaleString();
    endedAt.textContent = '—';
    updateStats();
    editor.focus();
  }

  function stop() {
    if (!isRecording) return;
    isRecording = false;
    sessionEnd = new Date();
    status.textContent = 'Đã dừng';
    endedAt.textContent = sessionEnd.toLocaleString();
    updateStats();
  }

  function mark() {
    if (!isRecording) return;
    pushLog('mark', { note: 'User mark' });
  }

  // === Event wiring ===
  el('btnStart').addEventListener('click', start);
  el('btnStop').addEventListener('click', stop);
  el('btnMark').addEventListener('click', mark);

  editor.addEventListener('keydown', (e) => {
    if (!isRecording) return;
    if (e.key === 'Backspace' || e.key === 'Delete') deleteCount++;
    pushLog('keydown', { key: e.key, ctrl: e.ctrlKey, alt: e.altKey, shift: e.shiftKey, meta: e.metaKey });
  });

  editor.addEventListener('input', (e) => {
    if (!isRecording) return;
    pushLog('input', { inputType: e.inputType ?? null });
  });

  editor.addEventListener('paste', (e) => {
    if (!isRecording) return;
    let pastedLen = 0;
    try {
      const text = (e.clipboardData || window.clipboardData).getData('text');
      pastedLen = text ? text.length : 0;
    } catch {}
    pasteCount++;
    pushLog('paste', { pastedLen });
  });

  editor.addEventListener('cut', () => { if (isRecording) pushLog('cut'); });
  document.addEventListener('selectionchange', () => {
    if (!isRecording) return;
    // chỉ log nếu selection trong editor
    if (document.activeElement === editor) pushLog('selection');
  });

  window.addEventListener('focus', () => { if (isRecording) pushLog('focus'); });
  window.addEventListener('blur', () => { if (isRecording) pushLog('blur'); });
  document.addEventListener('visibilitychange', () => {
    if (isRecording) pushLog('visibility', { hidden: document.hidden });
  });

  // === Export / Local storage ===
  function download(filename, content, mime='application/json') {
    const blob = new Blob([content], { type: mime });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url; a.download = filename; a.click();
    URL.revokeObjectURL(url);
  }

  el('btnExportJson').addEventListener('click', () => {
    const payload = {
      meta: {
        tool: 'Writer Logger',
        version: '1.0',
        pid: participantId.value || null,
        prompt: prompt.value || null,
        startedAt: sessionStart ? sessionStart.toISOString() : null,
        endedAt: sessionEnd ? sessionEnd.toISOString() : null,
        idleThresholdMs,
        textFinalLen: editor.value.length,
        textFinal: editor.value
      },
      logs
    };
    download(`log_${participantId.value || 'anon'}_${Date.now()}.json`, JSON.stringify(payload, null, 2));
  });

  el('btnExportCsv').addEventListener('click', () => {
    const headers = ['t','type','pid','caretStart','caretEnd','textLen','key','inputType','ctrl','alt','shift','meta','pastedLen','hidden','note'];
    const rows = [headers.join(',')];
    for (const L of logs) {
      const row = headers.map(h => {
        const v = (L[h] !== undefined) ? L[h] : '';
        const s = String(v).replace(/"/g,'""');
        return `"${s}"`;
      }).join(',');
      rows.push(row);
    }
    download(`log_${participantId.value || 'anon'}_${Date.now()}.csv`, rows.join('\n'), 'text/csv');
  });

  el('btnSaveLocal').addEventListener('click', () => {
    const snapshot = {
      ts: Date.now(),
      sessionStart: sessionStart ? sessionStart.toISOString() : null,
      sessionEnd: sessionEnd ? sessionEnd.toISOString() : null,
      isRecording,
      pasteCount, deleteCount, bursts, pauses, idleThresholdMs,
      text: editor.value,
      participantId: participantId.value,
      prompt: prompt.value,
      logs
    };
    localStorage.setItem('writerLogger.snapshot', JSON.stringify(snapshot));
    alert('Đã lưu tạm vào trình duyệt.');
  });

  el('btnLoadLocal').addEventListener('click', () => {
    const data = localStorage.getItem('writerLogger.snapshot');
    if (!data) return alert('Chưa có dữ liệu lưu tạm.');
    try {
      const s = JSON.parse(data);
      editor.value = s.text || '';
      participantId.value = s.participantId || '';
      prompt.value = s.prompt || '';
      pasteCount = s.pasteCount || 0;
      deleteCount = s.deleteCount || 0;
      bursts = s.bursts || 0;
      pauses = s.pauses || 0;
      idleThresholdMs = s.idleThresholdMs || 5000;
      if (s.sessionStart) sessionStart = new Date(s.sessionStart);
      if (s.sessionEnd) sessionEnd = new Date(s.sessionEnd);
      lastEventTs = null; // reset
      logs.length = 0; logs.push(...(s.logs || []));
      status.textContent = s.isRecording ? 'Đang ghi (khôi phục)' : 'Khôi phục';
      startedAt.textContent = s.sessionStart ? new Date(s.sessionStart).toLocaleString() : '—';
      endedAt.textContent = s.sessionEnd ? new Date(s.sessionEnd).toLocaleString() : '—';
      updateStats();
      alert('Đã khôi phục dữ liệu.');
    } catch (e) { alert('Không thể khôi phục: ' + e.message); }
  });

  el('btnClearLocal').addEventListener('click', () => {
    localStorage.removeItem('writerLogger.snapshot');
    alert('Đã xóa dữ liệu lưu tạm trong trình duyệt.');
  });

  // Khởi tạo
  updateStats();
})();
</script>
</body>
</html>

