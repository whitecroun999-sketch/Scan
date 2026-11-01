```html

<!DOCTYPE html>

<html lang="ru">

<head>

  <meta charset="UTF-8">

  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">

  <title>Сканер Оборудования</title>

  <script src="https://unpkg.com/html5-qrcode@2.3.8/html5-qrcode.min.js"></script>

  <style>

    :root {

      --bg-color: #f8fafc;

      --card-bg: #ffffff;

      --text-main: #0f172a;

      --text-sub: #64748b;

      --primary: #2563eb;

      --primary-hover: #1d4ed8;

      --border: #e2e8f0;

      --success: #16a34a;

    }

    * { box-sizing: border-box; margin: 0; padding: 0; }

    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background: var(--bg-color); color: var(--text-main); padding: 16px; }

    

    .container { max-width: 480px; margin: 0 auto; }

    .card { background: var(--card-bg); border-radius: 12px; padding: 16px; border: 1px solid var(--border); margin-bottom: 16px; }

    

    h1 { font-size: 18px; text-align: center; margin-bottom: 16px; color: var(--text-main); }

    

    #reader { width: 100%; border-radius: 8px; overflow: hidden; border: none !important; }

    #reader video { object-fit: cover; border-radius: 8px; }

    .form-group { margin-bottom: 12px; }

    label { font-size: 12px; color: var(--text-sub); display: block; margin-bottom: 4px; font-weight: 600; }

    input, select { width: 100%; padding: 10px 12px; border: 1px solid var(--border); border-radius: 6px; font-size: 14px; background: #fff; }

    

    button { width: 100%; background: var(--primary); color: white; border: none; padding: 12px; border-radius: 6px; font-size: 14px; font-weight: 600; cursor: pointer; transition: background 0.2s; }

    button:active { background: var(--primary-hover); }

    button:disabled { opacity: 0.6; cursor: not-allowed; }

    .status-badge { display: inline-block; padding: 4px 8px; border-radius: 4px; font-size: 12px; font-weight: 600; background: #f1f5f9; color: var(--text-sub); }

    .hidden { display: none; }

    #toast { margin-top: 12px; padding: 10px; border-radius: 6px; font-size: 13px; text-align: center; }

    .toast-success { background: #dcfce7; color: #166534; }

    .toast-error { background: #fee2e2; color: #991b1b; }

  </style>

</head>

<body>

<div class="container">

  <h1>Учет и обслуживание</h1>

  <div class="card">

    <div id="reader"></div>

  </div>

  <div id="result-card" class="card hidden">

    <div class="form-group">

      <label>Серийный номер</label>

      <input type="text" id="res-serial" readonly>

    </div>

    

    <div class="form-group">

      <label>Наименование</label>

      <input type="text" id="res-name" readonly>

    </div>

    <div class="form-group">

      <label>Последняя чистка</label>

      <input type="text" id="res-last-cleaned" readonly>

    </div>

    <div class="form-group">

      <label for="res-status">Статус оборудования</label>

      <select id="res-status">

        <option value="В работе">В работе</option>

        <option value="Обслужено">Обслужено</option>

        <option value="Требует ремонта">Требует ремонта</option>

        <option value="Списано">Списано</option>

      </select>

    </div>

    <button id="btn-save" onclick="updateData()">Сохранить изменения</button>

  </div>

  <div id="toast" class="hidden"></div>

</div>

<script>

  // Укажите URL развернутого Web App Google Apps Script

  const API_URL = '<span type="placeholder" placeholder-type="file"></span>'; 

  let html5QrcodeScanner = null;

  function showToast(message, isSuccess = true) {

    const toast = document.getElementById('toast');

    toast.innerText = message;

    toast.className = isSuccess ? 'toast-success' : 'toast-error';

    toast.classList.remove('hidden');

    setTimeout(() => toast.classList.add('hidden'), 4000);

  }

  function onScanSuccess(decodedText) {

    if (html5QrcodeScanner) {

      html5QrcodeScanner.pause(true);

    }

    fetchEquipmentData(decodedText);

  }

  async function fetchEquipmentData(serial) {

    showToast("Загрузка данных...", true);

    try {

      const response = await fetch(`${API_URL}?serial=${encodeURIComponent(serial)}`);

      const result = await response.json();

      if (result.success) {

        document.getElementById('res-serial').value = result.data.serialNumber;

        document.getElementById('res-name').value = result.data.name;

        document.getElementById('res-last-cleaned').value = result.data.lastCleaned || 'Нет данных';

        document.getElementById('res-status').value = result.data.status || 'В работе';

        

        document.getElementById('result-card').classList.remove('hidden');

        showToast("Данные найдены", true);

      } else {

        showToast(result.message || "Запись не найдена", false);

        resumeScanner();

      }

    } catch (err) {

      showToast("Ошибка сети при получении данных", false);

      resumeScanner();

    }

  }

  async function updateData() {

    const btn = document.getElementById('btn-save');

    const serial = document.getElementById('res-serial').value;

    const status = document.getElementById('res-status').value;

    btn.disabled = true;

    showToast("Сохранение...", true);

    try {

      const response = await fetch(API_URL, {

        method: 'POST',

        headers: { 'Content-Type': 'text/plain;charset=utf-8' },

        body: JSON.stringify({ serialNumber: serial, status: status })

      });

      const result = await response.json();

      if (result.success) {

        showToast("Обновлено успешно!", true);

        document.getElementById('res-last-cleaned').value = result.updatedAt;

        setTimeout(() => {

          document.getElementById('result-card').classList.add('hidden');

          resumeScanner();

        }, 1500);

      } else {

        showToast(result.message || "Ошибка обновления", false);

      }

    } catch (err) {

      showToast("Ошибка сети при сохранении", false);

    } finally {

      btn.disabled = false;

    }

  }

  function resumeScanner() {

    if (html5QrcodeScanner) {

      html5QrcodeScanner.resume();

    }

  }

  function initScanner() {

    html5QrcodeScanner = new Html5QrcodeScanner(

      "reader", 

      { 

        fps: 10, 

        qrbox: { width: 250, height: 150 },

        facingMode: "environment"

      },

      /* verbose= */ false

    );

    html5QrcodeScanner.render(onScanSuccess);

  }

  document.addEventListener("DOMContentLoaded", initScanner);

</script>

</body>

</html>
