# チラシ/クーポン施策一元管理モック：実装着手仕様（次アクション詳細）

## 目的
前回のロードマップを、**すぐ実装に着手できる仕様**に落とし込む。
本ドキュメントでは次の5項目を確定する。

1. 権限マトリクス（ロール×操作）
2. 状態遷移図と禁止遷移
3. 公開後変更イベントのスキーマ（payload）
4. 保存/公開バリデーション仕様
5. 監査ログ項目の最小必須セット

---

## 1) 権限マトリクス（ロール×操作）

### ロール定義
- **PLANNER（営業企画）**: 施策ヘッダ責任者。公開オペレーション担当。
- **BUYER（バイヤー）**: 商品明細責任者。商品/価格/JAN更新担当。
- **STORE_VIEWER（店舗）**: 閲覧専用。
- **ADMIN（管理者）**: 例外運用・ロック解除・全データ管理。

### 操作一覧と可否
| 操作ID | 操作 | PLANNER | BUYER | STORE_VIEWER | ADMIN |
|---|---|---:|---:|---:|---:|
| P01 | 施策作成 | ✅ | ❌ | ❌ | ✅ |
| P02 | 施策編集（公開前） | ✅ | ❌ | ❌ | ✅ |
| P03 | 施策削除（公開前のみ） | ✅ | ❌ | ❌ | ✅ |
| P04 | 対象店舗設定（store_set/store_ids） | ✅ | ❌ | ❌ | ✅ |
| P05 | 商品明細作成/編集（公開前） | ✅ | ✅ | ❌ | ✅ |
| P06 | 商品明細削除（公開前） | ✅ | ✅ | ❌ | ✅ |
| P07 | レイアウト枠編集 | ✅ | ✅ | ❌ | ✅ |
| P08 | 承認申請（承認待ちへ遷移） | ✅ | ✅ | ❌ | ✅ |
| P09 | 承認（確定へ遷移） | ❌ | ❌ | ❌ | ✅ |
| P10 | 店舗公開 | ✅ | ❌ | ❌ | ✅ |
| P11 | 公開後変更（通常） | ❌ | ❌ | ❌ | ❌ |
| P12 | 緊急変更申請（公開後） | ✅ | ✅ | ❌ | ✅ |
| P13 | 緊急変更承認/実施（公開後） | ❌ | ❌ | ❌ | ✅ |
| P14 | BY締切ロック解除 | ❌ | ❌ | ❌ | ✅ |
| P15 | 監査ログ閲覧 | ✅ | ✅ | ✅（自店関連のみ） | ✅ |
| P16 | Export/Import | ❌（本番不可） | ❌（本番不可） | ❌ | ✅（運用限定） |

### 補足ルール
- 公開後の直接編集は禁止（P11 = 全ロール不可）。
- 公開後は「緊急変更申請 → ADMIN 承認 → 反映」のみ許可。
- STORE_VIEWER は閲覧対象を自店関連データに限定。

---

## 2) 状態遷移図（施策）と禁止遷移

### ステータス定義
- `DRAFT`（企画中）
- `BUYER_INPUT`（BY入力中）
- `COMP_PENDING`（カンプ作成中）
- `PENDING_APPROVAL`（承認待ち）
- `LOCKED`（確定）
- `PUBLISHED`（店舗公開済）
- `EMERGENCY_REQUESTED`（緊急変更申請中）
- `ARCHIVED`（終了/保管）

### 正常遷移
```text
DRAFT
  -> BUYER_INPUT
  -> ARCHIVED

BUYER_INPUT
  -> COMP_PENDING
  -> DRAFT

COMP_PENDING
  -> PENDING_APPROVAL
  -> BUYER_INPUT

PENDING_APPROVAL
  -> LOCKED
  -> COMP_PENDING

LOCKED
  -> PUBLISHED
  -> COMP_PENDING   (ADMINのみ、差戻し時)

PUBLISHED
  -> EMERGENCY_REQUESTED
  -> ARCHIVED

EMERGENCY_REQUESTED
  -> PUBLISHED      (ADMIN承認で再公開)
  -> LOCKED         (却下・再調整)

ARCHIVED
  -> (遷移不可)
```

### 禁止遷移（必須制約）
- `PUBLISHED -> BUYER_INPUT`（公開済を入力中へ戻す）
- `PUBLISHED -> DRAFT`（公開済を下書きへ戻す）
- `LOCKED -> DRAFT`（確定から下書きへのジャンプ）
- `ARCHIVED -> *`（アーカイブから復帰）
- `* -> PUBLISHED` ただし `LOCKED` 経由以外は禁止

### 権限付き遷移制約
- `PENDING_APPROVAL -> LOCKED` は ADMIN のみ。
- `LOCKED -> PUBLISHED` は PLANNER または ADMIN。
- `PUBLISHED -> EMERGENCY_REQUESTED` は申請者（PLANNER/BUYER/ADMIN）。
- `EMERGENCY_REQUESTED -> PUBLISHED` は ADMIN のみ。

---

## 3) 公開後変更イベントスキーマ（payload）

### イベント名
- `promotion.post_publish_changed.v1`

### 発火条件
- 施策が `PUBLISHED` の状態で、以下のいずれかに変更が入ったとき。
  - 施策ヘッダ（対象店、発行日、版、締切、状態）
  - 商品明細（JAN、商品名、価格、掲載枠、表示順、ステータス）
  - レイアウト枠（slot_id, x/y/w/h）

### JSONスキーマ（実装用）
```json
{
  "event_id": "evt_01J...",
  "event_type": "promotion.post_publish_changed.v1",
  "occurred_at": "2026-02-11T10:15:30+09:00",
  "actor": {
    "user_id": "u_123",
    "role": "PLANNER",
    "name": "営業企画A"
  },
  "promotion": {
    "promo_id": "P-20260214-001",
    "campaign": "バレンタイン特集",
    "edition": "A版",
    "issue_date": "2026-02-14",
    "published_at": "2026-02-10T18:00:00+09:00"
  },
  "change_summary": {
    "severity": "HIGH",
    "changed_fields": [
      "items.price",
      "items.slot"
    ],
    "reason": "仕入条件変更"
  },
  "diffs": [
    {
      "entity": "item",
      "entity_id": "I-00045",
      "field": "price",
      "before": 298,
      "after": 278
    }
  ],
  "impact": {
    "store_scope": {
      "store_set": "道東A",
      "store_ids": ["S004", "S005", "S006"]
    },
    "affected_store_count": 3,
    "effective_from": "2026-02-12T09:00:00+09:00"
  },
  "approval": {
    "flow": "EMERGENCY_CHANGE",
    "requested": true,
    "approved_by": "u_admin_01",
    "approved_at": "2026-02-11T10:20:00+09:00"
  },
  "trace": {
    "request_id": "req-abc123",
    "source": "web-app"
  }
}
```

### 配信要件
- 遅延許容: 発火から**5分以内に通知作成**。
- 冪等性: `event_id` で重複排除。
- 再送: 失敗時は指数バックオフで3回。

---

## 4) 保存/公開バリデーション仕様

### A. 保存時バリデーション（下書き保存）
| コード | ルール | レベル |
|---|---|---|
| V001 | `promo_id` 必須・一意 | ERROR |
| V002 | `kind/campaign/issue_date/store_set` 必須 | ERROR |
| V003 | `store_set=特定10店` の場合 `store_ids` 1件以上必須 | ERROR |
| V004 | `buyer_deadline <= issue_date` | ERROR |
| V005 | `slots >= 1` | ERROR |
| V006 | 商品明細の `jan` は13桁数字 | ERROR |
| V007 | 価格は 0 以上の整数 | ERROR |
| V008 | `slot` がレイアウト枠IDに存在しない場合 | WARN |

### B. 公開前バリデーション（公開ボタン押下時）
| コード | ルール | レベル |
|---|---|---|
| P001 | ステータスが `LOCKED` であること | ERROR |
| P002 | 未承認の緊急変更申請が存在しないこと | ERROR |
| P003 | 対象商品が1件以上あること | ERROR |
| P004 | 全商品に `jan/product/price/slot` があること | ERROR |
| P005 | 商品 `item_status=NG` が残っていないこと | ERROR |
| P006 | レイアウト重複（枠重なり率閾値超え）がないこと | WARN |
| P007 | BY締切超過後の編集履歴がある場合、管理者承認ログ必須 | ERROR |

### C. 締切ガード
- `now > buyer_deadline` の場合:
  - PLANNER/BUYER の編集不可（UI非活性 + API拒否）
  - ADMIN のみ `lock_override=true` 指定で更新可
- override 実行時は監査ログに理由必須

---

## 5) 監査ログ項目の最小必須セット

### 必須フィールド
| 項目 | 型 | 必須 | 説明 |
|---|---|---:|---|
| log_id | string | ✅ | 監査ログID（UUID推奨） |
| ts | datetime | ✅ | 発生時刻（ISO8601, TZ付き） |
| actor_user_id | string | ✅ | 実行ユーザーID |
| actor_role | enum | ✅ | 実行時ロール |
| action | enum | ✅ | CREATE/UPDATE/DELETE/PUBLISH/APPROVE/REJECT/OVERRIDE |
| entity | enum | ✅ | promotion/item/layout/approval/notification |
| entity_id | string | ✅ | 対象レコードID |
| promo_id | string | ✅ | 施策ID（横断検索キー） |
| before_json | json | △ | 変更前（UPDATE/DELETE時は必須） |
| after_json | json | △ | 変更後（CREATE/UPDATE時は必須） |
| reason | string | △ | 変更理由（公開後変更・override時必須） |
| request_id | string | ✅ | API追跡ID |
| source_ip | string | ✅ | 送信元IP |
| user_agent | string | ✅ | クライアント識別 |

### 保持・参照要件
- 保持期間: 最低13か月。
- 検索軸: `promo_id`, `entity`, `actor_user_id`, 期間。
- エクスポート: CSV/JSON（改ざん防止のためハッシュ付与推奨）。

---

## 実装順（2スプリント想定）

### Sprint 1
1. RBAC実装（APIガード + UI非活性）
2. 状態遷移ガード（禁止遷移のサーバー側拒否）
3. 保存/公開バリデーション（ERROR/WARN分離）

### Sprint 2
1. 公開後変更イベント発火 + 通知基盤接続
2. 監査ログの必須項目拡張（before/after/reason/request_id）
3. 運用ダッシュボードに「公開後変更件数」を追加

---

## 完了判定（DoD）
- 権限外操作は UI/API の両方で拒否される。
- 禁止遷移を全ケースで拒否できる自動テストが通る。
- 公開後変更でイベントが発火し、5分以内に通知キューへ投入される。
- 監査ログから任意変更の before/after と実行者が追跡できる。
