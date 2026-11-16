# JWT

https://www.jwt.io/

<!-- Mantener aquí la captura de jwt.io del documento original -->

JWT (JSON Web Token) es un estándar para la creación de tokens de acceso. Se utiliza principalmente para autenticación y autorización en APIs REST.

> Antes de este documento, repasa la regla 5 de [API-REST.md](API-REST.md): una API REST es **stateless**, así que no puede usar la sesión del servidor como hacíamos en MVC con `formLogin`.

## Sesión (MVC) vs token (API REST)

| | MVC con `formLogin` | API REST con JWT |
|---|---|---|
| Dónde se guarda quién eres | En la sesión del **servidor** | En el **token**, que guarda el cliente |
| Qué envía el cliente en cada petición | La cookie `JSESSIONID` | La cabecera `Authorization: Bearer <token>` |
| El servidor recuerda al usuario | Sí | No: lo lee del token en cada petición |
| Cerrar sesión | Se invalida la sesión | El token caduca (`exp`) |

## Funcionamiento

1. **Autenticación:** el usuario envía sus credenciales (`POST /auth/login`).
2. **Generación:** el servidor comprueba las credenciales, genera un JWT firmado y lo devuelve al cliente.
3. **Autorización:** el cliente envía el JWT en las siguientes peticiones, en la cabecera `Authorization: Bearer <token>`.
4. **Verificación:** el servidor comprueba la firma del JWT antes de conceder acceso.

<!-- Mantener aquí el diagrama Browser/Server del documento original -->

## Necesidad en HTTP

- **Seguridad:** solo las peticiones con un token válido pueden acceder a recursos protegidos.
- **Eficiencia:** el servidor no necesita almacenar sesiones.
- **Escalabilidad:** varios servidores pueden atender al mismo cliente sin compartir sesiones.

## Partes

Un JWT está compuesto por tres partes, separadas por un punto (`.`):

1. Header (encabezado)
2. Payload (carga útil)
3. Signature (firma)

<!-- Mantener aquí el diagrama Header/Payload/Signature del documento original -->

### Estructura de un JWT

Un JWT completo podría verse así:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### 1. Header (encabezado)

El encabezado indica el tipo de token (JWT) y el algoritmo de firma utilizado (por ejemplo, HMAC SHA256 o RSA).

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

Este JSON se codifica en Base64Url para formar la primera parte del JWT.

### 2. Payload (carga útil)

El payload contiene las reclamaciones (*claims*): declaraciones sobre una entidad (normalmente, el usuario) y datos adicionales. Hay tres tipos:

- **Registrados (Registered Claims):** claims predefinidos, no obligatorios pero recomendados, como `iss` (emisor), `exp` (expiración), `sub` (sujeto) y `aud` (audiencia).
- **Públicos (Public Claims):** para información personalizada. Deben evitar colisiones de nombres usando URLs o URNs como nombres.
- **Privados (Private Claims):** para compartir información entre partes que han acordado usarlos y que no están registrados.

```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "iat": 1516239022,
  "exp": 1516242622
}
```

- `sub` (*subject*): identificador del usuario al que pertenece el token (su id o su username).
- `name`: nombre del usuario.
- `admin`: si el usuario tiene permisos de administrador.
- `iat` (*issued at*): fecha de emisión, como timestamp UNIX (segundos desde el 1 de enero de 1970).
- `exp` (*expiration*): fecha de caducidad, en el mismo formato. A partir de ese momento el token deja de ser válido.

Este JSON también se codifica en Base64Url para formar la segunda parte del JWT.

> ⚠️ **El payload no está cifrado, solo codificado.** Cualquiera que tenga el token puede leerlo (pruébalo en jwt.io). **Nunca** metas contraseñas ni datos sensibles en el payload.

### 3. Signature (firma)

Para crear la firma, se toman el header y el payload codificados en Base64Url, se unen con un punto (`.`) y se firma esa cadena con el algoritmo indicado en el header y una clave secreta.

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret)
```

La firma garantiza que el token **no se ha modificado**: si alguien cambia el payload (por ejemplo, `"admin": true`), la firma deja de coincidir y el servidor rechaza el token. Por eso la clave secreta nunca sale del servidor.
