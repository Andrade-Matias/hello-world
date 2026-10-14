# Semana 3 — Nginx, PHP-FPM y MariaDB (mentalidad admin)

**Objetivo:** levantar el stack que ya conocés como dev, pero depurarlo con `systemctl`, logs y tickets de producción.

**VMs:**

- `srv-web` — `10.10.10.20` (recomendado dedicada; puede ser la misma `srv-core` si RAM aprieta).
- Red: gateway `10.10.10.1`, DNS funcional (semana 2).

**Docs:**

- [Nginx — Beginner’s Guide](https://nginx.org/en/docs/beginners_guide.html)
- [PHP-FPM](https://www.php.net/manual/en/install.fpm.php)
- [MariaDB — Backup and restore](https://mariadb.com/kb/en/backing-up-and-restoring-databases/)

---

## Martes — Nginx instalación y virtual host (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | `sites-available` / `sites-enabled`, `root`, `index`, `location` |
| 25–45 | Logs: `/var/log/nginx/access.log`, `error.log` |
| 45–60 | `nginx -t`, `systemctl reload nginx` vs `restart` |

### Bloque práctica (~60 min)

```bash
sudo apt update
sudo apt install -y nginx
sudo mkdir -p /var/www/app/public
echo '<?php phpinfo();' | sudo tee /var/www/app/public/index.php
```

Crear sitio `app.conf` con `root /var/www/app/public;` y `index index.php index.html;` (PHP se activa mañana).

**Entregable:** `curl http://127.0.0.1` devuelve HTML (aunque PHP aún no corra).

---

## Miércoles — PHP-FPM y socket (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | PHP-FPM pools: `www.conf`, usuario `www-data` |
| 25–45 | Nginx `fastcgi_pass unix:/run/php/php8.2-fpm.sock` (versión según `ls /run/php/`) |
| 45–60 | Permisos: Nginx no lee archivos que no puede abrir |

### Bloque práctica (~60 min)

```bash
sudo apt install -y php-fpm php-mysql
sudo nginx -t && sudo systemctl reload nginx
```

Configurar `location ~ \.php$` con `fastcgi_param SCRIPT_FILENAME` correcto.

Verificar: `curl -s http://127.0.0.1/index.php | head` muestra salida PHP.

---

## Jueves — MariaDB como servicio (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | Instalación, `bind-address`, socket `/run/mysqld/mysqld.sock` |
| 25–45 | Usuarios: app vs admin; `mysql -u root` |
| 45–60 | Backup lógico: `mariadb-dump` |

### Ticket T-007 — Error connecting to database

**Prioridad:** Alta  
**Síntoma:** La app PHP muestra `SQLSTATE[HY000] [2002] Connection refused` o access denied.

**App de prueba** en `/var/www/app/public/db.php`:

```php
<?php
$pdo = new PDO('mysql:host=127.0.0.1;dbname=appdb', 'appuser', 'wrongpass');
echo "connected";
```

**Setup:**

```bash
sudo apt install -y mariadb-server
sudo mysql -e "CREATE DATABASE appdb; CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'LabPass123!'; GRANT ALL ON appdb.* TO 'appuser'@'localhost'; FLUSH PRIVILEGES;"
# Dejar password incorrecto en db.php a propósito
```

**Criterios de cierre:**

- `db.php` imprime `connected` sin exponer password en logs.
- Servicio `mariadb` enabled y arranca tras reboot (verificar con `systemctl is-enabled`).

### Bloque práctica (~60 min)

Resolver T-007; corregir credenciales o host (`127.0.0.1` vs `localhost` socket).

---

## Viernes — Logs y correlación (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–20 | Nginx: códigos 4xx/5xx en access log |
| 20–40 | PHP-FPM: `/var/log/php8.2-fpm.log`, pool `slowlog` (opcional) |
| 40–60 | MariaDB: `journalctl -u mariadb` |

### Ticket T-008 — Sitio lento y errores intermitentes

**Prioridad:** Media  
**Síntoma:** Usuarios reportan lentitud; access log muestra muchos `499`/`504` o tiempos altos.

**Setup (simular carga / error):**

```bash
# Script lento en PHP
sudo tee /var/www/app/public/slow.php << 'EOF'
<?php
sleep(5);
echo "done";
EOF
```

Opcional: bajar `pm.max_children` en FPM a `1` para ver cola.

**Tu trabajo:**

1. Una petición: `time curl -s http://127.0.0.1/slow.php`
2. Mientras corre, en otra terminal: `sudo tail -f /var/log/nginx/access.log` y log FPM.
3. Cierre: qué capa causó la lentitud y qué harías en prod (timeout, pool, cache).

### Bloque práctica (~60 min)

Resolver T-008 con informe de 5 líneas + captura de línea de log relevante.

---

## Sábado — Dependencias systemd (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–30 | Orden de arranque: `After=`, `Requires=`, `Wants=` |
| 30–60 | Unidad nginx que arranca tras `network-online.target` |

### Bloque práctica (~60 min)

| Min | Tarea |
|-----|--------|
| 0–30 | `systemctl stop mariadb php8.2-fpm nginx`; arrancar solo nginx y ver fallo PHP |
| 30–60 | Definir orden de arranque manual documentado: `mariadb` → `php8.2-fpm` → `nginx` |

Script de smoke test (guardar en `/usr/local/bin/smoke-web.sh`):

```bash
#!/bin/bash
set -e
curl -sf http://127.0.0.1/index.php >/dev/null
curl -sf http://127.0.0.1/db.php >/dev/null
echo "smoke ok"
```

---

## Domingo — Incidente integrado (2 h)

### Ticket T-009 — Página en blanco / HTTP 500

**Prioridad:** Crítica  
**Síntoma:** Tras “deploy”, la home devuelve 500 o blanco; Nginx sigue “verde”.

**Setup (romper una de estas, o varias):**

```bash
# 1) Syntax error PHP
echo '<?php broken' | sudo tee /var/www/app/public/index.php

# 2) Permisos
sudo chown root:root /var/www/app/public/index.php
sudo chmod 600 /var/www/app/public/index.php

# 3) Socket FPM desalineado en nginx (cambiar a puerto inexistente en fastcgi_pass)
```

**Criterios de cierre (120 min):**

| Fase | Tiempo sugerido | Acción |
|------|-----------------|--------|
| Impacto | 10 min | ¿Qué URL? ¿Todos los usuarios? |
| Diagnóstico | 40 min | `curl -I`, `tail error.log`, `journalctl -u php8.2-fpm`, `php -l` |
| Fix | 30 min | Cambio mínimo + prueba smoke |
| Cierre | 20 min | Mensaje al “cliente” + prevención (CI `php -l`, permisos deploy) |
| Extra | 20 min | `mariadb-dump appdb > /backup/appdb-$(date +%F).sql` |

---

## Checklist fin semana 3

- [ ] Nginx + PHP-FPM + MariaDB activos y habilitados
- [ ] T-007, T-008, T-009 resueltos con bitácora
- [ ] Runbook de 1 página: “Web stack down” (5 comandos en orden)

---

## Soluciones

<details>
<summary>T-007 — Solución esperada</summary>

```bash
sudo systemctl status mariadb
# Corregir db.php:
# password LabPass123!
# o host 127.0.0.1 vs localhost según PDO
php -r "new PDO('mysql:host=127.0.0.1;dbname=appdb','appuser','LabPass123!'); echo ok;"
```

Si `Connection refused`: MariaDB parado o `bind-address` incorrecto.

</details>

<details>
<summary>T-008 — Solución esperada</summary>

Identificar `sleep(5)` o pool FPM saturado. Mitigación: subir `pm.max_children`, quitar script lento, timeouts Nginx `fastcgi_read_timeout` solo si está justificado.

</details>

<details>
<summary>T-009 — Solución esperada</summary>

```bash
sudo tail -20 /var/log/nginx/error.log
php -l /var/www/app/public/index.php
sudo chown www-data:www-data /var/www/app/public/index.php
sudo chmod 644 /var/www/app/public/index.php
# Verificar fastcgi_pass coincide con ls /run/php/php*-fpm.sock
sudo nginx -t && sudo systemctl reload nginx
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/
```

HTTP 200 al final.

</details>
