# 第十一章：完整 API 參考

## 本章概述

本章提供 Make Bridge 所有 API 端點的完整參考文檔，包括請求格式、參數說明、回應結構和錯誤處理。所有 API 都遵循 RESTful 設計原則。

> 📚 **實作參考指南**
> 
> 本章為純參考文檔。如需實際實作範例，請參考：
> 
> - **基礎實作**：第4章_基礎整合實作.md - JWT 認證和 API 調用範例
> - **初級專案**：第5章_完整專案實作_初級.md - 客戶回饋系統實作
> - **中級專案**：第6章_完整專案實作_中級.md - CRM 集成系統實作
> - **高級專案**：第7章_完整專案實作_高級.md - 企業級多租戶平台
> - **進階 API**：第9章_Make_API_進階應用.md - 直接 Make API 使用
> - **架構設計**：第10章_架構設計與最佳實踐.md - 企業級架構指南
> - **問題排解**：第12章_疑難排解與維護.md - API 錯誤處理

## 11.1 API 基礎資訊

### 11.1.1 基礎 URL

```
生產環境: https://api.make.com
測試環境: https://api-sandbox.make.com
```

### 11.1.2 認證方式

所有 API 請求都需要在 Header 中包含認證資訊：

> 🔑 **認證實作參考**：第4章 4.0.1 JWT 生成標準模式 - 查看完整的認證程式碼實作

```http
Authorization: Bearer {JWT_TOKEN}
X-API-Key: {YOUR_API_KEY}
```

### 11.1.3 通用回應格式

成功回應：
```json
{
    "success": true,
    "data": {
        // 回應資料
    },
    "meta": {
        "timestamp": "2024-01-15T10:30:00Z",
        "version": "1.0"
    }
}
```

錯誤回應：
```json
{
    "success": false,
    "error": {
        "code": "ERROR_CODE",
        "message": "錯誤描述",
        "details": {}
    },
    "meta": {
        "timestamp": "2024-01-15T10:30:00Z",
        "request_id": "req_123456"
    }
}
```

## 11.2 Bridge 核心 API

### 11.2.1 初始化 Bridge

**端點**: `POST /v1/bridge/init`

**描述**: 初始化 Make Bridge 實例

**請求參數**:
```json
{
    "app_id": "string",
    "user_id": "string",
    "config": {
        "theme": {
            "primary_color": "#007bff",
            "secondary_color": "#6c757d"
        },
        "features": {
            "allow_custom_templates": true,
            "enable_webhooks": true
        }
    }
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "bridge_id": "brg_abc123",
        "jwt_token": "eyJhbGciOiJIUzI1NiIs...",
        "expires_at": "2024-01-15T10:32:00Z",
        "config": {
            "integration_url": "https://integrations.make.com/embed/bridge",
            "allowed_templates": ["template_123", "template_456"]
        }
    }
}
```

**錯誤碼**:
- `INVALID_APP_ID`: 無效的應用程式 ID
- `USER_NOT_FOUND`: 找不到使用者
- `INVALID_CONFIG`: 設定格式錯誤

### 11.2.2 更新 Bridge 設定

**端點**: `PUT /v1/bridge/{bridge_id}/config`

**描述**: 更新現有 Bridge 實例的設定

**路徑參數**:
- `bridge_id`: Bridge 實例 ID

**請求參數**:
```json
{
    "theme": {
        "primary_color": "#28a745"
    },
    "features": {
        "enable_analytics": true
    }
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "bridge_id": "brg_abc123",
        "updated_at": "2024-01-15T10:35:00Z",
        "config": {
            // 更新後的完整設定
        }
    }
}
```

### 11.2.3 獲取 Bridge 狀態

**端點**: `GET /v1/bridge/{bridge_id}/status`

**描述**: 獲取 Bridge 實例的當前狀態

**回應範例**:
```json
{
    "success": true,
    "data": {
        "bridge_id": "brg_abc123",
        "status": "active",
        "created_at": "2024-01-15T10:00:00Z",
        "last_activity": "2024-01-15T10:30:00Z",
        "scenarios_count": 5,
        "active_connections": 3
    }
}
```

### 11.2.4 初始化整合流程

**端點**: `POST /portal/api/bridge/integrations/init/{publicVersionId}`

**描述**: 使用特定模板的公開版本 ID 初始化整合流程

**路徑參數**:
- `publicVersionId`: 模板的公開版本識別碼

**請求參數**:
```json
{
    "customerId": "string",
    "operationsLimit": 10000,
    "redirectUri": "https://your-app.com/callback"
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "flowId": "flow_abc123",
        "integrationUrl": "https://integrations.make.com/embed/...",
        "expiresAt": "2024-01-15T10:32:00Z"
    }
}
```

### 11.2.5 檢查初始化狀態

**端點**: `GET /portal/api/bridge/integrations/check-init/{flowId}`

**描述**: 檢查整合流程的初始化狀態

**路徑參數**:
- `flowId`: 流程識別碼

**回應範例**:
```json
{
    "success": true,
    "data": {
        "flowId": "flow_abc123",
        "status": "completed",
        "scenarioId": "scn_456",
        "createdAt": "2024-01-15T10:30:00Z"
    }
}
```

## 11.3 模板 API

### 11.3.1 獲取模板列表

**端點**: `GET /v1/templates`

**描述**: 獲取可用的模板列表

**查詢參數**:
- `category`: 模板分類 (可選)
- `search`: 搜尋關鍵字 (可選)
- `page`: 頁碼 (預設: 1)
- `limit`: 每頁數量 (預設: 20, 最大: 100)
- `sort`: 排序方式 (created_at, updated_at, name)
- `order`: 排序順序 (asc, desc)

**回應範例**:
```json
{
    "success": true,
    "data": {
        "templates": [
            {
                "template_id": "tpl_123",
                "name": "Slack 通知整合",
                "description": "當特定事件發生時發送 Slack 通知",
                "category": "communication",
                "modules": [
                    {
                        "type": "webhook",
                        "name": "接收 Webhook"
                    },
                    {
                        "type": "slack",
                        "name": "發送訊息"
                    }
                ],
                "created_at": "2024-01-10T08:00:00Z",
                "updated_at": "2024-01-14T15:30:00Z"
            }
        ],
        "pagination": {
            "current_page": 1,
            "total_pages": 5,
            "total_items": 98,
            "items_per_page": 20
        }
    }
}
```

### 11.3.2 獲取模板詳情

**端點**: `GET /v1/templates/{template_id}`

**描述**: 獲取特定模板的詳細資訊

**回應範例**:
```json
{
    "success": true,
    "data": {
        "template_id": "tpl_123",
        "name": "Slack 通知整合",
        "description": "當特定事件發生時發送 Slack 通知",
        "category": "communication",
        "version": "1.2.0",
        "author": {
            "id": "usr_456",
            "name": "Make Team"
        },
        "modules": [
            {
                "id": "mod_1",
                "type": "webhook",
                "name": "接收 Webhook",
                "position": {
                    "x": 100,
                    "y": 100
                },
                "config": {
                    "method": "POST",
                    "headers": {
                        "Content-Type": "application/json"
                    }
                }
            }
        ],
        "connections": [
            {
                "source": "mod_1",
                "target": "mod_2"
            }
        ],
        "variables": [
            {
                "name": "webhook_url",
                "type": "string",
                "required": true,
                "description": "Webhook URL"
            }
        ]
    }
}
```

### 11.3.3 建立自訂模板

**端點**: `POST /v1/templates`

**描述**: 建立新的自訂模板

**請求參數**:
```json
{
    "name": "我的自訂模板",
    "description": "模板描述",
    "category": "custom",
    "modules": [
        {
            "type": "webhook",
            "name": "Webhook 觸發器",
            "config": {
                "method": "POST"
            }
        }
    ],
    "connections": [],
    "variables": []
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "template_id": "tpl_custom_789",
        "name": "我的自訂模板",
        "created_at": "2024-01-15T11:00:00Z",
        "status": "draft"
    }
}
```

### 11.3.4 更新模板

**端點**: `PUT /v1/templates/{template_id}`

**描述**: 更新現有模板

**請求參數**:
```json
{
    "name": "更新的模板名稱",
    "description": "更新的描述",
    "modules": [
        // 更新的模組配置
    ]
}
```

### 11.3.5 刪除模板

**端點**: `DELETE /v1/templates/{template_id}`

**描述**: 刪除自訂模板

**回應範例**:
```json
{
    "success": true,
    "data": {
        "template_id": "tpl_custom_789",
        "deleted_at": "2024-01-15T11:30:00Z"
    }
}
```

## 11.4 場景 API

### 11.4.1 建立場景

**端點**: `POST /v1/scenarios`

**描述**: 從模板建立新場景

**請求參數**:
```json
{
    "template_id": "tpl_123",
    "name": "我的 Slack 通知",
    "config": {
        "webhook_url": "https://hooks.slack.com/services/...",
        "channel": "#general"
    },
    "team_id": 12345,
    "folder_id": 67890
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "scenario_id": "scn_456",
        "name": "我的 Slack 通知",
        "status": "inactive",
        "created_at": "2024-01-15T11:45:00Z",
        "webhook_url": "https://hook.make.com/abc123xyz",
        "team_id": 12345,
        "folder_id": 67890
    }
}
```

### 11.4.2 獲取場景列表

**端點**: `GET /v1/scenarios`

**描述**: 獲取使用者的場景列表

**查詢參數**:
- `team_id`: 團隊 ID (必填)
- `folder_id`: 資料夾 ID (可選)
- `status`: 場景狀態 (active, inactive, error)
- `page`: 頁碼
- `limit`: 每頁數量

**回應範例**:
```json
{
    "success": true,
    "data": {
        "scenarios": [
            {
                "scenario_id": "scn_456",
                "name": "我的 Slack 通知",
                "status": "active",
                "last_run": "2024-01-15T11:50:00Z",
                "runs_count": 152,
                "error_count": 2,
                "template": {
                    "id": "tpl_123",
                    "name": "Slack 通知整合"
                }
            }
        ],
        "pagination": {
            "current_page": 1,
            "total_pages": 3,
            "total_items": 45
        }
    }
}
```

### 11.4.3 獲取場景詳情

**端點**: `GET /v1/scenarios/{scenario_id}`

**描述**: 獲取場景的詳細資訊

**回應範例**:
```json
{
    "success": true,
    "data": {
        "scenario_id": "scn_456",
        "name": "我的 Slack 通知",
        "status": "active",
        "created_at": "2024-01-15T11:45:00Z",
        "updated_at": "2024-01-15T12:00:00Z",
        "webhook_url": "https://hook.make.com/abc123xyz",
        "config": {
            "scheduling": {
                "type": "interval",
                "interval": 15
            },
            "error_handling": {
                "continue_on_error": false,
                "retry_attempts": 3
            }
        },
        "modules": [
            // 場景中的模組配置
        ],
        "statistics": {
            "total_runs": 152,
            "successful_runs": 150,
            "failed_runs": 2,
            "average_duration": 1.5,
            "data_processed": "15.2 MB"
        }
    }
}
```

### 11.4.4 更新場景

**端點**: `PUT /v1/scenarios/{scenario_id}`

**描述**: 更新場景設定

**請求參數**:
```json
{
    "name": "更新的場景名稱",
    "status": "active",
    "config": {
        "scheduling": {
            "type": "cron",
            "expression": "0 9 * * *"
        }
    }
}
```

### 11.4.5 執行場景

**端點**: `POST /v1/scenarios/{scenario_id}/run`

**描述**: 手動執行場景

**請求參數**:
```json
{
    "data": {
        "message": "測試訊息",
        "priority": "high"
    }
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "execution_id": "exec_789",
        "scenario_id": "scn_456",
        "status": "running",
        "started_at": "2024-01-15T12:15:00Z"
    }
}
```

### 11.4.6 獲取執行歷史

**端點**: `GET /v1/scenarios/{scenario_id}/executions`

**描述**: 獲取場景的執行歷史

**查詢參數**:
- `status`: 執行狀態 (success, error, running)
- `start_date`: 開始日期
- `end_date`: 結束日期
- `page`: 頁碼
- `limit`: 每頁數量

**回應範例**:
```json
{
    "success": true,
    "data": {
        "executions": [
            {
                "execution_id": "exec_789",
                "status": "success",
                "started_at": "2024-01-15T12:15:00Z",
                "completed_at": "2024-01-15T12:15:02Z",
                "duration": 2000,
                "operations_used": 3,
                "data_processed": "1.2 KB"
            }
        ],
        "pagination": {
            "current_page": 1,
            "total_pages": 10,
            "total_items": 152
        }
    }
}
```

### 11.4.7 升級場景

**端點**: `PUT /portal/api/bridge/scenarios/{scenarioId}/upgrade`

**描述**: 升級場景到新版本的模板

**路徑參數**:
- `scenarioId`: 場景 ID

**請求參數**:
```json
{
    "targetVersion": "1.2.0",
    "preserveData": true
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "scenarioId": "scn_456",
        "upgradedVersion": "1.2.0",
        "upgradedAt": "2024-01-15T12:25:00Z"
    }
}
```

### 11.4.8 獲取場景藍圖

**端點**: `GET /portal/api/bridge/scenarios/{scenarioId}/blueprint`

**描述**: 獲取場景的完整藍圖配置

**回應範例**:
```json
{
    "success": true,
    "data": {
        "blueprint": {
            "modules": [...],
            "connections": [...],
            "metadata": {...}
        }
    }
}
```

### 11.4.9 獲取場景日誌

**端點**: `GET /portal/api/bridge/scenarios/{scenarioId}/logs`

**描述**: 獲取場景的執行日誌

**查詢參數**:
- `limit`: 日誌數量限制 (預設: 50)
- `offset`: 分頁偏移
- `level`: 日誌級別 (info, warning, error)

**回應範例**:
```json
{
    "success": true,
    "data": {
        "logs": [
            {
                "executionId": "exec_789",
                "timestamp": "2024-01-15T12:15:00Z",
                "level": "info",
                "message": "Scenario executed successfully",
                "details": {...}
            }
        ]
    }
}
```

### 11.4.10 獲取特定執行日誌

**端點**: `GET /portal/api/bridge/scenarios/{scenarioId}/logs/{executionId}`

**描述**: 獲取特定執行的詳細日誌

**回應範例**:
```json
{
    "success": true,
    "data": {
        "executionId": "exec_789",
        "scenarioId": "scn_456",
        "startedAt": "2024-01-15T12:15:00Z",
        "completedAt": "2024-01-15T12:15:02Z",
        "status": "success",
        "modules": [
            {
                "moduleId": "mod_1",
                "status": "success",
                "duration": 120,
                "input": {...},
                "output": {...}
            }
        ]
    }
}
```

### 11.4.11 獲取場景使用統計

**端點**: `GET /portal/api/bridge/scenarios/{scenarioId}/usage`

**描述**: 獲取場景的使用統計資料

**回應範例**:
```json
{
    "success": true,
    "data": {
        "period": {
            "start": "2024-01-01T00:00:00Z",
            "end": "2024-01-15T23:59:59Z"
        },
        "totalExecutions": 152,
        "successfulExecutions": 150,
        "failedExecutions": 2,
        "operationsUsed": 456,
        "averageDuration": 1.5
    }
}
```

### 11.4.12 刪除場景

**端點**: `DELETE /v1/scenarios/{scenario_id}`

**描述**: 刪除場景

**回應範例**:
```json
{
    "success": true,
    "data": {
        "scenario_id": "scn_456",
        "deleted_at": "2024-01-15T12:30:00Z"
    }
}
```

## 11.5 連接 API

### 11.5.1 獲取連接列表

**端點**: `GET /v1/connections`

**描述**: 獲取使用者的連接列表

**查詢參數**:
- `team_id`: 團隊 ID (必填)
- `service`: 服務名稱 (如 slack, google, etc.)
- `status`: 連接狀態

**回應範例**:
```json
{
    "success": true,
    "data": {
        "connections": [
            {
                "connection_id": "conn_123",
                "name": "我的 Slack 工作區",
                "service": "slack",
                "status": "active",
                "created_at": "2024-01-10T09:00:00Z",
                "expires_at": null,
                "scopes": ["chat:write", "channels:read"]
            }
        ]
    }
}
```

### 11.5.2 建立連接

**端點**: `POST /v1/connections`

**描述**: 建立新的服務連接

**請求參數**:
```json
{
    "service": "slack",
    "name": "新的 Slack 連接",
    "team_id": 12345,
    "auth_data": {
        "access_token": "xoxb-...",
        "workspace_id": "T12345"
    }
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "connection_id": "conn_456",
        "name": "新的 Slack 連接",
        "service": "slack",
        "status": "active",
        "created_at": "2024-01-15T13:00:00Z"
    }
}
```

### 11.5.3 測試連接

**端點**: `POST /v1/connections/{connection_id}/test`

**描述**: 測試連接是否正常

**回應範例**:
```json
{
    "success": true,
    "data": {
        "connection_id": "conn_123",
        "status": "success",
        "tested_at": "2024-01-15T13:15:00Z",
        "details": {
            "workspace_name": "My Workspace",
            "user": "integration-bot"
        }
    }
}
```

### 11.5.4 更新連接

**端點**: `PUT /v1/connections/{connection_id}`

**描述**: 更新連接資訊

**請求參數**:
```json
{
    "name": "更新的連接名稱",
    "auth_data": {
        "access_token": "xoxb-new-token"
    }
}
```

### 11.5.5 刪除連接

**端點**: `DELETE /v1/connections/{connection_id}`

**描述**: 刪除連接

**回應範例**:
```json
{
    "success": true,
    "data": {
        "connection_id": "conn_123",
        "deleted_at": "2024-01-15T13:30:00Z",
        "affected_scenarios": ["scn_456", "scn_789"]
    }
}
```

## 11.6 Webhook API

### 11.6.1 獲取 Webhook 資訊

**端點**: `GET /v1/webhooks/{webhook_id}`

**描述**: 獲取 Webhook 的詳細資訊

**回應範例**:
```json
{
    "success": true,
    "data": {
        "webhook_id": "whk_123",
        "url": "https://hook.make.com/abc123xyz",
        "scenario_id": "scn_456",
        "status": "active",
        "created_at": "2024-01-15T11:45:00Z",
        "last_triggered": "2024-01-15T13:45:00Z",
        "trigger_count": 25
    }
}
```

### 11.6.2 觸發 Webhook

**端點**: `POST /webhooks/{webhook_path}`

**描述**: 觸發 Webhook (此端點不需要認證)

**請求參數**:
```json
{
    "event": "user.created",
    "data": {
        "user_id": "usr_123",
        "email": "user@example.com",
        "name": "John Doe"
    }
}
```

**回應範例**:
```json
{
    "success": true,
    "message": "Webhook received",
    "execution_id": "exec_999"
}
```

### 11.6.3 獲取 Webhook 歷史

**端點**: `GET /v1/webhooks/{webhook_id}/history`

**描述**: 獲取 Webhook 的觸發歷史

**查詢參數**:
- `start_date`: 開始日期
- `end_date`: 結束日期
- `status`: 狀態篩選
- `page`: 頁碼
- `limit`: 每頁數量

**回應範例**:
```json
{
    "success": true,
    "data": {
        "history": [
            {
                "trigger_id": "trg_456",
                "triggered_at": "2024-01-15T13:45:00Z",
                "source_ip": "192.168.1.100",
                "payload_size": "2.5 KB",
                "execution_id": "exec_999",
                "status": "success"
            }
        ],
        "pagination": {
            "current_page": 1,
            "total_pages": 3,
            "total_items": 25
        }
    }
}
```

## 11.7 客戶管理 API

### 11.7.1 獲取客戶列表

**端點**: `GET /portal/api/bridge/customers`

**描述**: 獲取 Bridge 應用的客戶列表

**查詢參數**:
- `page`: 頁碼 (預設: 1)
- `limit`: 每頁數量 (預設: 20, 最大: 100)
- `search`: 搜尋關鍵字
- `status`: 客戶狀態 (active, inactive)

**回應範例**:
```json
{
    "success": true,
    "data": {
        "customers": [
            {
                "customerId": "cust_123",
                "email": "customer@example.com",
                "name": "John Doe",
                "status": "active",
                "operationsLimit": 10000,
                "operationsUsed": 2500,
                "scenarios": [
                    {
                        "scenarioId": "scn_456",
                        "name": "My Integration",
                        "status": "active"
                    }
                ],
                "createdAt": "2024-01-10T08:00:00Z",
                "lastActivity": "2024-01-15T14:30:00Z"
            }
        ],
        "pagination": {
            "currentPage": 1,
            "totalPages": 5,
            "totalItems": 95,
            "itemsPerPage": 20
        }
    }
}
```

### 11.7.2 獲取特定客戶詳情

**端點**: `GET /portal/api/bridge/customers/{customerId}`

**描述**: 獲取特定客戶的詳細資訊

**回應範例**:
```json
{
    "success": true,
    "data": {
        "customerId": "cust_123",
        "email": "customer@example.com",
        "name": "John Doe",
        "status": "active",
        "operationsLimit": 10000,
        "operationsUsed": 2500,
        "usageHistory": [
            {
                "date": "2024-01-15",
                "operations": 250
            }
        ],
        "scenarios": [
            {
                "scenarioId": "scn_456",
                "name": "My Integration",
                "status": "active",
                "lastRun": "2024-01-15T14:30:00Z"
            }
        ],
        "createdAt": "2024-01-10T08:00:00Z"
    }
}
```

### 11.7.3 更新客戶資訊

**端點**: `PUT /portal/api/bridge/customers/{customerId}`

**描述**: 更新客戶的基本資訊和限制

**請求參數**:
```json
{
    "name": "Updated Name",
    "operationsLimit": 15000,
    "status": "active"
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "customerId": "cust_123",
        "updatedFields": ["name", "operationsLimit"],
        "updatedAt": "2024-01-15T15:00:00Z"
    }
}
```

## 11.8 團隊與權限 API

### 11.7.1 獲取團隊資訊

**端點**: `GET /v1/teams/{team_id}`

**描述**: 獲取團隊詳細資訊

**回應範例**:
```json
{
    "success": true,
    "data": {
        "team_id": 12345,
        "name": "My Team",
        "created_at": "2024-01-01T00:00:00Z",
        "members_count": 5,
        "plan": {
            "name": "Professional",
            "operations_limit": 10000,
            "operations_used": 3500
        }
    }
}
```

### 11.7.2 獲取團隊成員

**端點**: `GET /v1/teams/{team_id}/members`

**描述**: 獲取團隊成員列表

**回應範例**:
```json
{
    "success": true,
    "data": {
        "members": [
            {
                "user_id": "usr_123",
                "name": "John Doe",
                "email": "john@example.com",
                "role": "admin",
                "joined_at": "2024-01-05T10:00:00Z"
            }
        ]
    }
}
```

### 11.7.3 邀請團隊成員

**端點**: `POST /v1/teams/{team_id}/invitations`

**描述**: 邀請新成員加入團隊

**請求參數**:
```json
{
    "email": "newmember@example.com",
    "role": "member",
    "message": "歡迎加入我們的團隊！"
}
```

**回應範例**:
```json
{
    "success": true,
    "data": {
        "invitation_id": "inv_789",
        "email": "newmember@example.com",
        "status": "pending",
        "expires_at": "2024-01-22T14:00:00Z"
    }
}
```

## 11.8 資料與分析 API

### 11.8.1 獲取使用統計

**端點**: `GET /v1/analytics/usage`

**描述**: 獲取使用統計資料

**查詢參數**:
- `team_id`: 團隊 ID
- `start_date`: 開始日期
- `end_date`: 結束日期
- `granularity`: 資料粒度 (hour, day, week, month)

**回應範例**:
```json
{
    "success": true,
    "data": {
        "period": {
            "start": "2024-01-01T00:00:00Z",
            "end": "2024-01-15T23:59:59Z"
        },
        "statistics": {
            "total_operations": 3500,
            "successful_executions": 3450,
            "failed_executions": 50,
            "active_scenarios": 15,
            "data_processed": "125.5 MB"
        },
        "timeline": [
            {
                "date": "2024-01-15",
                "operations": 250,
                "executions": 45,
                "errors": 2
            }
        ]
    }
}
```

### 11.8.2 獲取場景分析

**端點**: `GET /v1/analytics/scenarios/{scenario_id}`

**描述**: 獲取特定場景的分析資料

**回應範例**:
```json
{
    "success": true,
    "data": {
        "scenario_id": "scn_456",
        "period": {
            "start": "2024-01-01T00:00:00Z",
            "end": "2024-01-15T23:59:59Z"
        },
        "performance": {
            "average_duration": 1.5,
            "min_duration": 0.8,
            "max_duration": 3.2,
            "success_rate": 98.5
        },
        "modules_performance": [
            {
                "module_id": "mod_1",
                "module_name": "Webhook",
                "average_duration": 0.2,
                "operations_count": 450
            }
        ]
    }
}
```

## 11.9 錯誤碼參考

### 11.9.1 通用錯誤碼

| 錯誤碼 | HTTP 狀態碼 | 描述 |
|--------|------------|------|
| `UNAUTHORIZED` | 401 | 未提供認證資訊或認證失敗 |
| `FORBIDDEN` | 403 | 無權限執行此操作 |
| `NOT_FOUND` | 404 | 請求的資源不存在 |
| `RATE_LIMITED` | 429 | 超過速率限制 |
| `INTERNAL_ERROR` | 500 | 內部伺服器錯誤 |
| `SERVICE_UNAVAILABLE` | 503 | 服務暫時不可用 |

### 11.9.2 業務錯誤碼

| 錯誤碼 | HTTP 狀態碼 | 描述 |
|--------|------------|------|
| `INVALID_TEMPLATE` | 400 | 無效的模板配置 |
| `SCENARIO_LIMIT_EXCEEDED` | 402 | 超過場景數量限制 |
| `OPERATIONS_LIMIT_EXCEEDED` | 402 | 超過操作數限制 |
| `INVALID_WEBHOOK_PAYLOAD` | 400 | 無效的 Webhook 資料 |
| `CONNECTION_FAILED` | 400 | 連接建立失敗 |
| `EXECUTION_TIMEOUT` | 408 | 執行超時 |

## 11.10 速率限制

### 11.10.1 限制規則

- 一般 API: 1000 請求/小時
- 建立資源 API: 100 請求/小時
- 執行場景 API: 根據訂閱計劃而定

### 11.10.2 速率限制標頭

所有 API 回應都包含以下標頭：

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1642284000
```

## 11.11 SDK 範例

### 11.11.1 JavaScript SDK

```javascript
const MakeBridge = require('@make/bridge-sdk');

const bridge = new MakeBridge({
    apiKey: 'your-api-key',
    appId: 'your-app-id'
});

// 初始化 Bridge
const session = await bridge.init({
    userId: 'user_123',
    config: {
        theme: {
            primaryColor: '#007bff'
        }
    }
});

// 建立場景
const scenario = await bridge.scenarios.create({
    templateId: 'tpl_123',
    name: 'My Automation',
    teamId: 12345
});

// 執行場景
const execution = await bridge.scenarios.run(scenario.id, {
    data: {
        message: 'Hello World'
    }
});
```

### 11.11.2 Python SDK

```python
from make_bridge import MakeBridge

bridge = MakeBridge(
    api_key='your-api-key',
    app_id='your-app-id'
)

# 初始化 Bridge
session = bridge.init(
    user_id='user_123',
    config={
        'theme': {
            'primary_color': '#007bff'
        }
    }
)

# 建立場景
scenario = bridge.scenarios.create(
    template_id='tpl_123',
    name='My Automation',
    team_id=12345
)

# 執行場景
execution = bridge.scenarios.run(
    scenario_id=scenario['id'],
    data={
        'message': 'Hello World'
    }
)
```

## 總結

本章提供了 Make Bridge 完整的 API 參考文檔，涵蓋所有核心功能的端點、參數和回應格式。開發者可以根據這些資訊整合 Make Bridge 到自己的應用程式中。

## 下一章預告

在第十二章中，我們將探討疑難排解與維護，幫助你解決常見問題並確保系統穩定運行。