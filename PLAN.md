# Plan: "Mis Finanzas" — artefacto web de control financiero personal

## Contexto
El repo `Mis-finanzas` está vacío. Objetivo: una herramienta pro, privada, accesible desde móvil/PC, para controlar **lo que gano, lo que gasto, lo que debo y lo que me deben**, con presupuestos, metas, varias cuentas/monedas, donaciones e impuestos, y un **asesor** que detecte gastos que sobran y dé consejos para cumplir objetivos.

Decisiones del usuario:
- Formato: **Artefacto web en claude.ai** con base de datos compartida (persistente, sin servidor).
- Entrada de datos: manual en la app + importar CSV/Excel + pedírselo a Claude en el chat.
- Módulos v1: todos (ingresos/gastos, presupuestos, deudas y ahorro, cuentas/monedas, donaciones e impuestos).

## Arquitectura
- **Un solo `index.html`** (HTML + JS inline, Chart.js desde cdnjs con versión fija), publicado con la herramienta Artifact. Fuente versionada también en el repo (`artifact/index.html`).
- **Capacidades del runtime** (cargar skill `artifact-capabilities` antes de escribir código):
  - `db` → base de datos persistente (fuente de verdad; Claude puede leer/escribir con `ArtifactData`).
  - Capacidad de "preguntar a Claude" → botón **Asesor** que envía el resumen del mes y devuelve consejos.
  - Lectura de archivos → importar CSV/XLSX (parseo en el navegador con SheetJS desde cdnjs).
- Diseño: cargar skill `artifact-design` + `dataviz`; modo claro/oscuro, responsive a 360px, tokens de color en `:root`.

## Modelo de datos (colecciones en `db`)
| Colección | Campos clave |
|---|---|
| `cuentas` | nombre, tipo (banco/efectivo/tarjeta/inversión), moneda, saldo_inicial |
| `movimientos` | fecha, tipo (ingreso/gasto/transferencia), monto, moneda, cuenta_id, categoría, nota, etiqueta `necesario/prescindible`, deducible (bool) |
| `categorias` | nombre, tipo, icono, es_esencial |
| `presupuestos` | mes, categoría, tope |
| `deudas` | tipo (`debo` / `me_deben`), contraparte, monto, saldo, tasa, cuota, vencimiento, pagos[] |
| `metas` | nombre, objetivo, aportado, fecha_límite |
| `donaciones` | (vía movimientos con categoría Donación + certificado) |
| `config` | moneda base, tasas de cambio manuales, país/régimen fiscal |

## Pantallas (pestañas)
1. **Resumen**: patrimonio neto (activos − deudas + me deben), flujo del mes, % ahorro, gráfico ingresos vs gastos 6 meses, alertas.
2. **Movimientos**: formulario rápido (≤3 toques), lista filtrable, importar CSV/Excel con mapeo de columnas y auto-categorización por reglas (palabras clave).
3. **Presupuestos**: barras por categoría con semáforo (≤80% verde, ≤100% ámbar, >100% rojo).
4. **Deudas**: "Debo" y "Me deben", calendario de pagos, estrategia avalancha/bola de nieve con fecha estimada de liberación.
5. **Metas y ahorro**: progreso, aporte mensual necesario.
6. **Impuestos y donaciones**: total deducible del año, ingresos acumulados vs umbrales configurables, exportar resumen CSV.
7. **Asesor**:
   - Reglas locales (sin IA): gastos prescindibles > X% del ingreso, categorías sobre presupuesto, suscripciones recurrentes, gastos hormiga, deuda con mayor tasa.
   - Botón "Analizar con Claude": consejos personalizados para cumplir metas.

## Notificaciones (solo lo que sea viable; se valida en la Fase 4)
Por orden de viabilidad:
1. **Avisos dentro de la app** (seguro): centro de alertas en el Resumen con insignia: cuota de deuda que vence en ≤3 días, presupuesto al 80%/100%, cobro pendiente de "me deben", meta atrasada.
2. **Notificaciones del navegador** (si el entorno del artefacto lo permite): `Notification API` al abrir la página. Si el permiso está bloqueado en el iframe, se descarta y se queda en el punto 1.
3. **Recordatorios en Google Calendar** (conector disponible): al crear una deuda o un cobro, opción "Agendar recordatorio", y yo creo el evento con aviso.
4. **Resumen periódico por correo** (opcional): tarea programada semanal que lee la `db`, arma el resumen (gastos prescindibles, vencimientos, avance de metas) y te lo envía por Gmail. Solo si lo apruebas al llegar a esa fase.

## Fases de entrega
1. **Base** (v0.1): estructura, `db`, cuentas, movimientos manuales, Resumen con gráficos.
2. **Control** (v0.2): presupuestos, deudas (debo/me deben), metas.
3. **Datos** (v0.3): importación CSV/Excel, multimoneda con tasa manual.
4. **Inteligencia** (v0.4): Asesor (reglas + Claude), impuestos y donaciones, exportación, notificaciones.
Cada fase se publica en la misma URL del artefacto y se commitea en `claude/elegant-gates-s2cqug`.

## Supuestos a confirmar al iniciar
- Moneda base COP con USD secundaria y país Colombia para la sección fiscal (configurable en `config`).

## Verificación
- Probar la página con Playwright (Chromium preinstalado) a 360px y 1280px, claro/oscuro: crear cuenta, movimiento, presupuesto, deuda; revisar que los totales cuadren.
- Importar un CSV de ejemplo y validar categorización.
- Usar `ArtifactData` (list/query) para confirmar que los datos persisten y que puedo registrar un gasto desde el chat.
- Publicar y abrir el artefacto; revisar consola sin errores.

## Agregado: tarjetas de crédito (incluido en v0.1)
- Cupo, día de corte, día límite de pago, tasa E.A. y cuota de manejo por tarjeta.
- Compras a cuotas (1–72) con valor de cuota, saldo pendiente y fecha de terminación.
- Pagos proyectados de los próximos 6 meses, estado (pendiente / vence pronto / vencido / pagado) y alertas en el Resumen.
- "Registrar pago" crea una transferencia desde tu cuenta hacia la tarjeta.

## Fase 2 entregada (v0.2)
- Presupuestos: tope mensual por categoría, semáforo, marca del día en cada barra, "te quedan X por semana", sugerencia automática (promedio de 3 meses, -20% en prescindibles) y regla 50/30/20.
- Deudas: "Lo que debo" y "Lo que me deben", abonos/cobros con historial (opcionalmente registrados en una cuenta), fecha del próximo pago, plazo e intereses estimados, carga de cuotas vs ingresos y orden de pago avalancha / bola de nieve (incluye tarjetas).
- Metas: progreso, aporte mensual necesario, estado (a tiempo / atrasada / vencida / lograda) y comparación con el ahorro promedio.
- Los préstamos recibidos/otorgados y sus cobros no cuentan como ingreso ni gasto del mes. El patrimonio neto ya resta deudas y suma lo que te deben.
- Nuevas alertas en el Resumen: presupuestos al 80% o excedidos, pagos y cobros próximos o atrasados, metas atrasadas.

## Fase 3 entregada (v0.3)
- Importar extractos CSV/Excel en 3 pasos (archivo → columnas → revisión). Detecta encabezados, fechas (dd/mm/aaaa, aaaa-mm-dd, "05 oct 2026", seriales de Excel), montos colombianos ("1.234.567,89", negativos entre paréntesis) y columnas de débito/crédito separadas.
- Categorización automática con diccionario de comercios colombianos y reglas aprendidas (colección `reglas`) al corregir categorías. También sugiere la categoría al escribir la nota en el formulario manual.
- Duplicados marcados y excluidos por defecto; pagos a tarjeta detectados como transferencias; en extractos de tarjeta los pagos recibidos se excluyen.
- "Deshacer importación" por lote desde Movimientos.
- Multimoneda COP/USD/EUR: cada movimiento en moneda extranjera guarda su tasa; los saldos usan la tasa configurada.
- Movimientos suscritos por año (hasta 1000 por año) para soportar historiales grandes.
