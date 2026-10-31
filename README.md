# Semana-6-
# FarmaRed: Sistema Automatizado de Reposición de Inventario

Solución de integración basada en el patrón arquitectural **Remote Procedure Invocation (RPI)** para la gestión y reposición automática de existencias farmacéuticas en sucursales[span_0](start_span)[span_0](end_span). Desarrollado como parte de la asignatura **Integración de Sistemas (ISWZ3104)** en la Universidad de Las Américas (UDLA)[span_1](start_span)[span_1](end_span).

---

## 1. Descripción del Proyecto

El sistema automatiza el monitoreo de inventarios distribuidos en múltiples locales de la cadena "FarmaRed" conectando un pipeline en **Google Colab** directamente con **Google Sheets API v4**[span_2](start_span)[span_2](end_span)[span_3](start_span)[span_3](end_span). El flujo evalúa existencias, valida la integridad de los datos, calcula el reabastecimiento bajo una regla estricta de umbrales, garantiza idempotencia evitando órdenes duplicadas y despacha alertas a canales de mensajería (modo consola local o webhooks de Discord/Slack)[span_4](start_span)[span_4](end_span)[span_5](start_span)[span_5](end_span)[span_6](start_span)[span_6](end_span).

---

## 2. Arquitectura y Reglas del Negocio

### Patrón de Integración
* **Remote Procedure Invocation (RPI) sobre SaaS:** Se invocan métodos remotos de la API de Google Drive y Google Sheets (`gspread`) utilizando autenticación delegada por tokens OAuth2[span_7](start_span)[span_7](end_span)[span_8](start_span)[span_8](end_span).

### Lógica Operativa
1. **Validación Defensiva (Actividad B):**
   * Encabezados requeridos: `codigo`, `producto`, `local`, `stock_actual`, `stock_minimo`, `stock_maximo`[span_9](start_span)[span_9](end_span).
   * Verificación de campos obligatorios no vacíos[span_10](start_span)[span_10](end_span).
   * Valores enteros no negativos y consistencia relacional (`stock_maximo >= stock_minimo`)[span_11](start_span)[span_11](end_span).
2. **Cálculo de Reposición (Actividad C):**
   * Comparación estricta: Se genera solicitud únicamente si $\text{stock\_actual} < \text{stock\_minimo}$[span_12](start_span)[span_12](end_span). Si el stock es igual al mínimo, no se pide[span_13](start_span)[span_13](end_span).
   * Cantidad a reabastecer: $\text{cantidad\_a\_pedir} = \text{stock\_maximo} - \text{stock\_actual}$[span_14](start_span)[span_14](end_span).
3. **Idempotencia y Control de Duplicados (Actividad D):**
   * No se generan órdenes redundantes si ya existe una orden con estado `SOLICITADA` para la misma tupla `(codigo, local)`[span_15](start_span)[span_15](end_span).
   * La deduplicación se evalúa tanto contra el historial persistido en la pestaña `Reposicion` como dentro del lote procesado en memoria durante la ejecución activa[span_16](start_span)[span_16](end_span).
   * Si una orden previa cambia a estado `RECIBIDA` y el inventario continúa bajo el mínimo, el sistema habilita la creación de un nuevo pedido[span_17](start_span)[span_17](end_span)[span_18](start_span)[span_18](end_span).
4. **Sistema de Alertas (Actividad E):**
   * **Modo Prueba:** Imprime el detalle en consola sin depender de endpoints de red externos[span_19](start_span)[span_19](end_span).
   * **Webhooks (Opcional):** Despacho HTTP POST a Discord/Slack empleando credenciales protegidas vía `userdata.get('WEBHOOK_URL')`[span_20](start_span)[span_20](end_span)[span_21](start_span)[span_21](end_span).
   * **Widget Interactivo:** Botón desarrollado con `ipywidgets` para simular alertas sin escribir en Google Sheets ni consumir cuotas de API[span_22](start_span)[span_22](end_span).

---

## 3. Estructura de Datos en Google Sheets

El libro nativo de Google Sheets debe contener exactamente las siguientes hojas y encabezados[span_23](start_span)[span_23](end_span):

### Pestaña: `Inventario`
Almacena el catálogo de productos por sucursal y sus umbrales operativos[span_24](start_span)[span_24](end_span)[span_25](start_span)[span_25](end_span).

| codigo | producto | local | stock_actual | stock_minimo | stock_maximo |
| :--- | :--- | :--- | :--- | :--- | :--- |
| MED001 | Paracetamol 500 mg | Quito Norte | 12 | 20 | 100 |
| MED002 | Ibuprofeno 400 mg | Quito Norte | 35 | 20 | 80 |
| MED003 | Amoxicilina 500 mg | Quito Sur | 8 | 15 | 60 |
| MED004 | Loratadina 10 mg | Guayaquil | 25 | 25 | 90 |
| MED005 | Omeprazol 20 mg | Quito Norte | 4 | 10 | 50 |
| MED006 | Vitamina C 500 mg | Cuenca | 60 | 30 | 100 |
| MED007 | Losartán 50 mg | Guayaquil | 9 | 20 | 80 |
| MED008 | Metformina 850 mg | Quito Sur | 18 | 25 | 120 |

### Pestaña: `Reposicion`
Bitácora transaccional donde el script añade las solicitudes generadas[span_26](start_span)[span_26](end_span)[span_27](start_span)[span_27](end_span).

| fecha | codigo | producto | local | cantidad_a_pedir | estado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| *(Generado automáticamente)*[span_28](start_span)[span_28](end_span) | MED001[span_29](start_span)[span_29](end_span)[span_30](start_span)[span_30](end_span) | Paracetamol 500 mg[span_31](start_span)[span_31](end_span) | Quito Norte[span_32](start_span)[span_32](end_span) | 88[span_33](start_span)[span_33](end_span)[span_34](start_span)[span_34](end_span) | SOLICITADA[span_35](start_span)[span_35](end_span) |

---

## 4. Requisitos y Ejecución

### Requisitos Previos
* Cuenta de Google con acceso a Google Drive y Google Colab[span_36](start_span)[span_36](end_span)[span_37](start_span)[span_37](end_span).
* Libro de cálculo nativo creado en Google Sheets (no archivo `.xlsx` en modo visor)[span_38](start_span)[span_38](end_span)[span_39](start_span)[span_39](end_span).
* Entorno estándar de Colab (no requiere instalaciones adicionales con `pip install -U`)[span_40](start_span)[span_40](end_span)[span_41](start_span)[span_41](end_span).

### Configuración del SPREADSHEET_ID
Extrae el identificador del libro desde la barra de direcciones de tu navegador[span_42](start_span)[span_42](end_span):
```text
[https://docs.google.com/spreadsheets/d/1YdO6SUrrKGpJbf3AiJ635uWdLk6obkHHzt7Jt-biZIY/edit](https://docs.google.com/spreadsheets/d/1YdO6SUrrKGpJbf3AiJ635uWdLk6obkHHzt7Jt-biZIY/edit)
                                       └────────────────── SPREADSHEET_ID ─────────────────┘
