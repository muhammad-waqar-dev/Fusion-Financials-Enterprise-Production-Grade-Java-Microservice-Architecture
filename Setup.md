# Setup Guide: Step by Step

Prereqs: Node 18+, npm, JDK 17, Docker, Git, an Auth0 tenant (free works).

## Step 1: Auth0 configuration
1. **Applications -> Create -> Single Page Application** `platform-shell`.
   - Allowed Callback URLs: `http://localhost:3000/callback`
   - Allowed Logout URLs: `http://localhost:3000/login`
   - Allowed Web Origins: `http://localhost:3000`
   - Enable Refresh Token Rotation.
2. **APIs -> Create API** `platform-api`, Identifier (audience) `https://api.platform.local`, enable RBAC + "Add permissions in access token".
3. Add permissions (`finance:read`, `finance:write`, ...), create roles, assign to a test user.
4. Note `DOMAIN`, `CLIENT_ID`, `AUDIENCE`.

## Step 2: Shell (host) micro-frontend
```bash
mkdir shell-webui && cd shell-webui
npm init -y
npm i react react-dom react-router-dom@6 @auth0/auth0-react axios zustand
npm i -D webpack webpack-cli webpack-dev-server html-webpack-plugin dotenv-webpack \
  typescript ts-loader babel-loader @babel/core @babel/preset-env @babel/preset-react \
  @babel/preset-typescript external-remotes-plugin cross-env
```
`.env.development`
```
REACT_APP_AUTH0_DOMAIN=your-tenant.auth0.com
REACT_APP_AUTH0_CLIENT_ID=xxxx
REACT_APP_AUTH0_AUDIENCE=https://api.platform.local
REACT_APP_AUTH0_CALLBACK_URL=http://localhost:3000/callback
REACT_APP_API_BASE_URL=http://localhost:8080/api
```
`webpack.config.js` (key part)
```js
const { ModuleFederationPlugin } = require('webpack').container;
const deps = require('./package.json').dependencies;
const shared = ['react', 'react-dom', 'react-router-dom', '@auth0/auth0-react']
  .reduce((a, k) => ({ ...a, [k]: { singleton: true, requiredVersion: deps[k] } }), {});

new ModuleFederationPlugin({
  name: 'container',
  filename: 'remoteEntry.js',
  remotes: {
    finance: 'finance@[window.REMOTES.finance]/remoteEntry.js', // runtime URL
  },
  exposes: {
    './useUserProfile': './src/hooks/useUserProfile.ts',
    './apiClient': './src/services/apiClient.ts',
    './EnvironmentProperties': './src/utils/envProperties.ts',
  },
  shared,
});
```
Serve `public/remotes.json` and load it into `window.REMOTES` in `index.html` before the bundle starts, so URLs differ per environment.

### Auth + redirection code (shell)
`src/modules/auth0/Auth0ProviderWithNavigate.tsx`
```tsx
import { Auth0Provider, AppState } from '@auth0/auth0-react';
import { useNavigate } from 'react-router-dom';

const safe = (p?: string) => (p && p.startsWith('/') && !p.startsWith('//') ? p : '/home');

export const Auth0ProviderWithNavigate = ({ children }: { children: React.ReactNode }) => {
  const navigate = useNavigate();
  return (
    <Auth0Provider
      domain={process.env.REACT_APP_AUTH0_DOMAIN!}
      clientId={process.env.REACT_APP_AUTH0_CLIENT_ID!}
      authorizationParams={{
        redirect_uri: process.env.REACT_APP_AUTH0_CALLBACK_URL,
        audience: process.env.REACT_APP_AUTH0_AUDIENCE,
      }}
      useRefreshTokens
      cacheLocation="memory"
      onRedirectCallback={(s?: AppState) => navigate(safe(s?.returnTo), { replace: true })}
    >
      {children}
    </Auth0Provider>
  );
};
```
`src/modules/auth0/PrivateRoute.tsx`
```tsx
import { useAuth0 } from '@auth0/auth0-react';
import { Outlet } from 'react-router-dom';
import { useEffect } from 'react';

export default function PrivateRoute() {
  const { isLoading, isAuthenticated, loginWithRedirect } = useAuth0();
  useEffect(() => {
    if (!isLoading && !isAuthenticated) {
      loginWithRedirect({
        appState: { returnTo: window.location.pathname + window.location.search },
      });
    }
  }, [isLoading, isAuthenticated, loginWithRedirect]);
  return isAuthenticated ? <Outlet /> : <div>Redirecting to login…</div>;
}
```
`src/App.tsx`
```tsx
const FinanceRoutes = lazy(() => import('finance/Routes'));

<Auth0ProviderWithNavigate>
  <Suspense fallback={<Spinner />}>
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route path="/callback" element={<CallbackPage />} />   {/* then navigate handled by onRedirectCallback */}
      <Route element={<PrivateRoute />}>
        <Route path="/home" element={<Home />} />
        <Route path="/finance/*" element={<ErrorBoundary><FinanceRoutes /></ErrorBoundary>} />
        <Route path="/forbidden" element={<Forbidden />} />
      </Route>
      <Route path="/" element={<Navigate to="/home" replace />} />
      <Route path="*" element={<NotFound />} />
    </Routes>
  </Suspense>
</Auth0ProviderWithNavigate>
```
`src/services/apiClient.ts` (token-aware, exposed to remotes)
```ts
import axios from 'axios';
let getToken: () => Promise<string>;
let relogin: () => void;
export const bindAuth = (g: typeof getToken, r: typeof relogin) => { getToken = g; relogin = r; };

export const apiClient = axios.create({ baseURL: process.env.REACT_APP_API_BASE_URL });
apiClient.interceptors.request.use(async c => { c.headers.Authorization = `Bearer ${await getToken()}`; return c; });
apiClient.interceptors.response.use(r => r, e => {
  if (e.response?.status === 401) relogin();
  if (e.response?.status === 403) window.location.assign('/forbidden');
  return Promise.reject(e);
});
```
Call `bindAuth(getAccessTokenSilently, () => loginWithRedirect({appState:{returnTo: location.pathname}}))` once after login.

## Step 3: Remote micro-frontend
```bash
mkdir finance-webui && cd finance-webui   # same deps as shell, minus Auth0 provider
```
`webpack.config.js`
```js
new ModuleFederationPlugin({
  name: 'finance',
  filename: 'remoteEntry.js',
  exposes: { './Routes': './src/Routes.tsx' },
  remotes: { container: 'container@http://localhost:3000/remoteEntry.js' }, // to use apiClient
  shared,   // same singleton config
});
// devServer: { port: 3001, headers: { 'Access-Control-Allow-Origin': '*' } }
```
`src/Routes.tsx` – **relative routes only**
```tsx
export default function FinanceRoutes() {
  return (<Routes>
    <Route index element={<ItemList />} />
    <Route path="items/:id" element={<ItemDetail />} />
  </Routes>);
}
```
Standalone dev: `src/bootstrap.tsx` wraps in its own `BrowserRouter` + dev-only mock auth; `src/index.ts` = `import('./bootstrap')` (async boundary required by Module Federation).

## Step 4: Backend services (Spring Boot)
Generate with start.spring.io (Java 17, Gradle, Web, Security, OAuth2 Resource Server, Validation, Data JPA, Actuator, Flyway, PostgreSQL).

`application.yml`
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://your-tenant.auth0.com/
          audiences: https://api.platform.local
server.port: 8081
```
`SecurityConfig.java`
```java
@Configuration @EnableMethodSecurity
class SecurityConfig {
  @Bean SecurityFilterChain chain(HttpSecurity http) throws Exception {
    return http
      .csrf(c -> c.disable())
      .authorizeHttpRequests(a -> a
          .requestMatchers("/actuator/health/**").permitAll()
          .anyRequest().authenticated())
      .oauth2ResourceServer(o -> o.jwt(j -> j.jwtAuthenticationConverter(permissionsConverter())))
      .build();
  }
  // map JWT "permissions" claim -> authorities "SCOPE_/PERM_"
}
```
Controller: `@PreAuthorize("hasAuthority('PERM_finance:read')")`.

## Step 5: API Gateway
Spring Cloud Gateway project, same resource-server config, `application.yml`:
```yaml
spring.cloud.gateway.routes:
  - id: finance
    uri: http://localhost:8081
    predicates: [ Path=/api/finance/** ]
    filters: [ StripPrefix=1 ]
  - id: drug
    uri: http://localhost:8082
    predicates: [ Path=/api/drug/** ]
    filters: [ StripPrefix=1 ]
spring.cloud.gateway.globalcors.cors-configurations.'[/**]':
  allowedOrigins: ["http://localhost:3000"]
  allowedMethods: ["*"]
  allowedHeaders: ["*"]
```

## Step 6: Local orchestration
`infra/docker-compose.yml`
```yaml
services:
  postgres: { image: postgres:16, environment: { POSTGRES_PASSWORD: dev }, ports: ["5432:5432"] }
  gateway:  { build: ../api-gateway, ports: ["8080:8080"] }
  finance-service: { build: ../finance-service, environment: { SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/finance } }
```
Run order: `docker compose up -d` -> `npm start` in each remote (3001, 3002…) -> `npm start` in shell (3000) -> open `http://localhost:3000`.

## Step 7: Verify checklist
- [ ] Opening `/finance/items/5` logged-out redirects to Auth0, then returns to `/finance/items/5`.
- [ ] `/callback` never stays on screen; bad `returnTo=//evil.com` falls back to `/home`.
- [ ] Network calls carry `Authorization: Bearer`; curl without it -> 401 at gateway.
- [ ] User without `finance:read` gets 403 -> `/forbidden`.
- [ ] Stopping a remote shows the ErrorBoundary only; the shell stays up.
- [ ] Logout lands on `/login`; back button does not show protected data.
- [ ] Shell and remote use a single React instance (check `React.version` once, no "invalid hook call").

## Step 8: CI/CD & deploy (per repo)
1. `azure-pipeline.yml`: install -> lint -> test -> build -> publish artifact.
2. Frontends: upload `dist/` to blob/S3; invalidate CDN for `remoteEntry.js` only.
3. Services: Gradle build -> Docker image -> registry -> deploy (Terraform in `infra/`).
4. Update `remotes.json` per environment; promote dev -> qa -> uat -> prd.

## Suggested build order
1. Auth0 + shell with login/redirect + one dummy private page.
2. One service + gateway with JWT validation; call it from the shell.
3. First remote loaded via Module Federation.
4. Permissions/403 handling, error boundaries, observability.
5. Second service/remote, messaging, CI/CD.
