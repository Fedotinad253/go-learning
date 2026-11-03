# 18. OAuth 2.0 и OpenID Connect

В файле 06 ты делал свою аутентификацию: bcrypt + JWT. В компаниях часто используют внешний сервер авторизации (Keycloak, Auth0, Okta, Google, корпоративный SSO), а сервисы только проверяют его токены. На собеседованиях спрашивают, как это устроено.

**Когда проходить:** после файлов 06 и 17.

## Роли и термины

- **Resource Owner** — пользователь, которому принадлежат данные.
- **Client** — приложение, которое хочет доступ (веб-сайт, мобильное приложение, другой сервис).
- **Authorization Server** — выдаёт токены (Keycloak, Google). Пользователь вводит пароль только здесь.
- **Resource Server** — твой API, который принимает токены и отдаёт данные.

**OAuth 2.0** — протокол **авторизации**: «приложение X может читать мои задачи». Он не говорит, кто пользователь.
**OpenID Connect (OIDC)** — слой **аутентификации** поверх OAuth 2.0: добавляет **ID token** (JWT с данными пользователя), эндпоинт `userinfo` и **discovery**: `https://<issuer>/.well-known/openid-configuration`, где перечислены все адреса и ключи.

## Токены

| Токен | Для кого | Срок | Что внутри |
|---|---|---|---|
| Access token | для API (Resource Server) | короткий: 5–15 минут | права (`scope`), `sub`, `aud`, `exp`; JWT или непрозрачная строка |
| Refresh token | для Authorization Server, чтобы получить новый access | долгий: часы или дни | хранить надёжно, при каждом использовании выдаётся новый (rotation) |
| ID token | для клиента: узнать, кто вошёл | короткий | `sub`, `email`, `name`, `nonce`; **не отправляй его в API** вместо access token |

### Как проверять JWT access token в API
1. Подпись — открытым ключом из **JWKS** (`jwks_uri` из discovery). Ключ выбирается по заголовку `kid`. Ключи кэшируются и периодически обновляются.
2. Разрешай только ожидаемые алгоритмы (`RS256`, `ES256`). `alg: none` и подмена на `HS256` — классические уязвимости.
3. `iss` — наш issuer. `aud` — наш API. `exp` и `nbf` — срок действия.
4. Права: `scope` или роли из claims.

## Flows (способы получить токен)

- **Authorization Code + PKCE** — основной для веб- и мобильных приложений. Используй его.
- **Client Credentials** — сервис к сервису, без пользователя: сервис отдаёт свой `client_id` и `client_secret`, получает access token.
- **Refresh Token** — обновить access token без повторного входа.
- **Device Code** — для устройств без браузера (Smart TV, CLI).
- **Implicit** и **Password (ROPC)** — устарели, в OAuth 2.1 и в Security BCP (RFC 9700) их не рекомендуют. Если спросят, скажи, почему: токен в URL и пароль в руках у клиента.

### Authorization Code + PKCE по шагам
```
1. Пользователь жмёт «Войти» → наш сервер генерирует state и code_verifier,
   редиректит в Keycloak: /auth?response_type=code&client_id=…&redirect_uri=…
   &scope=openid profile&state=…&code_challenge=SHA256(verifier)&code_challenge_method=S256
2. Пользователь вводит логин и пароль в Keycloak
3. Keycloak редиректит обратно: /callback?code=…&state=…
4. Наш сервер проверяет state, обменивает code + code_verifier на токены (POST /token)
5. Проверяет ID token, создаёт сессию, ставит пользователю cookie
```

- **state** — защита от CSRF: ответ должен прийти на запрос, который начали мы.
- **PKCE** — защита от перехвата `code`: без `code_verifier` его не обменять на токены.
- **nonce** — защита ID token от повторного использования.

### Где хранить токены в браузере
Самый безопасный вариант — **BFF (Backend for Frontend)**: токены хранит сервер, браузер получает только сессионную cookie с флагами `HttpOnly`, `Secure`, `SameSite=Lax`. Хранить токены в `localStorage` плохо: их украдёт любой XSS.

## Запуск Keycloak

```bash
docker run -d --name keycloak -p 8081:8080 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest start-dev
```
Открой http://localhost:8081, войди как `admin` / `admin` и:
1. Создай realm `tasks`.
2. Создай client `tasks-web`: Client authentication **On**, Standard flow **On**, Valid redirect URIs `http://localhost:8080/callback`. Скопируй secret со вкладки Credentials.
3. Создай пользователя, задай ему пароль (вкладка Credentials, Temporary **Off**).

Issuer будет `http://localhost:8081/realms/tasks`. Открой `http://localhost:8081/realms/tasks/.well-known/openid-configuration` и посмотри, что там.

## OIDC из Go

Библиотеки: `golang.org/x/oauth2` (сам OAuth-протокол) и `github.com/coreos/go-oidc/v3/oidc` (discovery, проверка ID token и JWKS).

### Вход пользователя (Authorization Code + PKCE)
```go
import (
    "github.com/coreos/go-oidc/v3/oidc"
    "golang.org/x/oauth2"
)

provider, err := oidc.NewProvider(ctx, "http://localhost:8081/realms/tasks")
if err != nil { return err }

oauthCfg := oauth2.Config{
    ClientID:     "tasks-web",
    ClientSecret: os.Getenv("OIDC_CLIENT_SECRET"),
    Endpoint:     provider.Endpoint(),
    RedirectURL:  "http://localhost:8080/callback",
    Scopes:       []string{oidc.ScopeOpenID, "profile", "email"},
}
idVerifier := provider.Verifier(&oidc.Config{ClientID: oauthCfg.ClientID})

mux.HandleFunc("GET /login", func(w http.ResponseWriter, r *http.Request) {
    state := randomString(32)
    pkce := oauth2.GenerateVerifier()
    setTempCookie(w, "oauth_state", state)  // HttpOnly, Secure, живёт 5 минут
    setTempCookie(w, "oauth_pkce", pkce)
    url := oauthCfg.AuthCodeURL(state, oauth2.S256ChallengeOption(pkce))
    http.Redirect(w, r, url, http.StatusFound)
})

mux.HandleFunc("GET /callback", func(w http.ResponseWriter, r *http.Request) {
    if r.URL.Query().Get("state") != cookieValue(r, "oauth_state") {
        http.Error(w, "bad state", http.StatusBadRequest)
        return
    }
    tok, err := oauthCfg.Exchange(r.Context(), r.URL.Query().Get("code"),
        oauth2.VerifierOption(cookieValue(r, "oauth_pkce")))
    if err != nil {
        http.Error(w, "exchange failed", http.StatusUnauthorized)
        return
    }
    rawID, ok := tok.Extra("id_token").(string)
    if !ok {
        http.Error(w, "no id_token", http.StatusUnauthorized)
        return
    }
    idToken, err := idVerifier.Verify(r.Context(), rawID)
    if err != nil {
        http.Error(w, "invalid id_token", http.StatusUnauthorized)
        return
    }
    var claims struct {
        Sub   string `json:"sub"`
        Email string `json:"email"`
    }
    if err := idToken.Claims(&claims); err != nil { /* ... */ }

    // создать сессию (например, в Redis), сохранить в ней tok.AccessToken и tok.RefreshToken,
    // отдать браузеру только id сессии в HttpOnly cookie
})
```
Для краткости здесь нет `nonce`. В своём коде добавь его: `oidc.Nonce(n)` в `AuthCodeURL` и сравнение с `idToken.Nonce` после проверки.

### Проверка access token в API (middleware)
```go
// В Keycloak добавь клиенту Audience mapper, чтобы в aud access token был "tasks-api".
// По умолчанию там "account".
atVerifier := provider.Verifier(&oidc.Config{ClientID: "tasks-api"})

func Auth(v *oidc.IDTokenVerifier) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            raw, ok := strings.CutPrefix(r.Header.Get("Authorization"), "Bearer ")
            if !ok {
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return
            }
            tok, err := v.Verify(r.Context(), raw) // подпись по JWKS, iss, aud, exp
            if err != nil {
                http.Error(w, "unauthorized", http.StatusUnauthorized)
                return
            }
            ctx := context.WithValue(r.Context(), userIDKey{}, tok.Subject)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```
`go-oidc` создан для ID token, но так же проверяет и JWT access token. Альтернатива: `github.com/golang-jwt/jwt/v5` + `github.com/MicahParks/keyfunc/v3` (JWKS с автообновлением ключей).

### Сервис к сервису (Client Credentials)
```go
import "golang.org/x/oauth2/clientcredentials"

cc := clientcredentials.Config{
    ClientID:     "notifier",
    ClientSecret: os.Getenv("NOTIFIER_SECRET"),
    TokenURL:     provider.Endpoint().TokenURL,
}
httpClient := cc.Client(ctx) // сам получает токен, кэширует и обновляет по истечении
resp, err := httpClient.Get("http://tasks-api:8080/internal/tasks")
```

## Ресурсы
- oauth.net/2 — обзор всех flows со ссылками на RFC
- RFC 6749 (OAuth 2.0), RFC 7636 (PKCE), RFC 9700 (Security Best Current Practice)
- openid.net/developers/how-connect-works
- jwt.io — раскодировать токен и посмотреть claims
- Книга *OAuth 2 in Action* (Justin Richer, Antonio Sanso)
- keycloak.org/documentation — Getting started с Docker

## Задачи
- [ ] ⭐ **Разбери токены.** Получи токен из Keycloak через `curl` (client credentials), раскодируй на jwt.io. Найди `iss`, `aud`, `exp`, `scope`, `kid`. Сравни `kid` с ключами из `jwks_uri`.
- [ ] ⭐⭐ **Вход через Keycloak.** Сделай в Tasks API `GET /login`, `GET /callback`, `POST /logout` по коду выше: state, PKCE, nonce, сессия в Redis, HttpOnly cookie. `GET /me` возвращает email пользователя.
- [ ] ⭐⭐ **Resource Server.** Middleware проверки access token для всех `/tasks`. Каждый пользователь видит только свои задачи (`owner_id = sub`). Роль `admin` из claims видит все. Тесты: нет токена, протухший токен, чужой `aud`, токен с подписью другим ключом.
- [ ] ⭐⭐ **Refresh.** Если access token в сессии истёк, сервер сам обновляет его через refresh token. Проверь с временем жизни access token 1 минута.
- [ ] ⭐⭐ **Client Credentials.** Сервис `notifier` ходит во внутренний эндпоинт Tasks API со своим токеном. Tasks API принимает только токены со scope `tasks.read`.
- [ ] ⭐⭐⭐ **Свой JWKS-кэш.** Без `go-oidc`: скачай `jwks_uri`, распарси ключи RSA, проверь подпись через `golang-jwt/jwt/v5`, кэшируй ключи и перезагружай их, если пришёл неизвестный `kid` (не чаще раза в минуту). Протестируй ротацию ключей в Keycloak.

## Вопросы для самопроверки
1. Чем OAuth 2.0 отличается от OpenID Connect?
2. Чем access token отличается от ID token? Почему ID token нельзя отправлять в API?
3. Зачем нужны `state`, PKCE и `nonce`?
4. Как API проверяет JWT, не обращаясь к серверу авторизации на каждый запрос? Как тогда отозвать токен?
5. Какой flow выберешь для SPA, для мобильного приложения, для общения двух сервисов?
6. Где хранить токены в браузере и почему не в `localStorage`?
