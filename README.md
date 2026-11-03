# Proyecto Integrador — Curso de Automatización con IA (Coderhouse)

Repositorio del proyecto integrador del curso de Business Process Automation / Automatización con IA de Coderhouse (Comisión #96925).

Este proyecto **crece módulo a módulo**: el flujo de n8n que vive acá es la base sobre la que se van sumando capas en cada checkpoint (memoria, integraciones, RAG, voz, etc.) hasta el Proyecto Final Integrador. No se rehace desde cero en cada instancia: se importa el `.json` anterior y se extiende.

---

## Checkpoint 1 (M1) — Agente autónomo básico funcional

**Archivo del entregable:** [`checkpoint1_joaquin_peralta.json`](./checkpoint1_joaquin_peralta.json)

### ¿Qué hace?

Es un **agente de Triaje de Soporte**: recibe por chat un mensaje desestructurado de un cliente, lo entiende, lo **clasifica** (categoría + prioridad), redacta un resumen y **registra el caso** de forma autónoma en una planilla de Google Sheets. Al final emite un reporte de observabilidad por Gmail para supervisión humana.

El agente está construido como un **Tools Agent** que funciona con el ciclo **ReAct** (Reason + Act): el propio modelo decide, de forma probabilística, cuándo usar su herramienta de registro, guiándose por la descripción semántica de la misma — no es un router con ramas fijas.

### Categorías y prioridades que maneja

| Categoría | Descripción |
|-----------|-------------|
| `TECNICO` | Problemas de uso, fallas, errores del producto o servicio |
| `FACTURACION` | Pagos, cobros, facturas, reembolsos, precios |
| `RECLAMO_URGENTE` | Clientes molestos, pedidos de cancelación, problemas que requieren atención inmediata |

Prioridad asignada: `ALTA` · `MEDIA` · `BAJA`

---

## Arquitectura del flujo

```
[Chat Trigger] → [AI Agent (Tools Agent)] → [Gmail: reporte de observabilidad]
                      │           │
             [Anthropic Chat]  [Google Sheets Tool]
```

### Componentes

1. **Disparador — Chat Trigger** (`@n8n/n8n-nodes-langchain.chatTrigger`)
   Captura el mensaje inicial del usuario.

2. **AI Agent — el cerebro** (`@n8n/n8n-nodes-langchain.agent`)
   Configurado en modo **Tools Agent**. Contiene el System Prompt modular y el guardrail de iteraciones.
   - **Guardrail:** `maxIterations: 6` (dentro del rango 5–10 exigido, para evitar bucles infinitos).
   - **System Prompt modular:** estructurado en `ROL → ÁMBITO → OBJETIVO → REGLAS → ESCALAMIENTO → LO QUE NO PODÉS HACER`, con límites explícitos de alcance.

3. **Modelo de lenguaje — Anthropic Chat Model** (`@n8n/n8n-nodes-langchain.lmChatAnthropic`)
   Conectado al agente mediante el conector nativo `ai_languageModel`.

4. **Herramienta — Google Sheets Tool** (`n8n-nodes-base.googleSheetsTool`)
   Conectada **lateralmente** al agente como `ai_tool` (no como nodo secuencial del lienzo). Operación `Append Row`.
   - Tiene una **descripción semántica extensa** que le explica al modelo en qué casos de negocio debe activarla y en cuáles no.
   - Columnas mapeadas: `Fecha` (fijada con `{{ $now }}`), `Categoria`, `Prioridad`, `Resumen` y `Mensaje_Original` (estas definidas por el agente vía `$fromAI`).

5. **Observabilidad — Gmail** (`n8n-nodes-base.gmail`)
   Nodo final conectado a la salida principal del agente. Envía un reporte automático con el resultado de la ejecución, como punto de supervisión humana.

---

## Cómo importar y usar el flujo

1. Descargar [`checkpoint1_joaquin_peralta.json`](./checkpoint1_joaquin_peralta.json).
2. En n8n: menú `···` → **Import from File** → seleccionar el `.json`.
3. Configurar las credenciales (ver abajo).
4. Ajustar en el nodo de Google Sheets el **Document** y la **Sheet** de destino (apuntar a una planilla propia con los encabezados: `Fecha | Categoria | Prioridad | Resumen | Mensaje_Original`).
5. En el nodo Gmail, poner la dirección de destino del reporte.
6. Abrir el chat del Trigger y probar con un mensaje de soporte.

### Credenciales necesarias

- **Anthropic API** (para el Chat Model).
- **Google Sheets OAuth2** (para la herramienta de registro).
- **Gmail OAuth2** (para el reporte de observabilidad).

> Nota: el `.json` exportado **no contiene** las API keys ni los tokens — n8n solo exporta las referencias internas a las credenciales. Hay que reconfigurarlas al importar.

---

## Evidencia de la prueba de ejecución

Se ejecutó el flujo manualmente (`Execute Workflow`) con el siguiente prompt de prueba:

> *"Hola, hace tres días que no puedo entrar a mi cuenta, me tira error cada vez que pongo la contraseña. Ya probé de todo y estoy cansado de que no funcione."*

**Resultado verificado:**

- ✅ Todos los nodos se ejecutaron correctamente (flujo en verde, sin errores de sintaxis).
- ✅ El agente abrió de forma **probabilística** la ramificación de la herramienta y la invocó con éxito.
- ✅ Clasificación correcta: **Categoría `TECNICO` · Prioridad `ALTA`** (el agente captó el tono de malestar del cliente para asignar la prioridad).
- ✅ Se escribió una fila nueva en la planilla de Google Sheets con las 5 columnas completas.
- ✅ Se recibió el reporte de observabilidad por Gmail sin errores.

<!-- Para dejarlo aún más completo, podés subir tus capturas al repo y descomentar estas líneas:
![Flujo en verde tras la ejecución](./evidencia/flujo-verde.png)
![Fila registrada en Google Sheets](./evidencia/sheet.png)
![Reporte de observabilidad recibido por Gmail](./evidencia/mail.png)
-->

---

## Roadmap del proyecto integrador

| Módulo | Qué se suma |
|--------|-------------|
| **M1** ✅ | Agente base (Trigger → AI Agent + System Prompt + 1 Tool → Observabilidad) |
| M2 | Arquitectura multi-agente (Manager + workers como sub-workflows) |
| M3 | Memoria y contexto por `Session_ID` |
| M4 | Integraciones reales (CRM / Calendario / Workspace vía OAuth2) |
| M5 | RAG / base documental (vector store) |
| M6 | Voz (STT / TTS) |
| … | … hasta el Proyecto Final Integrador (M11) |

---

*Autor: Joaquín Peralta — Coderhouse, Comisión #97320*
