# Public Content Discord — Documentación

**ID del flujo:** `9xR4HH6grUfCZF5yFHam4`

**Resumen:**
Este workflow automatiza la publicación de contenido desde una hoja de Google Sheets hacia Discord. Escucha filas nuevas en la hoja "Publicaciones pendientes"; si la columna `Publicada` no está marcada con `✔`, envía el contenido al usuario (o canal) en Discord y marca la fila como publicada (escribe `✔` y copia campos relevantes). Ideal para equipos que crean posts en una hoja de cálculo y quieren un canal de validación/replicación en Discord.

---

## Tabla de contenidos

1. Objetivo
2. Requisitos previos
3. Diagrama lógico del flujo
4. Descripción nodo a nodo
5. Plantillas y mapeo de columnas
6. Ejemplo de ejecución
7. Configuración de credenciales y permisos (Google & Discord)
8. Uso, pruebas y despliegue
9. Manejo de errores y logging
10. Mejoras recomendadas
11. Registro de cambios / notas

---

## 1) Objetivo

Permitir que el equipo escriba publicaciones en Google Sheets y que, al agregarse una fila nueva, el contenido se reenvíe automáticamente a Discord para revisión o publicación. Luego se actualiza la fila para evitar reenvíos.

## 2) Requisitos previos

* Instancia de n8n conectada a Internet.
* Credenciales OAuth2 para Google Sheets (con permisos de lectura/escritura en la hoja indicada).
* Credenciales del bot de Discord configuradas en n8n (token con permisos de enviar mensajes a usuarios o canales).
* La hoja de cálculo debe tener las columnas usadas por el flujo: `Titulo`, `Contenido`, `Imagenes`, `Publicada`.
* Permisos del bot: `Send Messages` y, si envía imágenes/embeds, permisos de `Embed Links` y `Attach Files`.

## 3) Diagrama lógico del flujo

* `Google Sheets Trigger` (rowAdded, everyMinute) → `Filter` (Publicada != ✔) → `Create a channel` (discord node: send message to user) → `Update row in sheet` (marca como publicada y copia campos)

## 4) Descripción nodo a nodo

### 4.1 Google Sheets Trigger

* **Tipo:** `googleSheetsTrigger` (v1)
* **Parametros principales:**

  * `pollTimes`: `everyMinute` (actualmente chequea la hoja cada minuto).
  * `documentId`: `17te7D7ky6O_u2CyVOmBQj53fGpfONuQhKSJwAb4X6pw` (ID de la hoja de Google).
  * `sheetName`: `gid=0` (Hoja 1).
  * `event`: `rowAdded`.
* **Uso:** Dispara cuando se añade una fila nueva en la hoja.
* **Credencial:** `googleSheetsTriggerOAuth2Api`.

**Nota:** Este trigger puede configurarse para `rowUpdated` o usar `onChange` según el comportamiento deseado.

### 4.2 Filter

* **Tipo:** `filter` (v2.3)
* **Condición:** Rechaza filas donde `Publicada === '✔'`. Se evalúa la expresión: `={{ $json.Publicada }}` not equals `✔`.
* **Propósito:** Evitar reenviar filas que ya se han marcado como publicadas.
* **Consideración:** La comparación es case-sensitive (`caseSensitive: true`). Si la hoja puede contener otros tipos de marcas (por ejemplo `TRUE`, `true`, `X`), actualiza la lógica.

### 4.3 Create a channel (Discord node — send message)

* **Tipo:** `discord` (resource: message)
* **Parámetros:**

  * `guildId`: `1206784516442820628` (ID del servidor; requerido para algunos tipos de mensajes o webhooks).
  * `sendTo`: `user` (envía DM). `userId`: `809015116175376414`.
  * `content`: plantilla (multi-línea):

    ```text
    =#  {{ $json.Titulo }}

    {{ $json.Contenido }}

    {{ $json.Imagenes }}
    ```
* **Credencial:** `discordBotApi`.

**Notas prácticas:**

* La plantilla comienza con `=`; confirma que n8n esté evaluando correctamente las expresiones. En muchos nodos, la evaluación con `={{ ... }}` es la forma común.
* Si deseas publicar en un canal, cambia `sendTo` a `channel` y agrega `channelId`.
* Para enriquecer el mensaje, considera usar embeds y adjuntar imágenes si la columna `Imagenes` contiene URLs directas.

### 4.4 Update row in sheet

* **Tipo:** `googleSheets` (v4.7)
* **Operación:** `update`
* **Parámetros:**

  * `documentId`, `sheetName` (mismo documento/hoja que el trigger).
  * `columns.mappingMode`: `defineBelow` con mapeo:

    * `Publicada`: `✔`
    * `Titulo`: `={{ $('Filter').item.json.Titulo }}`
    * `Contenido`: `={{ $('Filter').item.json.Contenido }}`
    * `Imagenes`: `={{ $('Filter').item.json.Imagenes }}`
  * `matchingColumns`: `Titulo` (usa la columna `Titulo` para localizar la fila a actualizar).
* **Credencial:** `googleSheetsOAuth2Api`.

**Importante:** Usar `Titulo` como columna de matching solo es seguro si `Titulo` es único. Si no lo es, la actualización puede afectar múltiples filas o la fila equivocada. Se recomienda usar `row_number` o una columna con ID único para matching.

## 5) Plantillas y mapeo de columnas

* **Plantilla actual (Discord):**

  * Encabezado con `# {{ $json.Titulo }}`
  * Cuerpo con `{{ $json.Contenido }}`
  * Imágenes con `{{ $json.Imagenes }}`

* **Mapa de columnas en `Update row in sheet`:**

  * `Publicada` = `✔`
  * `Titulo` = el mismo título (copiado)
  * `Contenido` = contenido enviado
  * `Imagenes` = imágenes enviadas

## 6) Ejemplo de ejecución

**Fila añadida en Sheets:**

| Titulo         | Contenido             |                                   Imagenes | Publicada |
| -------------- | --------------------- | -----------------------------------------: | --------- |
| Lanzamiento v2 | Publicamos la v2 hoy. | [https://.../img.jpg](https://.../img.jpg) |           |

**Flujo:**

1. Trigger detecta la fila nueva → pasa `Filter` (Publicada está vacío) → envía DM al usuario con la plantilla y la imagen URL → `Update row in sheet` escribe `✔` en `Publicada` y copia los valores.

## 7) Configuración de credenciales y permisos (Google & Discord)

### Google Sheets

1. Crear credenciales OAuth2 en Google Cloud (OAuth client ID) y autorizar scopes: `spreadsheets.readonly` y `spreadsheets` (write).
2. Configurar la credencial en n8n (`googleSheetsTriggerOAuth2Api` y `googleSheetsOAuth2Api`).
3. Asegurarse de que la cuenta tenga acceso a la hoja especificada.

### Discord

1. Crear una aplicación y un Bot en Discord Developer Portal.
2. Generar token del bot y configurar credencial `discordBotApi` en n8n.
3. Invitar el bot al servidor con los permisos adecuados (`Send Messages`, `Embed Links`, `Attach Files` si aplica).
4. Obtener el `userId` o `channelId` de destino.

## 8) Uso, pruebas y despliegue

### Pruebas

* Añade una fila de prueba en la hoja y observa la ejecución en "Executions" en n8n.
* Ejecuta manualmente los nodos: prueba el `Filter` y el `Create a channel` para verificar la plantilla.

### Producción

* Ajusta `pollTimes` si necesitas menos frecuencia (cada minuto puede ser costoso o causar límites API).
* Activa el workflow desde la UI (actualmente `active: false` en el JSON exportado).

## 9) Manejo de errores y logging

* Añadir un nodo `Error Trigger` para capturar fallos y notificar a un canal de logs en Discord o Slack.
* Validar que las URLs en `Imagenes` sean accesibles antes de enviar; si no, enviar una versión sin imagen.
* Registrar `execution.data` en un storage (S3/Google Drive) para auditoría.

## 10) Mejoras recomendadas

* **Deduplicación robusta:** En lugar de `Titulo` para `matchingColumns`, usar `row_number` o un ID único generado al insertar la fila.
* **Soporte de imágenes:** Si `Imagenes` contiene URLs separadas por comas, añadir parsing para adjuntar cada imagen en Discord (nodo `HTTP Request` para fetch si necesitas validar).
* **Embeds ricos:** Usar embedding de Discord para mostrar título, contenido, autor, y miniatura en lugar de texto plano.
* **Control de flujo:** Añadir un `Delay` o `Rate Limit` si se espera un flujo alto de publicaciones para evitar rate limits del bot.
* **Moderación:** Añadir un paso de aprobación humana (por ejemplo, mandar a un canal de revisión y esperar reacción antes de publicar en canal público).
* **Soporte multicanal:** Mapear una columna `Destino` en la hoja para publicar en diferentes canales según categoría.

## 11) Registro de cambios / notas

* `versionId`: `59d65055-55e1-459f-9193-38119a71a9a2`
* `instanceId`: `b4ce499dd54e0958a9a7273d6b016310d08a100c93eb53087c344e754aea6959`
* `active`: `false` (verificar antes de activar en producción).

