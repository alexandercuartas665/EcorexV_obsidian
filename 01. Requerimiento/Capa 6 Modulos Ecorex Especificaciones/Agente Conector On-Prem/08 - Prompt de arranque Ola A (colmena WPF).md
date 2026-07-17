---
tipo: prompt-arranque
proyecto: Agente Conector On-Prem
proposito: Prompt auto-contenido para que una sesion nueva construya la OLA A del agente: la cascara visual "colmena" en WPF (hexagonos, fondo transparente, poligono de Configuracion siempre lleno, sub-agentes que se llenan al activarse). Sin logica real todavia.
fecha: 2026-07-15
ola: A - Cascara visual (WPF)
estado: listo para delegar
---

# 08 - Prompt de arranque Ola A (colmena WPF)

> Primera ola del agente: **lo visual**. Entrega la colmena que se ve y se siente, con un
> ViewModel preparado para que la **Ola B** (cliente SignalR real) solo enchufe datos sin
> rehacer la UI. Decisiones que lo gobiernan: [[06 - Decision de stack y expansion a colmena multi-agente]]
> (D7: .NET 10 + C#, Windows-first; colmena WPF) y [[07 - Prior-art Doom (orquestador drone + sub-agente navegador WebView2 + MCP)]]
> (prior-art del cliente).

## Olas siguientes (contexto, NO son de esta ola)

- **Ola B**: cliente SignalR real + handshake autenticado (ClientId + secreto).
- **Ola C**: sub-agente **Gateway de datos** (docs 01-05 ya lo especifican).
- **Ola D**: sub-agente **Archivos**. **Ola E**: sub-agente **Navegador** (WebView2 + MCP,
  reusando el prior-art Doom del doc 07).

## El prompt (copiar tal cual)

```text
=========================================================================
PROMPT DE ARRANQUE - Agente Conector On-Prem, OLA A: Cascara visual "colmena" (WPF)
=========================================================================

ROL
Eres un ingeniero .NET 10 / C# construyendo la app de escritorio del "Agente Conector
On-Prem" de ECOREX. Esta ola es SOLO la CASCARA VISUAL (la colmena) + el panel de
configuracion + los estados visuales de los sub-agentes. NO conectes SignalR real ni
ejecutes sub-agentes todavia: eso son olas siguientes. Construye lo que se VE.

UBICACION Y STACK (ya decidido)
- Repo (monorepo): C:\DesarrolloIA\ECOREX.tareas  (remote EcorexV; junto a la web)
- El agente vive en:  apps/agent/   (NUEVO)
    - Proyecto de esta ola: Ecorex.Agent.Gui  (WPF, net10.0-windows)
    - Contratos compartidos (crear vacio o minimo): libs/Ecorex.Contracts.Agent (net10.0)
  El agente solo referencia libs/Ecorex.Contracts.Agent; NUNCA internals del backend.
- Stack: .NET 10, C#, WPF (net10.0-windows). Windows-first (D7 confirmada). No Linux.

AISLAMIENTO (hay otras sesiones en el repo)
- Trabaja en un git worktree propio:
    git worktree add ../ecorex-agent-gui -b feat/agente-colmena-gui
- Es codigo GREENFIELD (apps/agent no existe); aun asi, no toques apps/backend. Si creas o
  tocas un .sln, cuida no romper el build del backend.

LECTURA OBLIGATORIA ANTES DE CODEAR
1. El capitulo completo (en el vault de documentacion):
   ...\OBSIDIAN.tareas\01. Requerimiento\Capa 6 Modulos Ecorex Especificaciones\
   Agente Conector On-Prem\   (docs 00 a 07)
   - Doc 06 = decision de stack + el modelo COLMENA (orquestador + sub-agentes efimeros +
     GUI panal + identidad por ClientId).
   - Doc 07 = prior-art del sistema Doom (ucClienteDrone / DroneClientService / consumoweb).
2. Prior-art visual del cliente (mirar el layout, NO copiar 1:1, es VB.NET 4.8):
   C:\Desarrollo\core\Doom\Modulos\Cliente\ucClienteDrone.xaml (+ .vb)

CONCEPTO VISUAL (lo que imagina el usuario)
- Ventana tipo "panal": un mosaico de HEXAGONOS, fondo TRANSPARENTE (sin marco de Windows).
- UN hexagono SIEMPRE LLENO = CONFIGURACION (el ancla): desde ahi se pega el ClientId para
  conectar con el servidor (la app web) y se ve el estado.
- Los demas hexagonos = SUB-AGENTES (Gateway de datos / Navegador web / Archivos): por
  defecto VACIOS (apagados). A medida que se "activan", se LLENAN (color + glow + animacion).
- Cuando un sub-agente atiende una peticion: pulso/animacion; al terminar: se vacia de nuevo.
- Estetica: translucida, moderna, con brillo suave; es una app de bandeja/escritorio, NO usa
  el look del prototipo web (ese es de la web). Puede tomar los colores de marca de ECOREX.

ALCANCE DE ESTA OLA (lo que SI entra)
1. Proyecto WPF Ecorex.Agent.Gui que compila y ARRANCA mostrando la colmena.
2. Ventana sin borde y transparente:
   WindowStyle=None, AllowsTransparency=True, Background=Transparent, ResizeMode ajustable;
   arrastrable (mouse-drag por el fondo), boton cerrar/minimizar propios, minimizar a la
   bandeja del sistema (tray icon).
3. Layout de HEXAGONOS (teselado tipo panal): un control/UserControl reutilizable "HexTile"
   con estados visuales: Vacio (inactivo) / Lleno (activo) / Atendiendo (pulso) / Error.
   Usar Path con geometria hexagonal; opacidad/relleno segun estado; animaciones suaves
   (Storyboard) al cambiar de estado.
4. Hexagono CONFIGURACION siempre lleno: al hacer click abre un panel/flyout con:
   - ClientId (input), URL del hub (input), Estado (En linea / Offline - por ahora visual),
     boton "Probar conexion" (stub que solo cambia el estado visual).
   - Persistencia local OPCIONAL del ClientId/URL (si la haces, cifra con DPAPI; si no, deja
     el hook listo). No metas secretos en el repo.
5. Hexagonos de sub-agentes (Gateway, Navegador, Archivos) en estado Vacio, con su icono/nombre.
6. Un modo DEMO para VER el llenado: un boton oculto / atajo que simule "activar sub-agente X"
   y "atender/terminar", para demostrar las transiciones de estado (esto se reemplaza por el
   SignalR real en la Ola B). Dejalo claramente marcado como mock.

FUERA DE ALCANCE (NO en esta ola; solo dejar hooks/interfaces)
- Conexion SignalR real, handshake, ClientId real -> Ola B.
- Ejecucion real de sub-agentes (WebView2/MCP, gateway BD, archivos) -> Olas C+.
- Whitelist de seguridad, auditoria, instalador/servicio Windows -> olas posteriores.
  (Deja interfaces/placeholders donde encajaran, pero no los implementes.)

REGLAS
- C# + .NET 10, proyecto net10.0-windows. MVVM ligero (o code-behind limpio); estados de la
  colmena en un ViewModel para que la Ola B solo conecte datos reales.
- Nada de credenciales/secretos en el repo. DPAPI si persistes local.
- No romper el build del backend. El agente referencia solo Ecorex.Contracts.Agent.

ENTREGA
- Ecorex.Agent.Gui compila y CORRE en Windows mostrando la colmena translucida.
- Verificacion: ejecuta la app y toma SCREENSHOT(S) mostrando: (a) la colmena con el hexagono
  Configuracion lleno y los demas vacios, (b) el panel de configuracion abierto, (c) la demo de
  "activar sub-agente" con el hexagono llenandose. Adjunta las capturas.
- Commit(s) en la rama feat/agente-colmena-gui con mensaje en espanol
  (ej. "feat(agente-gui): cascara visual colmena WPF + panel config + estados de sub-agente").
- Actualiza PROGRESO.md del repo con lo hecho y lo que queda para la Ola B (SignalR).
- Deja la rama lista para PR; no hagas push a rama compartida sin avisar.

FORMA DE TRABAJO
- Antes de codear, hazme las preguntas que necesites (paleta/colores de marca, si quieres
  tray icon, cuantos hexagonos de sub-agente mostrar). Confirma que entendiste el modelo colmena
  (config siempre lleno; sub-agentes se llenan al activarse).
- Avanza en pasos verificables; corre la app seguido. Reporta al final: que se ve, como se probo
  (con capturas), y que queda para la Ola B.
=========================================================================
```

Relacionado: [[00 - INDICE - Agente Conector On-Prem]],
[[06 - Decision de stack y expansion a colmena multi-agente]],
[[07 - Prior-art Doom (orquestador drone + sub-agente navegador WebView2 + MCP)]],
[[05 - Plan de trabajo por olas (para sub-agente)]].
