<!DOCTYPE html>
<html lang="ar" dir="rtl">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>نظام الحضور الشامل - النسخة القاضية</title>
    <script src="https://unpkg.com/html5-qrcode"></script>
    <style>
        :root { --primary: #1a2a3a; --accent: #3498db; --success: #2ecc71; --danger: #e74c3c; --bg: #f4f7f6; }
        body { font-family: 'Segoe UI', Tahoma, sans-serif; background: var(--bg); margin: 0; padding-bottom: 80px; }
        .bottom-nav { background: var(--primary); color: white; display: flex; position: fixed; bottom: 0; width: 100%; height: 70px; z-index: 1000; box-shadow: 0 -2px 10px rgba(0,0,0,0.2); }
        .nav-btn { flex: 1; display: flex; flex-direction: column; align-items: center; justify-content: center; cursor: pointer; opacity: 0.6; transition: 0.3s; }
        .nav-btn.active { opacity: 1; background: rgba(255,255,255,0.1); border-top: 4px solid var(--accent); }
        .container { max-width: 1100px; margin: auto; padding: 15px; }
        .card { background: white; border-radius: 15px; padding: 20px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); margin-bottom: 20px; }
        .header-bar { background: var(--accent); color: white; padding: 15px; border-radius: 12px; margin-bottom: 15px; display: flex; justify-content: space-between; align-items: center; }
        select { padding: 10px; border-radius: 8px; border: none; font-weight: bold; cursor: pointer; }
        #reader { width: 100%; max-width: 500px; margin: auto; border-radius: 15px; border: 5px solid var(--primary); overflow: hidden; }
        .status-msg { margin-top: 15px; padding: 15px; border-radius: 10px; font-weight: bold; text-align: center; font-size: 1.2rem; transition: 0.3s; }
        .bulk-input { width: 100%; height: 120px; padding: 12px; border-radius: 10px; border: 2px solid #ddd; margin-top: 10px; resize: none; box-sizing: border-box; }
        .table-wrapper { overflow-x: auto; margin-top: 15px; border-radius: 10px; }
        table { width: 100%; border-collapse: collapse; background: white; min-width: 600px; }
        th, td { border: 1px solid #eee; padding: 12px 8px; text-align: center; font-size: 14px; }
        th { background: var(--primary); color: white; }
        .is-present { background: #d4edda !important; font-weight: bold; color: #155724; }
        .calc-col { background: #e3f2fd; font-weight: bold; color: var(--primary); border-left: 2px solid var(--accent); }
        .day-check { display: inline-block; margin: 5px; padding: 8px 12px; background: #eee; border-radius: 20px; cursor: pointer; font-size: 13px; }
        .day-check input { margin-left: 5px; }
        .btn { padding: 12px 20px; border-radius: 8px; border: none; cursor: pointer; font-weight: bold; width: 100%; margin-top: 10px; }
        .btn-green { background: var(--success); color: white; }
        .btn-danger { background: var(--danger); color: white; }
        .btn-accent { background: var(--accent); color: white; }
        @media print { .no-print, .bottom-nav { display: none !important; } .card { box-shadow: none; border: 1px solid #ccc; } }
    </style>
</head>
<body>

<div class="container">
    <div class="header-bar no-print">
        <span>الفصل النشط:</span>
        <select id="classSelector" onchange="switchClass()"></select>
    </div>

    <div id="page-scan" class="page">
        <div class="card" style="text-align: center;">
            <h2 id="scan-title">📸 تسجيل الحضور</h2>
            <div id="reader"></div>
            <div id="scan-status" class="status-msg" style="background: #f0f0f0;">جاهز للمسح...</div>
        </div>
    </div>

    <div id="page-admin" class="page" style="display: none;">
        <div class="card no-print">
            <h3>⚙️ إعدادات الفصل</h3>
            <div style="display: flex; gap: 10px; margin-bottom: 10px;">
                <input type="text" id="new-class-input" placeholder="اسم فصل جديد..." style="flex:1; padding:10px; border-radius:8px; border:1px solid #ccc;">
                <button class="btn btn-accent" style="width:auto; margin:0;" onclick="addNewClass()">إضافة</button>
            </div>
            
            <p><b>اختر أيام الخدمة لهذا الفصل:</b></p>
            <div id="days-config">
                </div>
            <button class="btn btn-danger" onclick="deleteCurrentClass()">🗑️ حذف الفصل الحالي</button>
        </div>

        <div class="card no-print">
            <h3>👥 إضافة أسماء (دفعة واحدة)</h3>
            <textarea id="bulk-names" class="bulk-input" placeholder="ضع الأسماء هنا، كل اسم في سطر..."></textarea>
            <button class="btn btn-green" onclick="processBulkAdd()">إضافة الأسماء 🚀</button>
        </div>

        <div class="card">
            <h3 id="report-title">📅 تقرير الحضور</h3>
            <div class="table-wrapper">
                <table>
                    <thead><tr id="thead-row"></tr></thead>
                    <tbody id="tbody-rows"></tbody>
                </table>
            </div>
            <button class="btn btn-accent no-print" onclick="window.print()">طباعة الكشف 🖨️</button>
        </div>
    </div>
</div>

<nav class="bottom-nav no-print">
    <div class="nav-btn active" id="nav-scan" onclick="showPage('scan')"><span>📸</span><span>الحضور</span></div>
    <div class="nav-btn" id="nav-admin" onclick="showPage('admin')"><span>📊</span><span>الإدارة</span></div>
</nav>

<script>
    const DAYS_AR = ["السبت", "الأحد", "الاثنين", "الثلاثاء", "الأربعاء", "الخميس", "الجمعة"];
    
    let appData = JSON.parse(localStorage.getItem('ASHBAL_PRO_MAX')) || {
        activeClass: "أشبال",
        classes: { "أشبال": { members: [], logs: {}, activeDays: ["السبت", "الاثنين", "الخميس"] } }
    };

    function notify(type) {
        let ctx = new (window.AudioContext || window.webkitAudioContext)();
        let osc = ctx.createOscillator();
        let gain = ctx.createGain();
        osc.connect(gain); gain.connect(ctx.destination);
        if(type === 'success') {
            osc.frequency.value = 1000; gain.gain.value = 0.2;
            osc.start(); osc.stop(ctx.currentTime + 0.2);
            if (navigator.vibrate) navigator.vibrate(300);
        } else {
            osc.frequency.value = 300; gain.gain.value = 0.3;
            osc.start(); osc.stop(ctx.currentTime + 0.4);
        }
    }

    function save() {
        localStorage.setItem('ASHBAL_PRO_MAX', JSON.stringify(appData));
        refreshUI();
    }

    function showPage(p) {
        document.querySelectorAll('.page').forEach(pg => pg.style.display = 'none');
        document.getElementById('page-' + p).style.display = 'block';
        document.querySelectorAll('.nav-btn').forEach(b => b.classList.remove('active'));
        document.getElementById('nav-' + p).classList.add('active');
        if(p === 'admin') refreshUI();
    }

    function addNewClass() {
        let name = document.getElementById('new-class-input').value.trim();
        if(name && !appData.classes[name]) {
            appData.classes[name] = { members: [], logs: {}, activeDays: ["السبت"] };
            appData.activeClass = name;
            document.getElementById('new-class-input').value = '';
            save();
        }
    }

    function deleteCurrentClass() {
        if(Object.keys(appData.classes).length <= 1) return alert("لا يمكن حذف الفصل الوحيد!");
        if(confirm(`حذف فصل "${appData.activeClass}"؟`)) {
            delete appData.classes[appData.activeClass];
            appData.activeClass = Object.keys(appData.classes)[0];
            save();
        }
    }

    function switchClass() {
        appData.activeClass = document.getElementById('classSelector').value;
        save();
    }

    function toggleDay(day) {
        let days = appData.classes[appData.activeClass].activeDays;
        if(days.includes(day)) {
            appData.classes[appData.activeClass].activeDays = days.filter(d => d !== day);
        } else {
            appData.classes[appData.activeClass].activeDays.push(day);
        }
        save();
    }

    function processBulkAdd() {
        let lines = document.getElementById('bulk-names').value.split('\n');
        let current = appData.classes[appData.activeClass];
        lines.forEach(l => {
            let n = l.trim();
            if(n && !current.members.includes(n)) current.members.push(n);
        });
        document.getElementById('bulk-names').value = '';
        save();
    }

    function refreshUI() {
        const selector = document.getElementById('classSelector');
        selector.innerHTML = Object.keys(appData.classes).map(c => `<option value="${c}" ${c === appData.activeClass ? 'selected' : ''}>${c}</option>`).join('');
        
        const current = appData.classes[appData.activeClass];
        document.getElementById('scan-title').innerText = `تسجيل: ${appData.activeClass}`;

        // تحديث مربعات اختيار الأيام
        document.getElementById('days-config').innerHTML = DAYS_AR.map(d => `
            <label class="day-check" style="background:${current.activeDays.includes(d)?'#3498db':'#eee'}; color:${current.activeDays.includes(d)?'white':'black'}">
                <input type="checkbox" ${current.activeDays.includes(d)?'checked':''} onchange="toggleDay('${d}')"> ${d}
            </label>
        `).join('');

        const now = new Date(), m = now.getMonth(), y = now.getFullYear();
        const headRow = document.getElementById('thead-row'), bodyRows = document.getElementById('tbody-rows');
        
        let hHtml = '<th>الاسم</th>', dates = [];
        let totalInMonth = new Date(y, m + 1, 0).getDate();

        for(let i=1; i<=totalInMonth; i++) {
            let dObj = new Date(y, m, i);
            let dayName = dObj.toLocaleDateString('ar-EG', {weekday:'long'});
            if(current.activeDays.includes(dayName)) {
                hHtml += `<th>${i}<br><small>${dayName.substring(0,2)}</small></th>`;
                dates.push(i);
            }
        }
        hHtml += '<th class="calc-col">العدد</th><th class="calc-col">%</th>';
        headRow.innerHTML = hHtml;

        bodyRows.innerHTML = current.members.sort().map(name => {
            let row = `<td><b>${name}</b></td>`, count = 0;
            dates.forEach(day => {
                let key = `${y}-${m+1}-${day}`;
                let ok = current.logs[name] && current.logs[name].includes(key);
                if(ok) count++;
                row += `<td class="${ok ? 'is-present' : ''}">${ok ? '✔' : '-'}</td>`;
            });
            let percent = dates.length > 0 ? ((count / dates.length) * 100).toFixed(0) : 0;
            row += `<td class="calc-col">${count}</td><td class="calc-col">${percent}%</td>`;
            return `<tr>${row}</tr>`;
        }).join('');
    }

    const scanner = new Html5QrcodeScanner("reader", { fps: 20, qrbox: 250 });
    scanner.render(text => {
        let name = text.trim(), now = new Date();
        let dateKey = `${now.getFullYear()}-${now.getMonth()+1}-${now.getDate()}`;
        let current = appData.classes[appData.activeClass];
        let status = document.getElementById('scan-status');

        if(current.members.includes(name)) {
            if(!current.logs[name]) current.logs[name] = [];
            if(!current.logs[name].includes(dateKey)) {
                current.logs[name].push(dateKey);
                notify('success'); save();
                status.innerText = `✅ تم: ${name}`; status.style.background = "#d4edda";
            } else {
                notify('fail');
                status.innerText = `⚠️ مسجل مسبقاً: ${name}`; status.style.background = "#fff3cd";
            }
        } else {
            status.innerText = "❌ الاسم غير موجود!"; status.style.background = "#f8d7da";
        }
    });

    refreshUI();
</script>
</body>
</html>
