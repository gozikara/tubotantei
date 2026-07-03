# ツボ刑事ゲーム — Claude Code 引き継ぎ資料

## 0. 一番大事なこと(必ず読む)

- 作業は完了して**ローカルにコミット済み**だが、**リモートへのプッシュだけがブロックされた**。
- 原因: `gozikara/tubotantei` にインストールされている **Claude の GitHub App が読み取り専用**で、
  `git push`(`git-receive-pack` → 403 Forbidden)も GitHub API(`403 Resource not accessible by integration`)も
  書き込み拒否された。読み取り(clone/fetch)は可能。
- **やるべきこと**: 書き込み権限のある環境(=あなたのローカル Git 認証)でプッシュする。
  権限さえあれば内容はそのまま push するだけで完了する。

## 1. ブランチとコミット

- ブランチ: `claude/pokemon-anatomy-game-ili54y`
- 最新コミット: `3f19aa4`「サロン導線CTAとポケモン忠実化(突入演出/文字送り/技エフェクト)」
  (`5a12879` マップをポケモン風ルートに → `146f08c` クイズ作り直し → `ceb86e1` バトル画面のポケモン化 → `81db800` 初回実装)
- これらのコミットは**まだリモートに無い**。下記の bundle か zip から取り込むこと。
- バトル画面は本家ポケモン対戦画面(敵/味方ステータス枠・戦闘台・青枠メッセージ・2×2コマンド・
  技選択・こうかばつぐん/反撃/ひんし/けいけんち獲得)を再現。突入トランジション・文字送り・
  ダメージ数値ポップも実装済み。
- マップは見下ろしのポケモン風ルート(草原タイル・木・道・草むら・立て札・トレーナー)。
- クイズは admin.html のエピソード情報(原因筋・経穴)から作成、紛らわしい本物の選択肢＋症例ベース。
  選択肢はシャッフル表示、各問に hint 付き。

### ★ サロン導線(CTA)の設定 — ここだけ差し替える
`index.html`(および配布用 `js/data.js` を束ねた単一HTML)の JS 冒頭に `SALON` 定数がある:
```js
const SALON = { name: "セラピスト オンラインサロン", url: "https://example.com/salon" };
```
`name`(表示名)と `url`(入会/LPページ)を実際の値に変更するだけで導線が有効化される。
- 不正解時: その筋・ツボ名を出す軽いチップCTA(タップでサロンを開く)
- バトル終了時: 成績に応じて文言が変わる強めのCTAパネル
URL が未設定(example.com)の間はタップすると「未設定」の案内が出る仕様。

### 取り込み方法A: git bundle(コミット履歴ごと復元・推奨)
```bash
# 既存クローンがある場合
git fetch /path/to/tubotantei-handoff.bundle claude/pokemon-anatomy-game-ili54y
git checkout claude/pokemon-anatomy-game-ili54y   # or: git merge FETCH_HEAD
git push -u origin claude/pokemon-anatomy-game-ili54y
```

### 取り込み方法B: zip(作業ツリー一式)
`tubotantei-handoff.zip` を展開すると `.git` 込みの完全なリポジトリが得られる。
そのまま `git push -u origin claude/pokemon-anatomy-game-ili54y` すればよい。

## 2. 何を作ったか

アップロードされた「エピソード制作ツール」(`admin.html`)を土台に、**症状モンスターと戦いながら
筋肉(解剖)と経穴(ツボ)を学ぶスマホ向け PWA ゲーム**を実装した。ポケモンの4要素を全部入れた:

1. **マップ探索** — 人体シルエット上に事件ピン。クリアで次エリア開放(`mapPos` で座標指定)
2. **ストーリー** — patient / detective / monster の会話劇で事件を導入(吹き出し UI)
3. **バトル** — クイズで症状モンスターの HP を削る。HP=問題数、過半数正解で勝ち、全問正解でボーナス
4. **収集・図鑑** — 原因筋と経穴(位置・圧し方・経絡・注意・効果)を解決時に登録
5. **育成** — 経験値でレベルアップ→捜査ポイントを「診断眼(与ダメ増)」等に割り振り
6. **PWA** — manifest + Service Worker + アイコンで「ホーム画面追加」&オフライン対応

対象プレイヤーは**無資格のセラピスト・整体師**想定(実用寄り、国家試験レベルの難度は避けている)。

## 3. ファイル構成

```
index.html     ゲーム本体(画面・ロジック/約650行、依存なしのバニラJS)
js/data.js     エピソードデータ(window.EPISODES、EP1〜3収録)
manifest.json  PWA設定
sw.js          Service Worker(オフライン用、キャッシュ名 tubo-keiji-v1)
icons/         icon-192.png / icon-512.png / icon-180.png(PILで生成)
admin.html     元アップロードのエピソード制作ツール
README.md      利用者向け説明
HANDOFF.md     このファイル
```

## 4. 重要: js/data.js は破損していたので修復した

リポジトリに元々あった `js/data.js` は **EP3 のクイズ途中でファイルが切れており、
配列が閉じられていない不正な JS** だった(`window.EPISODES = [` が閉じられていなかった)。
EP3 の風池クイズ(場所・圧す向き・危険な頭痛のサイン)を補完し、配列を正しく閉じて有効化した。
現在は EP1〜3 とも各 quiz 5問・script 完備で、`node` で構文チェック済み。

## 5. エピソードの増やし方(データ構造)

`js/data.js` の `window.EPISODES` にオブジェクトを1つ追加するだけ。既存EPをコピーして値を変えるのが楽。

```js
{
  ep: 4,                       // 事件番号(ユニーク)
  title: "…",                  // 事件タイトル
  location: "…",               // マップのラベル
  mapPos: { x: 50, y: 45 },    // マップ上の位置(%指定、人体シルエット基準)
  patient: { name:"", age:0, gender:"", job:"", emoji:"🧍" },
  symptom: "…",                // 主訴
  trigger: "…",                // きっかけ
  gaveup:  "…",                // 諦めたこと・辛かったこと
  monster: { name:"", emoji:"👹", intro:"登場セリフ" },
  muscle:  { name:"", reading:"", info:"解説" },                 // ← 図鑑「原因筋」
  acupoint:{ name:"", reading:"", meridian:"", location:"",
             how:"圧し方", caution:"注意", effect:"効果" },      // ← 図鑑「経穴」
  script: [                    // 会話劇。speaker は narrator/patient/detective/monster
    { speaker:"narrator", text:"…" },
    { speaker:"patient",  text:"…" },
  ],
  quiz: [                      // 1問=1ダメージ。answer は正解のindex(0始まり)
    { q:"…", choices:["","","",""], answer:0, explain:"解説" },
  ],
}
```

- モンスターの HP は `quiz.length` から自動計算(全問正解で確実に撃破)。
- セーブは `localStorage`(キー `tuboKeijiSave_v2`)。データ構造を大きく変えるならキーの版番号を上げる。
- 運用フロー: `admin.html` でプロンプト生成 → claude.ai で台本・クイズ作成 → 上記形式で `js/data.js` に追記。

## 6. ローカル確認方法

```bash
cd tubotantei
python3 -m http.server 8899
# ブラウザで http://localhost:8899/ を開く(スマホ幅で確認推奨)
```
Service Worker は http(s) 配信が必要(file:// では登録されない)。GitHub Pages にそのまま置けば公開可能
(Settings → Pages でブランチ選択、ビルド不要)。

## 7. 動作確認済みの内容

Playwright(Chromium・iPhone相当ビューポート)で全フローを検証済み、JSエラーゼロ:
マップ表示(3ピン)→ 会話送り → バトル(HP 5/5)→ 勝利で +80XP/🪙+40 → EP1クリア&EP2開放 →
図鑑に原因筋+経穴が登録 → 育成画面のポイント割り振り。
