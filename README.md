# custom-defender-for-storage-resource-audit

Azure Policy カスタム定義 — 各 Storage アカウントに対してリソース単位の **Defender for Storage** が有効化されていることを監査します。

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisashin0728%2FCheck-DefenderForStorage--ResourceLevel%2Frefs%2Fheads%2Fmain%2Fazuredeploy.json)

---

## 概要

| 項目 | 値 |
|---|---|
| ポリシー名 | `custom-defender-for-storage-resource-audit` |
| 表示名 | `[Custom] Defender for Storage - Resource Level Audit` |
| カテゴリ | Security Center |
| バージョン | 1.0.0 |
| モード | All |
| Effect | AuditIfNotExists（デフォルト） / Disabled |
| 対象リソース | `Microsoft.Storage/storageAccounts` |
| チェック対象 | `Microsoft.Security/defenderForStorageSettings` |

---

## 動作

```
Storage アカウントが存在する
  └─ defenderForStorageSettings.isEnabled == true か？
       ├─ YES → Compliant
       └─ NO  → NonCompliant（AuditIfNotExists）
```

各 Storage アカウントに対して、リソース単位の Defender for Storage（`isEnabled: true`）が設定されているかを評価します。  
`evaluationDelay: AfterProvisioning` により、リソースプロビジョニング完了後に評価されます。

> **注意:** このポリシーはリソース単位の設定のみを評価します。サブスクリプションレベルの Defender for Storage プラン（`Microsoft.Security/pricings/StorageAccounts`）は評価対象外です。サブスクリプションレベルのチェックには [`Defender_StorageAccounts_Audit`](#関連ポリシー) を併用してください。

---

## パラメーター

| パラメーター | 型 | 許容値 | デフォルト | 説明 |
|---|---|---|---|---|
| `effect` | String | `AuditIfNotExists`, `Disabled` | `AuditIfNotExists` | ポリシーの効果。`Disabled` にするとポリシーが無効化されます。 |

---

## デプロイ方法

### ARM テンプレートによるデプロイ（推奨）

#### Deploy to Azure ボタン

以下のリンクをブラウザで開くと、Azure Portal から直接デプロイできます。

```
https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fhisashin0728%2FCheck-DefenderForStorage--ResourceLevel%2Frefs%2Fheads%2Fmain%2Fazuredeploy.json
```

#### Azure CLI によるデプロイ

```powershell
$mgId = "<Tenant Root Group ID>"  # 例: 464ded72-1950-4150-8811-b47ceb718344

az deployment mg create `
  --name "deploy-defender-storage-policy" `
  --management-group-id $mgId `
  --location japaneast `
  --template-file azuredeploy.json `
  --parameters azuredeploy.parameters.json
```

#### パラメーター

| パラメーター | 型 | デフォルト | 説明 |
|---|---|---|---|
| `policyEffect` | string | `AuditIfNotExists` | ポリシーの Effect（`AuditIfNotExists` / `Disabled`） |
| `assignPolicy` | bool | `true` | `true` にすると、定義作成と同時に管理グループへの割り当ても実施 |

#### ファイル構成

```
azuredeploy.json              # ARM テンプレート本体
azuredeploy.parameters.json   # パラメーターファイル
```

---

### Azure CLI（JSON 直接指定）によるデプロイ

### 前提条件

- Azure CLI がインストール済みであること
- ポリシー定義を作成する権限（`Microsoft.Authorization/policyDefinitions/write`）があること
- 管理グループ または サブスクリプションへのアクセス権があること

### Tenant Root Group（推奨）

```powershell
$mgId = "<Tenant Root Group ID>"

# policyRule と parameters を抽出
$policy = Get-Content "combined-policy.json" -Raw | ConvertFrom-Json
$policy.properties.policyRule | ConvertTo-Json -Depth 10 | Out-File "$env:TEMP\policy-rule.json" -Encoding utf8
$policy.properties.parameters | ConvertTo-Json -Depth 10 | Out-File "$env:TEMP\policy-params.json" -Encoding utf8

# ポリシー定義を管理グループに作成
az policy definition create `
  --name "custom-defender-for-storage-resource-audit" `
  --rules "$env:TEMP\policy-rule.json" `
  --params "$env:TEMP\policy-params.json" `
  --mode All `
  --display-name "[Custom] Defender for Storage - Resource Level Audit" `
  --description "各 Storage Account に対して、Defender for Storage がリソース単位（isEnabled: true）で有効化されていることを監査します。" `
  --management-group $mgId

# ポリシーを Tenant Root Group に割り当て
az policy assignment create `
  --name "dfs-resource-audit" `
  --display-name "[Custom] Defender for Storage - Resource Level Audit" `
  --policy "/providers/Microsoft.Management/managementGroups/$mgId/providers/Microsoft.Authorization/policyDefinitions/custom-defender-for-storage-resource-audit" `
  --scope "/providers/Microsoft.Management/managementGroups/$mgId" `
  --enforcement-mode Default
```

### サブスクリプション単位

```powershell
$subscriptionId = "<Subscription ID>"

$policy = Get-Content "combined-policy.json" -Raw | ConvertFrom-Json
$policy.properties.policyRule | ConvertTo-Json -Depth 10 | Out-File "$env:TEMP\policy-rule.json" -Encoding utf8
$policy.properties.parameters | ConvertTo-Json -Depth 10 | Out-File "$env:TEMP\policy-params.json" -Encoding utf8

az policy definition create `
  --name "custom-defender-for-storage-resource-audit" `
  --rules "$env:TEMP\policy-rule.json" `
  --params "$env:TEMP\policy-params.json" `
  --mode All `
  --display-name "[Custom] Defender for Storage - Resource Level Audit" `
  --subscription $subscriptionId

az policy assignment create `
  --name "dfs-resource-audit" `
  --policy "custom-defender-for-storage-resource-audit" `
  --scope "/subscriptions/$subscriptionId" `
  --enforcement-mode Default
```

---

## コンプライアンス評価

### オンデマンドスキャン（即時評価）

通常の評価サイクルは約24時間ですが、以下のコマンドで即時評価をトリガーできます。

```powershell
# スキャントリガー（同期：完了まで待機）
az policy state trigger-scan --subscription <Subscription ID>

# 結果確認
az policy state list `
  --subscription <Subscription ID> `
  --filter "policyAssignmentName eq 'dfs-resource-audit'" `
  --query "[].{resource:resourceId, compliant:complianceState}" `
  -o table
```

### NonCompliant リソースへの対応

NonCompliant と表示された Storage アカウントは、以下のいずれかで対応します。

**Azure Portal の場合:**  
対象 Storage アカウント → [Microsoft Defender for Cloud] → [Defender for Storage] を有効化

**Azure CLI の場合:**
```powershell
az rest --method put \
  --url "https://management.azure.com/subscriptions/<Sub>/resourceGroups/<RG>/providers/Microsoft.Storage/storageAccounts/<SA>/providers/Microsoft.Security/defenderForStorageSettings/current?api-version=2022-12-01-preview" \
  --body '{"properties":{"isEnabled":true,"overrideSubscriptionLevelSettings":true}}'
```

---

## サブスクリプション単位の Defender for Storage と組み合わせる場合

Defender for Storage はサブスクリプションレベルで一括有効化（`Microsoft.Security/pricings/StorageAccounts` の `pricingTier: Standard`）することもできます。  
その場合、リソース単位では `isEnabled: false` / `overrideSubscriptionLevelSettings: false` のまま実質的に保護されている Storage アカウントが存在します。

> **本ポリシー単独ではそのリソースも NonCompliant と判定されます**（リソース単位の `isEnabled: true` のみを評価するため）。

### 推奨アプローチ：Initiative（イニシアティブ）による組み合わせ

「サブスクリプション単位 **OR** リソース単位のどちらかで有効であれば Compliant」とみなしたい場合は、以下の2つのポリシーを **Policy Initiative（イニシアティブ）** にまとめることを推奨します。

| # | ポリシー | チェック内容 | NonCompliant の意味 |
|---|---|---|---|
| 1 | `custom-defender-for-storage-resource-audit` | リソース単位 `isEnabled: true` | そのリソース自身では未設定 |
| 2 | `Defender_StorageAccounts_Audit` | サブスクリプション `pricingTier: Standard` | サブスクリプション一括有効化も未設定 |

両方が NonCompliant のサブスクリプション・リソースが **真の未対応対象** です。

#### Initiative の作成例

```powershell
$mgId = "<Tenant Root Group ID>"

# イニシアティブ定義ファイルを作成
$initiative = @{
  properties = @{
    displayName = "[Custom] Defender for Storage - Combined Audit"
    description = "Defender for Storage のリソース単位・サブスクリプション単位の両方の有効化状態を監査します。"
    metadata    = @{ category = "Security Center"; version = "1.0.0" }
    policyDefinitions = @(
      @{
        policyDefinitionId = "/providers/Microsoft.Management/managementGroups/$mgId/providers/Microsoft.Authorization/policyDefinitions/custom-defender-for-storage-resource-audit"
        policyDefinitionReferenceId = "defenderStorageResourceLevel"
        parameters = @{ effect = @{ value = "AuditIfNotExists" } }
      },
      @{
        policyDefinitionId = "/providers/Microsoft.Management/managementGroups/$mgId/providers/Microsoft.Authorization/policyDefinitions/Defender_StorageAccounts_Audit"
        policyDefinitionReferenceId = "defenderStorageSubscriptionLevel"
        parameters = @{ effect = @{ value = "AuditIfNotExists" } }
      }
    )
  }
} | ConvertTo-Json -Depth 20

$initiative | Out-File "$env:TEMP\initiative.json" -Encoding utf8

az policy set-definition create `
  --name "custom-defender-for-storage-combined" `
  --definitions "$env:TEMP\initiative.json" `
  --management-group $mgId `
  --display-name "[Custom] Defender for Storage - Combined Audit"
```

#### コンプライアンス状態の解釈

```
サブスクリプション単位ポリシー  リソース単位ポリシー   解釈
─────────────────────────────────────────────────────────
Compliant                        Compliant            ✅ サブスク有効 かつ リソースも有効
Compliant                        NonCompliant         ✅ サブスク有効（リソース単位は未設定だが実質保護中）
NonCompliant                     Compliant            ✅ リソース単位で個別有効化済み
NonCompliant                     NonCompliant         ❌ どちらも未設定 → 要対応
```

> **補足:** Azure Policy は「イニシアティブ内の2ポリシーの AND / OR」を直接表現できません。上記の解釈は各ポリシーの結果を**手動で突き合わせる**運用を前提としています。

---

## 関連ポリシー

| ポリシー名 | チェック内容 | スコープ |
|---|---|---|
| `custom-defender-for-storage-resource-audit`（本ポリシー） | リソース単位の `isEnabled: true` | 各 Storage アカウント |
| `Defender_StorageAccounts_Audit` | サブスクリプションの Defender for Storage プランが `Standard` | サブスクリプション |

両ポリシーを組み合わせることで、リソース単位・サブスクリプション単位の両方の有効化状態を網羅的に監査できます。

---

## ファイル構成

```
.
├── azuredeploy.json              # ARM テンプレート（ポリシー定義 + 割り当て）
├── azuredeploy.parameters.json   # ARM テンプレート パラメーターファイル
└── combined-policy.json          # Azure Policy 定義ファイル（CLI デプロイ用）
```

---

## 参考リンク

- [Microsoft Defender for Storage とは](https://learn.microsoft.com/ja-jp/azure/defender-for-cloud/defender-for-storage-introduction)
- [Azure Policy の定義の構造](https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/definition-structure)
- [AuditIfNotExists の効果](https://learn.microsoft.com/ja-jp/azure/governance/policy/concepts/effects#auditifnotexists)
