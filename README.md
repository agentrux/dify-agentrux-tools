# Dify AgenTrux Tools プラグイン

Dify workflow から AgenTrux PubSub トピックへイベントを発行・読み取り・ファイルアップロードするための Tool プラグインです。

---

## Quick Start（5 分）

1. AgenTrux Console で **Script** を作成し、対象 Topic に **Grant** を付与
2. Script の **Activation Code (ac\_...)** を発行
3. Dify で本プラグインをインストール（下記ステップ 2）
4. プラグインの credential 画面で **base_url + Activation Code**（default topic UUID は任意）を入力して **Save**
5. Workflow に **Check Credential** ツールを配置、**Authorized by** で作成した credential を選び実行 → 全項目 `ok` なら完了

---

## 用語

| 用語 | 意味 |
|------|-----|
| **Script** | AgenTrux 上のエージェント単位。Topic に対する read/write Grant を持つ |
| **Grant** | Script が特定の Topic で read / write する権限 |
| **Activation Code (AC)** | 管理者が Script に対して発行する 1 回限りのコード。Dify の Save 時に消費され、永続クレデンシャルと交換される |

---

## 前提条件

| 対象 | 条件 |
|------|------|
| Dify | 1.13.3 以上 |
| plugin_daemon（self-hosted のみ） | `langgenius/dify-plugin-daemon:0.5.7-local` 以上推奨 |
| AgenTrux Console | Script 管理権限 |

### self-hosted で plugin_daemon を固定する

`/opt/dify/docker/docker-compose.yaml`:

```yaml
plugin_daemon:
  image: langgenius/dify-plugin-daemon:0.5.7-local
```

```bash
cd /opt/dify/docker
sudo docker compose pull plugin_daemon
sudo docker compose up -d plugin_daemon
```

Dify Cloud を使っている場合はこの設定は不要です。

---

## ステップ 1 — AgenTrux Console で準備

1. Script を作成
2. Script に対象 Topic の **read / write Grant** を付与
3. **Activation Code** を発行。後で Dify に貼り付ける

---

## ステップ 2 — プラグインをインストール（GitHub install）

Dify ワークスペースの管理者権限でログインし、以下の手順で本プラグインをインストールします。

1. 左メニュー → **Plugins** を開く
2. 画面右上の **「+ Install plugin」** ボタン
3. モーダルで **「GitHub」** タブを選択
4. **Repository URL** に以下を貼り付け:

   ```
   https://github.com/agentrux/dify-agentrux-tools
   ```

5. **Next** → Version ドロップダウンから **最新タグ**（例: `v0.8.2`）を選択 → **Next**
6. Package ドロップダウンから `dify-agentrux-tools-0.8.2.difypkg` を選択 → **Next**
7. プラグインの情報確認画面で **Install** をクリック
8. インストール完了後、Plugins 一覧に **AgenTrux Tools** が表示されれば成功

> **既存バージョンを差し替える場合**: 先に旧バージョンをアンインストールしてから新しい tag を install してください。v0.8.2 ではディスクキャッシュのパスが同一コンテナ内に残っていますが、**アンインストールで cwd ごと削除される**のでキャッシュも消える点に注意（AC の再発行が必要になります）。

### トラブル例

- **「Failed to request plugin daemon」で 400 エラー** → plugin_daemon が `0.5.3-local` のまま。`0.5.7-local` 以上に更新してください（「前提条件」セクション参照）
- **「dify_pkg yaml parse error」** → v0.8.1 以前の package にバグあり。**v0.8.2 以降** を使用してください
- **Dify Cloud** を利用している場合は plugin_daemon のバージョン選定は不要（Dify 側で管理）

---

## ステップ 3 — クレデンシャルを設定

Dify UI で本プラグインの credential / authorize 画面を開き、以下を入力:

| フィールド | 必須 | 内容 |
|-----------|------|------|
| **AgenTrux API URL** | ○ | `https://api.agentrux.com` |
| **Activation Code** | ○ | 発行済みの `ac_...` |
| **Default Topic UUID (write)** | ― | Publish / Upload のフォールバック用 Topic UUID |
| **Default Topic UUID (read)** | ― | Read のフォールバック用 Topic UUID |

**Save** をクリック → AC が消費され、永続クレデンシャルが内部で作成・キャッシュされます。

> **AC の扱い**: Save は**新しい AC 1 個につき 1 回**成功します。再保存は内部キャッシュが使われるので同じ AC で OK。ただしプラグインを**アンインストール→再インストール**した場合はキャッシュが消えるため、新しい AC を発行してください（セキュリティ上、AC は使い回せない仕様）。

### 複数 Topic / 複数 Script を扱う場合

1 credential = 1 Script です。複数の context を使い分ける場合は credential を複数作成し、分かりやすく命名:

- `AgenTrux: project-A (results)`
- `AgenTrux: project-B (admin)`

Workflow の Tool ノードの **Authorized by** で credential を選択して切り替えます。

---

## ステップ 4 — 動作確認（Check Credential）

1. Workflow に `AgenTrux Tools` → **Check Credential** ノードを追加
2. ノードの **Authorized by** で**ステップ 3 で作成した credential を選択**（これを忘れると「No credentials found」エラー）
3. 実行

戻り値:

```json
{
  "jwt_acquire": "ok",
  "topics_visible": 2,
  "default_topic_write_id": {"value": "019d95c4-...", "status": "ok (has write grant)"},
  "default_topic_read_id":  {"value": "019d95c4-...", "status": "ok (has read grant)"},
  "granted_topics": [
    {"topic_id": "019d95c4-...", "action": "write"},
    {"topic_id": "019d95c4-...", "action": "read"}
  ]
}
```

- `jwt_acquire: "ok"` → 認証 OK
- `status: "ok (has ... grant)"` → default UUID が正しく権限ありで設定されている
- `status: "warning (not ... -granted ...)"` → UUID タイポ or Grant 不足。Console で確認
- `topics_visible: 0` → Script に Grant がない

---

## ステップ 5 — ツールを使う

| ツール | 用途 | `topic_id` の扱い |
|-------|-----|-----|
| **Publish Event** | Topic にイベント発行 | Workflow UI で**入力必須**。Agent 等で省略された時のみ credential の `default_topic_write_id` にフォールバック |
| **Read Events** | Topic からイベント取得 | 任意。空時は credential の `default_topic_read_id` を使う |
| **Upload File** | Topic にファイル添付 | Workflow UI で**入力必須**。Agent 等で省略時のみ credential の `default_topic_write_id` にフォールバック |
| **List Topics** | 現 credential で使える全 Topic `[{topic_id, action}]` を返す | パラメータなし。Agent モードで LLM が自動呼び出しする想定 |
| **Check Credential** | 疎通診断 | パラメータなし |

### Publish Event の入出力

入力: `topic_id`, `event_type` (例 `dify.response`), `payload_json` (JSON 文字列), `correlation_id` (任意)
出力: `{event_id, sequence_no, topic_id}` — 実際に送信した topic_id が echo されるので、default フォールバックが発火した場合も可視化されます。

### Read Events の入出力

入力: `topic_id` (任意), `limit` (デフォルト 10), `event_type` (任意フィルタ)
出力: `{count, topic_id, events: [...]}`

### Upload File の入出力

入力: `topic_id`, `filename`, `content_type`, `content_base64`（最大 15 MB）
出力: `{object_id, download_url, topic_id}`

---

## トラブルシューティング

| 症状 | 対処 |
|------|------|
| Save 時「Activation failed」 | AC が既に消費済み。Console で**新 AC を発行**して再保存 |
| ツール実行時「No credentials found」 | ノードの **Authorized by** で credential が未選択か、プラグイン再インストール後にキャッシュが消失。credential を再選択 or 新 AC で再 Save |
| ツール実行時「This credential does not have '...' permission on topic 'X'」 | Script の Grant 範囲外。Console で grant 追加 or 別 credential を選択 |
| Topic フィールドがドロップダウンにならない | Dify 1.13.3 の UI 制約（現時点 Tool plugin は静的 options のみ対応）。UUID は **Check Credential** の `granted_topics` からコピー or credential の `default_topic_*_id` にプリセット |

### Dify ログ（self-hosted）

```bash
# plugin_daemon
docker logs docker-plugin_daemon-1 --tail 200

# dify api
docker logs docker-api-1 --tail 200 2>&1 | grep -iE 'agentrux|plugin|error'
```

---

## 付録 A: 既知の Dify 側制約（実装者向け）

プラグイン実装者・メンテナー向けのメモ。通常の利用者は読む必要はありません。

1. **Tool plugin パラメータの動的ドロップダウン未対応**  
   Dify 1.13.3 の web frontend には generic `useFetchDynamicOptions` フックが存在するが、**Tool-node レンダリング時に wire up されていない**（`useTriggerPluginDynamicOptions` は `provider_type: "trigger"` をハードコード）。backend の `/plugin/parameters/dynamic-options` / `_fetch_parameter_options` dispatch は正常に動作するが、UI が呼ばない。公式 Notion プラグインなども同様に静的 select や `form: llm` で回避している。
2. **plugin_daemon のバージョン選定がシビア**  
   - `0.5.3-local`: query-param binding バグで install 時 400 Bad Request
   - `0.5.8-local` / `latest-local`: heartbeat タイムアウトで Save 時 500（`dify_plugin 0.7.4` SDK との相性）
   - **`0.5.7-local` 推奨**（両バグなし）
3. **Activation Code は server-side で単発使用**  
   プラグインは AC の SHA256 をキーに `.agentrux_activated.json`（atomic write + 0600）でディスクキャッシュし、同じ AC での再 Save に冪等。ただしプラグイン再インストール時には cwd ごとキャッシュが消える。
4. **gevent 互換**  
   `dify_plugin` は `monkey.patch_all()` するため HTTP クライアントは stdlib `urllib.request` を使用（`httpx` は hang する）。

---

## リンク

- 配布リポジトリ: https://github.com/agentrux/dify-agentrux-tools
- ソース: メインリポ `plugins/dify-agentrux-tools/`
- 詳細設計書: `doc/詳細設計/15_Dify_AgenTrux_プラグイン.md`
