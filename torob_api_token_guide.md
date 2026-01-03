# Torob API Token Guide: JWT Validation
## Authentication for Product API v3

## 1. Introduction

All requests to the TorobAPI v3 endpoint include a JWT token in the `X-Torob-Token` header. By validating this token, you can ensure the request originates from Torob and prevent unauthorized access to your product data.

The token is signed using Torob's private key and can be verified using Torob's public key provided below.

## 2. Torob Public Key

Use this public key to validate JWT signatures. The algorithm used is **EdDSA (ed25519)**.

```pem
-----BEGIN PUBLIC KEY-----
MCowBQYDK2VwAyEAt6Mu4T0pBORY11W+QeM35UsmLO3vsf+6yKpFDEImFk0=
-----END PUBLIC KEY-----
```

## 3. Request Headers

Each request includes the following authentication headers:

| Header                  | Example                            | Description |
| ----------------------- | ---------------------------------- | ----------- |
| `X-Torob-Token`         | (base64 encoded JWT string)        | The JWT for authentication |
| `X-Torob-Token-Version` | `1`                                | Token version |

**Example Request:**
```bash
curl --header "Content-Type: application/json" \
     --header "Accept: application/json" \
     --header "X-Torob-Token: [jwt_token]" \
     --header "X-Torob-Token-Version: 1" \
     --request POST \
     --data '{"page": 1, "sort": "date_added_desc"}' \
     'https://example.com/torob_api/v3/products'
```

## 4. JWT Structure

The JWT contains the following structure:

```json
{
  "header": {"alg": "EdDSA", "typ": "JWT", "v": 1},
  "payload": {"aud": "api.example.com", "exp": 1730206744, "nbf": 1730206000}
}
```

### 4.1. Payload Claims

| Claim | Description |
| ----- | ----------- |
| `exp` | **Expiration Time**: Unix timestamp (seconds). Reject requests after this time. |
| `nbf` | **Not Before**: Unix timestamp (seconds). Reject requests before this time. |
| `aud` | **Audience**: Must match your API's hostname. Critical for security. |

### 4.2. Audience (aud) Examples

The `aud` value corresponds to the `Host` header of the request:

| API URL | Expected `aud` Value |
| ------- | -------------------- |
| `https://example.com/torob_api/v3/` | `example.com` |
| `https://api.example.com/v3/` | `api.example.com` |
| `https://api.example.com:8080/v3/` | `api.example.com:8080` |

> **Security Note**: Always validate the `aud` claim matches your API hostname. This prevents tokens generated for other shops from being used on your endpoint.

## 5. Validation Requirements

When validating the JWT:

1. **Verify the signature** using Torob's public key and the EdDSA algorithm
2. **Check `exp`**: Reject if current time > `exp`
3. **Check `nbf`**: Reject if current time < `nbf`
4. **Check `aud`**: Must exactly match your API's hostname

> **Important**: Ensure your server's clock is synchronized (NTP). Incorrect server time will cause valid tokens to be rejected.

## 6. Code Samples

### Python

Using the `pyjwt[crypto]` library:

```python
import jwt

PUBLIC_KEY = """
-----BEGIN PUBLIC KEY-----
MCowBQYDK2VwAyEAt6Mu4T0pBORY11W+QeM35UsmLO3vsf+6yKpFDEImFk0=
-----END PUBLIC KEY-----
"""

def validate_token(token: str):
    # exp and nbf fields are checked by PyJWT library
    jwt.decode(
        token,
        key=PUBLIC_KEY,
        algorithms=["EdDSA"],
        audience="[expected_aud_value]"  # Replace with your API hostname
    )
```

### Go

Using the `github.com/golang-jwt/jwt/v5` library:

```go
package main

import (
    "crypto"
    "fmt"
    "github.com/golang-jwt/jwt/v5"
    "log"
)

func verify(token string, parser *jwt.Parser, key crypto.PublicKey) (*jwt.Token, error) {
    parsedToken, err := parser.ParseWithClaims(token, &jwt.MapClaims{}, func(token *jwt.Token) (interface{}, error) {
        return key, nil
    })
    if err != nil {
        return nil, fmt.Errorf("unable to parse token: %v", err)
    }
    return parsedToken, nil
}

func main() {
    publicKey := []byte(`-----BEGIN PUBLIC KEY-----
MCowBQYDK2VwAyEAt6Mu4T0pBORY11W+QeM35UsmLO3vsf+6yKpFDEImFk0=
-----END PUBLIC KEY-----`)

    publicKeyPem, err := jwt.ParseEdPublicKeyFromPEM(publicKey)
    if err != nil {
        log.Fatal(err)
    }

    // aud, exp, and nbf fields are checked by jwt library
    parser := jwt.NewParser(
        jwt.WithAudience("[expected_aud_value]"),  // Replace with your API hostname
        jwt.WithExpirationRequired(),
        jwt.WithValidMethods([]string{"EdDSA"}),
    )

    // parser and publicKeyPem are constant - compute once
    token := "..."  // JWT token from request

    if _, err = verify(token, parser, publicKeyPem); err != nil {
        log.Fatal(err)
    }
}
```

### Java

Using the `io.jsonwebtoken:jjwt` library:

```java
import io.jsonwebtoken.*;
import io.jsonwebtoken.io.Decoders;
import java.security.*;
import java.security.spec.*;

public class JwtVerifier {
    private final JwtParser parser;

    public JwtVerifier() throws NoSuchAlgorithmException, InvalidKeySpecException {
        final var publicKeyString = "MCowBQYDK2VwAyEAt6Mu4T0pBORY11W+QeM35UsmLO3vsf+6yKpFDEImFk0=";
        KeySpec keySpec = new X509EncodedKeySpec(Decoders.BASE64.decode(publicKeyString));
        PublicKey publicKey = KeyFactory.getInstance("EdDSA").generatePublic(keySpec);

        // aud and exp fields are checked by library
        parser = Jwts.parser()
            .requireAudience("[expected_aud_value]")  // Replace with your API hostname
            .verifyWith(publicKey)
            .build();
    }

    public Jws<Claims> verifyToken(String token) {
        return parser.parseSignedClaims(token);
    }
}
```

### PHP

Using the `firebase/php-jwt` library:

```php
use Firebase\JWT\JWT;
use Firebase\JWT\Key;

define('TOROB_PUBLIC_KEY', 'MCowBQYDK2VwAyEAt6Mu4T0pBORY11W+QeM35UsmLO3vsf+6yKpFDEImFk0=');
define('TOROB_PUBLIC_KEY_SEED', base64_encode(substr(base64_decode(TOROB_PUBLIC_KEY), -32)));

function verify($jwt): object {
    // exp is checked by library but we should check aud manually
    $decoded = JWT::decode($jwt, new Key(TOROB_PUBLIC_KEY_SEED, 'EdDSA'));

    if ($decoded->aud !== "[expected_aud_value]") {  // Replace with your API hostname
        throw new \Exception("Invalid audience");
    }

    return $decoded;
}
```

## 7. Troubleshooting

| Issue | Solution |
| ----- | -------- |
| Token always rejected | Check server clock synchronization |
| Invalid signature | Ensure you're using the correct public key |
| Audience mismatch | Verify your API hostname matches the `aud` claim |
