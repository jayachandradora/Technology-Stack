

# Details of Kafka SASL Mechanisms & Security Protocol

---

# ✅ 1. SASL Mechanisms for Producing Messages to Kafka

Kafka supports several **SASL (Simple Authentication and Security Layer)** mechanisms for authentication between client (producer/consumer) and broker.

### **Common SASL Mechanisms in Kafka**

| Mechanism                             | Description                                                     | When Used                           |
| ------------------------------------- | --------------------------------------------------------------- | ----------------------------------- |
| **PLAIN**                             | Username/password in plain text (usually with SSL to secure it) | Simple auth, non-enterprise systems |
| **SCRAM-SHA-256** / **SCRAM-SHA-512** | Secure salted password hashing                                  | More secure alternative to PLAIN    |
| **GSSAPI (Kerberos)**                 | Kerberos-based authentication                                   | Enterprise AD/kerberos environments |
| **OAUTHBEARER**                       | OAuth 2.0 / OIDC tokens                                         | Cloud/Kafka SaaS/auth providers     |
| **AWS_MSK_IAM**                       | IAM-based authentication (for AWS MSK)                          | AWS MSK only                        |

---

# ✅ 2. SASL_OAUTHBEARER for Kafka Producers

When using **OAUTHBEARER**, the Kafka producer obtains a **JWT or access token** from an OAuth/OIDC provider (Keycloak, Okta, Auth0, etc.) and presents it to the Kafka broker during authentication.

### Example Producer Config for OAUTHBEARER

```properties
security.protocol=SASL_SSL
sasl.mechanism=OAUTHBEARER
sasl.login.callback.handler.class=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginCallbackHandler
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
    oauth.token.endpoint.uri="https://auth.example.com/oauth2/token" \
    oauth.client.id="kafka-producer" \
    oauth.client.secret="my-secret" \
    oauth.scope="kafka.write";
```

### What happens internally

1. **Producer uses the login callback handler** to get a token.
2. Kafka client sends token → Broker verifies token (JWT signature / introspection).
3. If valid, connection is authenticated.

---

# ✅ 3. `SASL_JAAS_CONFIG` (or `sasl.jaas.config`) Explained

`SASL_JAAS_CONFIG` is where you provide **credentials** for the SASL mechanism.

It always contains a Java login module.

### Example for SASL/PLAIN

```properties
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
  username="producer" \
  password="secret";
```

### Example for SCRAM

```properties
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="scramUser" \
  password="scramPwd";
```

### Example for OAuth

```properties
sasl.mechanism=OAUTHBEARER
sasl.jaas.config=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required \
  oauth.client.id="clientId" \
  oauth.client.secret="clientSecret";
```

👉 **It tells Kafka which LoginModule to use and with what config.**

---

# ✅ 4. `sasl.login.callback.handler.class` Explained

This property points to the **class that performs authentication token retrieval**.

Kafka uses callback handlers for mechanisms that require dynamic actions such as:

* Generating OAuth tokens
* Refreshing tokens
* Interacting with Kerberos

### Built-In Handler for OAUTHBEARER

```properties
sasl.login.callback.handler.class=org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginCallbackHandler
```

This class:

* contacts OAuth server
* gets access token
* caches token
* refreshes token on expiration

### For Custom Token Generation (e.g., your own JWT service)

```properties
sasl.login.callback.handler.class=com.mycompany.kafka.CustomBearerTokenHandler
```

You implement:

```java
public class CustomBearerTokenHandler
        implements AuthenticateCallbackHandler {
    @Override
    public void handle(Callback[] callbacks) {
        // Your logic to fetch / generate a JWT
    }
}
```

---

# ✅ 5. Summary Table

| Setting                             | Purpose                                                |
| ----------------------------------- | ------------------------------------------------------ |
| `sasl.mechanism`                    | Which authentication method Kafka should use           |
| `sasl.jaas.config`                  | Credentials or configuration needed for that mechanism |
| `sasl.login.callback.handler.class` | Custom logic for producing tokens (OAuth)              |

---

# Apache Kafka, the security.protocol

In Apache Kafka, the **`security.protocol`** setting defines *how* the client (producer/consumer/admin) connects to the brokers—specifically what security layer is used.

For producing messages, you can use any of the following **valid `security.protocol` values**:

---

## ✅ **Kafka `security.protocol` Options**

### **1. `PLAINTEXT`**

* No authentication
* No encryption
* **Not secure**, only use in internal/trusted networks

---

### **2. `SSL`**

* TLS encryption
* Optional client certificates for mutual auth
* Good for encrypted communication without SASL

---

### **3. `SASL_PLAINTEXT`**

* SASL authentication
* No encryption
* Use only in trusted networks
* With SASL mechanisms like:

  * `PLAIN`
  * `SCRAM-SHA-256`
  * `SCRAM-SHA-512`
  * `GSSAPI` (Kerberos)
  * `OAUTHBEARER`

---

### **4. `SASL_SSL`**  ⭐ *Most common secure option*

* **TLS encryption + SASL authentication**
* Recommended for production
* Supports the same SASL mechanisms as above
* Example:

  ```properties
  security.protocol=SASL_SSL
  sasl.mechanism=SCRAM-SHA-512
  ```

---

## Summary Table

| security.protocol | Encryption | Authentication  | Notes         |
| ----------------- | ---------- | --------------- | ------------- |
| `PLAINTEXT`       | ❌          | ❌               | No security   |
| `SSL`             | ✅          | Optional (mTLS) | TLS only      |
| `SASL_PLAINTEXT`  | ❌          | ✅ (SASL)        | Auth only     |
| `SASL_SSL`        | ✅          | ✅ (SASL)        | Best practice |

---

## Example Producer Config Using SASL_SSL (SCRAM)

```properties
bootstrap.servers=broker:9093
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required \
  username="user" \
  password="password";
```

---
