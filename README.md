# Portero

**Un lenguaje para describir firewalls que se compila a nftables y a BPF clásico, y que detecta errores lógicos en las reglas antes de aplicarlas.**

Proyecto final de Compiladores 2026-2 · Universidad Nacional de Colombia

```
define web_ports = {22, 80, 443}
define lan       = 10.0.0.0/8

policy input = drop

allow established
allow iface lo
allow tcp to port web_ports
allow tcp from lan to port 3306
allow icmp echo-request rate 5/s
```

## Requisitos

* Linux y Python 3.8 o superior. **El compilador no tiene dependencias externas.**
* Opcionales: `nftables` (validar y cargar la salida), `graphviz` (dibujos en el informe) y `tcpdump` (comparar el BPF).

```bash
sudo apt install nftables graphviz tcpdump      # opcionales
git clone <URL-de-este-repositorio>
cd <carpeta-clonada>
```

No hay paso de compilación ni de instalación.

## Uso

```bash
./ptr compilar ejemplos/01_servidor.portero                      # -> nftables
./ptr compilar --target bpf ejemplos/02_filtro_web.portero       # -> BPF optimizado
./ptr analizar ejemplos/03_con_anomalias.portero                 # extensión
./ptr informe  ejemplos/03_con_anomalias.portero -o informe.html # todas las fases
```

`./ptr` equivale a `python3 -m portero`.

| Comando | Qué muestra |
|---|---|
| `tokens` | análisis léxico |
| `ast [--dot]` | árbol de sintaxis |
| `tabla` | tabla de símbolos y tipos |
| `ir` | conjuntos de paquetes (representación intermedia) |
| `compilar --target nft` | ruleset para `nft -f` |
| `compilar --target bpf [-O0\|-O1] [--formato asm\|c\|decimal\|dot] [--cadena X]` | programa BPF |
| `analizar [--estricto]` | reglas sombreadas, cubiertas, redundantes y correlacionadas |
| `optimizar` | borra reglas inútiles y **verifica** que nada cambió |
| `equivalencia A B` | ¿dos firewalls deciden igual? Si no, da un contraejemplo |
| `simular` | qué regla decide un paquete concreto |
| `informe -o x.html` | página con todas las fases |
| `gramatica` | PRIMEROS, SIGUIENTES y verificación LL(1) |

Para probar la salida sin tocar el firewall del equipo (red aislada):

```bash
./ptr compilar ejemplos/01_servidor.portero -o /tmp/fw.nft
sudo unshare -n sh -c 'nft -f /tmp/fw.nft && nft list ruleset'
```

> ⚠️ No ejecutes `sudo nft -f` fuera de `unshare` en un servidor remoto: una política `drop` puede dejarte sin acceso.

## El lenguaje

| Construcción | Significado |
|---|---|
| `define nombre = valor` | constante: puerto, rango `1000-2000`, red IPv4/IPv6 o conjunto `{...}` |
| `policy input\|output\|forward = drop\|accept` | abre una cadena; las reglas siguientes son suyas |
| `allow` / `block` / `reject` *predicado* `[rate N/s\|m\|h\|d]` | regla; gana la primera que coincide |
| `match` *predicado* | filtro puro (solo BPF); varios `match` se unen con *o* |
| `tcp` `udp` `icmp [tipo]` `icmpv6 [tipo]` `ipv4` `ipv6` `any` | protocolo o familia |
| `from X` / `to X` / `to X port P` / `to port P` | direcciones y puertos |
| `iface nombre` | interfaz (de salida en `output`, de entrada en las demás) |
| `established` / `state new` | estado de la conexión |
| `and` `or` `not` `( )` | el `and` es opcional; precedencia: `not` > `and` > `or` |

La gramática completa está en [`portero/gramatica.py`](portero/gramatica.py) y se verifica con `./ptr gramatica`.

## Pruebas

```bash
make pruebas          # 32 pruebas; no necesitan root
make pruebas-kernel   # carga el nftables generado en el kernel e inyecta paquetes (root)
```

## Estructura

```
portero/
  lexer.py        análisis léxico            conjuntos.py   álgebra de paquetes (IR)
  parser.py       descendente recursivo      analisis.py    EXTENSIÓN
  arbol.py        AST                        backend_nft.py generación de nftables
  semantico.py    tipos, símbolos, IR        bpf/           CFG, código de salto, optimizador,
  gramatica.py    PRIMEROS/SIGUIENTES                       ensamblador y máquina virtual
  simulador.py    intérprete de referencia   informe.py     informe HTML
ejemplos/         programas de muestra y de error
tests/            unitarias, diferenciales y contra el kernel
docs/             reporte, guion del video, guía de estudio
demo/             script guiado para grabar
```

## Documentación

* [Reporte técnico](docs/REPORTE.md)
* [Guion del video](docs/GUION_VIDEO.md) y [herramientas de grabación](docs/HERRAMIENTAS_VIDEO.md)
* [Guía de estudio para las preguntas](docs/GUIA_PREGUNTAS.md)
* [Pendientes del equipo](docs/PENDIENTES.md)

## Referencias y uso de IA

La lista completa está en el [reporte](docs/REPORTE.md#9-referencias). El proyecto **no parte del código de otra aplicación**. La primera versión del código, las pruebas y los borradores de los documentos se generaron con asistencia de IA (Claude, de Anthropic); el detalle está en la sección de contribuciones del reporte.
