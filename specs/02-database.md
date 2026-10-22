# 運動会 リアルタイム得点管理・応援システム「SPO-CONNECT」仕様書

## 3. ユーザー識別・キャッシュ・データベース仕様

### 3.1 ユーザーID構造、属性分離およびオンライン再同期プロトコル

- **ユーザーIDフォーマット:** `[チームID]-[無作為生成ランダムID]` (例: `team_a-x8f2k9l1`)
- **識別情報と表示名の分離（不変性と可変性の分離）:**
  - ユーザーIDおよびQRコードの主たる識別情報は不変の識別子として扱い、名前（`user_name`）やチーム所属はサーバー側の動的な現行属性として厳密に分離する。
  - 参加者スマホ側のローカルストレージ（キャッシュ）には、`db_id`、`user_id`に加え、属性のバージョン番号（`attr_rev`）や最終更新日時を保持させる。
- **キオスク端末による名前変更時の反映とキャッシュ無効化:**
  - キオスク端末の「名前変更モード」でユーザー名が更新された際、サーバーDBおよびサーバー側キャッシュ（`user_cache.json`）の属性リビジョン（`attr_rev`）をインクリメントする。
  - キオスク等での変更時点でスマホ側がオフラインであった場合、一時的にスマホ側ローカルの古い名前が保持されるが、不変の `user_id` がQRコードの正本として機能するため、本人識別（出席・得点紐付け）が破綻することは構造的に防止される。
- **オンライン復帰時の属性再同期フロー:**
  - 参加者スマホがオンラインに復帰した際（または定期ポーリング時）、サーバー側の現行属性リビジョンとローカルの `attr_rev` を比較する。
  - 差異を検知した場合は、サーバー側の最新のユーザー名・チーム所属を自動取得してローカルキャッシュを上書き更新し、表示情報の不整合を速やかに解消する。

### 3.2 ユーザー名変更モード (キオスク端末)

1. キオスク端末を「名前変更モード」にする。
2. 変更対象の参加者QRコードを読み取る。
3. キオスク画面上にテキストボックスが出現。新しい名前を入力して確定ボタンを押すことで、DBおよびキャッシュJSON（`user_cache.json`）が更新される。

### 3.3 データベース構成、スキーマバージョン管理 & 自動マイグレーションプロトコル

単なる `CREATE TABLE IF NOT EXISTS` に依存するのではなく、アプリケーション起動時にデータベースのスキーマバージョンを検知し、安全に構造をアップデートするマイグレーション機構を必須とする。

- **スキーマバージョン管理テーブルの導入:**

```sql
-- スキーマバージョン管理テーブル
CREATE TABLE IF NOT EXISTS schema_versions (
    version INTEGER PRIMARY KEY,
    applied_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

- **起動時マイグレーション実行フロー:**
  - サーバー起動時（`src/database.js` の初期化プロセス内）、接続したSQLiteの `schema_versions` テーブルを確認し、現在の適用済み最新バージョン（例: v3）を特定する。
  - コード側に定義された目標スキーマバージョン（例: v4）との間に乖離がある場合、トランザクション内で自動マイグレーションスクリプトを順次実行する。
- **安全確保のための厳格な運用手順 (DR & Testing):**
  - **自動バックアップの強制:** マイグレーション処理を実行する直前に、必ず指定のバックアップディレクトリ（`./data/backups/`）へ、`VACUUM INTO` 方式によるアトミックなスナップショットを自動生成する。
  - **移行失敗時の即時停止とロールバック:** マイグレーション中にSQL構文エラーや不整合が発生した場合、即座にトランザクションをロールバックしてデータベース変更を破棄し、サーバーの起動を安全に停止（Panic Exit）してエラーログを出力する。
  - **移行テストの義務付け:** 本番投入前に、過去の古いスキーマバージョンで作成されたテスト用データファイルを用いたマイグレーションテストを必ず実施し、既存データ（ユーザーや過去の `score_logs`）が一切欠損することなく無傷でバージョンアップできることを検証完了することをリリース条件とする。

- **テーブル定義 (SQL):**

```sql
-- ユーザー管理テーブル
CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,          -- '[チームID]-[ランダムID]'
    name TEXT NOT NULL,
    team_id TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- チームマスター管理テーブル
CREATE TABLE IF NOT EXISTS teams (
    id TEXT PRIMARY KEY,               -- チームID（例: 'team_a', 'red_team'）※不変識別子
    name TEXT NOT NULL,                -- チーム表示名（例: '赤組'）
    color_code TEXT NOT NULL,          -- チームカラー（例: '#FF0000'）※HEX形式
    is_active INTEGER DEFAULT 1,       -- 有効フラグ (1: 有効, 0: 無効)
    display_order INTEGER DEFAULT 0,   -- 得点ボード等での表示順序
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 監査・スコアログテーブル
CREATE TABLE IF NOT EXISTS score_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    operation_id TEXT UNIQUE NOT NULL,    -- 二重計上防止用の一意な操作ID
    revision INTEGER NOT NULL,            -- サーバー側で採番される順序・リビジョン番号
    competition_id TEXT,                  -- 競技識別子
    round_id TEXT,                        -- ラウンド識別子
    session_id TEXT,                      -- 順位入力セッションID
    user_id TEXT,                         -- 個人紐付け
    team_id TEXT NOT NULL,
    points INTEGER NOT NULL,
    rank INTEGER,                         -- 順位モード時の順位
    reason TEXT NOT NULL,                 -- 'rank_result', 'bonus', 'cheer'等
    kiosk_id TEXT,                        -- 操作を行ったキオスク端末ID（監査用）
    client_occurred_at TEXT,              -- クライアント側（オフライン時等）で発生したローカル時刻
    created_at DATETIME DEFAULT (strftime('%Y-%m-%d %H:%M:%f', 'now')), -- サーバー側の正本時刻（UTC）
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE SET NULL
);

-- 順位入力セッション管理テーブル（競技・ラウンド・セッション完全分離版）
CREATE TABLE IF NOT EXISTS rank_sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    session_id TEXT NOT NULL,             -- 順位入力セッションID
    competition_id TEXT NOT NULL,         -- 競技識別子
    round_id TEXT NOT NULL,               -- ラウンド識別子
    rank INTEGER NOT NULL,
    user_id TEXT NOT NULL,
    team_id TEXT NOT NULL,
    status TEXT NOT NULL CHECK(status IN ('DRAFT', 'FINALIZED', 'CANCELLED')),
    operation_id TEXT,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(competition_id, round_id, user_id) -- 同一競技・ラウンド内での複数順位登録を物理阻止
);

-- システム状態・表示フラグ管理テーブル
CREATE TABLE IF NOT EXISTS system_state (
    key TEXT PRIMARY KEY,                 -- 例: 'board_visibility', 'current_phase'
    value TEXT NOT NULL,
    version INTEGER NOT NULL,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- シェイク集計・ボーナス履歴テーブル
CREATE TABLE IF NOT EXISTS cheer_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id TEXT NOT NULL,
    team_id TEXT NOT NULL,
    shake_count INTEGER NOT NULL,
    window_start DATETIME NOT NULL,
    window_end DATETIME NOT NULL,
    awarded_bonus INTEGER DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

- **同時書き込み制御・トランザクション・ビジー対策:**
  - **トランザクションの原子性 (Atomicity):** 順位確定に伴う結果保存と得点記録など、複数テーブルにまたがる変更は必ず単一のトランザクション内でアトミックに実行し、途中で失敗した場合はロールバックして整合性を保つ。
  - **SQLITE_BUSY 対策とリトライ戦略:** 100人のシェイク送信やキオスクの操作が競合して `SQLITE_BUSY` が発生した場合に備え、サーバー側では以下のパラメータを持つリトライ機構を実装する：
    - 最大再試行回数（例: 5回）
    - ジッター付き指数バックオフ（Exponential Backoff with Jitter）による待機
    - タイムアウト後の明確なエラー応答（二重計上を防ぐため、冪等な `operation_id` を用いて安全に再実行できる設計とする）
- **ディスク同期モード（`synchronous = NORMAL`）の選定理由:**
  - `synchronous = NORMAL` は、電源断等で直近のごく一部のコミット済みトランザクションが失われるリスクを許容する代わりに書き込み性能を向上させる設定である。
  - 本システムの運動会データにおける耐久性要件とパフォーマンスのトレードオフを慎重に検証し、商用環境でのリスクを許容した上で採用する。また、定期的なインメモリキャッシュのダンプおよび監査ログの非同期永続化を併用する。
- **負荷試験の義務付け:** 正式稼働前に、想定接続数（約100人）での同時書き込み時におけるエラー率、応答時間、および処理件数の負荷試験を実施し、パフォーマンスの妥当性を検証する。
- **チーム設定の運用・管理および動的反映ルール:**
  - **チームIDの不変性とホスト編集権限:** `teams.id` は不変のキーとして扱い、チーム表示名（`name`）、チームカラー（`color_code`）、表示順序はホストアプリ（`master.html` / 管理画面）からリアルタイムに編集可能とする。
  - **参加者スマホ画面（QRコード背景色）への反映:** 参加者のスマホ画面およびキオスク端末は、サーバーから取得した `teams` テーブルの `color_code` を読み込み、QRコード表示エリアやヘッダー背景色に動的に適用する。これにより、現場での目視による所属チーム判別を容易にする。
- **チーム移籍と過去得点の集計ルール:**
  - 参加者（`users`）が競技途中で別チームへ移籍した場合、`users.team_id` は最新の所属チームへ更新される。
  - **得点ログ（`score_logs`）における記録方針:** スコア発生時に書き込まれる `score_logs.team_id` は、**「操作発生時点で所属していたチームID」**を固定で保持・記憶する。過去に獲得したポイントを遡って新チームへ付け替える（再分類する）ことは行わず、獲得時点のチーム成績として不変のまま集計する。
- **整合性と外部キーの運用規則:**
  - `score_logs.user_id` はチーム一括ボーナスやシステム全体の補正など、個人に紐づかない操作において NULL を許可する。ただし、個人指定の操作の場合は外部キー制約により存在しないユーザーへの紐付けを完全に排除する。
  - すべての特権操作・キオスク操作において `kiosk_id` および `created_at` の記録を義務付け、障害発生時のリカバリや監査証跡の完全性を保証する。
- **データベース起動時の整合性検証とID突合:**
  - サーバー起動時、接続先のSQLiteファイル内にメタデータテーブル（または環境変数定義）が存在するか確認し、現在稼働中のシステムID（`db_id`）と一致することを義務付ける。
  - キオスク端末や参加者QRコードに含まれる `db_id` と、サーバーが現時点で稼働しているDBのIDが一致しない場合、サーバー側はすべての得点・データ操作リクエストを拒否（400 Bad Request）し、データの分断や誤ったデータベースへの書き込みを物理的に阻止する。
- **競技・セッションの独立性と集計ルール:**
  - **イベント単位の関連付け:** すべての順位結果および競技ボーナスの得点操作には、`competition_id`（競技ID）、`round_id`（ラウンドID）、および `session_id`（セッションID）を必須で付与する。
  - **運動会全体の得点再構築:** 総合得点は独立した各競技・セッションイベント（`score_logs`）の決定論的な集計（SUM）値として算出する。
  - **スコープの限定と波及防止:** 特定の競技結果に対する訂正・打消し操作（CANCELLED）が発生した場合、該当する `competition_id` および `session_id` に紐づく得点ログのみが無効化（またはマイナス計上）される。他の競技や時間帯の得点履歴には一切影響を及ぼさないことを保証する。
- **時刻の正本と表示・順序保証のルール:**
  - **正本はUTC:** データベースに記録される `created_at` は常にUTC（ISO 8601形式またはSQLite標準形式）を正本として保持する。日本時間（JST: UTC+9）への変換は、サーバーからデータを取得した後のフロントエンド（ブラウザ・ホストアプリ）の画面描画時に動的に行う。
  - **オフライン時の時刻分離:** キオスク端末やオフライン運用時に発生した操作については、端末側の発生時刻を `client_occurred_at` として保持し、サーバーがリクエストを受け付けて確定させた実際の順序・時刻を `created_at`（サーバーUTC）として厳密に切り分けて記録する。
  - **監査上の順序確定:** 監査ログや履歴タイムラインの前後関係の判定には、各端末の時計（不正確な可能性があるローカル時刻）ではなく、サーバー側でアトミックに採番される `revision`（リビジョン番号）および `id`（インクリメントID）を絶対的な順序の基準として使用する。

### 3.4 QRコード・ペイロード仕様、改ざん検証・互換性およびデータ隔離プロトコル

システム内で生成・提示・取り込みされるすべてのQRコード（参加者QR、キオスク未送信回収用QR等）は、データ構造の検証、バージョン互換性確保、および HMAC による改ざん防止・真正性検証を必須とする。

- **参加者識別用QR（スマホ画面表示）:**

```json
{
  "v": 1,
  "type": "user",
  "db_id": "2026_sports_day",
  "user_id": "team_a-x8f2k9l1",
  "ts": 1792584000,
  "hmac": "a3f8b9c..."
}
```

`ts` (タイムスタンプ) とサーバー/システム共有鍵から生成した `hmac` を付与し、クライアント側での不正な `user_id` や `db_id` の偽造・改ざんを検証する。

- **キオスク未送信・再照会用QR（通信エラー・オフラインフォールバック用）:**

```json
{
  "v": 1,
  "type": "reconciliation",
  "db_id": "2026_sports_day",
  "kiosk_id": "kiosk-a1b2c3",
  "op_id": "550e8400-e29b-41d4-a716-446655440000",
  "payload": {
    "user_id": "team_a-x8f2k9l1",
    "points": 1,
    "reason": "rank_result"
  },
  "hmac": "e7d1f2a..."
}
```

`op_id` (一意な操作UUID) による重複排除と、`hmac` による改ざん検証を組み合わせて運用する。

- **改ざん検証・重複排除・データ隔離プロトコル:**
  - **真正性・改ざん検証 (HMAC Verification):** ホストアプリおよびサーバーは、読み取ったペイロードの HMAC 署名を検証する。署名不一致または改ざんが検知されたデータは即座に無効化する。
  - **二重取り込み・複製防止 (Replay Attack Prevention):** 署名検証をパスしたデータであっても、含まれる `op_id`（または `user_id` + 操作タイムスタンプ）を `score_logs` の既存レコードおよびローカル処理済みインデックスと照合する。すでに処理済みの `op_id` は「適用済み（スキップ）」として扱い、二重取り込み・再送攻撃を構造的に防止する。
  - **互換性判定 (Schema & Version Validation):** ペイロード読み取り時、厳格な JSON Schema による型・構造チェックおよびサポート対象のメジャーバージョン（v）チェックを行う。
  - **不正データの隔離・警告フロー (Quarantine Workflow):**
    - 「署名不正（HMAC不一致）」「未知のスキーマ・バージョン」「`db_id` 不一致」「構造崩れ」を検知したデータは、自動でデータベースにマージ（適用）することは厳禁とする。
    - 該当データは「隔離バッファ（Quarantine Vault）」へ退避させ、ホストアプリのGUI上に「⚠️ 不正/非互換データ検知」として詳細理由（例: DB_ID不一致, HMAC検証失敗, 重複操作）とともに警告表示する。
    - 管理者が手動でデータ内容を確認・裁定（破棄または強制補正適用）できる運用設計とする。