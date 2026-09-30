# 🏛️ AI Council

**Orquestador manual de debates técnicos multi-LLM. Sin APIs, sin costo, model-agnostic.**

`ai_council.html` es una aplicación de una sola página (HTML + JS vanilla, cero
dependencias, cero build) que permite conducir debates estructurados entre varios LLMs
(ChatGPT, Meta, Gemini, Claude, Qwen y cualquier otro que agregues) usando sus interfaces
de chat gratuitas mediante copy-paste. El humano modera; la interfaz mantiene el estado
colectivo del proceso.

---

## ¿Por qué manual?

| Motivación | Detalle |
|---|---|
| **Costo cero** | Usa los tiers gratuitos de cada plataforma; no hay APIs pagas. |
| **Model-agnostic** | Cualquier LLM con chat web participa, sin integraciones. |
| **Control humano** | El moderador decide turnos, cierres, ausencias, adjuntos y conclusiones. |
| **Resiliencia** | Los participantes pueden ser intermitentes (límites de tokens) sin romper el estado colectivo. |

> Filosofía: **evidencia antes que documentación**. La transcripción completa
> es la evidencia histórica; los resúmenes son contexto operativo comprimido; las
> decisiones viven embebidas en el texto hasta que la evidencia justifique estructurarlas.

---

## Arranque rápido

1. Abrí `ai_council.html` en el navegador (Chrome/Edge/Firefox).
2. Pantalla de setup (todo **antes** de iniciar, para que ningún ciclo arranque con integrantes o contexto incorrectos):
   - **Tu nombre** como moderador (se recuerda entre sesiones), **Tema del debate**, **Contexto / Links**.
   - **Archivos de contexto** (opcionales): cualquier extensión se lee como `.txt` (tope 300 KB por archivo).
   - **Integrantes del consejo**: lista editable con defaults (ChatGPT, Meta, Gemini, Claude, Qwen); quitá, agregá LLMs o humanos.
   - **Primera vuelta a ciegas** (activada por defecto): en el Ciclo 1 nadie ve las respuestas de los demás.
   - **Resumidor**: quién cierra los ciclos y genera el `RESUMEN_ACUMULATIVO` (default: el último de la lista).
3. `Iniciar Debate` → el consejo queda armado y el primer turno genera su prompt.
4. Flujo por turno:
   - `📋 Copiar Prompt` → pegalo en el chat del LLM de turno.
   - Pegá la respuesta **completa** en el recuadro de respuesta.
   - Revisá/editá la **síntesis auto-extraída** (lo único que verá el siguiente).
   - `Guardar Respuesta y Continuar ➡️`.

**Atajos:** <kbd>Alt</kbd>+<kbd>C</kbd> copia el prompt (y deja el cursor en el recuadro de
respuesta) · <kbd>Ctrl</kbd>+<kbd>Enter</kbd> guarda la respuesta · <kbd>Esc</kbd> cierra ventanas.
Las confirmaciones de copiado son avisos que no bloquean (sin `alert`).

---

## Mecánica del consejo

### Rotación y ciclos
- **Orden base:** el array del consejo, con el **resumidor siempre cerrando el ciclo**.
- El ciclo **cierra recién cuando todos los participantes activos opinaron** y el resumidor produjo el acumulativo.
- Al cierre: modal para **intervenir como moderador** o **continuar al ciclo siguiente**.
- El indicador de turno muestra la rotación real simulada: `A continuación: X → Y → Z`.

### Registro de decisiones
El consejo **propone**; el moderador **decide**. El registro lo mantiene la interfaz (sin LLM),
viaja en cada prompt, en los snapshots de reincorporación, en el prompt de cierre y en el export.

- **Proponer:** cualquier participante (o el moderador en una intervención) escribe una línea
  `PROPONGO: <decisión concreta>`. La interfaz le asigna número (`D1`, `D2`…); quien propone
  cuenta como apoyo.
- **Posicionarse:** `APOYO: D1, D3` · `OBJETO: D2 — <motivo>`. **Sin motivo, la objeción no se
  registra** (queda un aviso). Vale la última postura de cada uno; el historial se conserva.
- **Retirar:** `RETIRO: D6 — <motivo>`, solo quien la propuso (estado `RETIRADA`). Si alguien
  objeta su propia propuesta, se interpreta como retiro.
- **Parser tolerante y sin silencios:** acepta varios IDs (`OBJETO: D4/D5/D6 — …`, `APOYO: D2, D3`),
  aclaraciones entre paréntesis (`OBJETO: D6 (mi postura anterior) — …`), `(refina Dn)` como
  reemplazo y propuestas de varias líneas o con listas. Toda línea que parezca una marca y no se
  pueda interpretar genera un aviso visible: nada se descarta en silencio.
- **Consenso visual:** cada tarjeta muestra `apoyos/integrantes` con color — verde **UNÁNIME**,
  turquesa **casi unánime** (falta uno, sin objeciones), naranja **disputada**. Las tarjetas se
  ordenan por prioridad y abajo hay un contador (unánimes sin resolver, casi, disputadas, en
  debate, cerradas, reemplazos pendientes).
- **Aviso al moderador:** un cartel verde arriba del chat avisa cuando hay propuestas unánimes o
  reemplazos esperando decisión; el cierre de ciclo lo repite. "Revisar y resolver" abre un panel
  para acordar/rechazar (o acordar todas las unánimes de una vez) y confirmar reemplazos.
- **Estados:**

  | Estado | Quién lo pone |
  |---|---|
  | `PROPUESTA` | automático (sin objeciones vigentes) |
  | `DISPUTADA` | automático (al menos una objeción vigente; vuelve a PROPUESTA si se levantan todas) |
  | `ACORDADA` / `RECHAZADA` | **solo el moderador**, con botones en el panel lateral |
  | `REEMPLAZADA` | el moderador confirma un `PROPONGO (reemplaza D2): ...` |

- **Acordar con constancia:** si hay objeciones vigentes o integrantes sin pronunciarse
  (incluidos ausentes), la interfaz avisa y, si confirmás, queda escrito en la decisión.
- **Nada se borra:** lo reemplazado queda `REEMPLAZADA`, con referencia a su sucesora.
- El resumidor ya **no** escribe "Decisiones tomadas": se refiere a las propuestas por número.
- En el Ciclo 1 a ciegas, cada uno solo ve sus propias propuestas y las del moderador.
- El export `.md` incluye el registro legible y un bloque `LEDGER_JSON` que permite
  **restaurarlo exacto** al retomar el debate.

### Primera vuelta a ciegas
- En el **Ciclo 1**, cada participante responde **sin ver** las respuestas de los demás LLMs
  (sí ve las intervenciones del moderador). Evita que el primero en hablar marque el rumbo
  y que el debate se llene de "coincido".
- El **resumidor** sí ve todo, porque necesita cerrar el ciclo (su opinión, por lo tanto, no es ciega).
- En el **Ciclo 2**, el prompt incluye las **posturas del Ciclo 1 tal cual** (armadas por la
  interfaz, sin depender del resumidor) y pide señalar coincidencias y **desacuerdos concretos**,
  cambiando de postura solo por un argumento, no por mayoría.
- Las reincorporaciones durante el Ciclo 1 respetan la ceguera (snapshot y novedades).
- Se desactiva con el checkbox del setup; el export `.md` indica si estuvo activa.

### Prompts largos
- **Regeneración reducida:** cuando un LLM pide archivos y el prompt se regenera en la misma
  vuelta, el segundo prompt lleva **solo los archivos + las instrucciones de cierre** (el contexto
  ya está en ese chat).
- **División en partes:** si un prompt supera el máximo configurable (default 12.000 caracteres),
  se divide en partes `[PARTE i/N]`: las intermedias le piden al LLM responder solo "OK i/N" y la
  última dice que ya tiene todo y que responda. `Alt+C` copia la parte actual y avanza a la siguiente.
- El registro viaja **compacto**: las propuestas reemplazadas, rechazadas o retiradas van en una línea.

### Cadena de síntesis (control de crecimiento del prompt)
- Todo LLM debe cerrar su respuesta con `RESUMEN:` (máx 2 líneas).
- La interfaz auto-extrae esa síntesis (marcadores `RESUMEN:` / `SÍNTESIS:` / `TL;DR`,
  fallback: últimas 2 líneas) y la muestra editable antes de guardar.
- El prompt del siguiente LLM = `Tema + Contexto + archivos disponibles + RESUMEN
  ACUMULATIVO + síntesis del ciclo actual + avisos + instrucciones`. Crece ~2 líneas por turno.

### Resumen acumulativo
- Lo genera el resumidor al cerrar cada ciclo (sección `RESUMEN_ACUMULATIVO actualizado:`),
  auto-capturado desde su respuesta.
- Debe ser exhaustivo para permitir retomados: ciclos y posturas, decisiones, pendientes,
  ideas **descartadas**, **contradicciones abiertas** y TAGs de ausencia.
- Si el resumidor no cierra (ausente), la interfaz compone una **síntesis automática de
  respaldo** desde la cadena del ciclo.

### Chequeo del acumulativo (sin LLM)
Antes de guardar el `RESUMEN_ACUMULATIVO`, la interfaz lo contrasta con la transcripción:

1. **Atribuciones imposibles:** algo atribuido a alguien en un ciclo donde no tiene mensajes.
2. **Atribuciones posiblemente sin respaldo** (heurística): si ≥40 % de las palabras clave de
   la línea (y al menos 3) no aparecen en lo que esa persona dijo en ese ciclo. Calibrada con un
   debate real: detectó una propuesta inventada ("Máquina de Estados Finita") sin marcar las
   atribuciones legítimas.
3. **Ciclos inexistentes** o rotulados como "actual" cuando no lo son.
4. **Omisiones:** quien habló en el ciclo actual y no figura; ciclo actual sin sección.
5. **Decisiones:** sección "Decisiones tomadas" o referencias a `D#` inexistentes.

6. **Afirmaciones sobre el registro:** estados que no coinciden ("D4 reemplazada" cuando está
   pendiente) y propuestas atribuidas a quien no las hizo.

**Condensación automática:** el resumidor ya no copia el estado de las propuestas (el registro
viaja en cada prompt). Si el acumulativo supera 6.000 caracteres, su próximo prompt le pide
condensarlo por debajo de 4.000 sin perder contradicciones, pendientes, descartes ni TAGs.

Los **TAGs** de ausencia y de retomado que el resumidor pierda se **vuelven a agregar solos**.
Si hay problemas, un modal ofrece: **pedir corrección al resumidor** (prompt listo en la misma
vuelta; su respuesta original queda en la transcripción), **editar vos**, o **guardar igual**
(los avisos se agregan al propio acumulativo, para que el resto de los LLM sepa qué está en duda).

### Turnos multi-prompt (preservación)
- Si un LLM solicita archivos y el prompt se regenera en la misma vuelta, la **respuesta
  intermedia y su síntesis se preservan** como mensaje propio en la transcripción y en la
  cadena del ciclo. Nada se pierde entre prompts del mismo turno.

---

## Archivos de contexto y adjuntos bajo demanda

- Subida en el setup; cualquier extensión (`.js`, `.mjs`, `.ps1`, `.txt`, …) se guarda y
  visualiza como **texto plano (.txt)**.
- Cada prompt lista `## ARCHIVOS DE CONTEXTO DISPONIBLES` y pide solicitar **todos los
  necesarios en una sola línea**: `ADJUNTAR: a.js, b.mjs` o `ADJUNTAR: *` para todos
  (cada pedido extra es una regeneración más para el moderador).
- La detección tolera markdown (`**ADJUNTAR:**`), nombres sin extensión y separadores `,` o `;`.
  Si piden un archivo que no está cargado, queda un aviso con la lista de disponibles.
- Al detectar el pedido, la interfaz ofrece dos caminos:
  1. **Regenerar el prompt ahora con el adjunto** (misma vuelta; el LLM recibe
     prompt inicial + listado + síntesis anteriores + el archivo pedido).
  2. **Guardar y continuar** (el adjunto viaja en el próximo prompt de ese LLM).
- **Archivos durante el debate:** botón `+ Agregar` en el panel de archivos. Si un LLM había
  pedido un archivo que no estaba cargado, al subirlo se le adjunta solo en su próximo prompt.
  Subir un archivo con el mismo nombre lo reemplaza y habilita reenviarlo.
- **Entrega única:** los archivos ya adjuntados no se re-envían (viven en el historial del
  chat externo del LLM); los pedidos nuevos viajan solos en la siguiente regeneración.
- **No propagación:** el adjunto y la mecánica de solicitud NO entran al prompt del
  siguiente LLM. Solo nutren el resumen si el solicitante lo mencionó en su propio `RESUMEN:`.
- El panel lateral mantiene la lista de archivos con visor (`ver`) durante todo el debate.

---

## Retomado de debates

- `Exportar .md` produce: metadatos, archivos de contexto, resumen acumulativo y
  transcripción completa con síntesis por mensaje.
- Para retomar: subí ese `.md` como **archivo de contexto** en un debate nuevo. La
  interfaz lo detecta (badge `🔄 debate retomable`), pre-carga el **Tema** y **siembra el
  `RESUMEN_ACUMULATIVO`** con tag `[DEBATE_RETOMADO desde <archivo>]`.
- Desde ahí basta escribir "retomamos el debate cargado": el consejo tiene todo el estado
  anterior, incluidas ausencias y decisiones.

---

## Protocolo de intermitencia (ausencias)

Pensado sobre todo para los límites de uso de los tiers gratuitos, pero sirve para cualquier ausencia.


1. **Marcar ausente:** se marca `Ausente`, se agrega `[TAG_AUSENCIA: Nombre - hora - Ciclo N]`
   al acumulativo, y se notifica a todos los activos en su prompt
   (pueden dejar notas `PARA [Nombre]: ...`).
2. **Reincorporar:** se genera un **snapshot dirigido** al ausente con: acumulativo + lo
   que se perdió desde su TAG + notas `PARA [Nombre]` que le dejaron.
3. **Cola de reincorporados:** el reincorporado habla **inmediatamente después del
   hablante actual**, antes que cualquier otro pendiente.
4. **Resincronización en vivo:** su prompt de turno incluye la sección
   `## NOVEDADES DESDE TU REINCORPORACIÓN:` con todo lo dicho después de su snapshot.
   > Regla general: *el snapshot reincorpora, el prompt de turno resincroniza, y la cola
   > minimiza la distancia entre ambos.*
5. **Anti-metatización:** el puntero de turno nunca salta con una respuesta pegada en el
   recuadro; los saltos protegidos solo ocurren con el recuadro vacío.

---

## Consejo dinámico

- **Pre-debate:** editor con defaults editables en la pantalla de setup.
- **En debate:** `+` agrega participantes al final de la rotación; `−` quita por número o
  nombre exacto (mínimo 1; confirmación si se quita al resumidor).
- Quitar al hablante actual reasigna el turno prolijamente al siguiente pendiente.

## Rol resumidor configurable

- Se elige en el setup (selector); por defecto, el último integrante de la lista.
- Toda la lógica de cierre (último del ciclo, caja de acumulativo, estilos, guards) usa
  `summarizerId`, sin hardcodeo.
- Sin resumidor (quitado en pleno debate), los ciclos cierran con síntesis automática de respaldo.

---

## Cierre y persistencia

- **Finalizar Debate:** elegís quién redacta conclusiones y próximos pasos (prompt copiado
  al portapapeles); opción de exportar antes de volver al inicio.
- **Exportar .md:** transcripción completa + síntesis por mensaje + resumen acumulativo +
  archivos de contexto.
- **Persistencia:** `localStorage` (`ai_council_state`; migra automáticamente desde la clave anterior `michelab_council_v4`) con migraciones no destructivas;
  sobrevive recargas y conserva debates en curso entre versiones. `saveState()` avisa si
  se excede la cuota (archivos grandes) y recomienda exportar como red de seguridad.

---

## Historial de versiones

| Versión | Hitos |
|---|---|
| v0.1 | Turnos fijos, ciclos, intervención, cierre con redactor, ausencias con TAGs y resúmenes de reincorporación. |
| v0.2 | Resumen acumulativo por ciclo (Qwen), aviso a todos los ausentes, export `.md`, Qwen cierra. |
| v0.2.1 | Cadena de síntesis `RESUMEN:`, salto de turno al deshabilitar hablante, timestamps de sistema. |
| v0.3 | Fin de debate → inicio limpio con export opcional; auto-captura del acumulativo; síntesis de respaldo; intervenciones al ciclo nuevo. |
| v0.4 | Rotación por ciclo (`spokenThisCycle`); cierre solo con todos los activos; el resumidor cierra incluido. |
| v0.5 | Consejo dinámico `+/−`, usuario configurable, migraciones, log sentinel. |
| v0.6 | Cola de reincorporados, preview de rotación, `NOVEDADES DESDE TU REINCORPORACIÓN`, saltos protegidos. |
| v0.7 | Setup pre-debate (integrantes + resumidor), archivos de contexto como `.txt`, adjuntos bajo demanda vía `ADJUNTAR:` sin propagar al siguiente, tope 300 KB y manejo de cuota. |
| v0.8 | Solicitudes con nombres de archivo, adjuntos entregados una sola vez, síntesis intermedias preservadas en turnos multi-prompt, retomado de debates desde `.md` exportado, resumidor configurable (default Qwen). |
| v0.8.1 | Seguridad y conservación: el contenido de los LLMs se muestra como texto (sin interpretar HTML), el acumulativo ya no se trunca en silencio (aviso a partir de 8000 caracteres), finalizar siempre exporta antes de borrar y "Cancelar" mantiene el debate. |
| v0.8.2 | Neutral para uso público: sin referencias a MicheLab en la interfaz, nombre del moderador recordado, resumidor por defecto = último de la lista, textos de ausencia neutrales, clave de almacenamiento `ai_council_state` con migración automática. |
| v0.9 | Primera vuelta a ciegas (opcional, activada por defecto): Ciclo 1 sin ver al resto, Ciclo 2 con las posturas ciegas tal cual y pedido explícito de desacuerdos. |
| v0.10 | Registro de decisiones: `PROPONGO` / `APOYO` / `OBJETO` con motivo; estados PROPUESTA, DISPUTADA, ACORDADA, RECHAZADA y REEMPLAZADA; solo el moderador resuelve, con constancia de objeciones y silencios; restauración exacta al retomar. |
| v0.11 | Chequeo determinista del acumulativo: atribuciones imposibles o sin respaldo, ciclos inexistentes, omisiones, "Decisiones tomadas" y `D#` inexistentes; TAGs perdidos re-agregados; prompt de corrección para el resumidor. |
| v0.12 | Menos fricción: todos los adjuntos en una línea (`ADJUNTAR: *`), detección tolerante y aviso de archivos inexistentes, atajos Alt+C / Ctrl+Enter / Esc, avisos no bloqueantes, `**RESUMEN:**` en negrita reconocido. |
| v0.13 | Tras el primer debate real con la herramienta: parser tolerante sin descartes silenciosos, `RETIRO:`, consenso visual (unánime / casi / disputada) con contador, cartel y resolución en lote; archivos durante el debate; regeneración reducida y prompts divididos en partes; condensación automática del acumulativo; chequeo de estados y autorías del registro; vuelta a ciegas sin números de propuesta; Esc ya no deja trabado el cierre de ciclo. |

---

## Limitaciones conocidas y próximos pasos

- Las decisiones ya no viven solo embebidas en texto: el registro de decisiones (v0.10) se agregó
  después de que un debate real mostrara disenso perdido y atribuciones inventadas en el resumen.
- Prueba de estrés real superada: debate técnico (3 ciclos, ausencias
  escalonadas, adjuntos encadenados y acuerdos refinados ciclo a ciclo) validó el
  protocolo, la preservación de turnos multi-prompt y el retomado desde export.
- No se agregan features especulativas: se usa en debates reales y se anota
  qué pierden o qué sobra en los resúmenes. **Evidence primero.**

---

*Un solo archivo, cero dependencias, cero costo. Nacido dentro de MicheLab.*