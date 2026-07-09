# SDK API 文档

本文档描述了 `github.com/swooder/apple-server-sdk` 的公共 API。包名是 `appstore`，SDK 涵盖两个主要领域：

- 用于 App Store Server API 请求的 JWT 签名 `Client`
- 用于验证 Apple 签名 JWS 有效负载的 `SignedDataVerifier`

## 安装

```bash
go get github.com/swooder/apple-server-sdk
```

```go
import appstore "github.com/swooder/apple-server-sdk"
```

## 核心概念

### 环境

SDK 支持两种 App Store Server 环境：

```go
appstore.EnvironmentProduction // "Production"
appstore.EnvironmentSandbox    // "Sandbox"
```

当 `ClientOptions.Environment` 设置为 `EnvironmentSandbox` 时，客户端使用沙盒基础 URL。在所有其他情况下，默认使用生产环境。

```go
fmt.Println(appstore.ProductionBaseURL)
fmt.Println(appstore.SandboxBaseURL)
```

### 时间戳

Apple 有效负载和请求日期字段以 `int64` Unix 时间戳表示。App Store Server 有效负载字段（如 `signedDate`、`purchaseDate` 和 `expiresDate`）通常以毫秒表示。

```go
start := time.Now().Add(-30 * 24 * time.Hour).UnixMilli()
end := time.Now().UnixMilli()
```

### 导入别名

模块路径包含连字符，而包名是 `appstore`。为了可读性，示例使用导入别名：

```go
import appstore "github.com/swooder/apple-server-sdk"
```

## Client

`Client` 向 App Store Server API 端点发送请求。它为每个请求创建一个 ES256 JWT，并将其添加为 `Authorization: Bearer <token>` 头。

### 创建客户端

```go
package billing

import (
	"context"
	"os"
	"time"

	appstore "github.com/swooder/apple-server-sdk"
)

func newAppStoreClient() (*appstore.Client, error) {
	privateKey, err := os.ReadFile("AuthKey_XXXXXX.p8")
	if err != nil {
		return nil, err
	}

	return appstore.NewClient(appstore.ClientOptions{
		PrivateKeyPEM: string(privateKey),
		KeyID:         os.Getenv("APPSTORE_KEY_ID"),
		IssuerID:      os.Getenv("APPSTORE_ISSUER_ID"),
		BundleID:      "com.example.app",
		Environment:   appstore.EnvironmentSandbox,
		TokenTTL:      20 * time.Minute,
	})
}

func example(ctx context.Context) error {
	client, err := newAppStoreClient()
	if err != nil {
		return err
	}

	_, err = client.GetTransactionInfo(ctx, "50000001232324334")
	return err
}
```

### ClientOptions

| 字段 | 必填 | 描述 |
| --- | --- | --- |
| `PrivateKeyPEM` | 是 | App Store Connect API 密钥 `.p8` 私钥内容。支持 PKCS#8 和 EC 私钥 PEM 格式。 |
| `KeyID` | 是 | App Store Connect API 密钥 ID。用作 JWT 头 `kid`。 |
| `IssuerID` | 是 | App Store Connect 发行者 ID。用作 JWT 有效负载 `iss`。 |
| `BundleID` | 是 | 应用程序包 ID。用作 JWT 有效负载 `bid`。 |
| `Environment` | 否 | 设置为 `EnvironmentSandbox` 时使用沙盒；否则使用生产环境。 |
| `BaseURL` | 否 | 覆盖测试或代理的基础 URL。为空时，URL 从 `Environment` 中选择。 |
| `HTTPClient` | 否 | 自定义 `*http.Client`。默认值为 `http.DefaultClient`。 |
| `TokenTTL` | 否 | JWT 有效期。默认值为 `20 * time.Minute`；不得超过 `1 * time.Hour`。 |
| `Now` | 否 | 用于确定性测试的时钟钩子。默认值为 `time.Now`。 |

### 创建 JWT

正常 API 调用会自动创建令牌。如果您需要手动创建令牌：

```go
token, err := client.CreateToken(time.Now())
if err != nil {
	return err
}

_ = token
```

JWT 头字段：

- `alg`: `ES256`
- `kid`: `ClientOptions.KeyID`
- `typ`: `JWT`

JWT 有效负载字段：

- `iss`: `ClientOptions.IssuerID`
- `iat`: 令牌发布时间
- `exp`: `iat + TokenTTL`
- `aud`: `appstoreconnect-v1`
- `bid`: `ClientOptions.BundleID`

## API 方法

### 获取交易信息

```go
info, err := client.GetTransactionInfo(ctx, transactionID)
if err != nil {
	return err
}

signedTransaction := info.SignedTransactionInfo
_ = signedTransaction
```

方法：

```go
GetTransactionInfo(ctx context.Context, transactionID string) (TransactionInfoResponse, error)
```

端点：

```text
GET /inApps/v1/transactions/{transactionID}
```

响应：

```go
type TransactionInfoResponse struct {
	SignedTransactionInfo string `json:"signedTransactionInfo"`
}
```

返回的 `SignedTransactionInfo` 通常应该直接传递给 `SignedDataVerifier.VerifyAndDecodeTransaction`。

### 获取交易历史

交易历史是分页的。当响应中 `HasMore == true` 时，将返回的 `Revision` 传递给下一个请求。

```go
func fetchTransactionHistory(ctx context.Context, client *appstore.Client, transactionID string) ([]appstore.JWSTransactionDecodedPayload, error) {
	verifier, err := appstore.NewSignedDataVerifier(appstore.SignedDataVerifierOptions{
		BundleID:    "com.example.app",
		Environment: appstore.EnvironmentSandbox,
	})
	if err != nil {
		return nil, err
	}

	var revision string
	var transactions []appstore.JWSTransactionDecodedPayload

	for {
		response, err := client.GetTransactionHistory(ctx, transactionID, revision, &appstore.TransactionHistoryRequest{
			StartDate: time.Now().Add(-90 * 24 * time.Hour).UnixMilli(),
			EndDate:   time.Now().UnixMilli(),
			ProductIDs: []string{
				"com.example.premium.monthly",
				"com.example.premium.yearly",
			},
			Sort: "ASCENDING",
		})
		if err != nil {
			return nil, err
		}

		for _, signed := range response.SignedTransactions {
			tx, err := verifier.VerifyAndDecodeTransaction(signed)
			if err != nil {
				return nil, err
			}
			transactions = append(transactions, tx)
		}

		if !response.HasMore {
			break
		}
		revision = response.Revision
	}

	return transactions, nil
}
```

方法：

```go
GetTransactionHistory(ctx context.Context, transactionID string, revision string, request *TransactionHistoryRequest) (HistoryResponse, error)
```

端点：

```text
GET /inApps/v2/history/{transactionID}
```

查询参数由 `revision` 和 `TransactionHistoryRequest` 生成。`request` 可以是 `nil`。

`TransactionHistoryRequest`：

| 字段 | 查询参数 | 描述 |
| --- | --- | --- |
| `StartDate` | `startDate` | 开始时间戳。当值为 `0` 时省略。 |
| `EndDate` | `endDate` | 结束时间戳。当值为 `0` 时省略。 |
| `ProductIDs` | `productId` | 每个值都作为单独的查询参数添加。 |
| `ProductTypes` | `productType` | 每个值都作为单独的查询参数添加。 |
| `Revoked` | `revoked` | 当值为 `nil` 时省略；否则发送 `true` 或 `false`。 |
| `SubscriptionGroupIdentifiers` | `subscriptionGroupIdentifier` | 每个值都作为单独的查询参数添加。 |
| `InAppOwnershipType` | `inAppOwnershipType` | 当值为空时省略。 |
| `Sort` | `sort` | 当值为空时省略。 |

响应：

```go
type HistoryResponse struct {
	Revision           string
	HasMore            bool
	BundleID           string
	AppAppleID         int64
	Environment        appstore.Environment
	SignedTransactions []string
}
```

### 获取所有订阅状态

```go
statusResponse, err := client.GetAllSubscriptionStatuses(ctx, anyTransactionID, 1, 4)
if err != nil {
	return err
}

for _, group := range statusResponse.Data {
	for _, item := range group.LastTransactions {
		tx, err := verifier.VerifyAndDecodeTransaction(item.SignedTransactionInfo)
		if err != nil {
			return err
		}

		renewal, err := verifier.VerifyAndDecodeRenewalInfo(item.SignedRenewalInfo)
		if err != nil {
			return err
		}

		_ = tx.ProductID
		_ = renewal.AutoRenewStatus
	}
}
```

方法：

```go
GetAllSubscriptionStatuses(ctx context.Context, anyTransactionID string, statuses ...int64) (StatusResponse, error)
```

端点：

```text
GET /inApps/v1/subscriptions/{anyTransactionID}
```

`statuses` 是可变参数。每个值都作为 `status=<value>` 查询参数添加。如果未提供状态，则不会发送状态过滤器。

### 获取退款历史

```go
var revision string

for {
	response, err := client.GetRefundHistory(ctx, transactionID, revision)
	if err != nil {
		return err
	}

	for _, signed := range response.SignedTransactions {
		tx, err := verifier.VerifyAndDecodeTransaction(signed)
		if err != nil {
			return err
		}
		_ = tx.RevocationDate
	}

	if !response.HasMore {
		break
	}
	revision = response.Revision
}
```

方法：

```go
GetRefundHistory(ctx context.Context, transactionID string, revision string) (RefundHistoryResponse, error)
```

端点：

```text
GET /inApps/v2/refund/lookup/{transactionID}
```

### 获取通知历史

```go
func fetchNotificationHistory(ctx context.Context, client *appstore.Client, verifier *appstore.SignedDataVerifier) error {
	request := appstore.NotificationHistoryRequest{
		StartDate:        time.Now().Add(-24 * time.Hour).UnixMilli(),
		EndDate:          time.Now().UnixMilli(),
		NotificationType: "DID_RENEW",
		OnlyFailures:     false,
	}

	var token string
	for {
		response, err := client.GetNotificationHistory(ctx, request, token)
		if err != nil {
			return err
		}

		for _, item := range response.NotificationHistory {
			notification, err := verifier.VerifyAndDecodeNotification(item.SignedPayload)
			if err != nil {
				return err
			}
			_ = notification.NotificationUUID
		}

		if !response.HasMore {
			break
		}
		token = response.PaginationToken
	}

	return nil
}
```

方法：

```go
GetNotificationHistory(ctx context.Context, request NotificationHistoryRequest, paginationToken string) (NotificationHistoryResponse, error)
```

端点：

```text
POST /inApps/v1/notifications/history
```

当 `paginationToken` 不为空时，它会作为查询参数添加。过滤器作为 JSON 请求体发送。

`NotificationHistoryRequest`：

| 字段 | 描述 |
| --- | --- |
| `StartDate` | 查询开始时间戳。 |
| `EndDate` | 查询结束时间戳。 |
| `NotificationType` | 可选的通知类型过滤器。 |
| `NotificationSubtype` | 可选的子类型过滤器。 |
| `TransactionID` | 可选的交易过滤器。 |
| `OnlyFailures` | 当值为 `true` 时，仅查询失败的发送尝试。 |

### 请求和检查测试通知

```go
sent, err := client.RequestTestNotification(ctx)
if err != nil {
	return err
}

status, err := client.GetTestNotificationStatus(ctx, sent.TestNotificationToken)
if err != nil {
	return err
}

if status.SignedPayload != "" {
	notification, err := verifier.VerifyAndDecodeNotification(status.SignedPayload)
	if err != nil {
		return err
	}
	_ = notification.NotificationType
}
```

方法：

```go
RequestTestNotification(ctx context.Context) (SendTestNotificationResponse, error)
GetTestNotificationStatus(ctx context.Context, testNotificationToken string) (CheckTestNotificationResponse, error)
```

端点：

```text
POST /inApps/v1/notifications/test
GET  /inApps/v1/notifications/test/{testNotificationToken}
```

### 发送消费信息

使用此方法向 Apple 发送消费信息，用于退款或消费请求流程。

```go
consumptionPercentage := int(64) //50% 是 50000，100% 是 100000
customerConsented := true
deliveryStatus := "Delivered|UNDELIVERED_QUALITY_ISSUE|UNDELIVERED_WRONG_ITEM|UNDELIVERED_SERVER_OUTAGE|UNDELIVERED_OTHER"
refundPreference := "DECLINE|GRANT_FULL|GRANT_PRORATED"
sampleContentProvided := false

err := client.SendConsumptionInformation(ctx, transactionID, appstore.ConsumptionRequest{
	ConsumptionPercentage:    consumptionPercentage,
	CustomerConsented:        customerConsented,
	RefundPreference:         refundPreference,
	DeliveryStatus:           deliveryStatus,
	SampleContentProvided:    sampleContentProvided,
})
if err != nil {
	return err
}
```

方法：

```go
SendConsumptionInformation(ctx context.Context, transactionID string, request ConsumptionRequest) error
```

端点：

```text
PUT /inApps/v2/transactions/consumption/{transactionID}
```

当 `ConsumptionRequest` 中的指针字段为 `nil` 时，它们会从 JSON 请求体中省略。这样您只需发送您实际知道的值。

### 为单个订阅者延长续订日期

```go
response, err := client.ExtendRenewalDate(ctx, originalTransactionID, appstore.ExtendRenewalDateRequest{
	ExtendByDays:      7,
	ExtendReasonCode:  1,
	RequestIdentifier: "support-case-12345",
})
if err != nil {
	return err
}

if response.Success {
	_ = response.EffectiveDate
}
```

方法：

```go
ExtendRenewalDate(ctx context.Context, originalTransactionID string, request ExtendRenewalDateRequest) (ExtendRenewalDateResponse, error)
```

端点：

```text
PUT /inApps/v1/subscriptions/extend/{originalTransactionID}
```

### 为所有活跃订阅者延长续订日期

```go
requestID := "campaign-2026-05"

started, err := client.ExtendRenewalDateForAllActiveSubscribers(ctx, appstore.MassExtendRenewalDateRequest{
	ProductID:              "com.example.premium.monthly",
	ExtendByDays:           7,
	ExtendReasonCode:       1,
	RequestIdentifier:      requestID,
	StorefrontCountryCodes: []string{"US", "TR"},
})
if err != nil {
	return err
}

status, err := client.GetStatusOfSubscriptionRenewalDateExtensions(ctx, "com.example.premium.monthly", started.RequestIdentifier)
if err != nil {
	return err
}

_ = status.Complete
```

方法：

```go
ExtendRenewalDateForAllActiveSubscribers(ctx context.Context, request MassExtendRenewalDateRequest) (MassExtendRenewalDateResponse, error)
GetStatusOfSubscriptionRenewalDateExtensions(ctx context.Context, productID, requestIdentifier string) (MassExtendRenewalDateStatusResponse, error)
```

端点：

```text
POST /inApps/v1/subscriptions/extend/mass
GET  /inApps/v1/subscriptions/extend/mass/{productID}/{requestIdentifier}
```

### 查找订单 ID

```go
lookup, err := client.LookUpOrderID(ctx, orderID)
if err != nil {
	return err
}

for _, signed := range lookup.SignedTransactions {
	tx, err := verifier.VerifyAndDecodeTransaction(signed)
	if err != nil {
		return err
	}
	_ = tx.OriginalTransactionID
}
```

方法：

```go
LookUpOrderID(ctx context.Context, orderID string) (OrderLookupResponse, error)
```

端点：

```text
GET /inApps/v1/lookup/{orderID}
```

## SignedDataVerifier

`SignedDataVerifier` 验证并解码 Apple 签名的 JWS 有效负载。成功处理必须验证签名、证书链、应用程序身份和环境；仅进行纯 JSON 解码是不够的。

### 创建验证器

```go
verifier, err := appstore.NewSignedDataVerifier(appstore.SignedDataVerifierOptions{
	BundleID:           "com.example.app",
	AppAppleID:         1234567890,
	Environment:        appstore.EnvironmentProduction,
	EnableOnlineChecks: true,
})
if err != nil {
	return err
}
```

对于沙盒环境：

```go
verifier, err := appstore.NewSignedDataVerifier(appstore.SignedDataVerifierOptions{
	BundleID:    "com.example.app",
	Environment: appstore.EnvironmentSandbox,
})
```

### SignedDataVerifierOptions

| 字段 | 描述 |
| --- | --- |
| `RootCertificates` | DER 格式的 Apple 根证书。当值为空时，SDK 使用其内置的 Apple 根证书。 |
| `BundleID` | 设置后，有效负载的包 ID 必须与此值匹配。 |
| `AppAppleID` | 用于生产验证中的 App Apple ID 验证。 |
| `Environment` | 设置后，有效负载的环境必须与此值匹配。 |
| `EnableOnlineChecks` | 启用当前日期证书验证以及叶证书和中间证书的 OCSP 吊销检查。默认为离线验证。 |
| `HTTPClient` | 启用在线检查时用于 OCSP 请求的可选 HTTP 客户端。默认值为具有 30 秒超时的客户端。 |
| `Now` | 当有效负载没有签名日期时，用于证书验证的时钟。默认值为 `time.Now`。 |

### 在线检查和 OCSP

默认情况下，验证是离线的：在 JWS 签名日期检查证书链，这与签名有效负载的历史有效性相匹配。当 `EnableOnlineChecks` 为 `true` 时，验证器的行为更像 Apple 的官方服务器库：

- 根据当前时间检查证书有效性。
- 为叶证书和中间证书请求 OCSP。
- 非 200 OCSP 响应、网络故障、响应过大和过期 OCSP 响应返回 `RetryableVerificationFailure`。
- 已吊销的证书返回 `InvalidCertificate`。
- 成功验证的证书链缓存 15 分钟，最多 32 个条目。

```go
verifier, err := appstore.NewSignedDataVerifier(appstore.SignedDataVerifierOptions{
	BundleID:           "com.example.app",
	AppAppleID:         1234567890,
	Environment:        appstore.EnvironmentProduction,
	EnableOnlineChecks: true,
	HTTPClient: &http.Client{
		Timeout: 10 * time.Second,
	},
})
```

### 验证规则

验证器检查：

- JWS 紧凑序列化正好有三个部分
- JWS 头 `alg` 是 `ES256`
- 头 `x5c` 证书链正好包含三个证书
- 证书可以被解析
- 根证书与受信任的 Apple 根证书之一匹配
- 叶证书具有 Apple 收据签名扩展
- 中间证书具有 Apple WWDR 扩展
- 在有效负载签名日期或启用 `EnableOnlineChecks` 时的当前时间，证书链有效
- 当启用 `EnableOnlineChecks` 时，叶证书和中间证书的 OCSP 状态良好
- ES256 签名对于叶公钥有效
- 包 ID、App Apple ID 和环境与验证器选项匹配

### 解码交易 JWS

```go
tx, err := verifier.VerifyAndDecodeTransaction(signedTransactionInfo)
if err != nil {
	return err
}

if tx.ExpiresDate > time.Now().UnixMilli() {
	// 订阅似乎处于活跃状态。
}
```

方法：

```go
VerifyAndDecodeTransaction(signedTransactionInfo string) (JWSTransactionDecodedPayload, error)
```

重要的解码字段：

| 字段 | 描述 |
| --- | --- |
| `TransactionID` | 交易 ID。 |
| `OriginalTransactionID` | 订阅链的原始交易 ID。 |
| `BundleID` | 应用程序包 ID。 |
| `ProductID` | 购买的产品 ID。 |
| `SubscriptionGroupIdentifier` | 订阅组 ID。 |
| `PurchaseDate` | 购买时间戳。 |
| `OriginalPurchaseDate` | 原始购买时间戳。 |
| `ExpiresDate` | 订阅到期时间戳。 |
| `RevocationReason` | 吊销原因，或 `nil`。 |
| `RevocationDate` | 吊销时间戳。 |
| `Environment` | `Production` 或 `Sandbox`。 |
| `AppAccountToken` | 与交易关联的应用端账户令牌。 |

### 解码续订信息 JWS

```go
renewal, err := verifier.VerifyAndDecodeRenewalInfo(signedRenewalInfo)
if err != nil {
	return err
}

if renewal.AutoRenewStatus != nil && *renewal.AutoRenewStatus == 1 {
	// 自动续订已启用。
}
```

方法：

```go
VerifyAndDecodeRenewalInfo(signedRenewalInfo string) (JWSRenewalInfoDecodedPayload, error)
```

续订信息有效负载不包含包 ID，因此此方法验证环境，但不执行交易风格的包 ID 验证。

重要字段：

| 字段 | 描述 |
| --- | --- |
| `OriginalTransactionID` | 订阅链 ID。 |
| `AutoRenewProductID` | 将续订的产品 ID。 |
| `ProductID` | 当前产品 ID。 |
| `AutoRenewStatus` | 自动续订状态，或 `nil`。 |
| `ExpirationIntent` | 到期原因，或 `nil`。 |
| `IsInBillingRetryPeriod` | 订阅是否处于账单重试状态。 |
| `GracePeriodExpiresDate` | 宽限期到期时间戳。 |
| `RenewalDate` | 下一次续订时间戳。 |
| `Environment` | 有效负载环境。 |

### 解码应用程序交易 JWS

```go
appTx, err := verifier.VerifyAndDecodeAppTransaction(signedAppTransaction)
if err != nil {
	return err
}

_ = appTx.OriginalApplicationVersion
```

方法：

```go
VerifyAndDecodeAppTransaction(signedAppTransaction string) (JWSAppTransactionDecodedPayload, error)
```

此方法验证应用程序交易有效负载，并将其 `BundleID`、`AppAppleID` 和 `ReceiptType` 与验证器选项进行比较。

### 解码通知 JWS

App Store Server Notifications V2 请求体包含一个 `signedPayload`。Webhook 处理程序应在信任任何字段之前验证此有效负载。

```go
func handleAppStoreNotification(w http.ResponseWriter, r *http.Request) {
	var body struct {
		SignedPayload string `json:"signedPayload"`
	}

	if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}

	notification, err := verifier.VerifyAndDecodeNotification(body.SignedPayload)
	if err != nil {
		http.Error(w, "invalid signed payload", http.StatusBadRequest)
		return
	}

	switch notification.NotificationType {
	case "DID_RENEW":
		if notification.Data != nil {
			tx, err := verifier.VerifyAndDecodeTransaction(notification.Data.SignedTransactionInfo)
			if err != nil {
				http.Error(w, "invalid transaction", http.StatusBadRequest)
				return
			}
			_ = tx.ProductID
		}
	case "EXPIRED":
		// 运行到期处理。
	}

	w.WriteHeader(http.StatusNoContent)
}
```

方法：

```go
VerifyAndDecodeNotification(signedPayload string) (ResponseBodyV2DecodedPayload, error)
```

验证器从以下字段之一中查找通知应用程序身份：

- `Data`
- `Summary`
- `ExternalPurchaseToken`

对于 `ExternalPurchaseToken`，环境从令牌 ID 中推断：以 `SANDBOX` 开头的 `ExternalPurchaseID` 值被视为沙盒；所有其他值被视为生产环境。

## 错误处理

### APIError

当 App Store Server API 返回非 2xx 响应时，SDK 返回 `*appstore.APIError`。

```go
info, err := client.GetTransactionInfo(ctx, transactionID)
if err != nil {
	var apiErr *appstore.APIError
	if errors.As(err, &apiErr) {
		log.Printf("appstore api failed: status=%d code=%d message=%q body=%s",
			apiErr.StatusCode,
			apiErr.ErrorCode,
			apiErr.Message,
			string(apiErr.Body),
		)
		return err
	}

	return err
}

_ = info
```

`APIError` 字段：

| 字段 | 描述 |
| --- | --- |
| `StatusCode` | HTTP 状态码。 |
| `ErrorCode` | Apple 错误响应体中的数字 `errorCode`。 |
| `Message` | Apple 错误消息。 |
| `Body` | 原始响应体的副本。 |

### VerificationError

JWS 验证失败作为 `*appstore.VerificationError` 返回。

```go
tx, err := verifier.VerifyAndDecodeTransaction(signedTransactionInfo)
if err != nil {
	var verificationErr *appstore.VerificationError
	if errors.As(err, &verificationErr) {
		switch verificationErr.Status {
		case appstore.InvalidAppIdentifier:
			return fmt.Errorf("wrong app payload: %w", err)
		case appstore.InvalidEnvironment:
			return fmt.Errorf("wrong app store environment: %w", err)
		case appstore.InvalidSignedData:
			return fmt.Errorf("malformed signed payload: %w", err)
		default:
			return fmt.Errorf("untrusted app store payload: %w", err)
		}
	}

	return err
}

_ = tx
```

验证状态：

| 状态 | 含义 |
| --- | --- |
| `VerificationOK` | 保留用于成功验证。它不会作为错误返回。 |
| `VerificationFailure` | 签名或一般验证失败。 |
| `RetryableVerificationFailure` | 保留用于可重试的验证失败。 |
| `InvalidAppIdentifier` | 包 ID 或 App Apple ID 与预期应用不匹配。 |
| `InvalidEnvironment` | 有效负载环境与预期环境不匹配。 |
| `InvalidChainLength` | JWS `x5c` 没有正好包含三个证书。 |
| `InvalidCertificate` | 证书解析、信任或有效性失败。 |
| `InvalidSignedData` | JWS 格式或有效负载解码失败。 |

## 端到端示例：通过交易 ID 进行权限检查

```go
package billing

import (
	"context"
	"errors"
	"os"
	"time"

	appstore "github.com/swooder/apple-server-sdk"
)

type Entitlement struct {
	ProductID string
	Active   bool
	Expires  time.Time
}

func LoadEntitlement(ctx context.Context, transactionID string) (Entitlement, error) {
	privateKey, err := os.ReadFile(os.Getenv("APPSTORE_PRIVATE_KEY_PATH"))
	if err != nil {
		return Entitlement{}, err
	}

	client, err := appstore.NewClient(appstore.ClientOptions{
		PrivateKeyPEM: string(privateKey),
		KeyID:         os.Getenv("APPSTORE_KEY_ID"),
		IssuerID:      os.Getenv("APPSTORE_ISSUER_ID"),
		BundleID:      "com.example.app",
		Environment:   appstore.EnvironmentProduction,
	})
	if err != nil {
		return Entitlement{}, err
	}

	verifier, err := appstore.NewSignedDataVerifier(appstore.SignedDataVerifierOptions{
		BundleID:    "com.example.app",
		AppAppleID:  1234567890,
		Environment: appstore.EnvironmentProduction,
	})
	if err != nil {
		return Entitlement{}, err
	}

	info, err := client.GetTransactionInfo(ctx, transactionID)
	if err != nil {
		var apiErr *appstore.APIError
		if errors.As(err, &apiErr) && apiErr.StatusCode == 404 {
			return Entitlement{}, nil
		}
		return Entitlement{}, err
	}

	tx, err := verifier.VerifyAndDecodeTransaction(info.SignedTransactionInfo)
	if err != nil {
		return Entitlement{}, err
	}

	expires := time.UnixMilli(tx.ExpiresDate)
	return Entitlement{
		ProductID: tx.ProductID,
		Active:   tx.RevocationDate == 0 && expires.After(time.Now()),
		Expires:  expires,
	}, nil
}
```

## 端到端示例：Webhook 处理程序

```go
package billing

import (
	"encoding/json"
	"net/http"

	appstore "github.com/swooder/apple-server-sdk"
)

type AppStoreWebhook struct {
	Verifier *appstore.SignedDataVerifier
}

func (h AppStoreWebhook) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	var body struct {
		SignedPayload string `json:"signedPayload"`
	}
	if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
		http.Error(w, "invalid json", http.StatusBadRequest)
		return
	}

	notification, err := h.Verifier.VerifyAndDecodeNotification(body.SignedPayload)
	if err != nil {
		http.Error(w, "invalid notification", http.StatusBadRequest)
		return
	}

	if notification.Data != nil && notification.Data.SignedTransactionInfo != "" {
		tx, err := h.Verifier.VerifyAndDecodeTransaction(notification.Data.SignedTransactionInfo)
		if err != nil {
			http.Error(w, "invalid transaction", http.StatusBadRequest)
			return
		}

		// 更新您自己系统中的权限或订阅状态。
		_ = tx.OriginalTransactionID
	}

	if notification.Data != nil && notification.Data.SignedRenewalInfo != "" {
		renewal, err := h.Verifier.VerifyAndDecodeRenewalInfo(notification.Data.SignedRenewalInfo)
		if err != nil {
			http.Error(w, "invalid renewal info", http.StatusBadRequest)
			return
		}

		// 更新续订策略或账单状态。
		_ = renewal.RenewalDate
	}

	w.WriteHeader(http.StatusNoContent)
}
```

## 测试和模拟

`ClientOptions.BaseURL`、`HTTPClient` 和 `Now` 使得在测试中轻松控制客户端行为。

```go
server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
	if r.Header.Get("Authorization") == "" {
		t.Fatal("missing authorization header")
	}

	switch r.URL.Path {
	case "/inApps/v1/transactions/tx-1":
		_, _ = w.Write([]byte(`{"signedTransactionInfo":"signed"}`))
	default:
		w.WriteHeader(http.StatusNotFound)
		_, _ = w.Write([]byte(`{"errorCode":4040010,"errorMessage":"not found"}`))
	}
}))
defer server.Close()

client, err := appstore.NewClient(appstore.ClientOptions{
	PrivateKeyPEM: testPrivateKeyPEM,
	KeyID:         "KEY123",
	IssuerID:      "issuer-1",
	BundleID:      "com.example.app",
	Environment:   appstore.EnvironmentSandbox,
	BaseURL:       server.URL,
	Now: func() time.Time {
		return time.Date(2026, 5, 4, 12, 0, 0, 0, time.UTC)
	},
})
if err != nil {
	t.Fatal(err)
}
```

`SignedDataVerifierOptions.RootCertificates` 可以设置为测试根证书。当它为空时，SDK 使用其内置的 Apple 根证书。

仓库还包含一个用于真实 Apple 沙盒交易 JWS 的跳过的黄金测试钩子。要启用它，请在以下位置放置一个一次性沙盒 `signedTransactionInfo` 紧凑 JWS：

```text
testdata/apple_sandbox_signed_transaction_info.jws
```

不要在此固定装置中使用生产用户数据。测试从固定装置有效负载中派生包 ID，根据内置的 Apple 根证书验证 JWS，并断言有效负载环境是 `Sandbox`。

## 安全说明

- 不要记录 `.p8` 私钥内容或将它们提交到源代码控制。
- 使用 API 响应中的数据之前，验证签名的 JWS 值。
- 纯 JSON 解码后不要信任 webhook 请求体；使用 `VerifyAndDecodeNotification`。
- 对于生产环境，同时设置 `BundleID`、`AppAppleID` 和 `Environment` 以进行更严格的验证。
- 为了更严格的生产验证，启用 `EnableOnlineChecks`，以便通过 OCSP 拒绝已吊销的 Apple 签名证书。
- 将沙盒和生产客户端/验证器实例分开，以减少操作错误。
- `APIError.Body` 可能包含用户或购买数据。在写入日志之前，根据您的日志记录策略对其进行掩码。

## 公共类型摘要

客户端和请求/响应类型：

| 类型 | 用途 |
| --- | --- |
| `Environment` | 生产和沙盒环境值。 |
| `ClientOptions` | `NewClient` 的配置。 |
| `Client` | App Store Server API 方法。 |
| `TransactionHistoryRequest` | 交易历史过滤器。 |
| `NotificationHistoryRequest` | 通知历史过滤器。 |
| `ConsumptionRequest` | 消费信息请求体。 |
| `ExtendRenewalDateRequest` | 单个订阅续订日期延长体。 |
| `MassExtendRenewalDateRequest` | 批量续订日期延长体。 |
| `TransactionInfoResponse` | 交易查找响应。 |
| `AppTransactionInfoResponse` | 签名应用交易响应模型。 |
| `HistoryResponse` | 交易历史响应。 |
| `StatusResponse` | 订阅状态响应。 |
| `SubscriptionGroupIdentifierItem` | 订阅状态响应中的组项。 |
| `LastTransactionsItem` | 订阅状态响应中的最后一项交易。 |
| `RefundHistoryResponse` | 退款历史响应。 |
| `OrderLookupResponse` | 订单 ID 查找响应。 |
| `SendTestNotificationResponse` | 测试通知创建响应。 |
| `CheckTestNotificationResponse` | 测试通知状态响应。 |
| `SendAttemptItem` | 通知发送尝试模型。 |
| `NotificationHistoryResponse` | 通知历史响应。 |
| `NotificationHistoryResponseItem` | 通知历史中的签名有效负载和发送尝试项。 |
| `ExtendRenewalDateResponse` | 单个订阅延长响应。 |
| `MassExtendRenewalDateResponse` | 批量延长启动响应。 |
| `MassExtendRenewalDateStatusResponse` | 批量延长状态响应。 |
| `APIError` | 非 2xx API 响应错误。 |

验证器和有效负载类型：

| 类型 | 用途 |
| --- | --- |
| `SignedDataVerifierOptions` | `NewSignedDataVerifier` 的配置。 |
| `SignedDataVerifier` | JWS 验证和解码方法。 |
| `JWSTransactionDecodedPayload` | 交易 JWS 有效负载。 |
| `JWSRenewalInfoDecodedPayload` | 续订信息 JWS 有效负载。 |
| `JWSAppTransactionDecodedPayload` | 应用交易 JWS 有效负载。 |
| `ResponseBodyV2DecodedPayload` | 通知 V2 签名有效负载。 |
| `NotificationData` | 通知数据部分。 |
| `NotificationSummary` | 通知摘要部分。 |
| `ExternalPurchaseToken` | 外部购买令牌部分。 |
| `JWSDecodedHeader` | JWS 头模型。 |
| `VerificationError` | JWS 验证错误。 |
| `VerificationStatus` | 验证错误类别。 |

根证书辅助函数：

```go
roots := appstore.DefaultAppleRootCertificates()
verifier, err := appstore.NewSignedDataVerifier(appstore.SignedDataVerifierOptions{
	RootCertificates: roots,
	BundleID:         "com.example.app",
	Environment:      appstore.EnvironmentSandbox,
})
```
