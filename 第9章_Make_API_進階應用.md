# 第9章：Make API 進階應用

當 Make Bridge 的預建功能無法滿足您的特殊需求時，直接使用 Make API 可以獲得最大的控制權和靈活性。本章將深入探討 Make API 的進階功能，幫助您建立完全客製化的自動化解決方案。

> 🎯 **低代碼友善提示**
> 
> 本章內容較為技術性，主要針對需要深度客製化的場景。如果您是初學者，建議先熟悉前幾章的 Make Bridge 方法，它能滿足大多數需求且更容易上手。

## 9.1 Make API 架構概覽

### 🏗️ API 架構設計

> 🔗 **API 基礎概念**
> 
> - **API (Application Programming Interface)**：應用程式之間溝通的橋樑，就像餐廳的服務生，負責在客人（您的應用）和廚房（Make 平台）之間傳遞訊息
> - **RESTful API**：一種 API 設計風格，使用標準的 HTTP 方法（GET、POST、PUT、DELETE）來執行不同操作
> - **程式化控制**：用程式碼來自動執行原本需要手動操作的任務

**📝 基於原文檔頁4、13-16：**Make API 提供了完整的 RESTful API 介面，讓您能夠程式化地控制 Make 平台的所有功能。

#### API 層級結構

```
Make API
├── Authentication Layer (認證層)
│   ├── API Token Management
│   ├── JWT Token Generation
│   └── OAuth 2.0 Support
├── Core APIs (核心 API)
│   ├── Organizations API
│   ├── Teams API
│   ├── Scenarios API
│   └── Templates API
├── Bridge APIs (Bridge 專用 API)
│   ├── Integrations API
│   ├── Instances API
│   └── Portal API
└── Supporting APIs (支援 API)
    ├── Connections API
    ├── Webhooks API
    ├── Data Stores API
    └── Keys API
```

### 🔐 進階認證機制

> 🔑 **認證方法比較**
> 
> - **API Key**：就像是一把長期有效的萬能鑰匙，適合伺服器對伺服器的通訊
> - **JWT Token**：像是有時效性的臨時通行證，安全性較高但需要定期更新
> - **OAuth 2.0**：像是委託授權機制，讓第三方應用能代表用戶執行特定操作
> - **Scopes**：權限範圍限制，就像是鑰匙只能開特定的門

#### API Token 管理系統

```javascript
// services/MakeAPIAuthService.js - 進階認證服務
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const NodeCache = require('node-cache');

class MakeAPIAuthService {
    constructor(config) {
        this.config = config;
        this.tokenCache = new NodeCache({ stdTTL: 110 }); // 快取 110 秒
        this.refreshTokens = new Map();
        
        // 初始化 OAuth 客戶端
        this.oauthClient = this.initializeOAuthClient();
    }

    /**
     * 多重認證策略管理
     */
    async authenticate(method = 'api_key', credentials) {
        switch (method) {
            case 'api_key':
                return this.authenticateWithAPIKey(credentials);
            case 'jwt':
                return this.authenticateWithJWT(credentials);
            case 'oauth':
                return this.authenticateWithOAuth(credentials);
            default:
                throw new Error(`不支援的認證方式: ${method}`);
        }
    }

    /**
     * API Key 認證（長期有效）
     */
    async authenticateWithAPIKey(credentials) {
        const { apiKey, scopes } = credentials;
        
        // 驗證 API Key
        const validation = await this.validateAPIKey(apiKey);
        if (!validation.valid) {
            throw new Error('無效的 API Key');
        }

        // 檢查權限範圍
        const hasRequiredScopes = this.checkScopes(validation.scopes, scopes);
        if (!hasRequiredScopes) {
            throw new Error('權限不足');
        }

        return {
            type: 'Bearer',
            token: apiKey,
            scopes: validation.scopes,
            expiresIn: null // API Key 不過期
        };
    }

    /**
     * JWT 認證（短期有效）
     */
    async authenticateWithJWT(credentials) {
        const { userId, scopes, expiresIn = '2m' } = credentials;
        
        // 檢查快取
        const cacheKey = `jwt_${userId}_${scopes.join('_')}`;
        const cachedToken = this.tokenCache.get(cacheKey);
        if (cachedToken) {
            return cachedToken;
        }

        // 生成新 Token
        const payload = {
            sub: userId,
            jti: crypto.randomUUID(),
            scopes: scopes,
            iat: Math.floor(Date.now() / 1000)
        };

        const token = jwt.sign(payload, this.config.jwtSecret, {
            expiresIn,
            keyid: this.config.keyId,
            algorithm: 'HS256'
        });

        const result = {
            type: 'Bearer',
            token,
            scopes,
            expiresIn: this.parseExpiresIn(expiresIn)
        };

        // 快取 Token
        this.tokenCache.set(cacheKey, result);

        return result;
    }

    /**
     * OAuth 2.0 認證流程
     */
    async authenticateWithOAuth(credentials) {
        const { code, redirectUri } = credentials;

        try {
            // 交換授權碼取得 Access Token
            const tokenResponse = await this.oauthClient.getToken({
                code,
                redirect_uri: redirectUri
            });

            const { access_token, refresh_token, expires_in, scope } = tokenResponse;

            // 儲存 Refresh Token
            if (refresh_token) {
                this.refreshTokens.set(access_token, refresh_token);
            }

            return {
                type: 'Bearer',
                token: access_token,
                refreshToken: refresh_token,
                scopes: scope.split(' '),
                expiresIn: expires_in
            };

        } catch (error) {
            throw new Error(`OAuth 認證失敗: ${error.message}`);
        }
    }

    /**
     * 自動更新 Token
     */
    async refreshAccessToken(expiredToken) {
        const refreshToken = this.refreshTokens.get(expiredToken);
        if (!refreshToken) {
            throw new Error('找不到 Refresh Token');
        }

        try {
            const tokenResponse = await this.oauthClient.refreshToken(refreshToken);
            const { access_token, expires_in } = tokenResponse;

            // 更新 Token 映射
            this.refreshTokens.delete(expiredToken);
            this.refreshTokens.set(access_token, refreshToken);

            return {
                type: 'Bearer',
                token: access_token,
                refreshToken: refreshToken,
                expiresIn: expires_in
            };

        } catch (error) {
            // Refresh Token 也過期了，需要重新授權
            this.refreshTokens.delete(expiredToken);
            throw new Error('需要重新授權');
        }
    }

    /**
     * 驗證 API Key
     */
    async validateAPIKey(apiKey) {
        // 這裡應該呼叫 Make API 來驗證
        // 為了示範，使用模擬邏輯
        const mockValidation = {
            valid: apiKey.startsWith('mk_'),
            scopes: ['scenarios:read', 'scenarios:write', 'templates:read'],
            userId: 'user_123',
            organizationId: 'org_456'
        };

        return mockValidation;
    }

    /**
     * 檢查權限範圍
     */
    checkScopes(availableScopes, requiredScopes) {
        if (!requiredScopes || requiredScopes.length === 0) {
            return true;
        }

        // 檢查是否有通配符權限
        if (availableScopes.includes('*')) {
            return true;
        }

        // 檢查每個必需的權限
        return requiredScopes.every(required => {
            // 直接匹配
            if (availableScopes.includes(required)) {
                return true;
            }

            // 檢查通配符匹配 (例如 scenarios:* 匹配 scenarios:read)
            const [resource] = required.split(':');
            return availableScopes.some(scope => 
                scope === `${resource}:*` || scope === '*'
            );
        });
    }

    /**
     * 建立認證請求頭
     */
    createAuthHeaders(authResult) {
        return {
            'Authorization': `${authResult.type} ${authResult.token}`,
            'X-Make-Organization-ID': this.config.organizationId,
            'X-Make-Team-ID': this.config.teamId
        };
    }

    /**
     * 初始化 OAuth 客戶端
     */
    initializeOAuthClient() {
        // 實際應用中使用如 simple-oauth2 等套件
        return {
            getToken: async (params) => {
                // 模擬 OAuth token 交換
                return {
                    access_token: 'oauth_access_token_' + Date.now(),
                    refresh_token: 'oauth_refresh_token_' + Date.now(),
                    expires_in: 3600,
                    scope: 'scenarios:read scenarios:write'
                };
            },
            refreshToken: async (refreshToken) => {
                // 模擬 token 更新
                return {
                    access_token: 'oauth_access_token_refreshed_' + Date.now(),
                    expires_in: 3600
                };
            }
        };
    }

    /**
     * 解析過期時間
     */
    parseExpiresIn(expiresIn) {
        if (typeof expiresIn === 'number') {
            return expiresIn;
        }

        const matches = expiresIn.match(/(\d+)([smhd])/);
        if (!matches) {
            return 120; // 預設 2 分鐘
        }

        const value = parseInt(matches[1]);
        const unit = matches[2];

        const multipliers = {
            's': 1,
            'm': 60,
            'h': 3600,
            'd': 86400
        };

        return value * (multipliers[unit] || 1);
    }
}

module.exports = MakeAPIAuthService;
```

### 🔄 API 請求管理器

```javascript
// services/MakeAPIClient.js - 進階 API 客戶端
const axios = require('axios');
const axiosRetry = require('axios-retry');
const pLimit = require('p-limit');
const EventEmitter = require('events');

class MakeAPIClient extends EventEmitter {
    constructor(config) {
        super();
        this.config = config;
        this.authService = new MakeAPIAuthService(config);
        
        // 初始化 HTTP 客戶端
        this.client = this.createHttpClient();
        
        // 速率限制器
        this.rateLimiter = pLimit(config.maxConcurrent || 10);
        
        // 請求統計
        this.stats = {
            totalRequests: 0,
            successfulRequests: 0,
            failedRequests: 0,
            retries: 0
        };
    }

    /**
     * 建立 HTTP 客戶端
     */
    createHttpClient() {
        const client = axios.create({
            baseURL: this.config.baseURL || 'https://eu2.make.com',
            timeout: this.config.timeout || 30000,
            headers: {
                'Content-Type': 'application/json',
                'User-Agent': `MakeAPIClient/1.0 (${this.config.appName || 'CustomApp'})`
            }
        });

        // 設定重試邏輯
        axiosRetry(client, {
            retries: 3,
            retryDelay: axiosRetry.exponentialDelay,
            retryCondition: (error) => {
                // 重試 5xx 錯誤和網路錯誤
                return axiosRetry.isNetworkOrIdempotentRequestError(error) ||
                       (error.response && error.response.status >= 500);
            },
            onRetry: (retryCount, error) => {
                this.stats.retries++;
                this.emit('retry', { retryCount, error });
                console.log(`重試請求 (${retryCount}/3):`, error.config.url);
            }
        });

        // 請求攔截器
        client.interceptors.request.use(
            async (config) => {
                // 添加認證
                const auth = await this.getAuthentication();
                Object.assign(config.headers, this.authService.createAuthHeaders(auth));
                
                // 添加請求 ID
                config.headers['X-Request-ID'] = this.generateRequestId();
                
                // 記錄請求
                this.emit('request', config);
                
                return config;
            },
            (error) => {
                this.emit('request-error', error);
                return Promise.reject(error);
            }
        );

        // 回應攔截器
        client.interceptors.response.use(
            (response) => {
                this.stats.successfulRequests++;
                this.emit('response', response);
                
                // 處理分頁資訊
                if (response.headers['x-pagination-total']) {
                    response.data._pagination = {
                        total: parseInt(response.headers['x-pagination-total']),
                        page: parseInt(response.headers['x-pagination-page'] || '1'),
                        limit: parseInt(response.headers['x-pagination-limit'] || '100')
                    };
                }
                
                return response;
            },
            async (error) => {
                this.stats.failedRequests++;
                
                // 處理 401 錯誤（Token 過期）
                if (error.response && error.response.status === 401) {
                    try {
                        // 嘗試更新 Token
                        await this.refreshAuthentication();
                        
                        // 重新發送原始請求
                        const originalRequest = error.config;
                        const auth = await this.getAuthentication();
                        Object.assign(originalRequest.headers, 
                            this.authService.createAuthHeaders(auth));
                        
                        return this.client.request(originalRequest);
                    } catch (refreshError) {
                        this.emit('auth-error', refreshError);
                        throw refreshError;
                    }
                }
                
                // 處理速率限制
                if (error.response && error.response.status === 429) {
                    const retryAfter = parseInt(error.response.headers['retry-after'] || '60');
                    this.emit('rate-limit', { retryAfter });
                    
                    // 自動等待並重試
                    await this.sleep(retryAfter * 1000);
                    return this.client.request(error.config);
                }
                
                this.emit('response-error', error);
                throw this.enhanceError(error);
            }
        );

        return client;
    }

    /**
     * 執行 API 請求
     */
    async request(options) {
        this.stats.totalRequests++;
        
        // 使用速率限制器
        return this.rateLimiter(async () => {
            try {
                const response = await this.client.request(options);
                return response.data;
            } catch (error) {
                throw error;
            }
        });
    }

    /**
     * GET 請求
     */
    async get(path, params = {}, options = {}) {
        return this.request({
            method: 'GET',
            url: path,
            params,
            ...options
        });
    }

    /**
     * POST 請求
     */
    async post(path, data = {}, options = {}) {
        return this.request({
            method: 'POST',
            url: path,
            data,
            ...options
        });
    }

    /**
     * PUT 請求
     */
    async put(path, data = {}, options = {}) {
        return this.request({
            method: 'PUT',
            url: path,
            data,
            ...options
        });
    }

    /**
     * DELETE 請求
     */
    async delete(path, options = {}) {
        return this.request({
            method: 'DELETE',
            url: path,
            ...options
        });
    }

    /**
     * 批次請求
     */
    async batch(requests) {
        const batchId = this.generateRequestId();
        this.emit('batch-start', { batchId, count: requests.length });
        
        const results = await Promise.allSettled(
            requests.map(req => this.request(req))
        );
        
        const successful = results.filter(r => r.status === 'fulfilled');
        const failed = results.filter(r => r.status === 'rejected');
        
        this.emit('batch-complete', {
            batchId,
            total: results.length,
            successful: successful.length,
            failed: failed.length
        });
        
        return {
            results,
            successful: successful.map(r => r.value),
            failed: failed.map(r => r.reason),
            summary: {
                total: results.length,
                successful: successful.length,
                failed: failed.length
            }
        };
    }

    /**
     * 分頁資料獲取
     */
    async *paginate(path, params = {}, options = {}) {
        let page = 1;
        let hasMore = true;
        
        while (hasMore) {
            const response = await this.get(path, {
                ...params,
                page,
                limit: params.limit || 100
            }, options);
            
            yield response;
            
            // 檢查是否有更多資料
            if (response._pagination) {
                const { total, limit } = response._pagination;
                hasMore = page * limit < total;
                page++;
            } else {
                hasMore = false;
            }
        }
    }

    /**
     * 取得所有分頁資料
     */
    async getAllPages(path, params = {}, options = {}) {
        const allData = [];
        
        for await (const page of this.paginate(path, params, options)) {
            if (Array.isArray(page)) {
                allData.push(...page);
            } else if (page.data && Array.isArray(page.data)) {
                allData.push(...page.data);
            } else {
                allData.push(page);
            }
        }
        
        return allData;
    }

    /**
     * 檔案上傳
     */
    async uploadFile(path, file, additionalData = {}) {
        const FormData = require('form-data');
        const formData = new FormData();
        
        // 添加檔案
        formData.append('file', file.buffer, {
            filename: file.originalname,
            contentType: file.mimetype
        });
        
        // 添加其他資料
        Object.entries(additionalData).forEach(([key, value]) => {
            formData.append(key, value);
        });
        
        return this.request({
            method: 'POST',
            url: path,
            data: formData,
            headers: formData.getHeaders()
        });
    }

    /**
     * WebSocket 連接
     */
    connectWebSocket(path, options = {}) {
        const WebSocket = require('ws');
        const wsUrl = this.config.baseURL.replace('https://', 'wss://') + path;
        
        const ws = new WebSocket(wsUrl, {
            headers: this.authService.createAuthHeaders(this.currentAuth)
        });
        
        ws.on('open', () => {
            this.emit('ws-open', { path });
        });
        
        ws.on('message', (data) => {
            try {
                const message = JSON.parse(data);
                this.emit('ws-message', message);
            } catch (error) {
                this.emit('ws-error', error);
            }
        });
        
        ws.on('error', (error) => {
            this.emit('ws-error', error);
        });
        
        ws.on('close', (code, reason) => {
            this.emit('ws-close', { code, reason });
        });
        
        return ws;
    }

    /**
     * 健康檢查
     */
    async healthCheck() {
        try {
            const response = await this.get('/api/health');
            return {
                healthy: true,
                ...response
            };
        } catch (error) {
            return {
                healthy: false,
                error: error.message
            };
        }
    }

    /**
     * 取得 API 使用統計
     */
    getStats() {
        return {
            ...this.stats,
            successRate: this.stats.totalRequests > 0 
                ? (this.stats.successfulRequests / this.stats.totalRequests * 100).toFixed(2) + '%'
                : '0%',
            errorRate: this.stats.totalRequests > 0
                ? (this.stats.failedRequests / this.stats.totalRequests * 100).toFixed(2) + '%'
                : '0%'
        };
    }

    /**
     * 重設統計資料
     */
    resetStats() {
        this.stats = {
            totalRequests: 0,
            successfulRequests: 0,
            failedRequests: 0,
            retries: 0
        };
    }

    /**
     * 取得認證資訊
     */
    async getAuthentication() {
        if (!this.currentAuth || this.isAuthExpired()) {
            this.currentAuth = await this.authService.authenticate(
                this.config.authMethod,
                this.config.authCredentials
            );
        }
        return this.currentAuth;
    }

    /**
     * 更新認證
     */
    async refreshAuthentication() {
        if (this.currentAuth && this.currentAuth.refreshToken) {
            this.currentAuth = await this.authService.refreshAccessToken(
                this.currentAuth.token
            );
        } else {
            this.currentAuth = await this.authService.authenticate(
                this.config.authMethod,
                this.config.authCredentials
            );
        }
        return this.currentAuth;
    }

    /**
     * 檢查認證是否過期
     */
    isAuthExpired() {
        if (!this.currentAuth || !this.currentAuth.expiresIn) {
            return false;
        }
        
        const expirationTime = this.authExpirationTime || 
            (Date.now() + this.currentAuth.expiresIn * 1000);
        
        return Date.now() >= expirationTime - 30000; // 提前 30 秒更新
    }

    /**
     * 增強錯誤資訊
     */
    enhanceError(error) {
        if (error.response) {
            const { status, statusText, data } = error.response;
            
            error.message = `API 錯誤 ${status}: ${statusText}`;
            error.statusCode = status;
            error.responseData = data;
            
            // 解析 Make API 錯誤格式
            if (data && data.error) {
                error.message = data.error.message || data.error;
                error.errorCode = data.error.code;
                error.errorDetails = data.error.details;
            }
        }
        
        return error;
    }

    /**
     * 生成請求 ID
     */
    generateRequestId() {
        return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }

    /**
     * 延遲執行
     */
    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

module.exports = MakeAPIClient;
```

## 9.2 核心 API 功能實作

### 📋 Scenarios API 進階操作

```javascript
// services/ScenariosAPIService.js - Scenarios 進階管理
class ScenariosAPIService {
    constructor(apiClient) {
        this.client = apiClient;
    }

    /**
     * 建立複雜的 Scenario
     */
    async createAdvancedScenario(scenarioData) {
        try {
            // 驗證 Scenario 結構
            this.validateScenarioStructure(scenarioData);
            
            // 預處理模組設定
            const processedData = this.preprocessScenarioData(scenarioData);
            
            // 建立 Scenario
            const scenario = await this.client.post('/api/v2/scenarios', processedData);
            
            // 設定進階功能
            if (scenarioData.advancedFeatures) {
                await this.configureAdvancedFeatures(scenario.id, scenarioData.advancedFeatures);
            }
            
            // 設定錯誤處理
            if (scenarioData.errorHandling) {
                await this.configureErrorHandling(scenario.id, scenarioData.errorHandling);
            }
            
            return scenario;
            
        } catch (error) {
            console.error('建立進階 Scenario 失敗:', error);
            throw error;
        }
    }

    /**
     * 複製 Scenario 並客製化
     */
    async cloneAndCustomize(sourceScenarioId, customizations) {
        try {
            // 取得原始 Scenario
            const sourceScenario = await this.client.get(`/api/v2/scenarios/${sourceScenarioId}`);
            
            // 複製基本結構
            const clonedData = this.deepClone(sourceScenario);
            delete clonedData.id;
            delete clonedData.createdAt;
            delete clonedData.updatedAt;
            
            // 應用客製化
            const customizedData = this.applyCustomizations(clonedData, customizations);
            
            // 建立新 Scenario
            const newScenario = await this.client.post('/api/v2/scenarios', customizedData);
            
            // 複製連接設定
            if (customizations.copyConnections) {
                await this.copyConnections(sourceScenarioId, newScenario.id);
            }
            
            return newScenario;
            
        } catch (error) {
            console.error('複製 Scenario 失敗:', error);
            throw error;
        }
    }

    /**
     * 批次更新多個 Scenarios
     */
    async batchUpdateScenarios(updates) {
        const results = {
            successful: [],
            failed: []
        };

        for (const update of updates) {
            try {
                const result = await this.updateScenario(update.scenarioId, update.changes);
                results.successful.push({
                    scenarioId: update.scenarioId,
                    result
                });
            } catch (error) {
                results.failed.push({
                    scenarioId: update.scenarioId,
                    error: error.message
                });
            }
        }

        return results;
    }

    /**
     * 進階執行控制
     */
    async executeWithControl(scenarioId, executionOptions) {
        const {
            inputData,
            executionMode = 'normal', // normal, test, debug
            breakpoints = [],
            timeout = 300000, // 5 分鐘
            priority = 'normal', // low, normal, high
            retryConfig
        } = executionOptions;

        try {
            // 建立執行實例
            const execution = await this.client.post(`/api/v2/scenarios/${scenarioId}/executions`, {
                mode: executionMode,
                input: inputData,
                priority,
                config: {
                    breakpoints,
                    timeout,
                    retry: retryConfig
                }
            });

            // 如果是 debug 模式，建立 WebSocket 連接監聽
            if (executionMode === 'debug') {
                const debugSession = await this.createDebugSession(execution.id);
                return { execution, debugSession };
            }

            // 等待執行完成
            const result = await this.waitForExecution(execution.id, timeout);
            
            return result;

        } catch (error) {
            console.error('執行 Scenario 失敗:', error);
            throw error;
        }
    }

    /**
     * 建立除錯會話
     */
    async createDebugSession(executionId) {
        const ws = this.client.connectWebSocket(`/ws/executions/${executionId}/debug`);
        
        return new Promise((resolve, reject) => {
            const debugInfo = {
                steps: [],
                variables: {},
                errors: []
            };

            ws.on('message', (data) => {
                const message = JSON.parse(data);
                
                switch (message.type) {
                    case 'step':
                        debugInfo.steps.push(message.data);
                        break;
                    case 'variable':
                        debugInfo.variables[message.data.name] = message.data.value;
                        break;
                    case 'error':
                        debugInfo.errors.push(message.data);
                        break;
                    case 'complete':
                        ws.close();
                        resolve(debugInfo);
                        break;
                }
            });

            ws.on('error', reject);
        });
    }

    /**
     * 等待執行完成
     */
    async waitForExecution(executionId, timeout) {
        const startTime = Date.now();
        
        while (Date.now() - startTime < timeout) {
            const execution = await this.client.get(`/api/v2/executions/${executionId}`);
            
            if (execution.status === 'success' || execution.status === 'error') {
                return execution;
            }
            
            // 等待 2 秒後再次檢查
            await this.sleep(2000);
        }
        
        throw new Error('執行超時');
    }

    /**
     * 分析 Scenario 效能
     */
    async analyzePerformance(scenarioId, timeRange = '7d') {
        try {
            // 取得執行歷史
            const executions = await this.client.get(`/api/v2/scenarios/${scenarioId}/executions`, {
                timeRange,
                limit: 1000
            });

            // 計算統計資料
            const stats = {
                totalExecutions: executions.length,
                successRate: 0,
                averageDuration: 0,
                peakHours: {},
                errorPatterns: {},
                modulePerformance: {}
            };

            // 分析成功率
            const successful = executions.filter(e => e.status === 'success').length;
            stats.successRate = (successful / executions.length * 100).toFixed(2);

            // 分析執行時間
            const durations = executions.map(e => e.duration).filter(d => d);
            stats.averageDuration = durations.reduce((a, b) => a + b, 0) / durations.length;

            // 分析尖峰時段
            executions.forEach(execution => {
                const hour = new Date(execution.startedAt).getHours();
                stats.peakHours[hour] = (stats.peakHours[hour] || 0) + 1;
            });

            // 分析錯誤模式
            const errors = executions.filter(e => e.status === 'error');
            errors.forEach(error => {
                const errorType = error.error?.type || 'unknown';
                stats.errorPatterns[errorType] = (stats.errorPatterns[errorType] || 0) + 1;
            });

            // 分析模組效能
            const moduleStats = await this.analyzeModulePerformance(scenarioId, executions);
            stats.modulePerformance = moduleStats;

            return stats;

        } catch (error) {
            console.error('分析 Scenario 效能失敗:', error);
            throw error;
        }
    }

    /**
     * 分析模組效能
     */
    async analyzeModulePerformance(scenarioId, executions) {
        const moduleStats = {};

        for (const execution of executions.slice(0, 100)) { // 分析最近 100 次
            if (execution.status === 'success' && execution.id) {
                try {
                    const details = await this.client.get(`/api/v2/executions/${execution.id}/details`);
                    
                    details.modules.forEach(module => {
                        if (!moduleStats[module.name]) {
                            moduleStats[module.name] = {
                                executions: 0,
                                totalDuration: 0,
                                errors: 0
                            };
                        }
                        
                        moduleStats[module.name].executions++;
                        moduleStats[module.name].totalDuration += module.duration || 0;
                        if (module.error) {
                            moduleStats[module.name].errors++;
                        }
                    });
                } catch (error) {
                    // 忽略個別執行詳情的錯誤
                }
            }
        }

        // 計算平均值
        Object.keys(moduleStats).forEach(moduleName => {
            const stats = moduleStats[moduleName];
            stats.averageDuration = stats.totalDuration / stats.executions;
            stats.errorRate = (stats.errors / stats.executions * 100).toFixed(2);
        });

        return moduleStats;
    }

    /**
     * 匯出 Scenario 為 JSON
     */
    async exportScenario(scenarioId, options = {}) {
        try {
            const scenario = await this.client.get(`/api/v2/scenarios/${scenarioId}`, {
                include: 'modules,connections,webhooks'
            });

            const exportData = {
                version: '2.0',
                exportedAt: new Date().toISOString(),
                scenario: scenario
            };

            // 包含連接資訊（移除敏感資料）
            if (options.includeConnections) {
                exportData.connections = await this.exportConnections(scenario.connections);
            }

            // 包含相關資料結構
            if (options.includeDataStructures) {
                exportData.dataStructures = await this.exportDataStructures(scenario);
            }

            return exportData;

        } catch (error) {
            console.error('匯出 Scenario 失敗:', error);
            throw error;
        }
    }

    /**
     * 匯入 Scenario
     */
    async importScenario(exportData, options = {}) {
        try {
            // 驗證匯入資料
            this.validateImportData(exportData);

            // 處理連接映射
            const connectionMapping = {};
            if (exportData.connections && options.mapConnections) {
                connectionMapping = await this.mapConnections(exportData.connections);
            }

            // 準備匯入資料
            const importData = this.prepareImportData(exportData.scenario, connectionMapping);

            // 建立新 Scenario
            const newScenario = await this.client.post('/api/v2/scenarios', importData);

            // 匯入資料結構
            if (exportData.dataStructures && options.importDataStructures) {
                await this.importDataStructures(exportData.dataStructures);
            }

            return newScenario;

        } catch (error) {
            console.error('匯入 Scenario 失敗:', error);
            throw error;
        }
    }

    /**
     * 驗證 Scenario 結構
     */
    validateScenarioStructure(scenarioData) {
        const requiredFields = ['name', 'modules'];
        
        requiredFields.forEach(field => {
            if (!scenarioData[field]) {
                throw new Error(`缺少必要欄位: ${field}`);
            }
        });

        // 驗證模組結構
        if (!Array.isArray(scenarioData.modules) || scenarioData.modules.length === 0) {
            throw new Error('Scenario 必須包含至少一個模組');
        }

        // 驗證模組連接
        this.validateModuleConnections(scenarioData.modules);
    }

    /**
     * 驗證模組連接
     */
    validateModuleConnections(modules) {
        const moduleIds = new Set(modules.map(m => m.id));
        
        modules.forEach(module => {
            if (module.mapper && module.mapper.connections) {
                Object.values(module.mapper.connections).forEach(connection => {
                    if (!moduleIds.has(connection.moduleId)) {
                        throw new Error(`無效的模組連接: ${connection.moduleId}`);
                    }
                });
            }
        });
    }

    /**
     * 深度複製物件
     */
    deepClone(obj) {
        return JSON.parse(JSON.stringify(obj));
    }

    /**
     * 延遲執行
     */
    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

module.exports = ScenariosAPIService;
```

### 🔗 Connections API 管理

```javascript
// services/ConnectionsAPIService.js - 連接管理服務
class ConnectionsAPIService {
    constructor(apiClient) {
        this.client = apiClient;
        this.connectionCache = new Map();
    }

    /**
     * 建立安全連接
     */
    async createSecureConnection(connectionData) {
        const {
            name,
            type,
            credentials,
            encryptionKey,
            testConnection = true
        } = connectionData;

        try {
            // 加密敏感資料
            const encryptedCredentials = await this.encryptCredentials(
                credentials,
                encryptionKey
            );

            // 建立連接
            const connection = await this.client.post('/api/v2/connections', {
                name,
                type,
                credentials: encryptedCredentials,
                metadata: {
                    encrypted: true,
                    createdBy: connectionData.userId,
                    createdAt: new Date().toISOString()
                }
            });

            // 測試連接
            if (testConnection) {
                const testResult = await this.testConnection(connection.id);
                if (!testResult.success) {
                    // 如果測試失敗，刪除連接
                    await this.deleteConnection(connection.id);
                    throw new Error(`連接測試失敗: ${testResult.error}`);
                }
            }

            // 快取連接資訊
            this.connectionCache.set(connection.id, {
                ...connection,
                credentials: credentials // 儲存未加密版本供本地使用
            });

            return connection;

        } catch (error) {
            console.error('建立安全連接失敗:', error);
            throw error;
        }
    }

    /**
     * 批次建立連接
     */
    async batchCreateConnections(connections) {
        const results = {
            successful: [],
            failed: []
        };

        // 使用並行處理但限制並發數
        const chunks = this.chunkArray(connections, 5);
        
        for (const chunk of chunks) {
            const promises = chunk.map(async (conn) => {
                try {
                    const result = await this.createSecureConnection(conn);
                    results.successful.push({
                        name: conn.name,
                        id: result.id
                    });
                } catch (error) {
                    results.failed.push({
                        name: conn.name,
                        error: error.message
                    });
                }
            });

            await Promise.all(promises);
        }

        return results;
    }

    /**
     * 更新連接認證
     */
    async updateConnectionCredentials(connectionId, newCredentials, options = {}) {
        try {
            // 取得現有連接
            const connection = await this.getConnection(connectionId);

            // 備份舊認證
            if (options.backup) {
                await this.backupCredentials(connection);
            }

            // 加密新認證
            const encryptedCredentials = await this.encryptCredentials(
                newCredentials,
                options.encryptionKey
            );

            // 更新連接
            const updated = await this.client.put(`/api/v2/connections/${connectionId}`, {
                credentials: encryptedCredentials,
                metadata: {
                    ...connection.metadata,
                    lastUpdated: new Date().toISOString(),
                    updatedBy: options.userId
                }
            });

            // 測試新認證
            if (options.testConnection !== false) {
                const testResult = await this.testConnection(connectionId);
                if (!testResult.success) {
                    // 還原舊認證
                    if (options.backup) {
                        await this.restoreCredentials(connectionId);
                    }
                    throw new Error('新認證測試失敗');
                }
            }

            // 更新快取
            this.connectionCache.delete(connectionId);

            return updated;

        } catch (error) {
            console.error('更新連接認證失敗:', error);
            throw error;
        }
    }

    /**
     * 連接健康檢查
     */
    async healthCheckConnections(connectionIds = []) {
        const healthStatus = {
            healthy: [],
            unhealthy: [],
            summary: {
                total: 0,
                healthy: 0,
                unhealthy: 0
            }
        };

        // 如果沒有指定，檢查所有連接
        if (connectionIds.length === 0) {
            const allConnections = await this.getAllConnections();
            connectionIds = allConnections.map(c => c.id);
        }

        healthStatus.summary.total = connectionIds.length;

        // 並行測試連接
        const testPromises = connectionIds.map(async (id) => {
            try {
                const result = await this.testConnection(id);
                if (result.success) {
                    healthStatus.healthy.push({
                        id,
                        status: 'healthy',
                        responseTime: result.responseTime
                    });
                    healthStatus.summary.healthy++;
                } else {
                    healthStatus.unhealthy.push({
                        id,
                        status: 'unhealthy',
                        error: result.error,
                        lastError: result.timestamp
                    });
                    healthStatus.summary.unhealthy++;
                }
            } catch (error) {
                healthStatus.unhealthy.push({
                    id,
                    status: 'error',
                    error: error.message
                });
                healthStatus.summary.unhealthy++;
            }
        });

        await Promise.all(testPromises);

        return healthStatus;
    }

    /**
     * 測試連接
     */
    async testConnection(connectionId) {
        const startTime = Date.now();
        
        try {
            const response = await this.client.post(
                `/api/v2/connections/${connectionId}/test`
            );
            
            return {
                success: response.success,
                responseTime: Date.now() - startTime,
                timestamp: new Date().toISOString(),
                details: response.details
            };
        } catch (error) {
            return {
                success: false,
                error: error.message,
                responseTime: Date.now() - startTime,
                timestamp: new Date().toISOString()
            };
        }
    }

    /**
     * 取得連接使用情況
     */
    async getConnectionUsage(connectionId, timeRange = '30d') {
        try {
            const usage = await this.client.get(
                `/api/v2/connections/${connectionId}/usage`,
                { timeRange }
            );

            // 分析使用模式
            const analysis = {
                totalUsage: usage.total,
                dailyAverage: usage.total / 30,
                peakUsage: Math.max(...usage.daily),
                scenarios: usage.scenarios || [],
                lastUsed: usage.lastUsed,
                trends: this.analyzeTrends(usage.daily)
            };

            return analysis;

        } catch (error) {
            console.error('取得連接使用情況失敗:', error);
            throw error;
        }
    }

    /**
     * 加密認證資料
     */
    async encryptCredentials(credentials, key) {
        const crypto = require('crypto');
        const algorithm = 'aes-256-gcm';
        
        // 生成初始化向量
        const iv = crypto.randomBytes(16);
        
        // 建立加密器
        const cipher = crypto.createCipheriv(algorithm, Buffer.from(key, 'hex'), iv);
        
        // 加密資料
        const encrypted = Buffer.concat([
            cipher.update(JSON.stringify(credentials), 'utf8'),
            cipher.final()
        ]);
        
        // 取得認證標籤
        const authTag = cipher.getAuthTag();
        
        // 組合結果
        return {
            encrypted: encrypted.toString('base64'),
            iv: iv.toString('base64'),
            authTag: authTag.toString('base64')
        };
    }

    /**
     * 解密認證資料
     */
    async decryptCredentials(encryptedData, key) {
        const crypto = require('crypto');
        const algorithm = 'aes-256-gcm';
        
        // 解析加密資料
        const { encrypted, iv, authTag } = encryptedData;
        
        // 建立解密器
        const decipher = crypto.createDecipheriv(
            algorithm,
            Buffer.from(key, 'hex'),
            Buffer.from(iv, 'base64')
        );
        
        // 設定認證標籤
        decipher.setAuthTag(Buffer.from(authTag, 'base64'));
        
        // 解密資料
        const decrypted = Buffer.concat([
            decipher.update(Buffer.from(encrypted, 'base64')),
            decipher.final()
        ]);
        
        return JSON.parse(decrypted.toString('utf8'));
    }

    /**
     * 分析使用趨勢
     */
    analyzeTrends(dailyData) {
        if (dailyData.length < 7) {
            return { trend: 'insufficient_data' };
        }

        // 計算移動平均
        const ma7 = this.movingAverage(dailyData, 7);
        const recent = ma7.slice(-7);
        const previous = ma7.slice(-14, -7);

        const recentAvg = recent.reduce((a, b) => a + b, 0) / recent.length;
        const previousAvg = previous.reduce((a, b) => a + b, 0) / previous.length;

        const change = ((recentAvg - previousAvg) / previousAvg) * 100;

        return {
            trend: change > 10 ? 'increasing' : change < -10 ? 'decreasing' : 'stable',
            changePercent: change.toFixed(2),
            movingAverage: ma7
        };
    }

    /**
     * 計算移動平均
     */
    movingAverage(data, period) {
        const result = [];
        for (let i = period - 1; i < data.length; i++) {
            const sum = data.slice(i - period + 1, i + 1).reduce((a, b) => a + b, 0);
            result.push(sum / period);
        }
        return result;
    }

    /**
     * 將陣列分組
     */
    chunkArray(array, size) {
        const chunks = [];
        for (let i = 0; i < array.length; i += size) {
            chunks.push(array.slice(i, i + size));
        }
        return chunks;
    }
}

module.exports = ConnectionsAPIService;
```

## 9.3 進階整合模式

### 🔄 事件驅動架構

```javascript
// services/EventDrivenIntegration.js - 事件驅動整合
const EventEmitter = require('events');

class EventDrivenMakeIntegration extends EventEmitter {
    constructor(config) {
        super();
        this.config = config;
        this.apiClient = new MakeAPIClient(config);
        this.webhookHandlers = new Map();
        this.eventQueue = [];
        this.processing = false;
    }

    /**
     * 註冊事件處理器
     */
    registerEventHandler(eventType, handler) {
        const handlers = this.webhookHandlers.get(eventType) || [];
        handlers.push(handler);
        this.webhookHandlers.set(eventType, handlers);

        // 如果是新的事件類型，建立對應的 Webhook
        if (handlers.length === 1) {
            this.createWebhookForEvent(eventType);
        }
    }

    /**
     * 建立事件對應的 Webhook
     */
    async createWebhookForEvent(eventType) {
        try {
            const webhook = await this.apiClient.post('/api/v2/webhooks', {
                name: `Event: ${eventType}`,
                url: `${this.config.webhookBaseUrl}/events/${eventType}`,
                events: [eventType],
                active: true,
                headers: {
                    'X-Event-Type': eventType,
                    'X-Webhook-Secret': this.config.webhookSecret
                }
            });

            this.emit('webhook-created', { eventType, webhookId: webhook.id });
            
            return webhook;

        } catch (error) {
            console.error(`建立 Webhook 失敗 (${eventType}):`, error);
            throw error;
        }
    }

    /**
     * 處理傳入的 Webhook 事件
     */
    async handleWebhookEvent(eventType, payload, headers) {
        // 驗證 Webhook 簽名
        if (!this.verifyWebhookSignature(payload, headers)) {
            throw new Error('無效的 Webhook 簽名');
        }

        // 將事件加入佇列
        const event = {
            id: this.generateEventId(),
            type: eventType,
            payload,
            timestamp: new Date().toISOString(),
            headers
        };

        this.eventQueue.push(event);
        this.emit('event-received', event);

        // 開始處理佇列
        if (!this.processing) {
            this.processEventQueue();
        }

        return { eventId: event.id, queued: true };
    }

    /**
     * 處理事件佇列
     */
    async processEventQueue() {
        if (this.processing || this.eventQueue.length === 0) {
            return;
        }

        this.processing = true;

        while (this.eventQueue.length > 0) {
            const event = this.eventQueue.shift();
            
            try {
                await this.processEvent(event);
                this.emit('event-processed', event);
            } catch (error) {
                console.error('處理事件失敗:', error);
                this.emit('event-error', { event, error });
                
                // 重試邏輯
                if (event.retryCount < 3) {
                    event.retryCount = (event.retryCount || 0) + 1;
                    this.eventQueue.push(event);
                }
            }
        }

        this.processing = false;
    }

    /**
     * 處理單一事件
     */
    async processEvent(event) {
        const handlers = this.webhookHandlers.get(event.type) || [];
        
        if (handlers.length === 0) {
            console.warn(`沒有處理器註冊給事件類型: ${event.type}`);
            return;
        }

        // 執行所有處理器
        const results = await Promise.allSettled(
            handlers.map(handler => handler(event))
        );

        // 記錄處理結果
        const successful = results.filter(r => r.status === 'fulfilled').length;
        const failed = results.filter(r => r.status === 'rejected').length;

        if (failed > 0) {
            throw new Error(`${failed} 個處理器執行失敗`);
        }

        return { successful, failed };
    }

    /**
     * 驗證 Webhook 簽名
     */
    verifyWebhookSignature(payload, headers) {
        const crypto = require('crypto');
        const signature = headers['x-webhook-signature'];
        
        if (!signature) {
            return false;
        }

        const expectedSignature = crypto
            .createHmac('sha256', this.config.webhookSecret)
            .update(JSON.stringify(payload))
            .digest('hex');

        return signature === expectedSignature;
    }

    /**
     * 生成事件 ID
     */
    generateEventId() {
        return `evt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }

    /**
     * 建立事件驅動的 Scenario
     */
    async createEventDrivenScenario(config) {
        const {
            name,
            triggerEvent,
            processingModules,
            outputEvents
        } = config;

        try {
            // 建立 Webhook 觸發器
            const trigger = {
                type: 'webhook',
                name: 'Event Trigger',
                config: {
                    webhookName: `${name} Trigger`,
                    dataStructure: this.getEventDataStructure(triggerEvent)
                }
            };

            // 建立處理模組
            const modules = [trigger, ...processingModules];

            // 添加事件發布模組
            if (outputEvents && outputEvents.length > 0) {
                outputEvents.forEach(eventType => {
                    modules.push({
                        type: 'http',
                        name: `Publish ${eventType}`,
                        config: {
                            url: `${this.config.eventBusUrl}/publish`,
                            method: 'POST',
                            headers: {
                                'Content-Type': 'application/json',
                                'X-Event-Type': eventType
                            },
                            body: {
                                eventType,
                                payload: '{{output}}',
                                timestamp: '{{now}}'
                            }
                        }
                    });
                });
            }

            // 建立 Scenario
            const scenario = await this.apiClient.post('/api/v2/scenarios', {
                name,
                modules,
                scheduling: {
                    type: 'instant'
                },
                errorHandling: {
                    type: 'rollback',
                    notifyOnError: true
                }
            });

            // 註冊事件處理器
            this.registerEventHandler(triggerEvent, async (event) => {
                await this.apiClient.post(`/api/v2/scenarios/${scenario.id}/executions`, {
                    input: event.payload
                });
            });

            return scenario;

        } catch (error) {
            console.error('建立事件驅動 Scenario 失敗:', error);
            throw error;
        }
    }

    /**
     * 取得事件資料結構
     */
    getEventDataStructure(eventType) {
        const structures = {
            'order.created': {
                orderId: 'string',
                customerId: 'string',
                items: 'array',
                total: 'number',
                timestamp: 'string'
            },
            'user.registered': {
                userId: 'string',
                email: 'string',
                name: 'string',
                registeredAt: 'string'
            },
            'payment.received': {
                paymentId: 'string',
                orderId: 'string',
                amount: 'number',
                currency: 'string',
                method: 'string',
                timestamp: 'string'
            }
        };

        return structures[eventType] || {};
    }
}

module.exports = EventDrivenMakeIntegration;
```

### 🌐 多租戶整合架構

```javascript
// services/MultiTenantIntegration.js - 多租戶整合
class MultiTenantMakeIntegration {
    constructor(config) {
        this.config = config;
        this.tenantClients = new Map();
        this.tenantConfigs = new Map();
        this.sharedResources = new Map();
    }

    /**
     * 初始化租戶
     */
    async initializeTenant(tenantId, tenantConfig) {
        try {
            // 建立租戶專用的 API 客戶端
            const client = new MakeAPIClient({
                ...this.config,
                ...tenantConfig,
                headers: {
                    'X-Tenant-ID': tenantId
                }
            });

            this.tenantClients.set(tenantId, client);
            this.tenantConfigs.set(tenantId, tenantConfig);

            // 初始化租戶資源
            await this.initializeTenantResources(tenantId);

            // 設定租戶限制
            await this.configureTenantLimits(tenantId, tenantConfig.limits);

            return {
                tenantId,
                status: 'initialized',
                timestamp: new Date().toISOString()
            };

        } catch (error) {
            console.error(`初始化租戶失敗 (${tenantId}):`, error);
            throw error;
        }
    }

    /**
     * 為租戶執行操作
     */
    async executeForTenant(tenantId, operation) {
        const client = this.tenantClients.get(tenantId);
        
        if (!client) {
            throw new Error(`租戶未初始化: ${tenantId}`);
        }

        // 檢查租戶配額
        await this.checkTenantQuota(tenantId, operation.type);

        // 添加租戶上下文
        const context = {
            tenantId,
            timestamp: new Date().toISOString(),
            requestId: this.generateRequestId()
        };

        try {
            // 執行操作
            const result = await operation.execute(client, context);
            
            // 更新使用統計
            await this.updateTenantUsage(tenantId, operation.type);
            
            return result;

        } catch (error) {
            // 記錄錯誤
            await this.logTenantError(tenantId, error, context);
            throw error;
        }
    }

    /**
     * 批次為多個租戶執行操作
     */
    async executeForMultipleTenants(tenantIds, operation) {
        const results = {
            successful: new Map(),
            failed: new Map()
        };

        // 並行執行但限制並發數
        const limit = pLimit(this.config.maxConcurrentTenants || 5);
        
        const promises = tenantIds.map(tenantId => 
            limit(async () => {
                try {
                    const result = await this.executeForTenant(tenantId, operation);
                    results.successful.set(tenantId, result);
                } catch (error) {
                    results.failed.set(tenantId, error);
                }
            })
        );

        await Promise.all(promises);

        return results;
    }

    /**
     * 建立租戶專用的 Scenario
     */
    async createTenantScenario(tenantId, scenarioConfig) {
        return this.executeForTenant(tenantId, {
            type: 'create_scenario',
            execute: async (client, context) => {
                // 添加租戶標識
                const tenantScenarioConfig = {
                    ...scenarioConfig,
                    name: `[${tenantId}] ${scenarioConfig.name}`,
                    metadata: {
                        ...scenarioConfig.metadata,
                        tenantId,
                        createdAt: context.timestamp
                    }
                };

                // 處理共享資源
                if (scenarioConfig.useSharedResources) {
                    tenantScenarioConfig.modules = await this.processSharedResources(
                        tenantId,
                        scenarioConfig.modules
                    );
                }

                // 建立 Scenario
                const scenario = await client.post('/api/v2/scenarios', tenantScenarioConfig);

                // 設定租戶專用的權限
                await this.configureTenantPermissions(tenantId, scenario.id);

                return scenario;
            }
        });
    }

    /**
     * 管理共享資源
     */
    async processSharedResources(tenantId, modules) {
        const processedModules = [];

        for (const module of modules) {
            if (module.sharedResource) {
                // 取得或建立共享資源
                const sharedResource = await this.getOrCreateSharedResource(
                    module.sharedResource
                );

                // 建立租戶專用的參考
                processedModules.push({
                    ...module,
                    config: {
                        ...module.config,
                        sharedResourceId: sharedResource.id,
                        tenantId
                    }
                });
            } else {
                processedModules.push(module);
            }
        }

        return processedModules;
    }

    /**
     * 取得或建立共享資源
     */
    async getOrCreateSharedResource(resourceConfig) {
        const resourceKey = this.generateResourceKey(resourceConfig);
        
        if (this.sharedResources.has(resourceKey)) {
            return this.sharedResources.get(resourceKey);
        }

        // 建立新的共享資源
        const resource = await this.createSharedResource(resourceConfig);
        this.sharedResources.set(resourceKey, resource);
        
        return resource;
    }

    /**
     * 租戶隔離的資料存取
     */
    async getTenantData(tenantId, dataType, filters = {}) {
        return this.executeForTenant(tenantId, {
            type: 'get_data',
            execute: async (client, context) => {
                // 自動添加租戶過濾
                const tenantFilters = {
                    ...filters,
                    'metadata.tenantId': tenantId
                };

                let endpoint;
                switch (dataType) {
                    case 'scenarios':
                        endpoint = '/api/v2/scenarios';
                        break;
                    case 'executions':
                        endpoint = '/api/v2/executions';
                        break;
                    case 'connections':
                        endpoint = '/api/v2/connections';
                        break;
                    default:
                        throw new Error(`未知的資料類型: ${dataType}`);
                }

                const data = await client.get(endpoint, tenantFilters);
                
                // 二次驗證租戶隔離
                return data.filter(item => 
                    item.metadata?.tenantId === tenantId
                );
            }
        });
    }

    /**
     * 跨租戶資料聚合
     */
    async aggregateAcrossTenants(tenantIds, aggregation) {
        const tenantData = new Map();

        // 收集各租戶資料
        for (const tenantId of tenantIds) {
            try {
                const data = await this.getTenantData(
                    tenantId,
                    aggregation.dataType,
                    aggregation.filters
                );
                tenantData.set(tenantId, data);
            } catch (error) {
                console.error(`取得租戶資料失敗 (${tenantId}):`, error);
                tenantData.set(tenantId, []);
            }
        }

        // 執行聚合
        return this.performAggregation(tenantData, aggregation);
    }

    /**
     * 執行資料聚合
     */
    performAggregation(tenantData, aggregation) {
        const results = {
            byTenant: {},
            total: {}
        };

        // 計算每個租戶的統計
        tenantData.forEach((data, tenantId) => {
            results.byTenant[tenantId] = this.calculateStats(data, aggregation.metrics);
        });

        // 計算總體統計
        const allData = Array.from(tenantData.values()).flat();
        results.total = this.calculateStats(allData, aggregation.metrics);

        return results;
    }

    /**
     * 計算統計資料
     */
    calculateStats(data, metrics) {
        const stats = {};

        metrics.forEach(metric => {
            switch (metric.type) {
                case 'count':
                    stats[metric.name] = data.length;
                    break;
                case 'sum':
                    stats[metric.name] = data.reduce((sum, item) => 
                        sum + (item[metric.field] || 0), 0
                    );
                    break;
                case 'average':
                    const sum = data.reduce((s, item) => s + (item[metric.field] || 0), 0);
                    stats[metric.name] = data.length > 0 ? sum / data.length : 0;
                    break;
                case 'unique':
                    stats[metric.name] = new Set(data.map(item => item[metric.field])).size;
                    break;
            }
        });

        return stats;
    }

    /**
     * 檢查租戶配額
     */
    async checkTenantQuota(tenantId, operationType) {
        const config = this.tenantConfigs.get(tenantId);
        const limits = config?.limits || {};

        // 取得當前使用量
        const usage = await this.getTenantUsage(tenantId);

        // 檢查各項限制
        const quotaChecks = {
            scenarios: usage.scenarios < (limits.maxScenarios || Infinity),
            executions: usage.monthlyExecutions < (limits.maxMonthlyExecutions || Infinity),
            storage: usage.storageBytes < (limits.maxStorageBytes || Infinity),
            api_calls: usage.dailyApiCalls < (limits.maxDailyApiCalls || Infinity)
        };

        // 根據操作類型檢查相應的配額
        const quotaMap = {
            create_scenario: ['scenarios'],
            execute_scenario: ['executions'],
            upload_file: ['storage'],
            api_call: ['api_calls']
        };

        const requiredQuotas = quotaMap[operationType] || [];
        const exceeded = requiredQuotas.filter(quota => !quotaChecks[quota]);

        if (exceeded.length > 0) {
            throw new Error(`超出配額限制: ${exceeded.join(', ')}`);
        }

        return true;
    }

    /**
     * 取得租戶使用量
     */
    async getTenantUsage(tenantId) {
        // 實際應用中應從資料庫或快取取得
        return {
            scenarios: 10,
            monthlyExecutions: 5000,
            storageBytes: 1024 * 1024 * 100, // 100MB
            dailyApiCalls: 1000
        };
    }

    /**
     * 更新租戶使用量
     */
    async updateTenantUsage(tenantId, operationType) {
        // 實際應用中應更新資料庫或快取
        console.log(`更新租戶使用量: ${tenantId} - ${operationType}`);
    }

    /**
     * 配置租戶限制
     */
    async configureTenantLimits(tenantId, limits) {
        // 實際應用中應儲存到資料庫
        console.log(`配置租戶限制: ${tenantId}`, limits);
    }

    /**
     * 生成資源鍵
     */
    generateResourceKey(resourceConfig) {
        return `${resourceConfig.type}_${resourceConfig.name}`;
    }

    /**
     * 生成請求 ID
     */
    generateRequestId() {
        return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    }
}

module.exports = MultiTenantMakeIntegration;
```

## 9.4 效能優化技巧

### ⚡ 快取策略

```javascript
// services/CacheStrategy.js - 快取策略實作
const NodeCache = require('node-cache');
const Redis = require('redis');

class MakeAPICacheStrategy {
    constructor(config) {
        this.config = config;
        
        // 多層快取
        this.l1Cache = new NodeCache({ stdTTL: 60 }); // 記憶體快取 (1分鐘)
        this.l2Cache = this.initializeRedis(); // Redis 快取
        
        // 快取統計
        this.stats = {
            hits: 0,
            misses: 0,
            l1Hits: 0,
            l2Hits: 0
        };
    }

    /**
     * 初始化 Redis
     */
    initializeRedis() {
        const client = Redis.createClient({
            url: this.config.redisUrl,
            retry_strategy: (options) => {
                if (options.error && options.error.code === 'ECONNREFUSED') {
                    return new Error('Redis 連接被拒絕');
                }
                if (options.total_retry_time > 1000 * 60 * 60) {
                    return new Error('Redis 重試超時');
                }
                if (options.attempt > 10) {
                    return undefined;
                }
                return Math.min(options.attempt * 100, 3000);
            }
        });

        client.on('error', (err) => {
            console.error('Redis 錯誤:', err);
        });

        return client;
    }

    /**
     * 快取裝飾器
     */
    cacheable(options = {}) {
        const {
            ttl = 300, // 5分鐘
            keyGenerator,
            condition,
            invalidate
        } = options;

        return (target, propertyName, descriptor) => {
            const originalMethod = descriptor.value;

            descriptor.value = async function (...args) {
                // 檢查是否應該使用快取
                if (condition && !condition(...args)) {
                    return originalMethod.apply(this, args);
                }

                // 生成快取鍵
                const cacheKey = keyGenerator 
                    ? keyGenerator(propertyName, ...args)
                    : this.generateCacheKey(propertyName, args);

                // 嘗試從快取取得
                const cachedResult = await this.get(cacheKey);
                if (cachedResult !== null) {
                    return cachedResult;
                }

                // 執行原始方法
                const result = await originalMethod.apply(this, args);

                // 儲存到快取
                if (result !== null && result !== undefined) {
                    await this.set(cacheKey, result, ttl);
                }

                // 處理快取失效
                if (invalidate) {
                    const keysToInvalidate = invalidate(result, ...args);
                    await this.invalidateKeys(keysToInvalidate);
                }

                return result;
            }.bind(this);

            return descriptor;
        };
    }

    /**
     * 從快取取得資料
     */
    async get(key) {
        this.stats.misses++;

        // L1 快取 (記憶體)
        const l1Result = this.l1Cache.get(key);
        if (l1Result !== undefined) {
            this.stats.hits++;
            this.stats.l1Hits++;
            return l1Result;
        }

        // L2 快取 (Redis)
        try {
            const l2Result = await this.getFromRedis(key);
            if (l2Result !== null) {
                this.stats.hits++;
                this.stats.l2Hits++;
                
                // 回填 L1 快取
                this.l1Cache.set(key, l2Result);
                
                return l2Result;
            }
        } catch (error) {
            console.error('Redis 讀取錯誤:', error);
        }

        return null;
    }

    /**
     * 設定快取資料
     */
    async set(key, value, ttl) {
        // 設定 L1 快取
        this.l1Cache.set(key, value, Math.min(ttl, 60));

        // 設定 L2 快取
        try {
            await this.setToRedis(key, value, ttl);
        } catch (error) {
            console.error('Redis 寫入錯誤:', error);
        }
    }

    /**
     * 批次取得快取
     */
    async mget(keys) {
        const results = new Map();
        const missingKeys = [];

        // 先從 L1 快取取得
        keys.forEach(key => {
            const value = this.l1Cache.get(key);
            if (value !== undefined) {
                results.set(key, value);
            } else {
                missingKeys.push(key);
            }
        });

        // 從 L2 快取取得缺失的項目
        if (missingKeys.length > 0) {
            try {
                const l2Results = await this.mgetFromRedis(missingKeys);
                l2Results.forEach((value, index) => {
                    if (value !== null) {
                        const key = missingKeys[index];
                        results.set(key, value);
                        // 回填 L1 快取
                        this.l1Cache.set(key, value);
                    }
                });
            } catch (error) {
                console.error('Redis 批次讀取錯誤:', error);
            }
        }

        return results;
    }

    /**
     * 快取預熱
     */
    async warmUp(dataLoader) {
        console.log('開始快取預熱...');
        
        try {
            const data = await dataLoader();
            const promises = [];

            for (const [key, value] of Object.entries(data)) {
                promises.push(this.set(key, value, 3600)); // 1小時
            }

            await Promise.all(promises);
            console.log(`快取預熱完成，載入 ${promises.length} 個項目`);
            
        } catch (error) {
            console.error('快取預熱失敗:', error);
        }
    }

    /**
     * 智能快取更新
     */
    async smartUpdate(key, updater, options = {}) {
        const {
            lockTimeout = 5000,
            staleWhileRevalidate = true
        } = options;

        // 取得鎖
        const lockKey = `lock:${key}`;
        const lockAcquired = await this.acquireLock(lockKey, lockTimeout);

        if (!lockAcquired) {
            // 如果無法取得鎖，返回舊資料（如果允許）
            if (staleWhileRevalidate) {
                const staleData = await this.get(key);
                if (staleData !== null) {
                    return staleData;
                }
            }
            throw new Error('無法取得更新鎖');
        }

        try {
            // 執行更新
            const newValue = await updater();
            
            // 更新快取
            await this.set(key, newValue, options.ttl || 300);
            
            return newValue;
            
        } finally {
            // 釋放鎖
            await this.releaseLock(lockKey);
        }
    }

    /**
     * 快取失效傳播
     */
    async invalidateKeys(patterns) {
        const keysToDelete = [];

        // 從 L1 快取收集鍵
        const l1Keys = this.l1Cache.keys();
        patterns.forEach(pattern => {
            const regex = new RegExp(pattern);
            l1Keys.forEach(key => {
                if (regex.test(key)) {
                    keysToDelete.push(key);
                }
            });
        });

        // 刪除 L1 快取
        keysToDelete.forEach(key => {
            this.l1Cache.del(key);
        });

        // 刪除 L2 快取
        try {
            await this.invalidateRedisKeys(patterns);
        } catch (error) {
            console.error('Redis 快取失效錯誤:', error);
        }

        return keysToDelete.length;
    }

    /**
     * 取得快取統計
     */
    getStats() {
        const hitRate = this.stats.hits > 0 
            ? (this.stats.hits / (this.stats.hits + this.stats.misses) * 100).toFixed(2)
            : 0;

        return {
            ...this.stats,
            hitRate: `${hitRate}%`,
            l1HitRate: this.stats.l1Hits > 0
                ? `${(this.stats.l1Hits / this.stats.hits * 100).toFixed(2)}%`
                : '0%',
            l2HitRate: this.stats.l2Hits > 0
                ? `${(this.stats.l2Hits / this.stats.hits * 100).toFixed(2)}%`
                : '0%'
        };
    }

    /**
     * Redis 操作輔助方法
     */
    async getFromRedis(key) {
        return new Promise((resolve, reject) => {
            this.l2Cache.get(key, (err, data) => {
                if (err) reject(err);
                else resolve(data ? JSON.parse(data) : null);
            });
        });
    }

    async setToRedis(key, value, ttl) {
        return new Promise((resolve, reject) => {
            this.l2Cache.setex(key, ttl, JSON.stringify(value), (err) => {
                if (err) reject(err);
                else resolve();
            });
        });
    }

    async mgetFromRedis(keys) {
        return new Promise((resolve, reject) => {
            this.l2Cache.mget(keys, (err, values) => {
                if (err) reject(err);
                else resolve(values.map(v => v ? JSON.parse(v) : null));
            });
        });
    }

    async invalidateRedisKeys(patterns) {
        // 使用 Redis SCAN 來找出符合的鍵
        const keysToDelete = [];
        
        for (const pattern of patterns) {
            const keys = await this.scanRedisKeys(pattern);
            keysToDelete.push(...keys);
        }

        if (keysToDelete.length > 0) {
            await new Promise((resolve, reject) => {
                this.l2Cache.del(keysToDelete, (err) => {
                    if (err) reject(err);
                    else resolve();
                });
            });
        }
    }

    async scanRedisKeys(pattern) {
        // 實作 Redis SCAN 邏輯
        const keys = [];
        let cursor = '0';
        
        do {
            const result = await new Promise((resolve, reject) => {
                this.l2Cache.scan(cursor, 'MATCH', pattern, 'COUNT', 100, (err, res) => {
                    if (err) reject(err);
                    else resolve(res);
                });
            });
            
            cursor = result[0];
            keys.push(...result[1]);
            
        } while (cursor !== '0');
        
        return keys;
    }

    /**
     * 分散式鎖實作
     */
    async acquireLock(lockKey, timeout) {
        const lockValue = Date.now() + timeout;
        
        return new Promise((resolve) => {
            this.l2Cache.set(lockKey, lockValue, 'NX', 'PX', timeout, (err, result) => {
                resolve(result === 'OK');
            });
        });
    }

    async releaseLock(lockKey) {
        return new Promise((resolve) => {
            this.l2Cache.del(lockKey, (err) => {
                resolve(!err);
            });
        });
    }

    /**
     * 生成快取鍵
     */
    generateCacheKey(method, args) {
        const crypto = require('crypto');
        const hash = crypto.createHash('md5');
        hash.update(method);
        hash.update(JSON.stringify(args));
        return `cache:${method}:${hash.digest('hex')}`;
    }
}

module.exports = MakeAPICacheStrategy;
```

## 9.5 本章總結

本章深入探討了 Make API 的進階應用，包括：

✅ **完整的 API 架構和認證機制**  
✅ **核心 API 功能的進階實作**  
✅ **事件驅動和多租戶整合模式**  
✅ **效能優化和快取策略**  
✅ **錯誤處理和重試機制**  

### 📈 技術要點

通過本章學習，您掌握了：
- 多種認證方式的實作和管理
- 進階的 API 請求處理和批次操作
- 事件驅動架構的設計和實作
- 多租戶系統的隔離和資源管理
- 多層快取策略和效能優化技巧

### 🚀 下一步

掌握了 Make API 的進階功能後，下一章將探討架構設計與最佳實踐，幫助您建立企業級的自動化解決方案。