# のれん承 ─ 外食チェーン経営記

会計・マクロ経済・住民ペルソナ需要モデルを備えた外食チェーン経営シミュレーション（単一 `index.html`、依存ライブラリなし）。

このリポジトリは **GitHub → Vercel（Webで公開） → Google Play（TWAでアプリ化）** の3段階で公開する前提で構成されています。

## ファイル構成

| ファイル | 役割 |
|---|---|
| `index.html` | ゲーム本体（このファイルだけで完結） |
| `manifest.json` | PWA（Webアプリ）マニフェスト。アプリ名・アイコン・テーマ色など |
| `icon.svg` / `icon-maskable.svg` | アプリアイコン（暖簾モチーフ）。TWA化の際にPWABuilderがここから各サイズのPNGを生成 |
| `.well-known/assetlinks.json` | **TWA化の後半で編集が必要**（Android アプリとこのWebサイトの所有関係を証明するファイル） |
| `vercel.json` | Vercelの配信設定（manifest/assetlinksのContent-Type指定） |

ローカル確認: `.claude/launch.json` の `noren-tycoon`（port **8798**）→ `C:\Users\81909\noren-tycoon` を配信。

---

## Step 1. GitHubにpush

このフォルダは既に `git init` 済み・ローカルコミット済みです（後述）。GitHub側の操作：

1. https://github.com/new で新規リポジトリを作成（例：`noren-tycoon`）。**READMEやgitignoreは追加しない**（空リポジトリにする）。
2. 作成後に表示されるリモートURLを使って、このフォルダから push：

```bash
cd /c/Users/81909/noren-tycoon
git remote add origin https://github.com/<あなたのユーザー名>/noren-tycoon.git
git branch -M main
git push -u origin main
```

`gh` CLI が使えるなら `gh repo create noren-tycoon --public --source=. --push` の1行でも作成できます（このマシンには未インストールでした）。

---

## Step 2. Vercelでデプロイ

1. https://vercel.com に GitHub アカウントでログイン。
2. 「Add New... → Project」→ 先ほどの `noren-tycoon` リポジトリを選択して Import。
3. Framework Preset は **Other**（静的サイトなのでビルド設定不要、Build Command は空のままでOK）。
4. Deploy をクリック。数十秒で `https://noren-tycoon-xxxx.vercel.app` のようなURLが発行されます。
5. 以後は `main` ブランチに push するたびに自動で再デプロイされます。

独自ドメイン（例: `noren-tycoon.com`）を使いたい場合は Vercel の Project → Settings → Domains から追加できます（TWA化にはドメインは必須ではなく、Vercelが発行する `*.vercel.app` のままでも可）。

**ここでできること**：発行されたURLをスマホのChromeで開き、メニューから「ホーム画面に追加」を選ぶと、既にPWAとして（ブラウザのアドレスバーなしで）起動できるはずです。この状態でゲームが問題なく遊べることを確認してください。

---

## Step 3. Google Play向けにTWA（Trusted Web Activity）を生成

TWAは「中身はWebサイトだが、Google Playで配布できるAndroidアプリの皮」です。Android StudioやNode.js不要で作れる **PWABuilder** を使います。

1. https://www.pwabuilder.com/ を開き、Step2で発行したVercelのURL（`https://xxxx.vercel.app`）を入力して「Start」。
2. マニフェスト・アイコン・Service Workerの診断が出ます。アイコンは `manifest.json` に登録済みのSVGから自動生成されますが、必要なら「Image Generator」（https://www.pwabuilder.com/imageGenerator）で `icon.svg` をアップロードし、Android用アイコン一式を作り直せます。
3. 「Package for stores」→「Android」を選択。
   - Package ID（例：`com.yourname.norentycoon`、Google Play上のアプリの一意なID）
   - App name / Short name
   - 署名鍵：**新規に作る**か、既存の `.jks` をアップロード。**新規作成した場合はここでダウンロードされる `.jks`（キーストア）ファイルとパスワードを絶対に紛失しないよう保管してください**（アップデート時に同じ鍵が必要）。
4. 生成された `.aab`（Android App Bundle）と署名情報一式（`signing.json` 等）がZIPでダウンロードされます。
5. ZIP内の `signing.json` または PWABuilder の画面に、**SHA-256フィンガープリント**が表示されます（例：`14:6D:E9:...`）。これをコピー。

---

## Step 4. assetlinks.json を更新して再デプロイ（重要）

Step 3 で取得した Package Name と SHA-256 フィンガープリントを、このリポジトリの `.well-known/assetlinks.json` に反映します。

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.yourname.norentycoon",
      "sha256_cert_fingerprints": ["14:6D:E9:...（実際の値）"]
    }
  }
]
```

編集したら `git add .well-known/assetlinks.json && git commit -m "assetlinks更新" && git push` でVercelに再デプロイ。これが公開されていないと、TWAのアプリを開いたときにブラウザのアドレスバーが表示されてしまいます（「検証済みアプリ」と認識されない）。

反映確認：`https://あなたのURL/.well-known/assetlinks.json` にブラウザでアクセスし、上記JSONが正しく表示されればOK。

---

## Step 5. Google Play Consoleで公開

1. https://play.google.com/console （Googleアカウントで登録、**初回$25の登録料**が必要）。
2. 「アプリを作成」→ Step 3 で決めた Package ID のアプリを新規作成。
3. 「テスト → 内部テスト」または「製品版」から新しいリリースを作成し、Step 3 で生成した `.aab` をアップロード。
4. ストア掲載情報（アプリ名・説明文・スクリーンショット・アイコン・プライバシーポリシーURLなど）を入力。スクリーンショットはスマホでゲーム画面をキャプチャして用意してください。
5. コンテンツのレーティング分類・データセーフティ（このゲームは外部通信なし、`localStorage` にセーブデータをブラウザ内保存するのみ）などの質問に回答。
6. 審査に提出。内部テストなら数時間、製品版審査は数日かかることがあります。

---

## 今後のアップデートの流れ

1. `index.html` 等をローカルで編集
2. `git add -A && git commit -m "更新内容" && git push`
3. Vercelが自動で再デプロイ（Webは即反映）
4. **Androidアプリ側の見た目や動作を変えたい場合**（TWAの外側の設定を変える場合のみ）は PWABuilder で再生成が必要。中身のWeb変更だけなら、TWAは常にVercel上の最新版を表示するため、**Playストアの再提出は基本的に不要**です（TWAはブラウザ相当のラッパーなので、Web側の更新が自動で反映されます）。
5. アプリの Package ID や署名鍵を変えない限り、既存ユーザーへの更新配信は維持されます。署名鍵（`.jks`）は絶対に失くさないこと。

---

## 補足：セーブデータについて

このゲームは `localStorage`（ブラウザのオリジン単位のローカル保存）にオートセーブします。`file://` で開いていた開発時と異なり、`https://` の実URLになることで保存の信頼性が上がります。TWAアプリ内でも同じオリジンの `localStorage` を使うため、Webブラウザ版とアプリ版でセーブは共有されません（別々のセーブ扱いになります）。
