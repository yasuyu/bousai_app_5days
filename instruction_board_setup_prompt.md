# 指示・発信ボード機能 統合プロンプト

## 概要
このプロンプトは、Flask アプリケーションに「指示・発信ボード」機能を完全に実装するための統合指示です。住民向けの緊急指示や情報発信を ID ベースで登録・管理・表示できるシステムです。

---

## 実装要件

### 1. バックエンド実装（app.py）

#### 1.1 必要な関数の追加

**Jinja2 グローバル関数の登録**
```python
# app = Flask(...) の直後に以下を追加
app.jinja_env.globals.update(max=max, min=min)
```

**日本時間取得関数**
```python
def get_japan_time():
    """日本時間（JST）の現在時刻を取得する"""
    JST = timezone(timedelta(hours=9))
    return datetime.now(JST).strftime("%Y年%m月%d日 %H:%M")
```

**指示ボードデータ保存関数**
```python
INSTRUCTIONS_FILE = os.path.join(APP_DIR, 'data', 'instructions.json')

def save_instructions():
    """指示ボードのデータをファイルに保存する"""
    try:
        with open(INSTRUCTIONS_FILE, 'w', encoding='utf-8') as f:
            json.dump(instructions, f, ensure_ascii=False, indent=2)
    except Exception:
        pass
```

**ログイン認証デコレータ**
```python
from functools import wraps

def login_required(f):
    """認証が必要なページに付けるデコレータ"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        if not session.get('logged_in'):
            return redirect(url_for('login', next=request.url))
        return f(*args, **kwargs)
    return decorated_function
```

#### 1.2 グローバル変数の初期化

```python
# JSON ファイルのパス定義
INSTRUCTIONS_FILE = os.path.join(APP_DIR, 'data', 'instructions.json')

# 日本標準時 (JST) の定義
JST = timezone(timedelta(hours=9))

# 指示ボードデータをメモリにロード
def load_json(path, default):
    """JSONファイルを読み込む（存在しない・壊れている場合は default を返す）"""
    try:
        with open(path, encoding='utf-8') as f:
            return json.load(f)
    except (FileNotFoundError, json.JSONDecodeError):
        return default

instructions = load_json(INSTRUCTIONS_FILE, [])
```

#### 1.3 ボード管理ルート `/board`

**ルートハンドラー関数 `board()`**

以下の機能を実装します：

- **GET リクエスト**: 住民向けの指示を表示
  - フィルター機能（日付で絞込）
  - ページネーション（1ページ 5 件）
  - 新しい順にソート
  
- **POST リクエスト**: 指示の追加・更新・削除
  - `action` パラメータで操作を判定（`add`, `update`, `delete`）
  - ID は必須（数字チェック）
  - 追加・更新時はコンテンツも必須
  - タイムスタンプ自動記録

**実装ポイント**：
```python
@app.route('/board', methods=['GET', 'POST'])
@login_required
def board():
    if request.method == 'POST':
        action = request.form.get('action', 'add')
        instruction_id = request.form.get('id', '').strip()
        content = request.form.get('content', '').strip()
        page = int(request.form.get('page', 1))
        
        # バリデーション
        if not instruction_id:
            # エラーメッセージをテンプレートに返す
            pass
        
        if action == 'delete':
            # ID（整数）で削除処理
            # 住民向け（target == '住民'）のデータのみ対象
            pass
        
        elif action in ['add', 'update']:
            # コンテンツのバリデーション
            # 既存 ID なら更新、新規なら追加
            # タイムスタンプを自動設定
            pass
    
    # GETリクエスト
    # 住民向けデータをフィルター、ソート、ページ分割
    # 過去の履歴用に日付リストを作成
```

---

### 2. テンプレート実装（templates/board.html）

#### 2.1 構造

```html
{% extends "base.html" %}

{% block title %}指示・発信ボード - 防災アプリ{% endblock %}

{% block content %}
<!-- スタイル定義 -->
<!-- ページコンテナ -->
<div class="board-container">
  <!-- ページタイトル -->
  <!-- ナビゲーションタブ -->
  <!-- 説明文 -->
  
  <!-- メッセージ表示（成功/エラー） -->
  
  <!-- フォーム：指示登録・更新・削除 -->
  <!-- 履歴テーブル -->
  <!-- ページネーション -->
</div>

<script>
<!-- JavaScript コード -->
</script>
{% endblock %}
```

#### 2.2 主要セクション

**1. ページタイトルと説明**
- タイトル: `⑤指示・発信ボード`
- 説明: `住民向けの発信を登録・確認できます`

**2. ナビゲーションタブ**
- ホーム → `/`
- 避難所検索 → `/shelter_search`
- 避難所登録 → `/shelter_register`
- 指示・発信ボード（アクティブ）

**3. フォームセクション**
- ID 入力フィールド（必須）
  - プレイスホルダー: `例: 101`
- コンテンツ（内容）入力フィールド（テキストエリア）
  - プレイスホルダー: `避難指示や重要な情報を入力`
  - 最小高さ: 100px
- ボタン
  - 登録/更新ボタン: `name="action" value="add"`
  - 削除ボタン: `name="action" value="delete"`

**4. 過去の履歴セクション**
- セクションタイトル: `過去の履歴`
- 日付フィルター
  - ドロップダウン（`<select>`）ですべての日付を表示
  - 現在選択されている日付をハイライト
- 履歴テーブル
  - カラム: 内容、時刻、ID
  - データ無し時: `登録されている発信はありません`
- ページネーション
  - 表示ページ数: 最大 7 ページ分（現在ページの ±2）
  - 状態: 最初・前・ページ番号・次・最後
  - ページ情報表示: `全 X 件中 Y ～ Z 件を表示`

#### 2.3 CSS スタイル

- ボード容器: `max-width: 1000px`
- 形式
  - ボード色: 青系 (`#1e5fa0`)
  - 背景色: ライトグレー (`#f5f5f5`, `#f9f9f9`)
  - ボーダー: `#ddd`, `#e0e0e0`
- ボタンスタイル
  - 登録/更新: 青色
  - 削除: 赤色 (`#d9534f`)
- テーブル: 標準的なホバー効果付き

#### 2.4 JavaScript 機能

**日付フィルター関数**
```javascript
function filterByDate(date) {
  if (date) {
    window.location.href = '?date=' + date;
  } else {
    window.location.href = '?';
  }
}
```

---

### 3. データファイル（`bousai_app/data/instructions.json`）

#### 3.1 ファイル構造

JSON 配列形式で、以下のスキーマに従うオブジェクトを格納：

```json
[
  {
    "id": <integer>,
    "target": "住民",
    "content": "<string>",
    "created_at": "<YYYY年MM月DD日 HH:MM 形式>",
    "updated_at": "<YYYY年MM月DD日 HH:MM 形式>"
  }
]
```

#### 3.2 注意事項

- `id`: 一意の整数（ユーザーが入力）
- `target`: "住民" に固定
- `content`: テキスト内容（複数行対応）
- `created_at`: 初回作成時刻（自動記録、変更不可）
- `updated_at`: 最終更新時刻（自動記録、毎回更新）

---

### 4. 機能仕様

#### 4.1 表示機能

1. **データフィルタリング**
   - `target == "住民"` のデータのみ表示
   
2. **ソート**
   - `updated_at`（または `created_at`）で降順（新しい順）

3. **ページネーション**
   - 1 ページ 5 件
   - 最大ページ数の計算: `(total + per_page - 1) // per_page`
   - ページ番号は現在ページの ±2 範囲を表示

4. **日付フィルター**
   - 年月単位（例: `2026年09月`）
   - フィルター用の日付リストは重複なし
   - `updated_at` または `created_at` の日付部分を使用

#### 4.2 追加/更新機能

1. **バリデーション**
   - ID: 必須、整数
   - コンテンツ: 必須、空白不可

2. **処理ロジック**
   - ID が既に存在 → 更新（`updated_at` を現在時刻に）
   - ID が存在しない → 新規追加
   - タイムスタンプ: 日本時間で自動記録

3. **エラーメッセージ**
   - ID 未入力: `IDを入力してください。`
   - コンテンツ未入力: `内容を入力してください。`
   - ID が整数でない: `IDは数字で入力してください。`

4. **成功メッセージ**
   - `データを登録・更新しました。`

#### 4.3 削除機能

1. **バリデーション**
   - ID: 必須、整数

2. **処理ロジック**
   - ID と `target == "住民"` の組み合わせで検索
   - マッチしたレコードを削除

3. **メッセージ**
   - 成功: `ID X のデータを削除しました。`
   - 失敗: `ID X が見つかりません。`
   - 整数エラー: `IDは数字で入力してください。`

---

### 5. 統合手順

#### ステップ 1: バックエンド準備
1. `app.py` に必要な import を追加（`datetime`, `timezone`, `timedelta`）
2. グローバル変数（`JST`, `INSTRUCTIONS_FILE`, `instructions`）を設定
3. ヘルパー関数を実装（`get_japan_time`, `save_instructions`, `login_required`）
4. Jinja2 グローバル関数を登録

#### ステップ 2: ルート実装
1. `/board` ルートを実装
2. GET/POST ロジックを完成させる
3. バリデーション、エラーハンドリングを追加

#### ステップ 3: テンプレート実装
1. `templates/board.html` を作成
2. HTML 構造、CSS スタイルを実装
3. Jinja2 テンプレート変数を正しく配置

#### ステップ 4: データファイル準備
1. `bousai_app/data/instructions.json` を作成
2. 初期データ（またはロード機能）を実装

#### ステップ 5: テスト
1. ブラウザで `/board` にアクセス
2. 追加・更新・削除機能を確認
3. ページネーション、フィルター機能を確認

---

## 設定確認チェックリスト

- [ ] `app.py` に `timezone`, `timedelta`, `JST` が定義されている
- [ ] `get_japan_time()` が実装されている
- [ ] `save_instructions()` が実装されている
- [ ] `login_required` デコレータが実装されている
- [ ] Jinja2 グローバル関数（`max`, `min`）が登録されている
- [ ] `/board` ルートが GET/POST に対応している
- [ ] `templates/board.html` が完全に実装されている
- [ ] `bousai_app/data/instructions.json` が存在する
- [ ] ページネーション、フィルター機能が動作する
- [ ] エラーメッセージが正しく表示される

---

## 主な技術スタック

- **バックエンド**: Flask
- **テンプレート**: Jinja2
- **データストレージ**: JSON ファイル
- **スタイリング**: CSS（インライン + スタイルブロック）
- **実行環境**: Python 3.x、日本時間対応

---

## 注意事項

1. **ファイルパス**: `bousai_app/data/` ディレクトリが存在することを確認
2. **エンコーディング**: すべてのファイルを UTF-8 で保存
3. **セッション**: `session['logged_in']` がセットされていることを前提
4. **タイムゾーン**: 日本時間（JST）固定
5. **ページネーション**: テンプレート内で `max()`, `min()` を使用するため、Jinja2 グローバル関数の登録が必須
