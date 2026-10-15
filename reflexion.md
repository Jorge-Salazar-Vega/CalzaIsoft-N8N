# Reflexión del equipo

---

## 1. ¿Qué casos fallaron y por qué? ¿Qué cambiaron para corregirlos?

### 1.1 Casos de prueba del agente

| Caso | Esperado | Qué pasó | Por qué falló | Qué cambiamos |
|---|---|---|---|---|
| Pedido que no existe (ej. "pedido 9999") | "No encontré ese pedido", sin inventar datos | El agente respondía con estado, productos y fecha de entrega que no existían. | Cuando la tool no devolvía datos, el modelo intentaba dar siempre una respuesta completa, y el prompt no le decía qué hacer en ese caso. | Agregamos al system prompt la regla de no inventar datos y de responder que no se encontró el pedido. |
| "¿Qué tipos de envío tienen?" | Responder con lo que dice la política de envíos | El agente hablaba solo de las paqueterías (Estafeta, DHL...) y les daba prioridad al responder sobre los tipos de envío. | Había dos fuentes que no coincidían: en la ingesta las políticas decían que existían envíos estándar y express, pero al diseñar la base de datos decidimos usar paqueterías externas, que son las que aparecen en los pedidos. | Quitamos de las políticas los tipos de envío estándar y express y dejamos solo los servicios de paquetería externos. |
| Cualquier mensaje al bot (Telegram no llegaba a n8n) | El agente responde en Telegram | ngrok devolvía 404 en todas las peticiones y el bot no contestaba. | n8n corre en Docker y no conocía su URL pública, así que registraba una URL local en Telegram. Además, el workflow no estaba activo. | Definimos `WEBHOOK_URL` con la URL de ngrok en el `docker-compose.yml`, reiniciamos n8n y activamos el workflow. |



### 1.2 Fallos técnicos durante la construcción

Borrador a confirmar y editar con lo que el equipo vivió:

| Fallo | Causa | Corrección |
|---|---|---|
| Telegram no llegaba a n8n: ngrok respondía 404 | n8n corre en Docker y registraba una URL local en Telegram. Faltaba `WEBHOOK_URL` y el workflow no estaba activo. | Se definió `WEBHOOK_URL` con la URL de ngrok en el `docker-compose.yml`, se reinició n8n y se activó el workflow. |
| `text-embedding-004` no aparecía en n8n | El modelo no estaba disponible en la instancia. | Se usó `gemini-embedding-2` (3072 dimensiones): se cambió la columna a `vector(3072)` y se quitó el índice HNSW, que no soporta más de 2000 dimensiones. |
| `fetch is not defined` en las tools de código | El entorno de los nodos Code de n8n 2.x no incluye `fetch`. | Se usó `this.helpers.httpRequest` y, para los tickets, un nodo Supabase en lugar de código. | El agente no encontraba pedidos que sí existían | La clave pública (`anon`) no atraviesa RLS, y las tablas lo tienen 

---

## 2. Una limitación del agente que no pudimos resolver

**La identidad del cliente es débil.** Se valida con número de pedido y correo, sin ninguna verificación real. Quien conozca esos dos datos puede ver el pedido de otra persona. Una solución real vincularía la cuenta del cliente con su chat de Telegram.

---

## 3. ¿Qué cambiaríamos si el agente fuera a producción con usuarios reales?

**Seguridad y privacidad**
- Verificar la identidad del cliente de verdad (vincular cuenta y chat) en lugar de pedir solo pedido y correo.
- Escribir políticas RLS por tabla en lugar de depender de una `service_role` con acceso total, y que las tools usen solo las funciones que necesitan.
- Defensa contra *prompt injection* y abuso: límites de mensajes por usuario, validación de entradas y registro de intentos raros.
- Tratar los datos personales (correos, pedidos, conversaciones) con política de retención, y avisar al cliente de qué se guarda.

**Confiabilidad**
- Manejo de errores y reintentos cuando fallen Gemini, Supabase o Telegram, con un mensaje de respaldo ("tuvimos un problema, intenta en unos minutos").
- Quitar la dependencia de ngrok: n8n con dominio y HTTPS fijos.
- Respaldos de Supabase y n8n, y un plan para escalar (modo *queue* de n8n, índice vectorial adecuado si crecen las políticas).
- Evitar tickets duplicados si el cliente repite el mensaje.

**Calidad del agente**
- Conjunto de pruebas automáticas con los casos de arriba, ejecutado cada vez que cambie el prompt o el modelo.
- Revisar conversaciones reales para ajustar los criterios de escalamiento (cuántos escalamientos fueron innecesarios y cuántos llegaron tarde).
- Versionar prompt y workflows en Git, y fijar la versión del modelo.

**Operación humana**
- Avisar al asesor cuando se crea un ticket (correo o Slack), con prioridad y tiempo de respuesta comprometido.
- Traspaso real: que el agente deje de responder mientras un humano atiende esa conversación.
- Soportar fotos y otros idiomas.

**Costos**
- Monitorear consumo de tokens y llamadas a embeddings, y poner límites.

---

## 4. ¿Qué hizo cada integrante, en concreto?

| Integrante | Aportes concretos |
|---|---|
| Jorge Silvano Salazar Vega (24040040) | Armó el workflow de ingesta (`workflows/02-ingesta.json`): un nodo Code que divide las políticas por secciones (un fragmento por cada `##`, con el tema como metadata), el Default Data Loader y el Supabase Vector Store en modo *Insert Documents* con embeddings `gemini-embedding-2`. Configuró la tool `buscar_politicas_rag` (nodo *Rag Politicas* en modo *retrieve-as-tool* sobre `policy_documents` con la función `match_policy_documents`) y su descripción para el agente. Configuró la memoria (*Simple Memory*, ventana de 10 mensajes, una sesión por chat de Telegram). |
| Abisaid Guadalupe Angulo Munguia (24040115) | Configuró el nodo AI Agent (Gemini): modelo, System Message y conexión de las tres tools. Programó la tool `consultar_estado_pedido`, que llama a la función `get_order_status` de Supabase con el número de pedido y el correo, y responde cuando el pedido no existe. Ejecutó los casos de prueba del 1 al 5 y guardó sus capturas en `evidencias/`. |
| Edgar Gael Reyez Quevedo (24040158) | Configuró la base de datos en Supabase: tablas (`customers`, `orders`, `order_items`, `products`, `product_variants`, `tickets`, `policy_documents`), funciones SQL, datos de prueba, `pgvector` y RLS. Configuró la tool `crear_ticket_escalacion` (nodo Supabase *Create a row* sobre `tickets`, con los campos llenados por el agente). Ejecutó los casos de prueba del 6 al 10 y guardó sus capturas en `evidencias/`. |