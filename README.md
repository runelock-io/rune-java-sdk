# Rune Spring Boot Starter

Use [Rune](https://github.com/runelock-io/rune) secrets in any Spring Boot backend.
Add one dependency, point it at your Rune server, and your DB passwords, API keys and JWT secrets come from Rune instead of `application.yml` or `.env` files.

**Requirements:** Java 17+, Spring Boot 3.x, a Rune server and a Rune token with read access to your namespace.

---

## 1. Add the dependency

In your backend's `pom.xml`:

```xml
<repositories>
    <repository>
        <id>github</id>
        <url>https://maven.pkg.github.com/runelock-io/rune-java-sdk</url>
    </repository>
</repositories>

<dependencies>
    <dependency>
        <groupId>io.rune</groupId>
        <artifactId>rune-spring-boot-starter</artifactId>
        <version>0.1.0</version>
    </dependency>
</dependencies>
```

> GitHub Packages needs a GitHub token (classic, `read:packages` scope) in `~/.m2/settings.xml` under `<server><id>github</id>...</server>` to download the dependency.

## 2. Store your secrets in Rune

Create your secrets in Rune under a namespace for your service, e.g. namespace `orders-service`:

| Key | Value |
|---|---|
| `db.password` | `s3cr3t-db-pass` |
| `redis.password` | `r3dis-pass` |
| `jwt.secret` | `super-long-signing-key` |
| `stripe.api-key` | `sk_live_...` |

## 3. Point your app at Rune

```yaml
# application.yml
rune:
  url: https://rune.yourcompany.com
  token: ${RUNE_TOKEN}          # comes from the environment, never commit it
  namespace: orders-service
```

## 4. Use the secrets

Every secret becomes a Spring property named `rune.<key>`. Use it anywhere you'd normally hardcode a secret:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/orders
    username: orders
    password: ${rune.db.password}

  data:
    redis:
      host: redis
      password: ${rune.redis.password}

app:
  jwt:
    secret: ${rune.jwt.secret}
```

Or inject them directly:

```java
@Service
public class PaymentService {

    @Value("${rune.stripe.api-key}")
    private String stripeApiKey;
}
```

Secrets are loaded at startup, **before** beans are created, so they work with `DataSource`, Redis, Kafka and anything else Spring Boot auto-configures. You don't need any extra code.

## 5. Run it

**Locally**
```bash
export RUNE_TOKEN=your-rune-token
mvn spring-boot:run
```

**Docker**
```bash
docker run -e RUNE_URL=https://rune.yourcompany.com \
           -e RUNE_TOKEN=your-rune-token \
           -e RUNE_NAMESPACE=orders-service \
           orders-service:latest
```

**Kubernetes**
```yaml
env:
  - name: RUNE_URL
    value: https://rune.yourcompany.com
  - name: RUNE_NAMESPACE
    value: orders-service
  - name: RUNE_TOKEN
    valueFrom:
      secretKeyRef:
        name: rune-token
        key: token
```

On startup you'll see:
```
[rune] Loaded 4 secret(s) from namespace 'orders-service'
```

---

## Fetching secrets at runtime

To look up a secret on demand (e.g. per tenant), inject `RuneClient`:

```java
@Service
public class TenantService {

    private final RuneClient rune;

    public TenantService(RuneClient rune) {
        this.rune = rune;
    }

    public String apiKeyFor(String tenantId) {
        return rune.getSecret("tenants", tenantId + ".api-key");
    }
}
```

## Configuration

| Property | Env variable | Default | Description |
|---|---|---|---|
| `rune.url` | `RUNE_URL` | — | Rune server URL (required) |
| `rune.token` | `RUNE_TOKEN` | — | Rune access token (required) |
| `rune.namespace` | `RUNE_NAMESPACE` | `default` | Namespace to load secrets from |
| `rune.enabled` | `RUNE_ENABLED` | `true` | Set `false` to turn Rune off |
| `rune.fail-fast` | `RUNE_FAIL_FAST` | `true` | App won't start if Rune is unreachable. Set `false` to start anyway |

## Tests

Turn Rune off in tests and provide dummy values:

```properties
# src/test/resources/application.properties
rune.enabled=false
rune.db.password=test
rune.jwt.secret=test-secret
```

## Notes

- Secrets are read **once at startup**. Restart the app after rotating a secret, or use `RuneClient` for values that change often.
- Don't name secrets `url`, `token`, `namespace`, `enabled` or `fail-fast`. Those names clash with the starter's own settings.
