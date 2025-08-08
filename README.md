<!DOCTYPE html>
<html lang="ko">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>요요 연습 타이머 & 기록</title>
<style>
  body {
    font-family: Arial, sans-serif;
    max-width: 600px;
    margin: 20px auto;
    padding: 10px;
  }
  h1, h2 {
    text-align: center;
  }
  #timer {
    font-size: 3em;
    text-align: center;
    margin: 10px 0;
  }
  button {
    margin: 5px;
    padding: 10px 20px;
    font-size: 1em;
  }
  input[type="number"], input[type="text"], select {
    padding: 5px;
    font-size: 1em;
    margin-left: 10px;
  }
  label {
    font-weight: bold;
  }
  #records {
    margin-top: 20px;
  }
  #records ul {
    list-style-type: none;
    padding: 0;
  }
  #records li {
    margin-bottom: 6px;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  #records button.deleteBtn {
    margin-left: 10px;
    padding: 3px 7px;
    font-size: 0.8em;
    color: white;
    background-color: #d33;
    border: none;
    border-radius: 3px;
    cursor: pointer;
  }
  #totalTime {
    font-weight: bold;
    margin-top: 10px;
    text-align: center;
  }
  #chartContainer {
    position: relative;
    width: 100%;
    max-width: 600px;
    height: 100px;  /* 작게 조절 */
    margin-top: 30px;
  }
  canvas {
    width: 100% !important;
    height: 100px !important;  /* 차트 높이 고정 */
  }
  #memo {
    width: 100%;
    margin-top: 5px;
    font-size: 1em;
  }
  #goalInput {
    width: 60px;
  }
  #langSelect {
    margin-left: 10px;
  }
  #exportBtn {
    float: right;
    margin-top: 10px;
    padding: 5px 10px;
  }
</style>
</head>
<body>

<h1 id="title">요요 연습 타이머 & 기록</h1>

<div id="timer">00:00.00</div>

<div style="text-align:center;">
  <button id="startStopBtn">시작</button>
  <button id="resetBtn">리셋</button>
  <button id="saveBtn" disabled>기록 저장</button>
</div>

<div style="margin-top:15px; text-align:center;">
  <label for="goalInput" id="goalLabel">오늘 목표 시간(분):</label>
  <input type="number" id="goalInput" min="1" max="600" value="30" />
</div>

<div>
  <label for="memo" id="memoLabel">메모:</label><br />
  <input type="text" id="memo" placeholder="연습 메모를 입력하세요" />
</div>

<div id="records">
  <h2 id="recordsTitle">오늘 연습 기록</h2>
  <ul id="recordList">
    <li>기록이 없습니다.</li>
  </ul>
  <div id="totalTime">총 연습 시간: 00:00.00</div>
  <button id="deleteAllBtn">전체 삭제</button>
  <button id="exportBtn">CSV 내보내기</button>
</div>

<div id="chartContainer">
  <canvas id="chart" height="100"></canvas>
</div>

<div style="margin-top: 20px; text-align:center;">
  <label for="langSelect">언어:</label>
  <select id="langSelect">
    <option value="ko" selected>한국어</option>
    <option value="en">English</option>
  </select>
</div>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
(() => {
  const texts = {
    ko: {
      title: '요요 연습 타이머 & 기록',
      start: '시작',
      stop: '멈춤',
      reset: '리셋',
      save: '기록 저장',
      delete: '삭제',
      deleteAllConfirm: '오늘 기록을 모두 삭제하시겠습니까?',
      noRecords: '기록이 없습니다.',
      totalTime: '총 연습 시간: ',
      goalLabel: '오늘 목표 시간(분):',
      memoLabel: '메모:',
      recordsTitle: '오늘 연습 기록',
      export: 'CSV 내보내기',
      alertGoalReached: '목표 시간을 달성했습니다! 수고했어요!'
    },
    en: {
      title: 'Yo-Yo Practice Timer & Records',
      start: 'Start',
      stop: 'Stop',
      reset: 'Reset',
      save: 'Save Record',
      delete: 'Delete',
      deleteAllConfirm: 'Delete all records for today?',
      noRecords: 'No records.',
      totalTime: 'Total Practice Time: ',
      goalLabel: 'Today\'s Goal (min):',
      memoLabel: 'Memo:',
      recordsTitle: 'Today\'s Practice Records',
      export: 'Export CSV',
      alertGoalReached: 'Goal time reached! Good job!'
    }
  };

  let currentLang = 'ko';

  const timerDisplay = document.getElementById('timer');
  const startStopBtn = document.getElementById('startStopBtn');
  const resetBtn = document.getElementById('resetBtn');
  const saveBtn = document.getElementById('saveBtn');
  const recordList = document.getElementById('recordList');
  const totalTimeDisplay = document.getElementById('totalTime');
  const deleteAllBtn = document.getElementById('deleteAllBtn');
  const goalInput = document.getElementById('goalInput');
  const memoInput = document.getElementById('memo');
  const titleEl = document.getElementById('title');
  const goalLabel = document.getElementById('goalLabel');
  const memoLabel = document.getElementById('memoLabel');
  const recordsTitle = document.getElementById('recordsTitle');
  const exportBtn = document.getElementById('exportBtn');
  const langSelect = document.getElementById('langSelect');

  let startTime = 0;
  let elapsedTime = 0;
  let timerInterval = null;
  let running = false;
  let goalReached = false;

  // 시간 문자열 변환 mm:ss.SS
  function timeToString(timeMs) {
    const ms = Math.floor((timeMs % 1000) / 10);
    const sec = Math.floor((timeMs / 1000) % 60);
    const min = Math.floor(timeMs / (1000 * 60));
    return (
      (min < 10 ? '0' + min : min) + ':' +
      (sec < 10 ? '0' + sec : sec) + '.' +
      (ms < 10 ? '0' + ms : ms)
    );
  }

  // 오늘 날짜 문자열 yyyy-mm-dd
  function getTodayDateStr() {
    const d = new Date();
    return d.toISOString().slice(0, 10);
  }

  // 로컬스토리지에서 기록 불러오기
  function getRecords() {
    const str = localStorage.getItem('yoyoRecords');
    return str ? JSON.parse(str) : [];
  }

  // 기록 저장
  function saveRecords(records) {
    localStorage.setItem('yoyoRecords', JSON.stringify(records));
  }

  // 타이머 시작
  function startTimer() {
    startTime = Date.now() - elapsedTime;
    timerInterval = setInterval(() => {
      elapsedTime = Date.now() - startTime;
      timerDisplay.textContent = timeToString(elapsedTime);
      checkGoal();
    }, 10);
    running = true;
    startStopBtn.textContent = texts[currentLang].stop;
    saveBtn.disabled = true;
    goalReached = false;
  }

  // 타이머 멈춤
  function stopTimer() {
    clearInterval(timerInterval);
    running = false;
    startStopBtn.textContent = texts[currentLang].start;
    saveBtn.disabled = elapsedTime === 0;
  }

  // 타이머 리셋
  function resetTimer() {
    clearInterval(timerInterval);
    elapsedTime = 0;
    timerDisplay.textContent = '00:00.00';
    running = false;
    startStopBtn.textContent = texts[currentLang].start;
    saveBtn.disabled = true;
    goalReached = false;
  }

  // 기록 저장 (현재 시간 + 메모)
  function saveRecord(timeMs, memo) {
    if (timeMs === 0) return;
    const today = getTodayDateStr();
    const records = getRecords();
    records.push({ time: timeMs, date: today, memo: memo || '' });
    saveRecords(records);
    renderRecords();
  }

  // 기록 삭제 (인덱스)
  function deleteRecord(index) {
    const today = getTodayDateStr();
    let records = getRecords();
    let todayRecords = records.filter(r => r.date === today);
    todayRecords.splice(index, 1);
    const otherRecords = records.filter(r => r.date !== today);
    const newRecords = otherRecords.concat(todayRecords);
    saveRecords(newRecords);
    renderRecords();
  }

  // 오늘 기록 전체 삭제
  function deleteAllRecords() {
    if (!confirm(texts[currentLang].deleteAllConfirm)) return;
    let records = getRecords();
    const today = getTodayDateStr();
    records = records.filter(r => r.date !== today);
    saveRecords(records);
    renderRecords();
  }

  // 기록 렌더링
  function renderRecords() {
    const today = getTodayDateStr();
    const records = getRecords();
    const todayRecords = records.filter(r => r.date === today);

    recordList.innerHTML = '';

    if (todayRecords.length === 0) {
      recordList.innerHTML = `<li>${texts[currentLang].noRecords}</li>`;
      totalTimeDisplay.textContent = texts[currentLang].totalTime + '00:00.00';
      updateChart([]);
      return;
    }

    todayRecords.forEach((record, idx) => {
      const li = document.createElement('li');
      const memoText = record.memo ? ` - ${record.memo}` : '';
      li.textContent = `${idx + 1}. ${timeToString(record.time)}${memoText}`;

      const delBtn = document.createElement('button');
      delBtn.textContent = texts[currentLang].delete;
      delBtn.className = 'deleteBtn';
      delBtn.onclick = () => deleteRecord(idx);

      li.appendChild(delBtn);
      recordList.appendChild(li);
    });

    const totalMs = todayRecords.reduce((sum, r) => sum + r.time, 0);
    totalTimeDisplay.textContent = texts[currentLang].totalTime + timeToString(totalMs);
    updateChart(todayRecords);
  }

  // 목표 시간 도달 알림 체크
  function checkGoal() {
    const goalMin = parseInt(goalInput.value);
    if (isNaN(goalMin) || goalMin < 1) {
      return;
    }
    const goalMs = goalMin * 60 * 1000;
    const todayRecords = getRecords().filter(r => r.date === getTodayDateStr());
    const totalTime = todayRecords.reduce((sum, r) => sum + r.time, 0) + elapsedTime;

    if (!goalReached && totalTime >= goalMs) {
      goalReached = true;
      alert(texts[currentLang].alertGoalReached);
    }
  }

  // CSV 내보내기
  function exportCSV() {
    const records = getRecords();
    if(records.length === 0){
      alert(texts[currentLang].noRecords);
      return;
    }
    let csv = '날짜,시간(ms),시간(분:초.밀리초),메모\n';
    records.forEach(r => {
      const line = `"${r.date}",${r.time},"${timeToString(r.time)}","${r.memo.replace(/"/g, '""')}"`;
      csv += line + '\n';
    });
    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);

    const a = document.createElement('a');
    a.href = url;
    a.download = 'yoyo_practice_records.csv';
    a.click();
    URL.revokeObjectURL(url);
  }

  // 차트 그리기
  const ctx = document.getElementById('chart').getContext('2d');
  let chartInstance = null;
  function updateChart(todayRecords) {
    const labels = todayRecords.map((_, i) => (i+1).toString());
    const data = todayRecords.map(r => r.time / 1000);

    if(chartInstance) {
      chartInstance.data.labels = labels;
      chartInstance.data.datasets[0].data = data;
      chartInstance.update();
    } else {
      chartInstance = new Chart(ctx, {
        type: 'bar',
        data: {
          labels: labels,
          datasets: [{
            label: currentLang === 'ko' ? '세션별 연습 시간(초)' : 'Practice Time per Session (sec)',
            data: data,
            backgroundColor: 'rgba(25, 118, 210, 0.7)',
            borderColor: 'rgba(25, 118, 210, 1)',
            borderWidth: 1
          }]
        },
        options: {
          scales: {
            y: {
              beginAtZero: true,
              title: {
                display: true,
                text: currentLang === 'ko' ? '시간(초)' : 'Time (sec)'
              }
            },
            x: {
              title: {
                display: true,
                text: currentLang === 'ko' ? '세션 번호' : 'Session Number'
              }
            }
          },
          plugins: {
            legend: {
              display: true
            },
            tooltip: {
              enabled: true
            }
          },
          responsive: true,
          maintainAspectRatio: false
        }
      });
    }
  }

  // 이벤트 핸들러 연결
  startStopBtn.addEventListener('click', () => {
    if (!running) {
      startTimer();
    } else {
      stopTimer();
    }
  });

  resetBtn.addEventListener('click', () => {
    resetTimer();
  });

  saveBtn.addEventListener('click', () => {
    saveRecord(elapsedTime, memoInput.value.trim());
    resetTimer();
    memoInput.value = '';
  });

  deleteAllBtn.addEventListener('click', deleteAllRecords);

  exportBtn.addEventListener('click', exportCSV);

  langSelect.addEventListener('change', () => {
    currentLang = langSelect.value;
    updateLanguage();
  });

  function updateLanguage() {
    titleEl.textContent = texts[currentLang].title;
    startStopBtn.textContent = running ? texts[currentLang].stop : texts[currentLang].start;
    resetBtn.textContent = texts[currentLang].reset;
    saveBtn.textContent = texts[currentLang].save;
    deleteAllBtn.textContent = currentLang === 'ko' ? '전체 삭제' : 'Delete All';
    goalLabel.textContent = texts[currentLang].goalLabel;
    memoLabel.textContent = texts[currentLang].memoLabel;
    recordsTitle.textContent = texts[currentLang].recordsTitle;
    exportBtn.textContent = texts[currentLang].export;
    renderRecords();
    updateChart(getRecords().filter(r => r.date === getTodayDateStr()));
  }

  // 초기화
  renderRecords();
  updateLanguage();

})();
</script>

</body>
</html>
