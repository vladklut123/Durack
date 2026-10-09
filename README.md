# Durack
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<meta name="theme-color" content="#111">
<title>Трекер Дурака</title>
<style>
  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  body {
    margin: 0; padding: 10px;
    background: #111; color: #eee;
    font: 14px/1.3 -apple-system, system-ui, sans-serif;
    user-select: none;
  }
  h1 { font-size: 16px; margin: 0 0 8px; display: flex; justify-content: space-between; align-items: center; }
  .reset { font-size: 13px; color: #f66; cursor: pointer; padding: 4px 8px; }
  .hint { font-size: 11px; color: #888; margin-bottom: 8px; }

  .suit-row {
    display: grid;
    grid-template-columns: repeat(9, 1fr);
    gap: 4 sympx;
    margin-bottom: 4px;
 : ' }
  .card {
    aspect-ratio: 2 / 3;
    border-radius: 5px;
    background: #333;
    border: 1px solid #444;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: bold;
    font-size: 15px;
    cursor: pointer;
    transition: transform 0.08s;
    position: relative;
  }
  .card:active { transform: scale(0.92); }
  .card.red { color: #ff6b6b; }
  .card.black { color: #fff; }
  .card.played {
    background: #1a1a1a;
    color: #555 !important;
    text-decoration: line-through;
    border-color: #222;
  }
  .card.mine {
    background: #2d6a2d;
    color: #fff !important;
    border-color: #4a4;
  }

  .summary {
    margin-top: 12px;
    padding-top: 10px;
    border-top: 1px solid #333;
    font-size: 14px;
  }
  .stat { margin-bottom: 6px; }
  .stat b { color: #6cf; }
  .stat .p { color: #888; }
  .stat .m { color: #6f6; }

  .remaining-title { font-weight: bold; margin: 10px 0 6px; }
  .remaining-line { margin: 3px 0; font-size: 15px; letter-spacing: 1px; }
  .remaining-line .suit { margin-right: 6px; font-size: 18px; }
  .remaining-line .ranks { color: #ccc; }

  .empty { color: #666; font-style: italic; }

  .toast {
    position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%);
    background: #333; color: #fff; padding: 8px 16px; border-radius: 20px;
    font-size: 13px; opacity: 0; transition: opacity 0.2s; pointer-events: none;
    z-index: 100;
  }
  .toast.show { opacity: 1; }

  /* Более компактный вид на очень узких экранах */
  @media (max-width: 360px) {
    .card { font-size: 13px; }
    .suit-row { gap: 3px; }
  }
</style>
</head>
<body>

<h1>
  🃏 Трекер Дурака
  <span class="reset" id="reset">⟲ Сброс</span>
</h1>
<div class="hint">Тап — в отбой · Долгий тап — у меня</div>

<div id="grid"></div>

<div class="summary">
  <div class="stat" id="stats"></div>
  <div class="remaining-title">Осталось в игре:</div>
  <div id="remaining"></div>
</div>

<div class="toast" id="toast"></div>

<script>
(function() {
  'use strict';

  const SUITS = [
    {♠', color: 'black' },
    { sym: '♥', color: 'red' },
    { sym: '♦', color: 'red' },
    { sym: '♣', color: 'black' }
  ];
  const RANKS = ['6','7','8','9','10','J','Q','K','A'];

  const STORAGE_KEY = 'durak_tracker_v1';
  const cardState = {};

  SUITS.forEach(s => RANKS.forEach(r => cardState[r + s.sym] = 'unknown'));

  // Загрузка
  try {
    const saved = JSON.parse(localStorage.getItem(STORAGE_KEY) || '{}');
    for (const k in saved) if (cardState.hasOwnProperty(k)) cardState[k] = saved[k];
  } catch(e) {}

  function save() {
    try { localStorage.setItem(STORAGE_KEY, JSON.stringify(cardState)); } catch(e) {}
  }

  // Тост-уведомление
  let toastTimer;
  function toast(msg) {
    const el = document.getElementById('toast');
    el.textContent = msg;
    el.classList.add('show');
    clearTimeout(toastTimer);
    toastTimer = setTimeout(() => el.classList.remove('show'), 1200);
  }

  // Обработка тапа и долгого тапа
  let pressTimer = null;
  let longPressed = false;
  const LONG_PRESS_MS = 350;

  function handlePressStart(e, key) {
    longPressed = false;
    pressTimer = setTimeout(() => {
      longPressed = true;
      cycleMine(key);
    }, LONG_PRESS_MS);
  }

  function handlePressEnd(e, key) {
    clearTimeout(pressTimer);
    if (longPressed) return;
    // Это был обычный тап
    cyclePlayed(key);
  }

  function cyclePlayed(key) {
    if (cardState[key] === 'played') {
      cardState[key] = 'unknown';
    } else if (cardState[key] === 'mine') {
      // не трогаем
      return;
    } else {
      cardState[key] = 'played';
    }
    save();
    render();
  }

  function cycleMine(key) {
    if (cardState[key] === 'mine') {
      cardState[key] = 'unknown';
    } else if (cardState[key] === 'played') {
      return;
    } else {
      cardState[key] = 'mine';
    }
    save();
    render();
    if (navigator.vibrate) navigator.vibrate(20);
    toast(cardState[key] === 'mine' ? `${key} — у тебя` : `${key} — снято`);
  }

  function render() {
    // Сетка
    let gridHtml = '';
    SUITS.forEach(s => {
      gridHtml += '<div class="suit-row">';
      RANKS.forEach(r => {
        const key = r + s.sym;
        const st = cardState[key];
        const cls = [
          'card',
          s.color === 'red' ? 'red' : 'black',
          st === 'played' ? 'played' : '',
          st === 'mine' ? 'mine' : ''
        ].filter(Boolean).join(' ');
        gridHtml += `<div class="${cls}" data-card="${key}">${r}</div>`;
      });
      gridHtml += '</div>';
    });
    document.getElementById('grid').innerHTML = gridHtml;

    // Статистика
    let unknown = 0, played = 0, mine = 0;
    for (const k in cardState) {
      if (cardState[k] === 'unknown') unknown++;
      else if (cardState[k] === 'played') played++;
      else mine++;
    }
    document.getElementById('stats').innerHTML =
      `<b>В игре:</b> ${unknown} &nbsp;·&nbsp; <span class="p">Отбой: ${played}</span> &nbsp;·&nbsp; <span class="m">У меня: ${mine}</span>`;

    // Оставшиеся
    const remaining = {};
    SUITS.forEach(s => remaining[s.sym] = []);
    for (const k in cardState) {
      if (cardState[k] !== 'unknown') continue;
      const suit = k.slice(-1);
      const rank = k.slice(0, -1);
      if (remaining[suit]) remaining[suit].push(rank);
    }

    let remHtml = '';
    let anyLeft = false;
    SUITS.forEach(s => {
      const ranks = remaining[s.sym];
      if (!ranks.length) return;
      anyLeft = true;
      const color = s.color === 'red' ? '#ff6b6b' : '#fff';
      remHtml += `<div class="remaining-line">
        <span class="suit" style="color:${color}">${s.sym}</span>
        <span class="ranks">${ranks.join(' ')}</span>
      </div>`;
    });
    if (!anyLeft) remHtml = '<div class="empty">— пусто —</div>';
    document.getElementById('remaining').innerHTML = remHtml;
  }

  // Делегирование событий
  document.getElementById('grid').addEventListener('touchstart', (e) => {
    const el = e.target.closest('[data-card]');
    if (!el) return;
    handlePressStart(e, el.dataset.card);
  }, { passive: true });

  document.getElementById('grid').addEventListener('touchend', (e) => {
    const el = e.target.closest('[data-card]');
    if (!el) return;
    e.preventDefault();
    handlePressEnd(e, el.dataset.card);
  });

  document.getElementById('grid').addEventListener('touchcancel', () => {
    clearTimeout(pressTimer);
  });

  // Fallback для мыши (десктоп)
  document.getElementById('grid').addEventListener('mousedown', (e) => {
    const el = e.target.closest('[data-card]');
    if (!el) return;
    handlePressStart(e, el.dataset.card);
  });
  document.getElementById('grid').addEventListener('mouseup', (e) => {
    const el = e.target.closest('[data-card]');
    if (!el) return;
    handlePressEnd(e, el.dataset.card);
  });

  document.getElementById('reset').addEventListener('click', () => {
    if (!confirm('Сбросить все отметки?')) return;
    for (const k in cardState) cardState[k] = 'unknown';
    save();
    render();
    toast('Сброшено');
  });

  render();
})();
</script>
</body>
</html>