# 🏛️ AI Council — MicheLab

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

> Filosofía del proyecto: **evidencia antes que documentación**. La transcripción completa
> es la evidencia histórica; los resúmenes son contexto operativo comprimido; las
> decisiones viven embebidas en el texto hasta que la evidencia justifique estructurarlas.

---

## Arranque rápido

1. Abrí `ai_council.html` en el navegador (Chrome/Edge/Firefox).
2. Pantalla de setup (todo **antes** de iniciar, para que ningún ciclo arranque con integrantes o contexto incorrectos):
   - **Nombre de Usuario** (default `Miche`), **Tema del debate**, **Contexto / Links**.
   - **Archivos de contexto** (opcionales): cualquier extensión se lee como `.txt` (tope 300 KB por archivo).
   - **Integrantes del consejo**: lista editable con defaults (ChatGPT, Meta, Gemini, Claude, Qwen); quitá, agregá LLMs o humanos.
   - **Resumidor**: quién cierra los ciclos y genera el `RESUMEN_ACUMULATIVO` (default Qwen).
3. `Iniciar Debate` → el consejo queda armado y el primer turno genera su prompt.
4. Flujo por turno:
   - `📋 Copiar Prompt` → pegalo en el chat del LLM de turno.
   - Pegá la respuesta **completa** en el recuadro de respuesta.
   - Revisá/editá la **síntesis auto-extraída** (lo único que verá el siguiente).
   - `Guardar Respuesta y Continuar ➡️`.

---

## Mecánica del consejo

### Rotación y ciclos
- **Orden base:** el array del consejo, con el **resumidor siempre cerrando el ciclo**.
- El ciclo **cierra recién cuando todos los participantes activos opinaron** y el resumidor produjo el acumulativo.
- Al cierre: modal para **intervenir como moderador** o **continuar al ciclo siguiente**.
- El indicador de turno muestra la rotación real simulada: `A continuación: X → Y → Z`.

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

### Turnos multi-prompt (preservación)
- Si un LLM solicita archivos y el prompt se regenera en la misma vuelta, la **respuesta
  intermedia y su síntesis se preservan** como mensaje propio en la transcripción y en la
  cadena del ciclo. Nada se pierde entre prompts del mismo turno.

---

## Archivos de contexto y adjuntos bajo demanda

- Subida en el setup; cualquier extensión (`.js`, `.mjs`, `.ps1`, `.txt`, …) se guarda y
  visualiza como **texto plano (.txt)**.
- Cada prompt lista `## ARCHIVOS DE CONTEXTO DISPONIBLES` con la instrucción de pedirlos
  con una línea `ADJUNTAR: <nombre exacto>`.
- Al detectar el pedido, la interfaz ofrece dos caminos:
  1. **Regenerar el prompt ahora con el adjunto** (misma vuelta; el LLM recibe
     prompt inicial + listado + síntesis anteriores + el archivo pedido).
  2. **Guardar y continuar** (el adjunto viaja en el próximo prompt de ese LLM).
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

## Protocolo de intermitencia (ausencias por límite de tokens)

1. **Deshabilitar:** se marca `Ausente`, se agrega `[TAG_AUSENCIA: Nombre - hora - Ciclo N]`
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

- Se elige en el setup (selector), con **Qwen por defecto** si está en la lista.
- Toda la lógica de cierre (último del ciclo, caja de acumulativo, estilos, guards) usa
  `summarizerId`, sin hardcodeo.
- Sin resumidor (quitado en pleno debate), los ciclos cierran con síntesis automática de respaldo.

---

## Cierre y persistencia

- **Finalizar Debate:** elegís quién redacta conclusiones y próximos pasos (prompt copiado
  al portapapeles); opción de exportar antes de volver al inicio.
- **Exportar .md:** transcripción completa + síntesis por mensaje + resumen acumulativo +
  archivos de contexto.
- **Persistencia:** `localStorage` (`michelab_council_v4`) con migraciones no destructivas;
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

---

## Limitaciones conocidas y próximos pasos

- Las **decisiones** viven embebidas en texto; una capa estructurada de decisiones (estado
  consolidado vs. delta) solo se agregará cuando debates reales muestren pérdida de
  contradicciones o decisiones provisionales en la compresión.
- Prueba de estrés real superada: debate de reconstrucción de WebMCP (3 ciclos, ausencias
  escalonadas, adjuntos encadenados y acuerdos refinados ciclo a ciclo) validó el
  protocolo, la preservación de turnos multi-prompt y el retomado desde export.
- No se agregan features especulativas: se usa en debates reales de MicheLab y se anota
  qué pierden o qué sobra en los resúmenes. **Evidence primero.**

---

*Uso interno MicheLab. Un solo archivo, cero dependencias, cero costo.*