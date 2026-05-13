# CHEMTEX 定制化修改记录

> 以下所有修改基于 WeKnora 官方版本，升级前请对照此列表重新应用。

---

## 1. 登录页面改造

**文件：** `frontend/src/views/auth/Login.vue`

| 修改 | 说明 |
|------|------|
| 删除左上角 WeKnora logo | 第 95-98 行 `<a class="header-logo">` 整个删除 |
| 删除右上角「官方网站」「GitHub」链接 | 第 101-116 行两个 `<a>` 标签删除，只保留语言切换 |
| `showcase-subtitle` 改为固定文本 | 第 128 行从 `{{ $t('platform.subtitle') }}` → `CHEMTEX企业级智能文档知识库` |
| 隐藏 email/password 登录表单 | 第 195 行用 `<template v-if="!oidcEnabled">` 包裹 `<t-form>` |
| 隐藏注册卡片 | 第 277 行 `v-if="oidcEnabled ? false : isRegisterMode"` |
| 登录窗口垂直居中 | CSS `.form-section`: `align-items: center`, `padding: 0 50px` |

---

## 2. OIDC SSO 修复

### 2.1 前端 token 处理

**文件：** `frontend/src/App.vue`

- **第 8 行** 新增 `import axios from 'axios'`
- **第 80-84 行** `persistOIDCLoginResponse` 中先清理旧 localStorage：
  ```typescript
  localStorage.removeItem('weknora_user')
  localStorage.removeItem('weknora_tenant')
  localStorage.removeItem('weknora_knowledge_bases')
  localStorage.removeItem('weknora_current_kb')
  ```
- **第 86 行** `setToken` 后设置 axios 默认 header：
  ```typescript
  axios.defaults.headers.common['Authorization'] = 'Bearer ' + response.token
  ```

**文件：** `frontend/src/utils/request.ts`

- **第 33-34 行** axios 拦截器优先使用全局默认 header：
  ```typescript
  const defaultAuth = axios.defaults.headers.common['Authorization'] as string;
  const token = defaultAuth ? defaultAuth.replace('Bearer ', '') : localStorage.getItem('weknora_token');
  ```

### 2.2 后端 cookie 支持

**文件：** `internal/handler/auth.go`

- OIDC callback 成功时设置 `weknora_token` 和 `weknora_refresh_token` cookie

**文件：** `internal/middleware/auth.go`

- JWT 中间件增加 cookie fallback：如果 `Authorization` header 为空，尝试从 `weknora_token` cookie 读取

---

## 3. 配置文件

### 3.1 `.env`

```ini
FRONTEND_PORT=8082
DISABLE_REGISTRATION=true
NEO4J_ENABLE=true
NEO4J_URI=bolt://neo4j:7687
ENABLE_GRAPH_RAG=true

# OIDC SSO
OIDC_AUTH_ENABLE=true
OIDC_AUTH_PROVIDER_DISPLAY_NAME=CHEMTEX SSO
OIDC_AUTH_ISSUER_URL=http://sso.chemtex.com:18080/realms/CHEMTEX_SSO
OIDC_AUTH_CLIENT_ID=weknora
OIDC_AUTH_CLIENT_SECRET=Uj9E35Ojub8O0tiQlKUOBvVC2vKQk2Oj
OIDC_AUTH_SCOPES=openid profile email
OIDC_USER_INFO_MAPPING_USER_NAME=preferred_username
OIDC_USER_INFO_MAPPING_EMAIL=email
```

### 3.2 `docker-compose.yml`

app service `environment:` 块末尾添加 OIDC 变量（共 12 个，见 `OIDC_AUTH_ENABLE` 到 `OIDC_USER_INFO_MAPPING_EMAIL`）

---

## 4. Keycloak 配置

**Realm:** CHEMTEX_SSO  
**Client ID:** weknora

| 配置项 | 值 |
|--------|-----|
| redirectUris | `http://kb.chemtex.com/api/v1/auth/oidc/callback` |
| | `http://kb.chemtex.com:8082/api/v1/auth/oidc/callback` |
| | `http://10.101.72.2:8082/api/v1/auth/oidc/callback` |
| webOrigins | `http://10.101.72.2:8082` |
| | `http://kb.chemtex.com` |
| | `http://kb.chemtex.com:8082` |
| defaultClientScopes | 必须包含 `email` scope |

---

## 升级流程

1. 从上游拉取新版本：
   ```bash
   git fetch upstream
   git checkout -b upgrade-<version> upstream/main
   ```

2. 对照本文档逐条重新应用修改

3. 构建并部署：
   ```bash
   docker compose build --no-cache app frontend
   docker compose up -d app frontend
   ```

4. 验证 SSO 登录和页面显示正常
