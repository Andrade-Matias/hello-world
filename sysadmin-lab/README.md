# Sysadmin lab — Dev PHP → Sysadmin junior

Plan de práctica con **Proxmox VE**, VMs **Debian 12**, base **Oracle Database** y cliente **SQL Developer** en tu PC.

**Plataforma (leer primero):** [plataforma-proxmox-oracle.md](./plataforma-proxmox-oracle.md)

| Semana | Archivo | Tickets |
|--------|---------|---------|
| 1 | [semana-01.md](./semana-01.md) | T-001 … T-003 |
| 2 | [semana-02.md](./semana-02.md) | T-004 … T-006 |
| 3 | [semana-03.md](./semana-03.md) | T-007 … T-009 |

**Ritmo:** martes a domingo, **2 h/día** (~1 h teoría + ~1 h práctica).

**VMs semanas 1–3:**

- `srv-core` (Debian 12, `10.10.10.10`) — semanas 1–2 y utilidades.
- `srv-web` (Debian 12, `10.10.10.20`) — semana 3 (Nginx + PHP-FPM).
- `srv-oracle` (`10.10.10.30`) — semana 3 (Oracle + listener); crear la VM en Proxmox antes del jueves de semana 3.

**Convención de tickets:** impacto → diagnóstico → cambio con rollback → cierre en 2–3 líneas. Snapshots en **Proxmox** antes de cada ticket que “rompe” el sistema.
