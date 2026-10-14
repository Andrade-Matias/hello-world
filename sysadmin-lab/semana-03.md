# Semana 3 — Nginx, PHP-FPM y Oracle (SQL Developer)

**Objetivo:** operar el stack web en Debian y diagnosticar la capa **Oracle** como sysadmin (listener, red, servicio), usando **SQL Developer** desde tu PC y SSH en `srv-oracle`.

**VMs (Proxmox):**

| VM | IP | Rol |
|----|-----|-----|
| `srv-web` | 10.10.10.20 | Nginx + PHP-FPM |
| `srv-oracle` | 10.10.10.30 | Oracle Database Free / XE |

**Cliente:** SQL Developer en tu workstation → `10.10.10.30:1521` / service `FREEPDB1` (ajustá a tu instalación).

**Docs:**

- [Nginx — Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html)
- [PHP-FPM](https://www.php.net/manual/en/install.fpm.php)
- [Oracle Database Free — Get started](https://www.oracle.com/database/free/get-started/)
- [SQL Developer](https://www.oracle.com/database/sqldeveloper/technologies/download/)
- Listener: documentación *Net Services* (Oracle Docs) — `lsnrctl`, `listener.ora`

**Pre-requisito:** [plataforma-proxmox-oracle.md](./plataforma-proxmox-oracle.md) — Oracle instalado y conexión SQL Developer OK antes del jueves.

---

## Martes — Nginx en `srv-web` (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | `sites-available` / `sites-enabled`, `root`, `index`, `location` |
| 25–45 | Logs: `/var/log/nginx/access.log`, `error.log` |
| 45–60 | `nginx -t`, `systemctl reload nginx` vs `restart` |

### Bloque práctica (~60 min)

En Proxmox: crear VM `srv-web` si no existe; snapshot `clean-install`.

```bash
sudo apt update
sudo apt install -y nginx
sudo mkdir -p /var/www/app/public
echo '<?php phpinfo();' | sudo tee /var/www/app/public/index.php
```

Crear sitio `app.conf` con `root /var/www/app/public;` y `index index.php index.html;`.

**Entregable:** desde tu PC `curl http://10.10.10.20` devuelve HTML.

---

## Miércoles — PHP-FPM (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | PHP-FPM pools: `www.conf`, usuario `www-data` |
| 25–45 | Nginx `fastcgi_pass unix:/run/php/php8.2-fpm.sock` |
| 45–60 | App en un servidor, BD en otro: latencia y firewall |

### Bloque práctica (~60 min)

```bash
sudo apt install -y php-fpm
# Opcional semana 3+: php-oci8 si querés PHP→Oracle (paquetes según Debian/PECL)
sudo nginx -t && sudo systemctl reload nginx
```

Configurar `location ~ \.php$` con `fastcgi_param SCRIPT_FILENAME` correcto.

Verificar: `curl -s http://127.0.0.1/index.php | head` en `srv-web`.

---

## Jueves — Oracle: listener, servicio y SQL Developer (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–20 | Instancia vs PDB; **service name** vs SID (conexión moderna = service) |
| 20–40 | Listener `1521`, `lsnrctl status`, logs de red Oracle |
| 40–50 | Usuario de aplicación en PDB: `CREATE USER`, `GRANT CONNECT, RESOURCE` |
| 50–60 | Sysadmin: `systemctl` de Oracle en Linux, espacio en `/u01`, swap |

### Preparación en `srv-oracle` (una vez)

Como usuario administrador de BD (ej. en SQL*Plus o SQL Developer con usuario privilegiado en el PDB):

```sql
CREATE USER appuser IDENTIFIED BY "LabPass123!" CONTAINER = CURRENT;
GRANT CONNECT, RESOURCE TO appuser;
-- opcional: cuota en tablespace
ALTER USER appuser QUOTA 100M ON USERS;
```

Probar en SQL Developer: conexión `appuser` → `FREEPDB1` @ `10.10.10.30`.

### Ticket T-007 — SQL Developer: ORA-12541 / connection refused

**Prioridad:** Alta  
**Síntoma:** Desde SQL Developer, **Test** falla: `ORA-12541: TNS:no listener` o timeout. La app PHP (si la usás) no conecta a Oracle.

**Setup (en `srv-oracle`, como root o oracle):**

```bash
# Simular listener caído
sudo systemctl stop oracle-free-23c    # ajustar nombre real: systemctl list-units '*oracle*'
# o: sudo -u oracle lsnrctl stop
```

Opcional segundo fallo: en `srv-oracle` firewall sin 1521:

```bash
sudo ufw deny 1521/tcp
```

**Criterios de cierre:**

- SQL Developer: **Test** en verde con `appuser` al PDB.
- Desde `srv-web`: `nc -zv 10.10.10.30 1521` → succeeded (herramienta de red, no sustituye SQL Developer).
- Servicio Oracle **enabled**; `lsnrctl status` muestra `READY`.
- Bitácora: nombre del unit systemd y puerto abierto.

**Pistas:** `ss -tlnp | grep 1521`, `journalctl -u oracle-free-23c -n 50`, `lsnrctl start`.

### Bloque práctica (~60 min)

Resolver T-007; snapshot Proxmox `post-T-007` en `srv-oracle`.

---

## Viernes — Logs web + correlación con BD (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–20 | Nginx: 4xx/5xx en access log |
| 20–40 | PHP-FPM log; errores OCI (si usás PHP-OCI): `ORA-` en `error.log` |
| 40–60 | Oracle: alert log (ubicación en `srv-oracle`), `V$SESSION` desde SQL Developer |

### Ticket T-008 — Sitio lento; BD responde tarde

**Prioridad:** Media  
**Síntoma:** Páginas PHP lentas; SQL Developer muestra consultas colgadas o el listener acepta pero la sesión tarda.

**Setup A — capa web (en `srv-web`):**

```bash
sudo tee /var/www/app/public/slow.php << 'EOF'
<?php
sleep(5);
echo "done";
EOF
```

**Setup B — capa Oracle (opcional, en SQL Developer como SYSDBA o DBA):**

```sql
-- Sesión de prueba que simula bloqueo/lentitud (solo lab)
-- Alternativa simple: en srv-oracle limitar CPU en Proxmox a 1 core temporalmente
```

**Tu trabajo:**

1. `time curl -s http://10.10.10.20/slow.php` — aislar si es PHP/web.
2. En SQL Developer: ejecutar `SELECT * FROM v$session WHERE username = 'APPUSER';` mientras probás conexión.
3. Cierre: ¿falla en Nginx, PHP-FPM o Oracle? Evidencia en una línea de log de cada capa que tocaste.

### Bloque práctica (~60 min)

Informe 5 líneas + captura SQL Developer o `tail` de log Nginx.

---

## Sábado — Dependencias y orden de arranque (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–30 | Oracle debe estar arriba antes que la app que depende de 1521 |
| 30–60 | `After=network-online.target` en units; arranque manual en incidentes |

### Bloque práctica (~60 min)

| Min | Tarea |
|-----|--------|
| 0–30 | Parar Oracle en `srv-oracle` y PHP/Nginx en `srv-web`; levantar solo Nginx y ver error en app |
| 30–60 | Documentar orden: **Oracle listener → PHP-FPM → Nginx** |

**Smoke desde `srv-web`:**

```bash
#!/bin/bash
set -e
curl -sf http://127.0.0.1/index.php >/dev/null
nc -zv 10.10.10.30 1521
echo "smoke ok"
```

Guardar como `/usr/local/bin/smoke-web.sh` y ejecutar tras cada cambio.

**SQL Developer:** script de comprobación manual — abrir conexión `appuser` y ejecutar `SELECT 1 FROM dual;`.

---

## Domingo — Incidente integrado (2 h)

### Ticket T-009 — HTTP 500 / app caída tras deploy

**Prioridad:** Crítica  
**Síntoma:** Tras deploy, la home devuelve 500 o blanco; Nginx “activo”; usuarios reportan que “la base no anda”.

**Romper una o más capas:**

**En `srv-web`:**

```bash
echo '<?php broken' | sudo tee /var/www/app/public/index.php
# o permisos root:root chmod 600 en index.php
# o fastcgi_pass apuntando a socket PHP incorrecto
```

**En `srv-oracle`:**

```bash
sudo systemctl stop oracle-free-23c   # o listener stop
# o cambiar password de appuser sin actualizar credenciales documentadas
```

**Criterios de cierre (120 min):**

| Fase | Tiempo | Acción |
|------|--------|--------|
| Impacto | 10 min | ¿Solo web o también SQL Developer? |
| Diagnóstico | 40 min | `curl -I`, Nginx error log, `php -l`, `lsnrctl status`, SQL Developer Test |
| Fix | 30 min | Cambio mínimo + smoke-web + `SELECT 1 FROM dual` |
| Cierre | 20 min | Mensaje al cliente + prevención |
| Extra | 20 min | Backup lógico lab: `expdp`/`Data Pump` o export SQL Developer del esquema `APPUSER` (según permisos) |

Snapshot Proxmox `pre-week-4` al terminar.

---

## Checklist fin semana 3

- [ ] `srv-web` y `srv-oracle` en Proxmox con snapshots documentados
- [ ] SQL Developer conecta a `FREEPDB1` con `appuser`
- [ ] T-007, T-008, T-009 en bitácora
- [ ] Runbook 1 página: “Web OK pero Oracle KO” (SSH en oracle + SQL Developer + puerto 1521)

---

## Soluciones

<details>
<summary>T-007 — Solución esperada</summary>

En `srv-oracle`:

```bash
sudo systemctl start oracle-free-23c    # nombre real según list-units
sudo -u oracle lsnrctl start
sudo -u oracle lsnrctl status
ss -tlnp | grep 1521
sudo ufw allow from 10.10.10.0/24 to any port 1521 proto tcp
```

SQL Developer: service name correcto (`FREEPDB1`), host `10.10.10.30`.

Si `ORA-01017`: reset password `appuser` en el PDB y actualizar credenciales guardadas en SQL Developer.

</details>

<details>
<summary>T-008 — Solución esperada</summary>

Si `slow.php` es la causa: eliminar o optimizar; ajustar `pm.max_children` en FPM si hay cola.

Si Oracle: restaurar CPU en Proxmox, revisar sesiones en `v$session`, alert log por errores de espacio o archiver.

Separar claramente en el informe capa web vs capa BD.

</details>

<details>
<summary>T-009 — Solución esperada</summary>

```bash
# srv-web
sudo tail -20 /var/log/nginx/error.log
php -l /var/www/app/public/index.php
sudo chown www-data:www-data /var/www/app/public/index.php
sudo chmod 644 /var/www/app/public/index.php
sudo nginx -t && sudo systemctl reload nginx

# srv-oracle
sudo systemctl start oracle-free-23c
sudo -u oracle lsnrctl start
```

SQL Developer Test OK; `curl` HTTP 200 en la home.

</details>

---

## Opcional — PHP conectando a Oracle (`db.php`)

Si querés unir tu perfil PHP con Oracle (no obligatorio para cerrar tickets):

```php
<?php
// Requiere OCI8 instalado en srv-web
$conn = oci_connect('appuser', 'LabPass123!', '10.10.10.30/FREEPDB1');
if (!$conn) {
    $e = oci_error();
    exit(htmlentities($e['message']));
}
echo "connected";
```

Diagnosticar con el mismo criterio que T-007: red 1521, service name, credenciales.
