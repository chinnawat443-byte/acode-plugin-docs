<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>ระบบเช็คชื่อและให้คะแนน</title>
  <style>
    * { box-sizing: border-box; font-family: Arial, sans-serif; }
    body {
      margin: 0;
      background: #f5f7fb;
      color: #1f2937;
    }
    .wrap {
      max-width: 980px;
      margin: 0 auto;
      padding: 12px;
    }
    .card {
      background: #fff;
      border-radius: 18px;
      padding: 14px;
      margin-bottom: 12px;
      box-shadow: 0 2px 10px rgba(0,0,0,.06);
    }
    h1, h2, h3 { margin: 0 0 8px 0; }
    .row { display: flex; gap: 10px; flex-wrap: wrap; }
    .col { flex: 1; min-width: 0; }
    input, textarea, button, select {
      width: 100%;
      border: 1px solid #d1d5db;
      border-radius: 12px;
      padding: 12px;
      font-size: 16px;
    }
    textarea { min-height: 100px; resize: vertical; }
    button {
      background: #111827;
      color: white;
      border: none;
      cursor: pointer;
    }
    button.outline {
      background: white;
      color: #111827;
      border: 1px solid #cbd5e1;
    }
    .badge {
      display: inline-block;
      padding: 6px 12px;
      border-radius: 999px;
      background: #e5e7eb;
      font-size: 14px;
      margin-right: 6px;
      margin-top: 6px;
    }
    .layout {
      display: grid;
      grid-template-columns: 320px 1fr;
      gap: 12px;
    }
    @media (max-width: 760px) {
      .layout { grid-template-columns: 1fr; }
    }
    .list {
      max-height: 60vh;
      overflow: auto;
      display: grid;
      gap: 8px;
    }
    .item {
      padding: 12px;
      border-radius: 14px;
      border: 1px solid #e5e7eb;
      background: white;
      cursor: pointer;
    }
    .item.active {
      background: #111827;
      color: white;
    }
    .small { font-size: 12px; opacity: .75; }
    .score-grid {
      display: grid;
      grid-template-columns: repeat(5,1fr);
      gap: 8px;
      margin-top: 10px;
    }
    .score-btn {
      padding: 12px 0;
      border-radius: 14px;
      border: 1px solid #cbd5e1;
      background: white;
      color: #111827;
      font-weight: bold;
    }
    .score-btn.active {
      background: #111827;
      color: white;
    }
    .section {
      border: 1px solid #e5e7eb;
      border-radius: 16px;
      padding: 12px;
      background: #fff;
      margin-bottom: 10px;
    }
    .muted { color: #6b7280; }
    .top-actions {
      display: grid;
      grid-template-columns: 1fr auto;
      gap: 10px;
    }
    @media (max-width: 760px) {
      .top-actions { grid-template-columns: 1fr; }
    }
    .checkline {
      display: flex;
      align-items: center;
      gap: 10px;
      padding: 12px;
      border: 1px solid #e5e7eb;
      border-radius: 14px;
    }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card">
      <h1>ระบบเช็คชื่อและให้คะแนนรายบุคคล</h1>
      <div>
        <span class="badge" id="countAll">ผู้เข้าร่วม 0 คน</span>
        <span class="badge" id="countCheckin">เช็คชื่อแล้ว 0 คน</span>
      </div>

      <div class="top-actions" style="margin-top:10px;">
        <input id="evaluator" placeholder="ชื่อผู้ประเมิน" />
        <button class="outline" onclick="toggleSetup()">จัดการรายชื่อ</button>
      </div>

      <div id="setupBox" class="card" style="display:none; margin-top:12px; border:1px dashed #cbd5e1; box-shadow:none;">
        <h3>วางรายชื่อใหม่ คนละ 1 บรรทัด</h3>
        <textarea id="bulkNames" placeholder="เช่น
สมชาย ใจดี
สมหญิง ใจงาม"></textarea>
        <div style="margin-top:10px;">
          <button onclick="loadNames()">โหลดรายชื่อเข้าระบบ</button>
        </div>
      </div>
    </div>

    <div class="layout">
      <div class="card">
        <h2>ค้นหารายชื่อ</h2>
        <input id="searchBox" placeholder="พิมพ์ชื่อหรือรหัส" oninput="renderList()" />
        <div class="list" id="participantList" style="margin-top:10px;"></div>
      </div>

      <div class="card" id="detailBox">
        <div id="detailContent"></div>
      </div>
    </div>
  </div>

  <script>
    let participants = [
      "ญาดา ไชยพิมูล",
      "วิลัยศิยา พันธุ์เงิน",
      "พิมพ์ชนก พันธ์รุพงศ์",
      "ศุภิสรา พรหมวิจิต",
      "อิสรีย์ แสนคำภา"
    ].map((name, i) => ({
      id: String(i + 1).padStart(3, "0"),
      name,
      checkedIn: false,
      scores: { creativity: 0, social: 0, team: 0 },
      note: "",
      saved: false,
      evaluator: ""
    }));

    let selectedId = participants[0]?.id || "";

    const dimensions = [
      { key: "creativity", title: "Creativity & Thinking", subtitle: "ความคิด + AI + การคิดแบบผู้ประกอบการ" },
      { key: "social", title: "Social Responsibility Mindset", subtitle: "ความเข้าใจสังคม + Impact" },
      { key: "team", title: "Team Contribution", subtitle: "บทบาทในทีม + Leadership" }
    ];

    function toggleSetup() {
      const box = document.getElementById("setupBox");
      box.style.display = box.style.display === "none" ? "block" : "none";
    }

    function loadNames() {
      const bulk = document.getElementById("bulkNames").value.trim();
      if (!bulk) return;

      const rows = bulk.split("\n").map(x => x.trim()).filter(Boolean);
      participants = rows.map((name, i) => ({
        id: String(i + 1).padStart(3, "0"),
        name,
        checkedIn: false,
        scores: { creativity: 0, social: 0, team: 0 },
        note: "",
        saved: false,
        evaluator: ""
      }));
      selectedId = participants[0]?.id || "";
      document.getElementById("setupBox").style.display = "none";
      renderAll();
    }

    function getSelected() {
      return participants.find(p => p.id === selectedId) || participants[0] || null;
    }

    function totalScore(p) {
      return p.scores.creativity + p.scores.social + p.scores.team;
    }

    function updateCounts() {
      document.getElementById("countAll").textContent = `ผู้เข้าร่วม ${participants.length} คน`;
      document.getElementById("countCheckin").textContent = `เช็คชื่อแล้ว ${participants.filter(p => p.checkedIn).length} คน`;
    }

    function renderList() {
      updateCounts();
      const q = document.getElementById("searchBox").value.trim().toLowerCase();
      const list = document.getElementById("participantList");

      const filtered = !q
        ? participants
        : participants.filter(p =>
            p.name.toLowerCase().includes(q) || p.id.toLowerCase().includes(q)
          );

      list.innerHTML = filtered.map(p => `
        <div class="item ${p.id === selectedId ? "active" : ""}" onclick="selectParticipant('${p.id}')">
          <div style="display:flex;justify-content:space-between;gap:10px;">
            <div>
              <div class="small">ID ${p.id}</div>
              <div style="font-weight:bold;">${escapeHtml(p.name)}</div>
            </div>
            <div style="text-align:right;">
              <div class="small">รวม</div>
              <div style="font-weight:bold;">${totalScore(p)}</div>
            </div>
          </div>
          <div style="margin-top:8px;">
            ${p.checkedIn ? '<span class="badge">มาแล้ว</span>' : ''}
            ${p.saved ? '<span class="badge">บันทึกแล้ว</span>' : ''}
          </div>
        </div>
      `).join("");
    }

    function selectParticipant(id) {
      selectedId = id;
      renderAll();
    }

    function setCheckedIn(val) {
      const p = getSelected();
      if (!p) return;
      p.checkedIn = val;
      renderAll();
    }

    function setScore(key, value) {
      const p = getSelected();
      if (!p) return;
      p.scores[key] = value;
      p.saved = false;
      renderAll();
    }

    function setNote(val) {
      const p = getSelected();
      if (!p) return;
      p.note = val;
      p.saved = false;
    }

    function saveCurrent() {
      const p = getSelected();
      if (!p) return;
      const evaluator = document.getElementById("evaluator").value.trim() || "ผู้ประเมิน";
      p.saved = true;
      p.checkedIn = true;
      p.evaluator = evaluator;
      renderAll();
    }

    function resetCurrent() {
      const p = getSelected();
      if (!p) return;
      p.scores = { creativity: 0, social: 0, team: 0 };
      p.note = "";
      p.saved = false;
      renderAll();
    }

    function exportText() {
      return participants
        .filter(p =>
          p.saved ||
          p.checkedIn ||
          totalScore(p) > 0 ||
          (p.note && p.note.trim() !== "")
        )
        .map(p =>
          `${p.id} | ${p.name} | เช็คชื่อ:${p.checkedIn ? "มา" : "ยังไม่มา"} | Creativity:${p.scores.creativity} | Social:${p.scores.social} | Team:${p.scores.team} | Total:${totalScore(p)} | หมายเหตุ:${p.note || "-"}`
        )
        .join("\n");
    }

    function copyExport() {
      const txt = exportText();
      navigator.clipboard.writeText(txt).then(() => {
        alert("คัดลอกข้อมูลแล้ว");
      }).catch(() => {
        alert("คัดลอกไม่สำเร็จ ลองกดค้างแล้วคัดลอกเอง");
      });
    }

    function renderDetail() {
      const box = document.getElementById("detailContent");
      const p = getSelected();

      if (!p) {
        box.innerHTML = "<div>ยังไม่มีข้อมูล</div>";
        return;
      }

      box.innerHTML = `
        <div style="display:flex;justify-content:space-between;gap:10px;align-items:flex-start;">
          <div>
            <div class="muted">รหัส ${p.id}</div>
            <h2 style="margin-top:4px;">${escapeHtml(p.name)}</h2>
          </div>
          <div class="badge" style="font-size:16px;">รวม ${totalScore(p)}/15</div>
        </div>

        <div class="checkline" style="margin:12px 0;">
          <input type="checkbox" id="checkin" ${p.checkedIn ? "checked" : ""} onchange="setCheckedIn(this.checked)" style="width:20px;height:20px;" />
          <label for="checkin" style="font-weight:bold;">เช็คชื่อแล้ว</label>
        </div>

        ${dimensions.map(d => `
          <div class="section">
            <div style="font-weight:bold;">${d.title}</div>
            <div class="muted" style="font-size:14px;">${d.subtitle}</div>
            <div class="score-grid">
              ${[1,2,3,4,5].map(n => `
                <button type="button" class="score-btn ${p.scores[d.key] === n ? "active" : ""}" onclick="setScore('${d.key}', ${n})">${n}</button>
              `).join("")}
            </div>
          </div>
        `).join("")}

        <div class="section">
          <div style="font-weight:bold; margin-bottom:8px;">หมายเหตุ</div>
          <textarea oninput="setNote(this.value)">${escapeHtml(p.note)}</textarea>
        </div>

        <div class="row">
          <div class="col"><button onclick="saveCurrent()">บันทึก</button></div>
          <div class="col"><button class="outline" onclick="resetCurrent()">ล้างคะแนน</button></div>
        </div>

        <div class="section" style="margin-top:12px;">
          <div style="font-weight:bold; margin-bottom:8px;">สรุปที่คัดลอกออกไปใช้ได้ทันที</div>
          <textarea readonly>${escapeHtml(exportText())}</textarea>
          <div style="margin-top:10px;">
            <button class="outline" onclick="copyExport()">คัดลอกข้อมูล</button>
          </div>
        </div>
      `;
    }

    function escapeHtml(str) {
      return String(str || "")
        .replaceAll("&", "&amp;")
        .replaceAll("<", "&lt;")
        .replaceAll(">", "&gt;")
        .replaceAll('"', "&quot;")
        .replaceAll("'", "&#039;");
    }

    function renderAll() {
      renderList();
      renderDetail();
    }

    renderAll();
  </script>
</body>
</html>
