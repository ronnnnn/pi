# dark/light モード切替 実装パターン

すべての生成 HTML はこの実装をそのまま採用する。差し替え不可の固定仕様として扱う (色値の調整は可)。

## 仕様

1. 初期テーマは `localStorage.theme` を最優先、なければ `prefers-color-scheme` に従う
2. テーマ状態は `<html data-theme="light|dark">` に保持する
3. 切替ボタンを押すと `data-theme` を反転し `localStorage` に保存する
4. CSS は `:root { ... }` (light) と `[data-theme="dark"] { ... }` (dark) の 2 セットを定義する
5. メタタグ `<meta name="color-scheme" content="light dark">` を必ず含める

## HTML 部分

```html
<!doctype html>
<html lang="ja" data-theme="light">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <meta name="color-scheme" content="light dark" />
    <!-- FOUC 防止: スタイル適用前にテーマを決定する -->
    <script>
      (function () {
        try {
          var saved = localStorage.getItem('theme');
          var prefersDark =
            window.matchMedia &&
            window.matchMedia('(prefers-color-scheme: dark)').matches;
          var theme = saved || (prefersDark ? 'dark' : 'light');
          document.documentElement.setAttribute('data-theme', theme);
        } catch (e) {}
      })();
    </script>
    ...
  </head>
</html>
```

## CSS 変数

```css
:root {
  --bg: #faf9f5;
  --surface: #ffffff;
  --border: #e3dacc;
  --text: #141413;
  --text-muted: #5b5a55;
  --accent: #d97757;
  --accent-fg: #ffffff;
  --code-bg: #f0eee6;
  --shadow: 0 1px 2px rgba(0, 0, 0, 0.06), 0 8px 24px rgba(0, 0, 0, 0.04);
}

[data-theme='dark'] {
  --bg: #1a1a18;
  --surface: #232320;
  --border: #3a3a36;
  --text: #f0eee6;
  --text-muted: #b1afa6;
  --accent: #f29670;
  --accent-fg: #1a1a18;
  --code-bg: #2a2a26;
  --shadow: 0 1px 2px rgba(0, 0, 0, 0.4), 0 8px 24px rgba(0, 0, 0, 0.3);
}

body {
  background: var(--bg);
  color: var(--text);
  transition:
    background 0.2s,
    color 0.2s;
}
```

## 切替ボタン

```html
<button
  id="theme-toggle"
  class="icon-btn"
  aria-label="Toggle theme"
  data-ja-aria-label="テーマ切替"
  data-en-aria-label="Toggle theme"
>
  <span class="icon" aria-hidden="true">
    <!-- インライン SVG (sun / moon) を切替 -->
    <svg class="icon-sun" viewBox="0 0 24 24" width="18" height="18">
      <circle cx="12" cy="12" r="4" fill="currentColor" />
      <g stroke="currentColor" stroke-width="2" stroke-linecap="round">
        <path
          d="M12 2v3M12 19v3M2 12h3M19 12h3M4.9 4.9l2.1 2.1M17 17l2.1 2.1M4.9 19.1L7 17M17 7l2.1-2.1"
        />
      </g>
    </svg>
    <svg class="icon-moon" viewBox="0 0 24 24" width="18" height="18">
      <path
        d="M21 12.8A9 9 0 1 1 11.2 3a7 7 0 0 0 9.8 9.8z"
        fill="currentColor"
      />
    </svg>
  </span>
</button>
```

```css
.icon-sun {
  display: none;
}
[data-theme='dark'] .icon-sun {
  display: inline;
}
[data-theme='dark'] .icon-moon {
  display: none;
}
```

## 切替 JS

```js
(function () {
  const btn = document.getElementById('theme-toggle');
  if (!btn) return;
  btn.addEventListener('click', () => {
    const cur = document.documentElement.getAttribute('data-theme') || 'light';
    const next = cur === 'light' ? 'dark' : 'light';
    document.documentElement.setAttribute('data-theme', next);
    try {
      localStorage.setItem('theme', next);
    } catch (e) {}
  });
})();
```

## SVG 内の色

SVG の `fill`/`stroke` は `currentColor` または `var(--accent)` などの CSS 変数を `style` 属性経由で指定し、テーマ変更で自動連動させる。

## 注意

- `transition` を多用しすぎるとテーマ切替時にチカチカする。`background-color` と `color` 程度に絞る
- 画像を使う場合、light/dark で別画像を用意し `<picture>` で出し分ける
- コードハイライト用クラス (`tok-kw` 等) も両テーマで定義する
