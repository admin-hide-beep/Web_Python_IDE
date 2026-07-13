# IDEウェブアプリ

## コンセプト

### IDE 部

- Python IDEとして、コードエディタ機能とPythonスクリプトの実行機能を有する
  - インストール不要なwebアプリ
  - 「GAS」「file:///」「localhost」「https」によるホスティングを想定
  - Python実行環境を含むメインのIDEコンテンツはsandbox属性を付与したiframe内で動作
    - これはセルフXSS等を行った際でもセッションジャックされない為のセキュリティ考慮

- GitHub のリポジトリにアクセスし、`Script_Explorer` 部分に表示する機能
  - ヘッダーの Gemini Nano 起動ボタンの右に GitHub ボタンを設ける
    - IDEウェブアプリが他のウェブページにiframeで埋め込まれている場合は非表示
    - 故にGASホスティング時は動作させない
    - ボタンはIDEのロードが完了するまで押せない様にする
  - PAT入力ではなく、ログイン画面ポップによる認証
    - Web版VSCode の GitHub 連携機能と同様の仕組み
    - Cloudflare Workers のサーバーレス関数で中継する仕組みとしたい
    - 認証情報はIDEフレームに渡してはならない
  - ホストフレームのCSPのみ "https://api.github.com" との通信を許可
  - 認証情報はローカルに保持せず、セッション終了とともに破棄
  - GitHubに変更をプッシュする機能

### AI Agent 部

- web版 Gemini をブックマークレットでブリッジする方法と、Chrome ブラウザ組み込みの Gemini nano を利用する方法に対応
  - Gemini の返答内容を解析する
    - 解析した結果、所定のコマンドが見つかればそれに応じた処理を実行する

# 各種コマンドのフォーマット

## Gemini からシステムへのスクリプト実行依頼 json フォーマット

```json
{
  "cmd": "script_run_request",
  "script": "Pythonスクリプトを記述"
}
```

## システムから Gemini へのスクリプト実行結果 json フォーマット

```json
{
  "cmd": "script_result",
  "result": "実行結果を記述"
}
```

## Gemini からシステムへの代理記憶依頼 json フォーマット

```json
{
  "cmd": "add_memory",
  "content":
  [
    {
      "key_1": "value_1"
    },
    {
      "key_2": "value_2"
    }
  ]
}
```

-  代理記憶する場合、ブックマークレットの代理記憶オブジェクトに新しいキーと値を追加します
  - 重複キーの場合は値を上書きします

- システムの代理記憶オブジェクトは以下のような構造になります

```json
{
  "cmd": "memory",
  "content":
  [
    {
      "key_1": "value_1"
    },
    {
      "key_2": "value_2"
    }
  ]
}
```

## Gemini からシステムへの代理記憶提示依頼 json フォーマット

```json
{
  "cmd": "memory_presentation"
}
```

## Gemini からユーザーへの検証依頼 json フォーマット

```json
{
  "cmd": "user_verification",
  "message": "ユーザー検証内容"
}
```

## Gemini からユーザーへの目的達成 json フォーマット

```json
{
  "cmd": "goal_achieved",
  "message": "目的達成内容"
}
```

## システムから Gemini へのコマンド json 多数検出通知フォーマット

```json
{
  "cmd": "Multiple_commands",
  "message": "システム通知: 複数のコマンド（JSON）が検出されました。一度に実行できるコマンドは一つだけです。順次実行するか、一つに絞ってください。"
}
```

## システムから Gemini への規定フォーマット json 未検出通知フォーマット

```json
{
  "cmd": "No_Command",
  "message": "出力の中にコマンドが発見できませんでした。意図したものであれば次のステップへ進んでください。意図したものでなければ修正してください。"
};
```