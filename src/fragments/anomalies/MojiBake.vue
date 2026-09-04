<script setup>
import { onMounted, onUnmounted } from "vue";
import { registerCleanup } from "../../store/game-store.js";

// x秒ごとにy文字の文字数を文字化けさせる。(上限10回)
// 文字化けは実際のDOMテキストノードを直接書き換えるため、
// 異変が終わった際に必ず元へ戻せるよう、元テキストを保持しておく。
let textCorruptionIntervalId = null;
const originalTexts = new Map(); // 書き換えたノード -> 元のテキスト
let unregisterCleanup = null;

const corruptChars = [
  "�",
  "□",
  "▢",
  "▣",
  "▤",
  "▥",
  "▦",
  "▧",
  "▨",
  "▩",
  "◼",
  "◻",
  "▪",
  "▫",
  "■",
  "□",
  "▬",
  "▭",
  "▮",
  "▯",
  "◆",
  "◇",
  "◈",
  "◉",
  "◊",
  "○",
  "●",
  "◐",
  "◑",
  "◒",
  "◓",
  "◔",
  "◕",
];

// DOMツリーからテキストノードを収集する関数
function getTextNodes(node, textNodes = []) {
  if (node.nodeType === Node.TEXT_NODE) {
    // 空白や改行のみのノードは除外
    if (node.textContent.trim().length > 0) {
      textNodes.push(node);
    }
  } else {
    // scriptタグとstyleタグは除外
    if (node.nodeName !== "SCRIPT" && node.nodeName !== "STYLE") {
      for (let child of node.childNodes) {
        getTextNodes(child, textNodes);
      }
    }
  }
  return textNodes;
}

// ランダムな文字化け文字を取得
function getRandomCorruptChar() {
  return corruptChars[Math.floor(Math.random() * corruptChars.length)];
}

function startTextCorruption(intervalSec = 3, countPer = 3) {
  const interval = 1000 * intervalSec;
  let executionCount = 0;
  const maxExecutions = 10;

  // テキストをランダムに文字化けさせる関数
  function corruptText() {
    if (executionCount >= maxExecutions) {
      clearInterval(intervalId);
      textCorruptionIntervalId = null;
      return;
    }

    // 現在のページのテキストノードを取得
    const textNodes = getTextNodes(document.body);

    if (textNodes.length === 0) {
      return;
    }

    // 文字化けさせる文字数分ループ
    for (let i = 0; i < countPer; i++) {
      // ランダムにテキストノードを選択
      const randomNode =
        textNodes[Math.floor(Math.random() * textNodes.length)];
      const text = randomNode.textContent;

      if (text.length === 0) continue;

      // 元のテキストを保存（初回のみ）
      if (!originalTexts.has(randomNode)) {
        originalTexts.set(randomNode, text);
      }

      // ランダムな位置の文字を文字化け文字に置き換え
      const randomIndex = Math.floor(Math.random() * text.length);
      const newText =
        text.substring(0, randomIndex) +
        getRandomCorruptChar() +
        text.substring(randomIndex + 1);

      randomNode.textContent = newText;
    }

    executionCount++;
  }

  // インターバルで実行（初回はintervalミリ秒後）
  const intervalId = setInterval(corruptText, interval);
  return intervalId;
}

// 書き換えたテキストをすべて元に戻す
function restoreOriginalTexts() {
  originalTexts.forEach((original, node) => {
    // 文字化けさせた文字が今も残っている場合のみ元に戻す。
    // （Vueの再描画で既に別の内容へ更新されたノードは上書きしない）
    try {
      node.textContent = original;
    } catch (e) {
      // ノードが既にDOMから切り離されている場合などは無視
    }
  });
  originalTexts.clear();
}

// 文字化けを止めて画面を元に戻す
function stopTextCorruption() {
  if (textCorruptionIntervalId) {
    clearInterval(textCorruptionIntervalId);
    textCorruptionIntervalId = null;
  }
  restoreOriginalTexts();
}

// コンポーネントがマウントされた後に文字化け機能を開始
onMounted(() => {
  textCorruptionIntervalId = startTextCorruption(1.5, 5); // 1.5秒ごとに5文字ずつ文字化け

  // ラウンド切り替え時（cleanupCurrentAnomaly）に同期的に呼ばれる。
  // ここで文字化けを停止し、元のテキストへ確実に復元する。
  unregisterCleanup = registerCleanup(() => {
    stopTextCorruption();
  });
});

// コンポーネントがアンマウントされる際のフォールバック
onUnmounted(() => {
  stopTextCorruption();
  if (unregisterCleanup) {
    unregisterCleanup();
    unregisterCleanup = null;
  }
});
</script>
