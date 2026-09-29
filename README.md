# 🏛️ AI Council — MicheLab

**Orquestador manual de debates técnicos multi-LLM. Sin APIs, sin costo, model-agnostic.**

`michelab_council.html` es una aplicación de una sola página (HTML + JS vanilla, cero
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
| **Control humano** | El moderador decide turnos, cierres, ausencias y conclusiones. |
| **Resiliencia** | Los participantes pueden ser intermitentes (límites de tokens) sin romper el estado colectivo. |

> Filosofía del proyecto: **evidencia antes que documentación**. La transcripción completa
> es la evidencia histórica; los resúmenes son contexto operativo comprimido; las
> decisiones viven embebidas en el texto hasta que la evidencia justifique estructurarlas.

---

## Arranque rápido

1. Abrí `michelab_council.html` en el navegador (Chrome/Edge/Firefox).
2. Pantalla de inicio: **Nombre de Usuario** (default `Miche`), **Tema del debate**, **Contexto / Links**.
3. `Iniciar Debate` → el consejo queda armado y el primer turno genera su prompt.
4. Flujo por turno:
   - `📋 Copiar Prompt` → pegalo en el chat del LLM de turno.
   - Pegá la respuesta **completa** en el recuadro de respuesta.
   - Revisá/editá la **síntesis auto-extraída** (lo único que verá el siguiente).
   - `Guardar Respuesta y Continuar ➡️`.

---

## Mecánica del consejo

### Rotación y ciclos
- **Orden base:** el array del consejo, con **Qwen siempre cerrando el ciclo** (es quien
  genera el `RESUMEN_ACUMULATIVO`).
- El ciclo **cierra recién cuando todos los participantes activos opinaron** y Qwen
  produjo el acumulativo.
- Al cierre: modal para **intervenir como moderador** o **continuar al ciclo siguiente**.
- El indicador de turno muestra la rotación real simulada: `A continuación: X → Y → Z`.

### Cadena de síntesis (control de crecimiento del prompt)
- Todo LLM debe cerrar su respuesta con `RESUMEN:` (máx 2 líneas).
- La interfaz auto-extrae esa síntesis (marcadores `RESUMEN:` / `SÍNTESIS:` / `TL;DR`,
  fallback: últimas 2 líneas) y la muestra editable antes de guardar.
- El prompt del siguiente LLM = `Tema + Contexto + RESUMEN ACUMULATIVO + síntesis del
  ciclo actual + avisos de ausencia + instrucciones`. Crece ~2 líneas por turno.

### Resumen acumulativo
- Lo genera Qwen al cerrar cada ciclo (sección `RESUMEN_ACUMULATIVO actualizado:`),
  auto-capturado desde su respuesta.
- Si Qwen no cierra (ausente), la interfaz compone una **síntesis automática de respaldo**
  desde la cadena del ciclo. Nada se pierde entre ciclos.

---

## Protocolo de intermitencia (ausencias por límite de tokens)

1. **Deshabilitar:** se marca `Ausente`, se agrega `[TAG_AUSENCIA: Nombre - hora - Ciclo N]`
   al acumulativo, y se notifica a todos los activos en su prompt
   (pueden dejar notas `PARA [Nombre]: ...`).
2. **Reincorporar:** se genera un **snapshot dirigido** al ausente con:
   acumulativo + lo que se perdió desde su TAG + notas `PARA [Nombre]` que le dejaron.
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

- `+` agrega participantes (LLM o humanos) al final de la rotación.
- `−` quita por número o nombre exacto (mínimo 1; confirmar si se quita a Qwen).
- Quitar al hablante actual reasigna el turno prolijamente al siguiente pendiente.

---

## Cierre y persistencia

- **Finalizar Debate:** elegís quién redacta conclusiones y próximos pasos (prompt copiado
  al portapapeles); opción de exportar antes de volver al inicio.
- **Exportar .md:** transcripción completa + síntesis por mensaje + resumen acumulativo.
- **Persistencia:** `localStorage` (`michelab_council_v4`) con migraciones no destructivas;
  sobrevive recargas y conserva debates en curso entre versiones.

---

## Historial de versiones

| Versión | Hitos |
|---|---|
| v0.1 | Turnos fijos, ciclos, intervención, cierre con redactor, ausencias con TAGs y resúmenes de reincorporación. |
| v0.2 | Resumen acumulativo por ciclo (Qwen), aviso a todos los ausentes, export `.md`, Qwen cierra. |
| v0.2.1 | Cadena de síntesis `RESUMEN:`, salto de turno al deshabilitar hablante, timestamps de sistema. |
| v0.3 | Fin de debate → inicio limpio con export opcional; auto-captura del acumulativo; síntesis de respaldo; intervenciones al ciclo nuevo. |
| v0.4 | Rotación por ciclo (`spokenThisCycle`); cierre solo con todos los activos; Qwen cierra incluido. |
| v0.5 | Consejo dinámico `+/−`, usuario configurable, migraciones, log sentinel. |
| v0.6 | Cola de reincorporados, preview de rotación, `NOVEDADES DESDE TU REINCORPORACIÓN`, saltos protegidos. |

---

## Limitaciones conocidas y próximos pasos

- Las **decisiones** viven embebidas en texto; una capa estructurada de decisiones
  (estado consolidado vs. delta) solo se agregará cuando debates reales muestren
  pérdida de contradicciones o decisiones provisionales en la compresión.
- Prueba de estrés pendiente: ciclos con contradicciones sucesivas, cambios de premisa
  y reincorporaciones escalonadas (A vuelve en C4, D vuelve en C5...).
- No se agregan features especulativas: se usa en debates reales de MicheLab y se
  anota qué pierde o sobra en los resúmenes. **Evidence primero.**

---

*Uso interno MicheLab. Un solo archivo, cero dependencias, cero costo.*