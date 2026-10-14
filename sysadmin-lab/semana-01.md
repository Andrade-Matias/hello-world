# Semana 1 — CLI, usuarios, permisos, systemd

**Objetivo:** operar Debian por SSH, entender permisos y recuperar un servicio con `systemd` y `journalctl`.

**VM:** `srv-core` — Debian 12 en **Proxmox**, IP `10.10.10.10/24`, hostname `srv-core.lab.local`. Guía de red y snapshots: [plataforma-proxmox-oracle.md](./plataforma-proxmox-oracle.md).

**Docs de la semana (ES/EN):**

- [Debian Administrator’s Handbook — Basic configuration](https://debian-handbook.info/browse/stable/sect.basic-configuration.html)
- `man bash`, `man chmod`, `man useradd`, `man systemctl`, `man journalctl`
- [wiki.debian.org/Systemd](https://wiki.debian.org/systemd)

---

## Martes — Instalación y primer acceso (2 h)

### Bloque teoría (~60 min)

| Min | Tema | Acción |
|-----|------|--------|
| 0–15 | FHS: `/etc`, `/var`, `/home`, `/tmp` | Dibujar en una hoja qué va en cada ruta |
| 15–30 | `root` vs usuario sin privilegios, `sudo` | Leer `man sudo`, notar `/etc/sudoers` y grupo `sudo` en Debian |
| 30–45 | Proxmox: bridge `vmbr0`, VirtIO disk/net, qcow2, VMID | Leer [Proxmox Wiki — QEMU/KVM](https://pve.proxmox.com/wiki/QEMU/KVM_Virtual_Machines) |
| 45–60 | SSH: clave pública vs contraseña | Generar clave en tu PC: `ssh-keygen -t ed25519` |

### Bloque práctica (~60 min)

| Min | Tarea |
|-----|--------|
| 0–20 | En Proxmox: crear VM `srv-core`, instalar Debian 12 (solo SSH), instalar `qemu-guest-agent`, snapshot `clean-install` |
| 20–40 | Desde tu PC: `ssh deploy@10.10.10.10` (o el usuario que creaste); copiar clave con `ssh-copy-id` |
| 40–50 | `sudo whoami` → `root`; editar `/etc/hostname` si hace falta; `sudo hostnamectl set-hostname srv-core` |
| 50–60 | Anotar en tu bitácora: IP, usuario, cómo entrás |

**Entregable:** una línea en bitácora con comando exacto de conexión.

---

## Miércoles — Usuarios, grupos y permisos (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–20 | `/etc/passwd`, `/etc/group`, `id`, `groups` |
| 20–40 | Permisos `rwx` en archivos y directorios; directorio necesita `x` para entrar |
| 40–50 | `umask` (default 022 → archivos 644, dirs 755) |
| 50–60 | Propiedad: `chown`, `chgrp`; recursivo con cuidado |

### Ticket T-001 — Deploy no puede publicar en `/var/www`

**Prioridad:** Media  
**Reportado por:** Equipo PHP  
**Síntoma:** Al hacer deploy, `git pull` o `rsync` falla con `Permission denied` en `/var/www/app`.

**Entorno preparado (si lab solo):** en `srv-core`:

```bash
sudo mkdir -p /var/www/app
echo '<?php echo "ok"; ?>' | sudo tee /var/www/app/index.php
sudo useradd -m -s /bin/bash deploy 2>/dev/null || true
sudo usermod -aG sudo deploy   # solo si deploy es tu usuario de práctica; en prod no siempre tiene sudo
```

**Criterios de cierre:**

- Usuario `deploy` puede crear/editar archivos en `/var/www/app` sin ser root.
- `www-data` (futuro Nginx) podrá leer el sitio (lectura para “otros” o grupo compartido).
- Documentás qué permisos numéricos quedaron y por qué.

**Pistas:** grupo compartido `www-data` + `deploy`; o ACL (`setfacl`); evitar `chmod 777`.

### Bloque práctica (~60 min)

Resolver T-001 y probar: `sudo -u deploy touch /var/www/app/test.txt`.

---

## Jueves — systemd y journal (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | Units: `.service`, `.timer`; estados `active`, `enabled`, `failed` |
| 25–45 | `systemctl status`, `start`, `stop`, `restart`, `enable`, `daemon-reload` |
| 45–60 | `journalctl -u nombre.service -e`, `-f`, `--since today` |

### Ticket T-002 — Servicio interno no arranca tras reinicio

**Prioridad:** Alta  
**Síntoma:** Tras `reboot`, el script de reportes no corre. Ayer funcionaba “a mano”.

**Escenario:** existe un unit mal configurado `report.service`.

**Setup (como root o con sudo):**

```bash
sudo tee /usr/local/bin/report.sh << 'EOF'
#!/bin/bash
echo "$(date) report ok" >> /var/log/report.log
EOF
sudo chmod +x /usr/local/bin/report.sh

sudo tee /etc/systemd/system/report.service << 'EOF'
[Unit]
Description=Daily report (ROTO para el ticket)
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/report.sh
# Falta: el script necesita crear log; permisos o ruta incorrecta a propósito:
ExecStart=/usr/local/bin/report-typo.sh

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable report.service
sudo systemctl start report.service   # fallará
```

**Tu trabajo:**

1. Confirmar fallo con `systemctl status report.service` y `journalctl -u report.service -n 30`.
2. Corregir unit o script; dejar `enabled` y verificar que `start` termina en `success`.
3. Cierre al usuario: qué falló y qué cambiaste.

**Pistas:** `ExecStart` incorrecto; después del arreglo, `daemon-reload` obligatorio.

### Bloque práctica (~60 min)

Resolver T-002; opcional: crear `report.timer` que ejecute el servicio cada hora (extra si te sobra tiempo).

---

## Viernes — APT y paquetes rotos (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | `apt update`, `upgrade`, `install`, `remove`, `autoremove` |
| 25–45 | `dpkg -l`, `dpkg -s`, estados `ii`, `iF` |
| 45–60 | Fuentes: `/etc/apt/sources.list`, `/etc/apt/sources.list.d/` |

### Bloque práctica (~60 min)

| Min | Tarea |
|-----|--------|
| 0–20 | Instalar `vim`, `htop`, `curl`, `git` |
| 20–40 | Simular rotura: `sudo dpkg -i` un paquete incompatible o interrumpir install (documentado en soluciones) |
| 40–60 | Recuperar con `sudo apt --fix-broken install` y verificar `dpkg -l \| grep ^iF` vacío |

**Entregable:** nota “qué rompí y cómo lo arreglé”.

---

## Sábado — Disco y búsqueda (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–20 | `df -h`, `du -sh`, `du -xhd1` |
| 20–40 | `find`, `locate` (instalar `plocate` si querés) |
| 40–60 | `tar`, `gzip`; rotación conceptual (`logrotate`) |

### Ticket T-003 — Servidor sin espacio, aplicaciones raras

**Prioridad:** Crítica  
**Síntoma:** SSH entra pero `apt update` falla; cron envía mails de “No space left on device”.

**Setup:**

```bash
sudo mkdir -p /var/log/app
sudo fallocate -l 2G /var/log/app/big.log 2>/dev/null || sudo dd if=/dev/zero of=/var/log/app/big.log bs=1M count=2048
# Si la VM es chica, usar count=512 (512M)
```

**Criterios de cierre:**

- `df -h` muestra espacio usable en `/` o en la partición afectada.
- Identificás el archivo o directorio culpable con evidencia (`du`).
- Proponés prevención: rotación, límite de logs, monitoring (adelanto semana 9).

### Bloque práctica (~60 min)

Resolver T-003; no borrar `/var/log` entero — solo lo necesario.

---

## Domingo — Repaso y simulacro (2 h)

| Min | Actividad |
|-----|-----------|
| 0–30 | Cheat sheet propio: 15 comandos de la semana |
| 30–60 | **Simulacro:** ticket aleatorio T-001 o T-003 sin mirar soluciones |
| 60–90 | **Simulacro:** T-002 (servicio failed) |
| 90–120 | Actualizar bitácora; snapshot Proxmox `pre-week-2` |

---

## Soluciones (intentar primero sin mirar)

<details>
<summary>T-001 — Solución esperada</summary>

Ejemplo válido (grupo compartido):

```bash
sudo groupadd webedit 2>/dev/null || true
sudo usermod -aG webedit deploy
sudo usermod -aG webedit www-data
sudo chown -R root:webedit /var/www/app
sudo chmod -R 2775 /var/www/app   # setgid para heredar grupo
```

Verificar: `sudo -u deploy touch /var/www/app/x.php` y `sudo -u www-data cat /var/www/app/x.php`.

Alternativa: ACL `setfacl -R -m u:deploy:rwx,g:www-data:rx /var/www/app`.

</details>

<details>
<summary>T-002 — Solución esperada</summary>

Corregir `ExecStart=/usr/local/bin/report.sh` y asegurar log escribible:

```bash
sudo touch /var/log/report.log
sudo chown root:root /var/log/report.log
sudo chmod 644 /var/log/report.log
sudo sed -i 's|report-typo.sh|report.sh|' /etc/systemd/system/report.service
sudo systemctl daemon-reload
sudo systemctl restart report.service
sudo systemctl status report.service
```

</details>

<details>
<summary>T-003 — Solución esperada</summary>

```bash
df -h
sudo du -xhd1 /var | sort -h
sudo rm -f /var/log/app/big.log
# o truncar: sudo truncate -s 0 /var/log/app/big.log
df -h
```

Cierre ejemplo: “Liberados ~2G por log de aplicación sin rotación; recomiendo logrotate y alerta de disco.”

</details>
