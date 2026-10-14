# Semana 2 — Red, SSH y firewall

**Objetivo:** diagnosticar conectividad por capas, endurecer SSH y abrir/cerrar puertos sin quedarte afuera.

**VM:** `srv-core` (`10.10.10.10`). Opcional segunda VM `srv-client` (`10.10.10.11`) para probar desde “otra máquina”.

**Docs:**

- [Debian Wiki — Network configuration](https://wiki.debian.org/NetworkConfiguration)
- `man ip`, `man ss`, `man sshd_config`
- [ufw](https://wiki.ubuntu.com/UncomplicatedFirewall) (mismo concepto en Debian con `ufw` package)
- [nftables wiki](https://wiki.nftables.org/)

---

## Martes — IP, ruta y interfaces (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–20 | IP, máscara, gateway, broadcast; tabla de rutas `ip route` |
| 20–40 | Debian 12: `/etc/network/interfaces` o NetworkManager (`nmtui` si está instalado) |
| 40–60 | Comandos: `ip addr`, `ip link`, `ping`, `traceroute` / `tracepath` |

### Bloque práctica (~60 min)

| Min | Tarea |
|-----|--------|
| 0–30 | Documentar en bitácora: IP, GW, DNS de `srv-core` |
| 30–45 | Si usás DHCP, reservar IP en hypervisor o pasar a estática en lab |
| 45–60 | Desde tu PC: ping a VM; desde VM: ping a gateway y `1.1.1.1` |

**Config estática de ejemplo** (`/etc/network/interfaces` en VM sin NM):

```
auto ens3
iface ens3 inet static
    address 10.10.10.10/24
    gateway 10.10.10.1
    dns-nameservers 10.10.10.1
```

(Ajustá `ens3` con `ip link`.)

---

## Miércoles — DNS y resolución de nombres (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | Recursivo vs autoritativo; registros A, AAA, CNAME |
| 25–45 | `/etc/resolv.conf`, `systemd-resolved` (`resolvectl status`) |
| 45–60 | `dig`, `getent hosts` |

### Ticket T-004 — “No resuelve nombres” tras cambio de red

**Prioridad:** Alta  
**Síntoma:** `ping 1.1.1.1` OK; `ping debian.org` → “Name or service not known”.

**Setup (simular DNS roto):**

```bash
# Backup
sudo cp /etc/resolv.conf /etc/resolv.conf.bak.$(date +%F)
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
# O vacío / nameserver inválido
```

**Criterios de cierre:**

- `dig debian.org` o `getent hosts debian.org` funciona.
- Explicás si el resolver es el hypervisor, `systemd-resolved` o `/etc/resolv.conf` manual.

### Bloque práctica (~60 min)

Resolver T-004; dejar DNS documentado para el resto del lab (`10.10.10.1` o `1.1.1.1`).

---

## Jueves — SSH seguro (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–25 | `/etc/ssh/sshd_config`: `PermitRootLogin`, `PasswordAuthentication`, `PubkeyAuthentication` |
| 25–45 | `ssh -v` para depurar; `authorized_keys` |
| 45–60 | Fail2ban: idea (ban tras N intentos); no obligatorio instalarlo hoy |

### Bloque práctica (~60 min)

| Min | Tarea |
|-----|--------|
| 0–25 | Confirmar login por clave desde tu PC |
| 25–45 | Crear `/etc/ssh/sshd_config.d/99-lab.conf` con `PasswordAuthentication no` |
| 45–55 | **Antes de cerrar sesión:** abrir segunda sesión SSH de prueba |
| 55–60 | `sudo systemctl reload ssh` y validar que la nueva sesión entra |

**Rollback:** revertir snippet y `reload` si te quedás afuera (consola del hypervisor).

---

## Viernes — Firewall (ufw) (2 h)

### Bloque teoría (~60 min)

| Min | Tema |
|-----|------|
| 0–30 | Política default deny/allow; puertos vs servicios |
| 30–50 | `ufw status verbose`, reglas numbered |
| 50–60 | Orden: permitir SSH **antes** de `ufw enable` |

### Ticket T-005 — Sitio web no accesible tras “hardening”

**Prioridad:** Alta  
**Síntoma:** Nginx escucha en 80 (lo verificarás en semana 3); desde otra máquina `curl http://10.10.10.10` timeout. SSH funciona.

**Setup:** instalar nginx ligero solo para el ticket:

```bash
sudo apt install -y nginx
sudo ufw default deny incoming
sudo ufw allow OpenSSH
sudo ufw --force enable
# Olvidar: sudo ufw allow 80/tcp
```

**Criterios de cierre:**

- `sudo ss -tlnp | grep ':80'` muestra nginx escuchando.
- Desde `srv-client` o tu PC: `curl -I http://10.10.10.10` respuesta HTTP.
- Regla mínima: solo puerto 80/tcp (y 22 ya permitido).

### Bloque práctica (~60 min)

Resolver T-005; documentar reglas finales `ufw status numbered`.

---

## Sábado — Troubleshooting en capas (2 h)

### Bloque teoría (~60 min)

Armá tu **tabla de diagnóstico** (copiar a bitácora):

| Capa | Comandos | OK significa |
|------|----------|----------------|
| Enlace | `ip link`, cable/VM network | interface UP |
| IP | `ip addr`, `ping GW` | hay IP y GW alcanzable |
| DNS | `dig`, `resolvectl` | resuelve nombres |
| Puerto | `ss -tlnp`, `nc -zv host port` | proceso escucha |
| App | `curl`, logs | respuesta correcta |

### Bloque práctica (~60 min)

Inventar 2 fallas pequeñas (DNS off, ufw bloquea 80) y cronometrar cuánto tardás en localizarlas con la tabla.

---

## Domingo — Ticket combinado (2 h)

### Ticket T-006 — “Me echaron del servidor”

**Prioridad:** Crítica  
**Narrativa:** Tras cambios de seguridad, el equipo reporta que **nadie entra por SSH** y el **monitoring HTTP** (curl interno) falla.

**Escenario sugerido (mezcla):** combiná errores de la semana — por ejemplo `PasswordAuthentication no` sin clave en `deploy`, **y** `ufw` sin regla 22, **o** DNS roto en script de deploy que usa hostname.

**Setup mínimo (elegí uno o dos):**

```bash
# A) Firewall sin SSH
sudo ufw reset
sudo ufw default deny incoming
sudo ufw allow 80/tcp
sudo ufw --force enable

# B) SSH solo root deshabilitado y clave borrada (simular)
# sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config.d/*.conf
```

**Criterios de cierre (2 h total):**

1. Recuperás acceso SSH por clave sin usar contraseña en producción real.
2. HTTP responde si aplica.
3. Post-mortem corto: qué capa falló primero y cómo lo detectaste.

**Simulacro:** 90 min resolución + 30 min post-mortem escrito.

---

## Soluciones

<details>
<summary>T-004 — Solución esperada</summary>

Restaurar resolver válido:

```bash
sudo tee /etc/resolv.conf << EOF
nameserver 1.1.1.1
nameserver 8.8.8.8
EOF
```

Si `systemd-resolved` está activo, configurar con `resolvectl dns` o `/etc/systemd/resolved.conf` y no pelear con resolv.conf symlink.

Verificar: `dig +short debian.org A`.

</details>

<details>
<summary>T-005 — Solución esperada</summary>

```bash
sudo ufw allow 80/tcp comment 'nginx lab'
sudo ufw status
curl -I http://127.0.0.1
```

Desde fuera: `curl -I http://10.10.10.10`.

</details>

<details>
<summary>T-006 — Enfoque esperado</summary>

1. Consola hypervisor si SSH imposible.
2. `ufw status` → `ufw allow OpenSSH` antes que nada.
3. `systemctl status ssh` y `sshd -t`.
4. Verificar `~deploy/.ssh/authorized_keys` y permisos `700`/`600`.
5. `curl localhost` vs firewall vs servicio.

</details>
