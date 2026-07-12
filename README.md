# 0. 概要

## 0-1. 基本的機能

- ブラウザ上で動作する、Python開発環境（IDE）です。
- Pyodide (WebAssembly) を使用しており、サーバーを介さずにブラウザローカルでPythonコードを実行できます。

## 0-2. 試験的機能

- web版Geminiと連携し、AIが生成したコードを即座に実行・検証する「サンドボックス・エージェント」としても機能します。
- 「サンドボックス・エージェント」機能を利用する場合、GeminiページとIDEページ間でPost Message通信する為のブックマークレットをGeminiページで実行する必要があります。

# 1. 特徴

## 1-1. ブラウザ完結のPython環境

- Pyodide採用: Python 3.x ランタイムがブラウザ上で動作します。
  *  サーバーサイドへのデータ送信がないため、プライバシー面でも安心です。
  *  ※ 「サンドボックス・エージェント」機能を利用する場合、通常のGemini利用時と同等のデータがプロンプト入力によってGoogle社へ送信されます。
  *  非エンタープライズ契約の場合、Google社はプロンプトに入力されたデータをレビューし、モデルの再学習に利用する可能性がある点を理解してください。

- 主要ライブラリ対応: numpy, pandas, matplotlib, scikit-learn などのデータサイエンス系ライブラリが利用可能です（micropipによるインストール対応）。

## 1-2. コードエディタ

- Monaco Editor搭載: VS Codeと同じエディタエンジンを採用。
  *  シンタックスハイライト、自動補完、アウトラインなどが利用できます。
- タブ型インターフェース: 複数のファイルを同時に開き、切り替えて編集可能です。

## 1-3. 高度なファイルシステム (VFS)

- 仮想ファイルシステム: Pyodideはブラウザメモリ上に仮想的なファイルシステムを持っています。
- ローカルフォルダマウント: Chrome等の「File System Access API」を使用し、PC内のローカルフォルダを直接マウントして読み書きできます。
- Drag & Drop: ファイルやフォルダをドラッグ＆ドロップでVFSにアップロード可能。
- ZIP圧縮・暗号化: VFS内のファイルをZIPとしてダウンロードしたり、パスワード付きZIPを作成・解凍できます。
  *  この際、セキュリティ上の懸念から「.bat」「.」「」

## 1-4. AI連携

### Gemini Bridge

- 双方向通信: 専用のブックマークレットを使用することで、Web版 Google Gemini のページと連携します。
- 自動実行: Geminiが生成したPythonコードを自動的にIDEに転送し、実行します。
  *  結果フィードバック: 実行結果（標準出力やエラー）やグラフ画像を自動的にGeminiのチャット欄にフィードバックします。

### Gemini nano

- Chrome に組み込まれている小型ローカルLLM、Gemini nano をエージェンティックに動作させ、コーディングを行わせる事が可能です
  - Gemini nano は32.5Bパラメータの計量モデルであり、その性能は限定的です

## 1-5. その他の機能

- Matplotlib描画: plt.show() で生成されたグラフを専用パネルに表示します。
- Web Serial API: ブラウザからシリアルポート（Arduinoやマイコン）に接続し、Pythonから制御可能です。
- HTML/JSプレビュー: Pythonだけでなく、HTML/JavaScriptのリアルタイムプレビューも可能です。

# 2. 使い方

## 2-1. IDE Webアプリとして利用する場合

1. GitHub PagesなどでホスティングされたURLにアクセスするだけで利用可能です。インストールは不要です。
2. 公開URL（ https://aki-coding.github.io/Pyodide_Web_IDE/prod/IDE/Pyodide_IDE.html ）にアクセスします。
3. エディタが表示され、すぐにPythonコードを記述・実行（▶ボタン）できます。

## 2-2. Geminiと連携する場合 (Bridge bookmarklet)

1. AIとの連携機能を使用するには、以下の手順でブックマークレットを設定してください。
2. ブックマークレットインストーラー（ https://aki-coding.github.io/Pyodide_Web_IDE/prod/bookmarklet/installer.html ） を開きます。
3. ブックマークレットをブックマークバーに登録します。
4. Google Gemini の Gem を作成します。
  *  ※Gemのカスタムプロンプトは ( https://aki-coding.github.io/Pyodide_Web_IDE/prod/prompt/GemCustomPrompt.txt ) の内容を登録してください
5. Google Gemini の Gem を開きます。
6. 登録したブックマークレットをクリックします。
7. IDEが別ウィンドウで開き、Geminiとの接続が確立されます。
8. IDEとGeminiを分割タブなどで同時に表示します。
9. Gemにプロンプトを与えてください。

# 3. 技術スタック

- Runtime: Pyodide (WebAssembly)
- Editor: Monaco Editor
- table: jspreadsheet
- Compression: zip.js
- Language: HTML5, CSS3, JavaScript (ES6+)

# 4. 注意事項

- セキュリティ: ブラウザのサンドボックス内で動作しますが、信頼できないコードを実行する際は注意してください。
- データ永続化: VFS（仮想ファイルシステム）の内容はメモリ上にあり、リロードすると消えます。重要なデータはこまめにローカルへダウンロードしてください。
- CSP制限: Gemini連携時は、クロスオリジン通信を行うため、適切なホスティング環境（GitHub Pages等）が必要です。

# 5. 免責事項

- このWebアプリは実験段階（Beta版）であり、予期せぬバグを含んでいる可能性があります。
- このアプリを利用することで生じたデータの損失、法的・社会的責任、その他一切のトラブルについて、開発者は責任を負いかねます。
  *  利用者は自己の責任において本アプリを利用するものとします。

# 6. ライセンス

- This project is licensed under the MIT License - see the LICENSE file for details.
