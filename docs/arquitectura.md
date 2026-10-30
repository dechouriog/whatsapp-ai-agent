# Arquitectura

Un solo proyecto en Node.js contiene el bot de WhatsApp y el panel web. Se despliegan juntos y comparten una base de datos SQLite.

## Recorrido de un mensaje

```
WhatsApp ──▶ Baileys (client.ts) ──▶ handler.ts
                                        │
     ┌──────────────────────────────────┤
     │  1. Identifica a la persona (resuelve la dirección @lid al número real)
     │  2. Guarda el mensaje en SQLite
     │  3. Filtro de entrada: anti-flood y recorte de mensajes desmedidos
     │  4. Espera unos segundos por si escribe más (máximo 30 s desde el primero)
     │     · notas de voz → transcripción · imágenes → descripción
     ▼
   generateReply (openrouter.ts)
     │  5. Prompt = reglas fijas + guion del negocio + fecha real + memoria de la persona
     │  6. Llama al modelo con las herramientas disponibles
     │  7. Ejecuta las herramientas que pida y vuelve a llamar (hasta 8 vueltas)
     │  8. Si una reserva falla, descarta cualquier "listo" escrito antes de saberlo
     ▼
   Filtro de salida (guardrails.ts)
     │  ¿precio fuera de la lista? ¿enlace no autorizado? ¿fuga del prompt?
     ▼
   humanize.ts: quita símbolos de bot y parte la respuesta en mensajes cortos
     │
     ▼
   WhatsApp   (+ guarda la respuesta en SQLite)

   En paralelo: watchdog.ts revisa cada 5 min que el bot siga respondiendo
   y avisa por WhatsApp a un número de alertas si algo falla.
```

## Las piezas

### Conexión a WhatsApp
- **`client.ts`**: abre la sesión de WhatsApp Web, genera el QR y reconecta solo. Descarga la versión del protocolo al arrancar, usa una huella de navegador conocida y espera más tiempo tras el código 440 para no entrar en bucle.
- **`handler.ts`**: recibe cada mensaje, agrupa ráfagas, transcribe audios, describe imágenes, llama al modelo y envía la respuesta. Si el panel puso la conversación en modo humano, el bot no responde.
- **`outbox.ts`**: cola de los mensajes que el equipo envía a mano desde el panel.

### El modelo
- **`system-prompt.ts`**: arma las instrucciones con reglas fijas (cómo escribir, cuándo usar cada herramienta, qué nunca prometer), el guion de la clínica y la fecha y hora reales en la zona del negocio.
- **`openrouter.ts`**: llama al modelo vía OpenRouter, ejecuta las herramientas que pida y devuelve el texto. El modelo se cambia con una variable de entorno, sin tocar código.

### Herramientas
| Herramienta | Qué hace |
|---|---|
| `listarServicios` | Catálogo real de servicios, precios y profesionales |
| `consultarDisponibilidad` | Huecos libres de un profesional; rechaza si ese profesional no da el servicio |
| `agendarCita` | Reserva solo si esa conversación consultó antes esa misma disponibilidad |
| `listarMisCitas` | Citas próximas del número que escribe |
| `reprogramarCita` / `cancelarCita` | Solo sobre citas de ese mismo chat |
| `calificar` | Puntúa el interés del paciente |
| `guardarLead` | Registra los datos y los envía al CRM cuando hay nombre y email |
| `escalarAAsesor` | Pausa el bot y avisa a una persona del equipo con un resumen |

### Agenda propia (`scheduling.ts`)
Sin Calendly ni servicios externos. Cruza el horario semanal de cada profesional, sus ausencias, las citas ya confirmadas y los festivos colombianos, calculados por año (fijos, trasladables por la Ley Emiliani y Semana Santa a partir de la fecha de Pascua). Antes de guardar vuelve a comprobar el hueco para evitar dobles reservas.

### Seguridad (`guardrails.ts`)
Capa independiente del modelo. A la entrada limita el flood. A la salida bloquea precios que no están en la lista oficial, enlaces fuera de los dominios permitidos, promesas de resultados y cualquier respuesta que contenga el canario del prompt (señal de que el modelo está revelando sus instrucciones). Si bloquea algo, envía un mensaje neutro.

### Autovigilancia (`watchdog.ts`)
Revisa que el bot siga respondiendo y que haya saldo en OpenRouter, cuenta los mensajes de emergencia y avisa por WhatsApp si algo se sale de lo normal.

### Base de datos (`db.ts`)
SQLite en modo WAL. Conversaciones, mensajes, estado de conexión, ajustes, profesionales, servicios, horarios, ausencias, citas, consultas de disponibilidad y los bloques del guion con su historial de versiones. La conexión se abre en la primera llamada, nunca al importar el módulo, para que `next build` no choque con el bot.

### Panel (Next.js)
Seis secciones: **Conversaciones** (en vivo, con opción de tomar el control), **Agenda**, **Equipo** (profesionales, horarios, ausencias y servicios), **Negocio** (guion por bloques con versiones y prueba en vivo), **Métricas** y **Ajustes**.

## Despliegue

VPS con EasyPanel y Nixpacks. Un solo proceso arranca el bot y el panel. La base de datos y la sesión de WhatsApp van en volúmenes persistentes, y el panel queda detrás de Cloudflare Access para que no esté abierto en internet.
