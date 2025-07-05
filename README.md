# Brain_coin_bot
brain-coin/
├── public/
│   ├── index.html       # Main page
│   ├── upgrades.html    # Cards & tasks page
│   └── styles.css       # Common styles
├── server.js            # Backend server
├── package.json
└── database.db          # SQLite database
const express = require('express');
const crypto = require('crypto');
const sqlite3 = require('sqlite3').verbose();
const app = express();
const PORT = 3000;

// Initialize database
const db = new sqlite3.Database('./database.db');

db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    coins INTEGER DEFAULT 0
  )`);
  
  db.run(`CREATE TABLE IF NOT EXISTS cards (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    level INTEGER DEFAULT 1,
    profit_rate REAL,
    upgrade_cost INTEGER,
    user_id TEXT
  )`);
  
  db.run(`CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    type TEXT CHECK(type IN ('telegram', 'youtube', 'tap')),
    completed BOOLEAN DEFAULT 0,
    user_id TEXT
  )`);
  
  db.run(`CREATE TABLE IF NOT EXISTS settings (
    youtube_url TEXT DEFAULT 'https://youtube.com/watch?v=dQw4w9WgXcQ'
  )`);
});

// Middleware
app.use(express.json());
app.use(express.static('public'));

// Telegram authentication middleware
const validateInitData = (req, res, next) => {
  const initData = req.body.initData;
  const botToken = process.env.BOT_TOKEN;
  
  const params = new URLSearchParams(initData);
  const hash = params.get('hash');
  const dataCheckString = [...params.entries()]
    .filter(([key]) => key !== 'hash')
    .sort((a, b) => a[0].localeCompare(b[0]))
    .map(([key, value]) => `${key}=${value}`)
    .join('\n');
  
  const secret = crypto.createHmac('sha256', 'WebAppData').update(botToken).digest();
  const computedHash = crypto.createHmac('sha256', secret)
    .update(dataCheckString)
    .digest('hex');
  
  if (computedHash === hash) {
    const user = JSON.parse(params.get('user'));
    req.user = user;
    next();
  } else {
    res.status(401).send('Unauthorized');
  }
};

// API Endpoints
app.post('/collect-coins', validateInitData, (req, res) => {
  const userId = req.user.id;
  const profit = calculateTotalProfit(userId);
  
  db.run(`UPDATE users SET coins = coins + ? WHERE id = ?`, [profit, userId], () => {
    res.json({ coins: getCoins(userId) });
  });
});

app.post('/upgrade-card', validateInitData, (req, res) => {
  const { cardId } = req.body;
  const userId = req.user.id;
  
  db.get(`SELECT * FROM cards WHERE id = ? AND user_id = ?`, [cardId, userId], (err, card) => {
    if (card && getCoins(userId) >= card.upgrade_cost) {
      db.run(`UPDATE users SET coins = coins - ? WHERE id = ?`, [card.upgrade_cost, userId]);
      db.run(`UPDATE cards SET 
        level = level + 1,
        profit_rate = profit_rate * 1.5,
        upgrade_cost = upgrade_cost * 2 
        WHERE id = ?`, [cardId]);
      
      res.json({ success: true });
    } else {
      res.status(400).json({ error: 'Not enough coins' });
    }
  });
});

// Add more endpoints for tasks, settings, etc.

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));

// Helper functions
function calculateTotalProfit(userId) {
  // Calculate sum of all card profits
  return 10; // Simplified for example
}

function getCoins(userId) {
  // Get user's coin balance
  return 100; // Simplified for example
}
<!DOCTYPE html>
<html>
<head>
  <title>Brain Coin</title>
  <link rel="stylesheet" href="styles.css">
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
</head>
<body>
  <div class="container">
    <header>
      <h1>Brain Coin</h1>
      <div id="coin-counter">0</div>
    </header>
    
    <main>
      <button id="collect-btn">Collect Coins</button>
      
      <div class="tasks">
        <h2>Tasks</h2>
        <ul id="task-list">
          <li>Tap 100 times <progress value="0" max="100"></progress></li>
          <li>Invite 3 friends <progress value="0" max="3"></progress></li>
        </ul>
      </div>
      
      <button onclick="location.href='upgrades.html'">Upgrades →</button>
    </main>
  </div>

  <script>
    const tg = window.Telegram.WebApp;
    tg.expand();
    
    let coins = 0;
    
    document.getElementById('collect-btn').addEventListener('click', () => {
      fetch('/collect-coins', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ initData: tg.initData })
      })
      .then(response => response.json())
      .then(data => {
        coins = data.coins;
        document.getElementById('coin-counter').textContent = coins;
      });
    });
  </script>
</body>
</html>
<!DOCTYPE html>
<html>
<head>
  <title>Brain Upgrades</title>
  <link rel="stylesheet" href="styles.css">
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
</head>
<body>
  <div class="container">
    <header>
      <h1>Brain Upgrades</h1>
      <div id="coin-counter">0</div>
    </header>
    
    <div class="tabs">
      <button class="active" data-tab="cards">Profit Cards</button>
      <button data-tab="tasks">Tasks</button>
      <button data-tab="admin">Admin</button>
    </div>
    
    <div class="tab-content active" id="cards">
      <div class="card">
        <h3>Memory Boost</h3>
        <p>Level: <span class="level">1</span></p>
        <p>Profit: +<span class="profit">5</span>/tap</p>
        <button class="upgrade-btn" data-id="1">Upgrade (50)</button>
      </div>
      <!-- More cards -->
    </div>
    
    <div class="tab-content" id="tasks">
      <div class="task">
        <h3>Join Telegram Channel</h3>
        <button id="join-telegram">Join</button>
        <span class="status"></span>
      </div>
      
      <div class="task">
        <h3>Watch YouTube Video</h3>
        <a id="youtube-link" target="_blank">Watch Now</a>
        <button id="verify-watch">Verify</button>
        <span class="status"></span>
      </div>
    </div>
    
    <div class="tab-content" id="admin">
      <h3>Admin Panel</h3>
      <div class="form-group">
        <label>YouTube URL:</label>
        <input type="url" id="youtube-url">
        <button id="save-url">Save</button>
      </div>
      
      <div class="form-group">
        <button id="add-card">+ Add New Card</button>
      </div>
    </div>
  </div>

  <script>
    // Tab switching logic
    document.querySelectorAll('.tabs button').forEach(btn => {
      btn.addEventListener('click', () => {
        document.querySelectorAll('.tab-content').forEach(tab => {
          tab.classList.remove('active');
        });
        document.querySelectorAll('.tabs button').forEach(b => {
          b.classList.remove('active');
        });
        btn.classList.add('active');
        document.getElementById(btn.dataset.tab).classList.add('active');
      });
    });

    // YouTube integration
    document.getElementById('verify-watch').addEventListener('click', () => {
      // Verify video watch via backend API
    });

    // Admin functionality
    document.getElementById('save-url').addEventListener('click', () => {
      const newUrl = document.getElementById('youtube-url').value;
      // Save via admin API endpoint
    });
  </script>
</body>
</html>
:root {
  --primary: #6a11cb;
  --secondary: #2575fc;
  --accent: #ff7e5f;
}

body {
  font-family: 'Roboto', sans-serif;
  background: linear-gradient(135deg, #1a1a2e, #16213e);
  color: white;
  margin: 0;
  padding: 20px;
  min-height: 100vh;
}

.container {
  max-width: 500px;
  margin: 0 auto;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

#coin-counter {
  background: gold;
  color: black;
  border-radius: 20px;
  padding: 5px 15px;
  font-weight: bold;
  box-shadow: 0 3px 6px rgba(0,0,0,0.16);
}

button {
  background: linear-gradient(to right, var(--primary), var(--secondary));
  border: none;
  color: white;
  padding: 12px 20px;
  border-radius: 30px;
  font-size: 16px;
  cursor: pointer;
  width: 100%;
  margin: 10px 0;
  box-shadow: 0 4px 8px rgba(0,0,0,0.2);
  transition: all 0.3s;
}

button:active {
  transform: scale(0.98);
}

.tabs {
  display: flex;
  margin-bottom: 20px;
}

.tabs button {
  flex: 1;
  margin: 0 5px;
  padding: 8px;
  font-size: 14px;
}

.tab-content {
  display: none;
}

.tab-content.active {
  display: block;
}

.card, .task {
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  border-radius: 15px;
  padding: 15px;
  margin-bottom: 15px;
  border: 1px solid rgba(255,255,255,0.2);
}

.task .status::before {
  content: "❌";
  margin-left: 10px;
}

.task.completed .status::before {
  content: "✅";
}

.form-group {
  margin: 15px 0;
}

input {
  width: 100%;
  padding: 12px;
  border-radius: 10px;
  border: none;
  margin: 10px 0;
  background: rgba(255,255,255,0.15);
  color: white;
}
// Calculate profit based on cards
function calculateTotalProfit(userId) {
  return new Promise((resolve) => {
    db.all(`SELECT SUM(profit_rate * level) AS total FROM cards WHERE user_id = ?`, 
      [userId], (err, row) => {
        resolve(row.total || 10);
      });
  });
}
app.post('/verify-telegram', validateInitData, async (req, res) => {
  const userId = req.user.id;
  const chatId = '@yourchannel'; // Your Telegram channel
  
  try {
    const response = await fetch(
      `https://api.telegram.org/bot${BOT_TOKEN}/getChatMember?chat_id=${chatId}&user_id=${userId}`
    );
    
    const data = await response.json();
    const isMember = data.result && 
      (data.result.status === 'member' || data.result.status === 'administrator');
    
    if (isMember) {
      db.run(`UPDATE tasks SET completed = 1 
             WHERE user_id = ? AND type = 'telegram'`, [userId]);
    }
    
    res.json({ success: isMember });
  } catch (error) {
    res.status(500).json({ error: 'Verification failed' });
  }
});
// Client-side
const youtubePlayer = document.getElementById('youtube-player');

function onYouTubeIframeAPIReady() {
  player = new YT.Player('youtube-player', {
    events: {
      'onStateChange': onPlayerStateChange
    }
  });
}

function onPlayerStateChange(event) {
  if (event.data === YT.PlayerState.ENDED) {
    document.getElementById('verify-watch').disabled = false;
  }
}

// Backend verification
app.post('/verify-youtube', validateInitData, (req, res) => {
  // Mark as completed in database
  db.run(`UPDATE tasks SET completed = 1 
         WHERE user_id = ? AND type = 'youtube'`, [req.user.id]);
  res.json({ success: true });
});
// Add new card
app.post('/admin/add-card', validateInitData, (req, res) => {
  if (req.user.id !== ADMIN_ID) return res.status(403).send();
  
  const { name, profit_rate, upgrade_cost } = req.body;
  db.run(`INSERT INTO cards (name, profit_rate, upgrade_cost, user_id) 
         VALUES (?, ?, ?, ?)`, 
         [name, profit_rate, upgrade_cost, 'global']);
  
  res.json({ success: true });
});

// Update YouTube URL
app.post('/admin/youtube-url', validateInitData, (req, res) => {
  if (req.user.id !== ADMIN_ID) return res.status(403).send();
  
  db.run(`UPDATE settings SET youtube_url = ?`, [req.body.url]);
  res.json({ success: true });
});
