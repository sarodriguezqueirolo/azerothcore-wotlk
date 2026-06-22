# Creación de cuentas de juego

## Requisitos

- Python 3
- Contenedor Docker `ac-database` corriendo
- Puerto `3307` accesible en `127.0.0.1`

## Credenciales de la base de datos

Definidas en `.env` en la raíz del proyecto:

| Variable | Valor |
|---|---|
| `DOCKER_DB_EXTERNAL_PORT` | `3307` |
| `DOCKER_DB_ROOT_PASSWORD` | `password` |

El contenedor expone MySQL en el puerto `3307` (mapeado al 3306 interno del contenedor).

## 1. Acceder a MySQL (opcional, para verificar)

```bash
mysql -h 127.0.0.1 -P 3307 -u root -p
# Password: password
```

O en una línea para scripts:

```bash
mysql -h 127.0.0.1 -P 3307 -u root -ppassword acore_auth
```

## 2. Crear cuenta

Ejecutá el siguiente script Python en la terminal. Cambiá `usuario` y `contraseña` por los valores que quieras.

```python
import hashlib, os, subprocess

# ── CONFIGURACIÓN ───────────────────────────────────
usuario = "MIUSUARIO"       # se guarda en minúscula en la DB
password = "MiPassword123"  # tal cual se tipea en el cliente
# ────────────────────────────────────────────────────

N = 0x894B645E89E1535BBDAD5B8B290650530801B18EBFBF5E8FAB3C82872A3E9BB7
g = 7

# El cliente WoW uppercases ambos antes del SRP6
u_upper = usuario.upper()
p_upper = password.upper()

salt = os.urandom(32)

inner = hashlib.sha1((u_upper + ":" + p_upper).encode()).digest()
x_bytes = hashlib.sha1(salt + inner).digest()
x_int = int.from_bytes(x_bytes, 'little')
verifier = pow(g, x_int, N)
verifier_bytes = verifier.to_bytes(32, 'little')

db = ["mysql", "-h", "127.0.0.1", "-P", "3307", "-u", "root", "-ppassword", "acore_auth"]

salt_hex = salt.hex()
ver_hex = verifier_bytes.hex()

# Eliminar cuenta si ya existe
subprocess.run(db + ["-e", f"DELETE FROM account_access WHERE id IN (SELECT id FROM account WHERE username='{usuario.lower()}')"])
subprocess.run(db + ["-e", f"DELETE FROM account WHERE username='{usuario.lower()}'"])

# Insertar cuenta nueva
subprocess.run(db + ["-e",
    f"INSERT INTO account (username, salt, verifier, expansion, joindate, last_ip) "
    f"VALUES ('{usuario.lower()}', UNHEX('{salt_hex}'), UNHEX('{ver_hex}'), 2, NOW(), '127.0.0.1')"
])

# Asignar GM level (opcional: nivel 3 = máximo, -1 = todos los reinos)
subprocess.run(db + ["-e",
    f"INSERT INTO account_access (id, gmlevel, RealmID) "
    f"SELECT id, 3, -1 FROM account WHERE username='{usuario.lower()}'"
])

print(f"Cuenta '{usuario}' creada correctamente.")
print(f"Password: {password}")
```

Copiá el script completo, pegalo en la terminal y presioná Enter.

## 3. Verificar

```bash
mysql -h 127.0.0.1 -P 3307 -u root -ppassword acore_auth \
  -e "SELECT id, username, gmlevel FROM account a LEFT JOIN account_access ag ON a.id=ag.id WHERE a.username='miusuario'"
```

## Notas técnicas

- El cliente WoW 3.3.5a **uppercases el username y el password** antes de calcular el hash SRP6.
- El hash interno es `SHA1(USERNAME_UPPER + ":" + PASSWORD_UPPER)`.
- El verifier es `7^x mod N` donde `x = SHA1(salt + SHA1(INNER))` interpretado como **little-endian**.
- El verifier se almacena en la DB en **little-endian** (OpenSSL `BN_lebin2bn` / `BN_bn2lebinpad`).
- La columna `username` usa collation `utf8mb4_unicode_ci` (case-insensitive).
- `expansion=2` permite Wrath of the Lich King.
