# DB設計書

## 1. 対象と保存方針

参照資料は `requirements.md` と `handout_app_spec.md`。MySQL（データを保存する仕組み）に `todos` テーブル（行と列で情報を保存する場所）を1つだけ作り、やること1件を1行に保存する。画面の詳しい設計は別の設計書で扱う。

## 2. todos のカラム一覧

カラム（保存する情報の項目）は次の9つ。`NULL` は「値が入っていない」状態、`NOT NULL` はそれを許さない指定。

| カラム | 型（値の種類） | 必須 | 既定値・自動設定 | 内容 |
|---|---|---|---|---|
| `id` | `BIGINT` | 必須 | 自動で連番 | 1件を見分ける番号。主キー（重複しない識別番号） |
| `title` | `VARCHAR(255)` | 必須 | なし | やること。255文字以内 |
| `detail` | `VARCHAR(255)` | 任意 | `NULL` | メモ。255文字以内 |
| `category` | `VARCHAR(255)` | 必須 | なし | ジャンル。指定の5種類から選ぶ |
| `priority` | `INT` | 必須 | `2` | 優先度。`1`＝高、`2`＝中、`3`＝低 |
| `due_date` | `DATE` | 任意 | `NULL` | 期限の日付 |
| `completed` | `BOOLEAN` | 必須 | `0` | 完了状態。`0`＝未完了、`1`＝完了 |
| `created_at` | `DATETIME` | 必須 | 登録時に自動設定 | 登録日時 |
| `updated_at` | `DATETIME` | 必須 | 登録時と更新時に自動設定 | 最終更新日時 |

`id`、`created_at`、`updated_at` は利用者の入力項目にしない。`priority` の保存時の既定値は `2` だが、登録画面では未選択から始め、利用者に選択を求める。`completed` は登録時に未完了とし、編集時に変更できる。

## 3. 値と入力の決まり

- `category` に保存できる値は「デザイン」「マーケティング」「プログラミング」「資格」「就職活動」の5つ。
- `title` は必須で255文字以内、`detail` は任意で255文字以内。空白だけの `title` は、半角・全角を問わず入力時に拒否する。
- `category` と `priority` は登録・編集時に選択必須。`priority` の保存値は `1`、`2`、`3` のいずれか。
- `due_date` は任意の日付。値がないときは `NULL` とする。
- これらの入力チェック（保存前に値を確かめる処理）と画面に出すエラー文はアプリ側で行う。データベース側でも `category`、`priority`、`completed` の許容値を制約（保存できる値の決まり）で守る。

## 4. CREATE TABLE 文

DDL（テーブルを作るSQL文）は次のとおり。MySQLで実行する。

```sql
CREATE TABLE todos (
    id BIGINT NOT NULL AUTO_INCREMENT,
    title VARCHAR(255) NOT NULL,
    detail VARCHAR(255) NULL,
    category VARCHAR(255) NOT NULL,
    priority INT NOT NULL DEFAULT 2,
    due_date DATE NULL,
    completed BOOLEAN NOT NULL DEFAULT FALSE,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    CONSTRAINT chk_todos_category CHECK (
        category IN ('デザイン', 'マーケティング', 'プログラミング', '資格', '就職活動')
    ),
    CONSTRAINT chk_todos_priority CHECK (priority IN (1, 2, 3)),
    CONSTRAINT chk_todos_completed CHECK (completed IN (0, 1))
) CHARACTER SET utf8mb4;
```

`AUTO_INCREMENT` は番号を自動で増やす指定。`CURRENT_TIMESTAMP` は保存時点の日時を自動で入れる指定。`ON UPDATE CURRENT_TIMESTAMP` は行を更新したときに日時を更新する指定。`CHECK` は指定外の値を保存しないための制約。

## 5. 要件とカラムの対応

| 要件・操作 | 対応カラム | 保存・参照のしかた |
|---|---|---|
| 1件を識別し、編集・削除の対象を決める | `id` | 自動採番した番号で対象行を特定する |
| やることの登録・編集・一覧表示 | `title` | 必須、255文字以内で保存する |
| メモの登録・編集 | `detail` | 任意、255文字以内で保存する |
| ジャンルの登録・編集・一覧表示・絞り込み | `category` | 指定の5種類を保存し、絞り込みでは完全一致で比べる |
| 優先度の登録・編集・一覧表示 | `priority` | `1`／`2`／`3` を保存し、表示時に高／中／低へ置き換える |
| 期限の登録・編集・一覧表示・並び替え・削除確認表示 | `due_date` | 日付を保存し、近い順／遠い順に並べる。未入力は `NULL` |
| 完了状態の編集・一覧表示 | `completed` | `0`／`1` を保存し、表示時に未完了／完了へ置き換える |
| 登録日時を記録する | `created_at` | 登録時にMySQLが自動で設定する |
| 最終更新日時を記録する | `updated_at` | 登録時と更新時にMySQLが自動で設定する |
| 削除 | `id` | 対象の行を削除する。復元用のカラムは設けない |

検索語・ジャンル・並び順はURLに保持し、検索結果を保存するカラムは設けない。操作ログ（操作の記録）はアプリのログに操作種別と `id` を1行ずつ残し、`todos` にログ用のカラムは設けない。`title` と `detail` の中身はログに残さない。

## 6. 未確定事項

資料では、期限が未入力の行を期限順のどこに置くか、同じ期限の行の順番は決まっていない。後続の設計で決める。
