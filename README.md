<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>CodePen Wall</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --bg-color-light: #f0f2f5;
      --card-bg-light: rgba(255,255,255,0.6);
      --text-color-light: #333;
      --accent-color-light: #0ebeff;
      --input-bg-light: #fff;
      --button-bg-light: #0ebeff;
      --button-text-light: #fff;

      --bg-color-dark: #1a1a2e;
      --card-bg-dark: rgba(40,42,54,0.65);
      --text-color-dark: #f0f2f5;
      --accent-color-dark: #0ebeff;
      --input-bg-dark: #333;
      --button-bg-dark: #162447;
      --button-text-dark: #e4f9f5;

      --blur: blur(8px);
      --transition: 0.3s;

      --bg-color: var(--bg-color-light);
      --card-bg: var(--card-bg-light);
      --text-color: var(--text-color-light);
      --accent-color: var(--accent-color-light);
      --input-bg: var(--input-bg-light);
      --button-bg: var(--button-bg-light);
      --button-text: var(--button-text-light);
    }

    body.dark-mode {
      --bg-color: var(--bg-color-dark);
      --card-bg: var(--card-bg-dark);
      --text-color: var(--text-color-dark);
      --accent-color: var(--accent-color-dark);
      --input-bg: var(--input-bg-dark);
      --button-bg: var(--button-bg-dark);
      --button-text: var(--button-text-dark);
    }

    body {
      font-family: system-ui, sans-serif;
      background-color: var(--bg-color);
      color: var(--text-color);
      min-height: 100vh;
      display: flex;
      flex-direction: column;
      transition: background-color var(--transition);
    }

    .app-header {
      background: var(--card-bg);
      backdrop-filter: var(--blur);
      position: sticky;
      top: 0;
      z-index: 10;
      padding: 1rem;
      display: flex;
      flex-wrap: wrap;
      gap: 10px;
      justify-content: center;
    }

    .form-container {
      display: flex;
      flex: 1;
      gap: 10px;
      max-width: 700px;
    }

    input[type="url"] {
      flex: 1;
      padding: 10px;
      border: 1px solid #ccc;
      border-radius: 6px;
      background: var(--input-bg);
      color: var(--text-color);
      transition: border var(--transition);
    }

    input[type="url"]:focus {
      border-color: var(--accent-color);
      outline: none;
    }

    button {
      background: var(--button-bg);
      color: var(--button-text);
      padding: 10px 16px;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      transition: background var(--transition);
    }

    button:hover {
      opacity: 0.9;
    }

    #pen-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
      gap: 20px;
      padding: 20px;
      width: 100%;
      max-width: 1400px;
      margin: 0 auto;
      flex: 1;
    }

    .pen-card {
      background: var(--card-bg);
      border-radius: 10px;
      overflow: hidden;
      box-shadow: 0 5px 15px rgba(0,0,0,0.1);
      display: flex;
      flex-direction: column;
    }

    .pen-card iframe {
      width: 100%;
      aspect-ratio: 16 / 10;
      border: none;
    }

    .pen-card .action-bar {
      text-align: center;
      padding: 10px;
      background: rgba(0,0,0,0.05);
    }

    .pen-card .action-bar a {
      text-decoration: none;
      background: var(--button-bg);
      color: var(--button-text);
      padding: 8px 16px;
      border-radius: 6px;
      font-weight: bold;
      display: inline-block;
      transition: background 0.3s;
    }

    .pen-card .action-bar a:hover {
      opacity: 0.85;
    }

    .controls {
      display: flex;
      gap: 10px;
      align-items: center;
    }

    .error-message {
      color: red;
      font-size: 0.9rem;
      text-align: center;
      padding: 5px 10px;
      display: none;
    }

    .theme-toggle {
      font-size: 1.2rem;
      cursor: pointer;
    }

    .app-footer {
      text-align: center;
      padding: 10px;
      font-size: 0.9rem;
      color: var(--text-color);
    }
  </style>
</head>
<body>

  <header class="app-header">
    <form id="add-pen-form" class="form-container">
      <input type="url" id="pen-url-input" placeholder="Paste CodePen URL..." required />
      <button type="submit">Add Pen</button>
    </form>
    <div class="controls">
      <button id="clear-all-btn">Clear All</button>
      <span id="theme-toggle" class="theme-toggle">☀️</span>
    </div>
  </header>

  <p id="error-feedback" class="error-message"></p>

  <main id="pen-grid"></main>

  <footer class="app-footer">
    CodePen Wall | Built with HTML5, CSS3, ES6 | AI Powered
  </footer>

  <script>
    document.addEventListener('DOMContentLoaded', () => {
      const form = document.getElementById('add-pen-form');
      const urlInput = document.getElementById('pen-url-input');
      const penGrid = document.getElementById('pen-grid');
      const clearAllBtn = document.getElementById('clear-all-btn');
      const themeToggle = document.getElementById('theme-toggle');
      const errorFeedback = document.getElementById('error-feedback');

      const STORAGE_KEY_PENS = 'codepenWallPens';
      const STORAGE_KEY_THEME = 'codepenWallTheme';

      const showError = (msg) => {
        errorFeedback.textContent = msg;
        errorFeedback.style.display = 'block';
      };

      const clearError = () => {
        errorFeedback.textContent = '';
        errorFeedback.style.display = 'none';
      };

      const parseCodePenUrl = (url) => {
        try {
          const parsed = new URL(url);
          if (parsed.hostname !== 'codepen.io') return null;

          const parts = parsed.pathname.split('/').filter(Boolean);
          if (parts.length >= 3 && ['pen', 'details', 'full', 'pres'].includes(parts[1])) {
            return { user: parts[0], slug: parts[2] };
          }
        } catch {
          return null;
        }
        return null;
      };

      const createPenEmbed = (url, user, slug) => {
        const penCard = document.createElement('div');
        penCard.className = 'pen-card';
        penCard.dataset.url = url;

        const embedUrl = `https://codepen.io/${user}/embed/${slug}?default-tab=result&theme-id=${document.body.classList.contains('dark-mode') ? 'dark' : 'light'}`;
        const viewUrl = `https://codepen.io/${user}/pen/${slug}`;

        const iframe = document.createElement('iframe');
        iframe.src = embedUrl;
        iframe.title = `CodePen Embed: ${slug}`;
        iframe.loading = 'lazy';
        iframe.sandbox = 'allow-scripts allow-same-origin allow-forms allow-popups allow-presentation';

        const actionBar = document.createElement('div');
        actionBar.className = 'action-bar';
        actionBar.innerHTML = `<a href="${viewUrl}" target="_blank">View in CodePen</a>`;

        penCard.appendChild(iframe);
        penCard.appendChild(actionBar);
        return penCard;
      };

      const addPen = (url) => {
        clearError();
        const parsed = parseCodePenUrl(url);
        if (!parsed) {
          showError('Invalid CodePen URL format.');
          return false;
        }

        if ([...penGrid.children].some(p => p.dataset.url === url)) {
          showError('This pen is already added.');
          return false;
        }

        const pen = createPenEmbed(url, parsed.user, parsed.slug);
        penGrid.appendChild(pen);
        saveUrl(url);
        return true;
      };

      const saveUrl = (url) => {
        const saved = JSON.parse(localStorage.getItem(STORAGE_KEY_PENS)) || [];
        if (!saved.includes(url)) {
          saved.push(url);
          localStorage.setItem(STORAGE_KEY_PENS, JSON.stringify(saved));
        }
      };

      const loadPens = () => {
        const saved = JSON.parse(localStorage.getItem(STORAGE_KEY_PENS)) || [];
        saved.forEach(addPen);
      };

      const clearPens = () => {
        localStorage.removeItem(STORAGE_KEY_PENS);
        penGrid.innerHTML = '';
        clearError();
      };

      const applyTheme = (theme) => {
        document.body.classList.toggle('dark-mode', theme === 'dark');
        themeToggle.textContent = theme === 'dark' ? '🌙' : '☀️';
        localStorage.setItem(STORAGE_KEY_THEME, theme);
      };

      const loadTheme = () => {
        const theme = localStorage.getItem(STORAGE_KEY_THEME) || 'light';
        applyTheme(theme);
      };

      form.addEventListener('submit', (e) => {
        e.preventDefault();
        const url = urlInput.value.trim();
        if (url && addPen(url)) {
          urlInput.value = '';
        }
      });

      clearAllBtn.addEventListener('click', clearPens);
      themeToggle.addEventListener('click', () => {
        const newTheme = document.body.classList.contains('dark-mode') ? 'light' : 'dark';
        applyTheme(newTheme);
      });
      urlInput.addEventListener('input', clearError);

      loadTheme();
      loadPens();
    });
  </script>

</body>
</html>
