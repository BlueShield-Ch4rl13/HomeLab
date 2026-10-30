# HomeLab

![Estado](https://img.shields.io/badge/estado-en%20preparaci%C3%B3n-E0A34A)
![Hipervisor](https://img.shields.io/badge/Proxmox%20VE-pfSense-2ee6f0)
![Equipos](https://img.shields.io/badge/blue%20%C2%B7%20purple%20%C2%B7%20red-a%20la%20vez-a78bfa)

El servidor físico donde se juntan los proyectos del perfil: el SOC de
[Infra-SocAnalyst](https://github.com/BlueShield-Ch4rl13/Infra-SocAnalyst), el
SIEM de [SplunkLab](https://github.com/BlueShield-Ch4rl13/SplunkLab), las reglas
de [DetectionLab](https://github.com/BlueShield-Ch4rl13/DetectionLab), la
respuesta automática de [ResponseLab](https://github.com/BlueShield-Ch4rl13/ResponseLab)
y el triage de [FtriageDFIR](https://github.com/BlueShield-Ch4rl13/FtriageDFIR),
con el atacante en su propia red. Blue team (detectar y responder), purple team
(Atomic Red Team contra las reglas) y red team (la Kali) encendidos a la vez, sin
apagar máquinas para cambiar de escenario.

> **Estado: en preparación.** Todavía no hay nada desplegado. Este repositorio
> recoge el plan: máquinas, red, orden de despliegue y la lista para darlo por
> bueno. Cada pieza se instala con la guía de su propio repositorio; aquí no se
> duplica.

## Hardware

Un mini PC propio en casa (no VPS ni nube), reacondicionado, con **64 GB de
RAM** y **Proxmox VE**. Las máquinas de abajo suman unos 49 GB, así que quedan
unos 15 GB para Proxmox y para crecer. Lo que más puede variar es el host SOAR
(según el modelo que cargue Ollama) y Splunk (según lo que se indexe).

## Red

Bridge `vmbr0` *VLAN-aware* en Proxmox y **pfSense** enrutando y filtrando entre
zonas, con el diseño de Infra-SocAnalyst. Direccionamiento `10.0.<VLAN>.0/24`.

```
VLAN 40 Atacante (Kali 10.0.40.10) ──► VLAN 20 DMZ (víctima 10.0.20.10, Suricata)
                                          │ agentes Wazuh, Falco, Tetragon
VLAN 10 Endpoints (10.0.10.0/24) ─────────┤
                                          ▼
VLAN 30 SOC   Wazuh manager 10.0.30.10 ──► ResponseLab 10.0.30.20:8443 ──► TheHive (10.0.30.11)
              Splunk 10.0.30.12 ─────────►        │        │
              Host SOAR 10.0.30.11               │        └──► listas EDL ──► pfSense
                                                  └──► API de Wazuh 55000: aislar, matar, triage…
VLAN 99 Gestión: Proxmox, pfSense y el navegador de administración
```

| VLAN | Zona | Qué hay |
|---|---|---|
| 10 | Endpoints | Equipos Windows con agente de Wazuh |
| 20 | DMZ | Víctima SSH/Web, Suricata |
| 30 | SOC | Wazuh, Splunk, host SOAR, ResponseLab |
| 40 | Atacante | Kali (red team) |
| 99 | Gestión | Proxmox, pfSense y el acceso a las consolas |

## Máquinas

| Máquina | VLAN | IP | De qué repositorio | vCPU | RAM | Disco |
|---|---|---|---|---|---|---|
| pfSense | todas | `.1` de cada una | Infra-SocAnalyst | 2 | 2 GB | 16 GB |
| Wazuh (manager, indexer y dashboard) + reglas | 30 | 10.0.30.10 | Infra-SocAnalyst · DetectionLab | 4 | 8 GB | 80 GB |
| Host SOAR (Shuffle, TheHive, Cortex, MISP, Ollama) | 30 | 10.0.30.11 | Infra-SocAnalyst | 6 | 20 GB | 120 GB |
| Splunk | 30 | 10.0.30.12 | SplunkLab | 4 | 8 GB | 60 GB |
| ResponseLab (motor, proxy TLS, copias) | 30 | 10.0.30.20 | ResponseLab | 1 | 1 GB | 10 GB |
| Víctima SSH/Web | 20 | 10.0.20.10 | Infra-SocAnalyst | 2 | 2 GB | 20 GB |
| Endpoint Windows | 10 | 10.0.10.20 | — | 2 | 4 GB | 60 GB |
| Kali | 40 | 10.0.40.10 | — | 2 | 4 GB | 40 GB |
| **Total** | | | | 23 (Proxmox reparte) | **49 GB** | **406 GB** |

Son un punto de partida, no una medida. La única pieza medida es el motor de
ResponseLab: unos 60 MB de RAM y 5 MB de base de datos con las alertas de los
escenarios.

## Orden de despliegue

| # | Qué | Guía |
|---|---|---|
| 1 | Proxmox, pfSense y las VLAN | [Infra-SocAnalyst › Arquitectura de red](https://github.com/BlueShield-Ch4rl13/Infra-SocAnalyst#readme) |
| 2 | Wazuh en 10.0.30.10 y las reglas Sigma convertidas | [DetectionLab › `deploy/wazuh/INSTALAR.md`](https://github.com/BlueShield-Ch4rl13/DetectionLab/blob/main/deploy/wazuh/INSTALAR.md) |
| 3 | Host SOAR en 10.0.30.11 (`SOC_BIND=10.0.30.11`) | [Infra-SocAnalyst › `compose/soar-stack.yml`](https://github.com/BlueShield-Ch4rl13/Infra-SocAnalyst/blob/main/compose/soar-stack.yml) |
| 4 | SplunkLab en 10.0.30.12 (`SPLUNK_BIND=10.0.30.12`, `make up`) | [SplunkLab › MANUAL.md](https://github.com/BlueShield-Ch4rl13/SplunkLab/blob/main/MANUAL.md) |
| 5 | ResponseLab en 10.0.30.20 | [ResponseLab › `docs/SERVIDOR.md`](https://github.com/BlueShield-Ch4rl13/ResponseLab/blob/main/docs/SERVIDOR.md) |
| 6 | Agentes de Wazuh y active response en la víctima y los endpoints | [ResponseLab › SERVIDOR.md, apartado 5](https://github.com/BlueShield-Ch4rl13/ResponseLab/blob/main/docs/SERVIDOR.md#5-conectar-el-laboratorio) |
| 7 | Escenarios en simulación, ataque de verdad desde la Kali y paso a producción | [ResponseLab › SERVIDOR.md, apartado 7](https://github.com/BlueShield-Ch4rl13/ResponseLab/blob/main/docs/SERVIDOR.md#7-de-simulaci%C3%B3n-a-producci%C3%B3n) |

## Reglas que no se negocian

- **Denegar por defecto entre VLAN** y abrir solo lo necesario. La matriz
  completa del motor está en
  [SERVIDOR.md › Cortafuegos](https://github.com/BlueShield-Ch4rl13/ResponseLab/blob/main/docs/SERVIDOR.md#6-cortafuegos-pfsense).
- **Nada del laboratorio se publica en Internet.** Ni las consolas ni el motor de
  respuesta, que ejecuta acciones de contención. Las consolas solo se abren desde
  la VLAN de gestión.
- Los puertos de cada servicio se publican en la IP de su máquina (`SOC_BIND`,
  `SPLUNK_BIND`, `RL_BIND`), nunca en `0.0.0.0`: Ollama no tiene autenticación.
- La VLAN atacante puede ir sin salida a Internet para un laboratorio aislado del
  todo.
- Secretos generados en cada máquina (`.env` con permisos 0600) y guardados fuera
  de la VM; ninguno se sube a los repositorios.

## Lo que se ve desde fuera

Lo público no sale de este servidor. Son vistas estáticas de solo lectura en
Cloudflare, generadas por GitHub Actions:

| Dónde | Qué |
|---|---|
| [cti.carlosvillalbalagos.com](https://cti.carlosvillalbalagos.com) | News CTI: IOCs, CVE priorizados y avisos de 26 fuentes |
| [dfir.carlosvillalbalagos.com](https://dfir.carlosvillalbalagos.com) | Panel de casos de FtriageDFIR |
| [box.carlosvillalbalagos.com](https://box.carlosvillalbalagos.com) | Portal de Malpipe: análisis de muestras en el navegador |

Cuando el laboratorio funcione, los datos que interese enseñar (incidentes,
decisiones por familia, resultados de los escenarios) saldrán igual: un JSON
estático generado desde la API, sin nombres de equipo ni direcciones.

## Lista para darlo por bueno

- [ ] pfSense enruta entre las cinco VLAN y la matriz de reglas está aplicada.
- [ ] Wazuh recibe eventos de la víctima y de un endpoint Windows.
- [ ] Shuffle, TheHive, Cortex y MISP responden en el host SOAR.
- [ ] SplunkLab está `healthy` y sus reglas activas saltan con `make ataque`.
- [ ] `comprobar_despliegue.py --conectividad` de ResponseLab sin ningún FALLO.
- [ ] La cadena de la demo de Infra-SocAnalyst (hydra, shell, `/etc/shadow`, cron)
      desde la Kali genera alertas de Wazuh, Falco y Tetragon.
- [ ] Una prueba de Atomic Red Team por táctica, con su resultado en
      [`purple/resultados.csv`](https://github.com/BlueShield-Ch4rl13/DetectionLab/blob/main/purple/resultados.csv)
      de DetectionLab.
- [ ] En producción, la víctima se aísla sola y el triage de FtriageDFIR vuelve al
      incidente.
- [ ] Copia de la VM con vzdump y de los `.env` fuera del servidor.

## Licencia

MIT — ver [`LICENSE`](LICENSE).
