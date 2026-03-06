# `.kiro` 仕様検証レポート

**検証日**: 2026-03-06
**対象仕様**: `tenant-management-backend`
**最終更新**: 2026-03-06

## 対象ファイル

- `.kiro/specs/tenant-management-backend/requirements.md` — 要件定義（14要件、非機能要件含む）
- `.kiro/specs/tenant-management-backend/design.md` — 技術設計（DDD、C#/.NET 8）

---

## 重大な問題

### ✅ 1. グローバルクエリフィルターの実装バグ（要件7.1, 7.4） - 修正済み

~~`TenantDbContext.OnModelCreating` で:~~

```csharp
// 修正前（問題あり）
var tenantId = Expression.Constant(_tenantContext.TenantId); // 問題箇所
```

~~`OnModelCreating` は DbContext **初期化時に一度だけ**実行されます。`Expression.Constant()` に渡す値はその時点でキャプチャされるため、リクエストごとに変わる `TenantId` が正しく反映されません。~~

~~**影響**: 他テナントのデータが返される可能性があり、**データ漏洩につながる重大なセキュリティリスク**です。~~

**修正内容**: ランタイムに評価されるクロージャーを使用するヘルパーメソッド `GetTenantIdFilter<TEntity>` を実装しました。

```csharp
// 修正後
private static LambdaExpression GetTenantIdFilter<TEntity>(ITenantContext tenantContext)
    where TEntity : class, ITenantEntity
{
    Expression<Func<TEntity, bool>> filter = e => e.TenantId == tenantContext.TenantId.Value;
    return filter;
}
```

---

### ✅ 2. トランザクション二重コミット問題（要件13.1〜13.3） - 修正済み

~~`TenantApplicationService.CreateTenantAsync` で:~~

```csharp
// 修正前（問題あり）
await _unitOfWork.CommitAsync(cancellationToken);        // テナント保存
await _auditService.LogAsync(..., cancellationToken);    // 内部でも CommitAsync を呼ぶ
```

~~`AuditService.LogAsync` 内でも `_unitOfWork.CommitAsync` を呼んでいます。~~

~~**影響**:~~
~~- テナント保存と監査ログ記録が**別トランザクション**になる~~
~~- テナント保存成功後に監査ログ失敗 → データ整合性が破れる~~

**修正内容**: 
1. `TenantApplicationService` 内で監査ログを直接作成し、同一トランザクション内で保存
2. `AuditService.LogAsync` からコミット処理を削除（トランザクション境界は呼び出し元が制御）

```csharp
// 修正後
await _tenantRepository.AddAsync(tenant, cancellationToken);
var auditLog = CreateAuditLog(...);
await _auditLogRepository.AddAsync(auditLog, cancellationToken);
await _unitOfWork.CommitAsync(cancellationToken); // 一度だけコミット
```

---

## 設計上の不整合

### ✅ 3. 要件3.3（テナント識別子の変更不可）の実装漏れ - 修正済み

~~要件「テナント識別子の変更を許可しない」に対して、設計には `UpdateName()` メソッドしかなく、識別子変更を試みた場合のエラー処理が未定義です。`UpdateTenantCommand` に識別子フィールドが含まれる場合のバリデーションが設計に記載されていません。~~

**修正内容**: `Tenant` エンティティに `UpdateIdentifier()` メソッドを追加し、常に `DomainException` をスローするように実装しました。

```csharp
public void UpdateIdentifier(TenantIdentifier newIdentifier)
{
    // 要件3.3: テナント識別子の変更を許可しない
    throw new DomainException("Tenant identifier cannot be changed after creation");
}
```

---

### ✅ 4. テナント管理API自体のテナント解決問題（要件5.1〜5.6との矛盾） - 修正済み

~~`TenantResolverMiddleware` はすべてのリクエストに適用されますが、テナント作成API（POST `/tenants`）自体はどのテナントに属するのかが不明です。~~

~~**影響**: システム管理者エンドポイントはミドルウェアをバイパスする仕組みが設計に存在しないため、テナント未作成の初期状態でAPIが呼び出せません。~~

**修正内容**: `TenantResolverMiddleware` にバイパスパスのリストを追加し、システム管理者エンドポイント（`/api/tenants`, `/api/system`, `/health`, `/swagger`）はテナント解決をスキップするように実装しました。

```csharp
private static readonly HashSet<string> _bypassPaths = new(StringComparer.OrdinalIgnoreCase)
{
    "/api/tenants",
    "/api/system",
    "/health",
    "/swagger"
};
```

---

### ✅ 5. AuditLog の不変性保証が不完全（要件13.5, 13.6） - 修正済み

~~要件「監査ログを変更不可能な形式で保存する」「削除を許可しない」と定義されています。しかし:~~

~~- `IAuditLogRepository` は `IRepository<AuditLog>` を継承しており、基底の `IRepository` に `UpdateAsync` や `DeleteAsync` が含まれている場合、不変性が破られる可能性がある~~
~~- DbContext レベルでの書き込み制御インターセプターの設計が不足~~

**修正内容**:
1. `IAuditLogRepository` から `IRepository<AuditLog>` の継承を削除し、読み取り専用メソッドのみを提供
2. `TenantDbContext.SaveChangesAsync` に監査ログの変更・削除を検出して例外をスローする処理を追加

```csharp
// 監査ログの不変性を保証（要件13.5, 13.6）
foreach (var entry in ChangeTracker.Entries<AuditLog>())
{
    if (entry.State == EntityState.Modified)
        throw new InvalidOperationException("Audit logs are immutable and cannot be modified");
    if (entry.State == EntityState.Deleted)
        throw new InvalidOperationException("Audit logs cannot be deleted");
}
```

---

### ✅ 6. 複数テナント所属時のJWT設計（要件8.3） - 修正済み

~~要件8.3で「複数テナント所属を考慮する」としていますが、`JwtTokenGenerator.GenerateToken(ApplicationUser user, Tenant tenant)` は単一テナントを受け取る設計のみです。~~

~~**影響**: 複数テナントユーザーがどのテナントでトークンを発行するかのシナリオが未設計です。~~

**修正内容**: `JwtTokenGenerator` に以下のメソッドを追加しました：
1. `GenerateTokenForMultiTenant()` - 複数テナント所属ユーザー用
2. `GenerateSwitchTenantToken()` - テナント切り替え用

```csharp
public string GenerateTokenForMultiTenant(ApplicationUser user, Tenant primaryTenant, IEnumerable<Tenant> additionalTenants)
{
    // プライマリテナントと追加テナント情報をクレームに含める
}
```

---

## 軽微な問題・改善点

### ✅ 7. Property の Validates 参照が不正確 - 修正済み

~~Design.md の Property 34〜40（監査ログ関連）が `"Audit Requirements"` と参照しており、具体的な要件番号（13.x, 14.x）を指定していません。他のプロパティとの一貫性がありません。~~

**修正内容**: すべての監査ログ関連プロパティに具体的な要件番号を追加しました。

- Property 34: Requirements 13.1, 13.2, 13.3
- Property 35: Requirements 13.4
- Property 36: Requirements 14.1
- Property 37: Requirements 14.2
- Property 38: Requirements 14.3
- Property 39: Requirements 14.4
- Property 40: Requirements 13.5, 13.6

---

### ✅ 8. AuditLog テーブルの FK 設計 - 修正済み

~~データモデル上 `AuditLog.TenantId` と `AuditLog.UserId` が FK として記載されていますが、テナント無効化後もログは残すべきです。~~

~~**修正方針**: FK 制約を `ON DELETE RESTRICT` にするか、参照整合性を持たない設計（FK なし）にすべきです。~~

**修正内容**: AuditLogsテーブルの設計を更新し、TenantIdとUserIdは外部キー制約を持たない設計に変更しました。テーブル定義に注記を追加しました。

---

### ✅ 9. SubdomainTenantResolver の境界値ロジック - 修正済み

~~```csharp
var parts = host.Split('.');
if (parts.Length < 2)
    return Task.FromResult(new TenantResolutionResult(false, null, "Invalid subdomain"));
```~~

~~`localhost` や IP アドレスへのアクセス時に誤動作する可能性があります。また `parts[0]` が空文字列になるケース（先頭がドット）の考慮が不足しています。~~

**修正内容**: 以下の改善を実装しました：
1. IPアドレス判定メソッド `IsIpAddress()` を追加
2. `localhost` の明示的なチェック
3. 最低3パート（subdomain.domain.tld）の検証
4. サブドメインが空文字列の場合のチェック

---

## 総評

| カテゴリ | 評価 |
|---------|------|
| 要件定義の網羅性 | 良好（14要件、非機能要件も定義済み） |
| DDD設計の質 | 良好（集約、値オブジェクト、ドメインイベントが適切に分離） |
| 要件↔設計のトレーサビリティ | 良好（40のPropertyで追跡） |
| 実装上のリスク | ✅ すべての重大なバグを修正済み |
| セキュリティ | ✅ データ漏洩リスクを解消 |

## 修正完了状況

| 優先度 | 問題 | 状態 |
|--------|------|------|
| 最優先 | 1. グローバルクエリフィルターの実装バグ | ✅ 修正完了 |
| 最優先 | 2. トランザクション二重コミット問題 | ✅ 修正完了 |
| 高 | 4. テナント管理APIのテナント解決問題 | ✅ 修正完了 |
| 高 | 5. AuditLog の不変性保証 | ✅ 修正完了 |
| 中 | 3. テナント識別子変更不可のバリデーション実装漏れ | ✅ 修正完了 |
| 中 | 6. 複数テナント所属時のJWT設計 | ✅ 修正完了 |
| 低 | 7. Property の Validates 参照の不正確さ | ✅ 修正完了 |
| 低 | 8. AuditLog テーブルの FK 設計 | ✅ 修正完了 |
| 低 | 9. SubdomainTenantResolver の境界値ロジック | ✅ 修正完了 |

**すべての指摘事項を修正しました。設計ドキュメントは本番実装に適した状態になっています。**
