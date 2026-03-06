# `.kiro` 仕様検証レポート

**検証日**: 2026-03-06
**対象仕様**: `tenant-management-backend`

## 対象ファイル

- `.kiro/specs/tenant-management-backend/requirements.md` — 要件定義（14要件、非機能要件含む）
- `.kiro/specs/tenant-management-backend/design.md` — 技術設計（DDD、C#/.NET 8）

---

## 重大な問題

### 1. グローバルクエリフィルターの実装バグ（要件7.1, 7.4）

`TenantDbContext.OnModelCreating` で:

```csharp
var tenantId = Expression.Constant(_tenantContext.TenantId); // 問題箇所
```

`OnModelCreating` は DbContext **初期化時に一度だけ**実行されます。`Expression.Constant()` に渡す値はその時点でキャプチャされるため、リクエストごとに変わる `TenantId` が正しく反映されません。

**影響**: 他テナントのデータが返される可能性があり、**データ漏洩につながる重大なセキュリティリスク**です。

**修正方針**: `Expression.Constant()` ではなく、ランタイムに評価されるクロージャーを使用する。

```csharp
// 修正例
modelBuilder.Entity(entityType.ClrType).HasQueryFilter(
    e => EF.Property<Guid>(e, nameof(ITenantEntity.TenantId)) == _tenantContext.TenantId
);
```

---

### 2. トランザクション二重コミット問題（要件13.1〜13.3）

`TenantApplicationService.CreateTenantAsync` で:

```csharp
await _unitOfWork.CommitAsync(cancellationToken);        // テナント保存
await _auditService.LogAsync(..., cancellationToken);    // 内部でも CommitAsync を呼ぶ
```

`AuditService.LogAsync` 内でも `_unitOfWork.CommitAsync` を呼んでいます。

**影響**:
- テナント保存と監査ログ記録が**別トランザクション**になる
- テナント保存成功後に監査ログ失敗 → データ整合性が破れる

**修正方針**: 監査ログ記録を同一トランザクション内で行うか、アウトボックスパターンを採用する。

---

## 設計上の不整合

### 3. 要件3.3（テナント識別子の変更不可）の実装漏れ

要件「テナント識別子の変更を許可しない」に対して、設計には `UpdateName()` メソッドしかなく、識別子変更を試みた場合のエラー処理が未定義です。`UpdateTenantCommand` に識別子フィールドが含まれる場合のバリデーションが設計に記載されていません。

---

### 4. テナント管理API自体のテナント解決問題（要件5.1〜5.6との矛盾）

`TenantResolverMiddleware` はすべてのリクエストに適用されますが、テナント作成API（POST `/tenants`）自体はどのテナントに属するのかが不明です。

**影響**: システム管理者エンドポイントはミドルウェアをバイパスする仕組みが設計に存在しないため、テナント未作成の初期状態でAPIが呼び出せません。

---

### 5. AuditLog の不変性保証が不完全（要件13.5, 13.6）

要件「監査ログを変更不可能な形式で保存する」「削除を許可しない」と定義されています。しかし:

- `IAuditLogRepository` は `IRepository<AuditLog>` を継承しており、基底の `IRepository` に `UpdateAsync` や `DeleteAsync` が含まれている場合、不変性が破られる可能性がある
- DbContext レベルでの書き込み制御インターセプターの設計が不足

---

### 6. 複数テナント所属時のJWT設計（要件8.3）

要件8.3で「複数テナント所属を考慮する」としていますが、`JwtTokenGenerator.GenerateToken(ApplicationUser user, Tenant tenant)` は単一テナントを受け取る設計のみです。

**影響**: 複数テナントユーザーがどのテナントでトークンを発行するかのシナリオが未設計です。

---

## 軽微な問題・改善点

### 7. Property の Validates 参照が不正確

Design.md の Property 34〜40（監査ログ関連）が `"Audit Requirements"` と参照しており、具体的な要件番号（13.x, 14.x）を指定していません。他のプロパティとの一貫性がありません。

---

### 8. AuditLog テーブルの FK 設計

データモデル上 `AuditLog.TenantId` と `AuditLog.UserId` が FK として記載されていますが、テナント無効化後もログは残すべきです。

**修正方針**: FK 制約を `ON DELETE RESTRICT` にするか、参照整合性を持たない設計（FK なし）にすべきです。

---

### 9. SubdomainTenantResolver の境界値ロジック

```csharp
var parts = host.Split('.');
if (parts.Length < 2)
    return Task.FromResult(new TenantResolutionResult(false, null, "Invalid subdomain"));
```

`localhost` や IP アドレスへのアクセス時に誤動作する可能性があります。また `parts[0]` が空文字列になるケース（先頭がドット）の考慮が不足しています。

---

## 総評

| カテゴリ | 評価 |
|---------|------|
| 要件定義の網羅性 | 良好（14要件、非機能要件も定義済み） |
| DDD設計の質 | 概ね良好（集約、値オブジェクト、ドメインイベントが適切に分離） |
| 要件↔設計のトレーサビリティ | 良好（40のPropertyで追跡） |
| 実装上のリスク | **重大なバグ2件あり**（クエリフィルター、トランザクション） |
| セキュリティ | グローバルクエリフィルターのバグはデータ漏洩リスク |

## 対応優先度

| 優先度 | 問題 |
|--------|------|
| 最優先 | 1. グローバルクエリフィルターの実装バグ（データ漏洩リスク） |
| 最優先 | 2. トランザクション二重コミット問題（データ整合性リスク） |
| 高 | 4. テナント管理APIのテナント解決問題 |
| 高 | 5. AuditLog の不変性保証 |
| 中 | 3. テナント識別子変更不可のバリデーション実装漏れ |
| 中 | 6. 複数テナント所属時のJWT設計 |
| 低 | 7. Property の Validates 参照の不正確さ |
| 低 | 8. AuditLog テーブルの FK 設計 |
| 低 | 9. SubdomainTenantResolver の境界値ロジック |
