# Sistema de Control de Viáticos — Especificación

Versión: borrador 1 · Fecha: 12 de octubre de 2026
Estado: para revisión del solicitante antes de construir.

## Instrucciones para Claude Code

1. Lee este documento completo y también `prototipo.html`, que muestra el flujo, los estados y las pantallas esperadas. El prototipo es referencia visual y de comportamiento. No reutilices su código como base de la aplicación.
2. Antes de escribir código, propón una arquitectura y un plan por etapas (sección 12) y espera mi aprobación.
3. Si algo de este documento es ambiguo o se contradice, pregunta. No lo resuelvas por tu cuenta. Las dudas ya identificadas están en la sección 13.
4. Construye una etapa a la vez, con pruebas, y detente al final de cada una para que yo la valide.
5. No construyas nada de la sección 11 (fuera de alcance).
6. Todo el texto de la interfaz va en español de México. Moneda en pesos mexicanos (MXN), fechas en formato día/mes/año y zona horaria America/Mexico_City.

## 1. Objetivo

Reemplazar el manejo de viáticos por correo electrónico con una aplicación web que tenga:

- Solicitud y autorización con varios niveles.
- Control del plazo de comprobación.
- Revisión de comprobantes fiscales.
- Módulo de reembolsos.
- Bitácora de todo lo que ocurre.

Usuarios: unas 50 personas. Se usará desde el navegador, en computadora y celular. Se alojará en la nube.

## 2. Roles y permisos

| Rol | Qué puede hacer |
|---|---|
| Empleado | Crear solicitudes de anticipo y de reembolso. Ver solo las suyas. Subir comprobantes. Corregir y reenviar solicitudes devueltas. |
| Jefe directo | Aprobar, devolver con comentarios o rechazar solicitudes de las personas a su cargo. Ver las solicitudes de su equipo y sus alertas de plazo. Enviar recordatorios. |
| Finanzas | Aprobar solicitudes que superan el umbral. Registrar la entrega del anticipo. Revisar comprobantes, cerrar la revisión, registrar reintegros y pagar reembolsos. Ver todas las solicitudes. Consultar reportes. |
| Administrador | Gestionar usuarios, jerarquías (quién es jefe de quién) y parámetros del sistema. Ver todo. No aprueba solicitudes. |

Reglas:
- Nadie puede aprobar su propia solicitud. Si el jefe directo solicita un viático, la aprobación sube al siguiente nivel definido en la jerarquía.
- Cada persona tiene un solo jefe directo, definido por el administrador.
- Un usuario solo ve los datos que su rol permite.

## 3. Parámetros editables por el administrador

Ninguno de estos valores debe estar fijo en el código. Cada cambio queda en la bitácora con quién lo hizo y cuándo.

| Parámetro | Valor inicial | Nota |
|---|---|---|
| Umbral para Finanzas | $50,000 MXN | Una solicitud mayor a este monto requiere también a Finanzas. |
| Plazo de comprobación | 2 días hábiles (48 horas hábiles) | Cuenta desde la fecha de regreso. |
| Días inhábiles | Lista editable | Además de sábados y domingos. |
| Tope diario sin CFDI, nacional | $750 MXN | Referencia fiscal. Validar con el contador. |
| Tope diario sin CFDI, extranjero | $1,500 MXN | Referencia fiscal. Validar con el contador. |
| Límite de pago en efectivo con CFDI | $2,000 MXN | Referencia fiscal. Validar con el contador. |

El umbral aplica sobre el monto total solicitado. Si cambia el parámetro, el cambio aplica a solicitudes que todavía no han sido resueltas por el jefe directo. Las que ya pasaron esa etapa conservan la ruta con la que se aprobaron.

## 4. Flujo de anticipo

Folio: `VT-0001`, consecutivo.

### Estados

`Por autorizar · Jefe` → `Por autorizar · Finanzas` (solo si el monto supera el umbral) → `Por entregar anticipo` → `Por comprobar` → `En revisión` → `Reintegro pendiente` (solo si sobró dinero) → `Cerrada`

Estados laterales: `Devuelta` (regresa al empleado) y `Rechazada` (final).

### Pasos

1. **Solicitud.** El empleado captura destino, ámbito (nacional o extranjero), fechas de salida y regreso, motivo y desglose por concepto (hospedaje, alimentos, transporte, otros). El total se calcula solo. El sistema muestra la ruta de aprobación que le tocará según el monto.
2. **Jefe directo.** Aprueba, devuelve o rechaza. Devolver y rechazar exigen comentario.
3. **Finanzas (condicional).** Solo si el monto supera el umbral. Mismas tres opciones.
4. **Entrega del anticipo.** Finanzas registra método (transferencia, cheque o efectivo) y referencia o folio. El folio es obligatorio.
5. **Comprobación.** El empleado sube sus comprobantes. Plazo de 2 días hábiles desde el regreso (sección 6).
6. **Revisión.** Finanzas acepta o rechaza cada comprobante. No se puede cerrar la revisión mientras haya comprobantes sin revisar.
7. **Cierre.** Se compara lo aceptado contra el anticipo:
   - Aceptado menor al anticipo: queda un **reintegro** pendiente por la diferencia. Finanzas confirma cuando recibe el dinero y la solicitud se cierra.
   - Aceptado mayor al anticipo: el excedente genera automáticamente un **reembolso** ligado al anticipo de origen, que pasa a autorización del jefe directo. El anticipo se cierra.
   - Aceptado igual al anticipo: se cierra.

### Devuelta

Si se devuelve, el empleado ve el comentario, corrige el monto y explica qué cambió. Al reenviar vuelve a la etapa que la devolvió.

## 5. Flujo de reembolso

Folio: `RB-0001`, consecutivo.

Se usa para gastos que la persona pagó de su bolsa sin anticipo, y para excedentes de anticipos.

### Estados

`Por autorizar · Jefe` → `Por autorizar · Finanzas` (si supera el umbral) → `Por pagar` → `Pagada`. Laterales: `Devuelta` y `Rechazada`.

### Pasos

1. El empleado captura la solicitud y adjunta comprobantes. En un reembolso el empleado puede agregar y quitar comprobantes mientras esté en `Por autorizar · Jefe` o `Devuelta`.
2. Autorización con las mismas reglas de monto y umbral.
3. Finanzas revisa comprobantes (acepta o rechaza cada uno) y registra el pago por el monto aceptado.
4. Un reembolso generado por excedente de un anticipo trae su origen visible y no pide comprobantes nuevos, porque ya se revisaron.

## 6. Plazo de comprobación

- El plazo es de **2 días hábiles** (48 horas hábiles), contado desde la **fecha de regreso** del viaje.
- Días hábiles: lunes a viernes, excluyendo los días inhábiles configurados.
- Estados del plazo para mostrar y alertar: faltan N días, vence mañana, vence hoy, vencida hace N días hábiles.
- **Por ahora solo se alerta.** No se bloquean anticipos nuevos ni se hace ningún descuento.
- Alertas: al empleado y a su jefe cuando el plazo vence mañana, vence hoy y está vencido (diaria mientras siga vencido). A Finanzas, un resumen diario de vencidas.
- Una solicitud que lleva más de 24 horas sin atenderse genera una alerta para quien debe actuar (jefe o Finanzas).
- Al enviar a revisión, la bitácora indica si la comprobación fue en plazo o fuera de plazo.
- Finanzas y el jefe pueden enviar un recordatorio manual. Queda en la bitácora.

## 7. Comprobantes y validaciones

Formatos aceptados: XML del CFDI, PDF e imágenes (fotos de tickets).

### Tipos de comprobante

| Tipo | Qué es | Tratamiento |
|---|---|---|
| CFDI | XML y PDF de la factura | Fuente confiable. Se leen los datos del XML. |
| Solo PDF | Factura en PDF sin XML | Se avisa "falta el XML". No es deducible hasta tener el XML. |
| Ticket o nota | Sin CFDI | Se marca como posible no deducible. Finanzas decide. |

### Lectura de documentos

- **XML:** se leen UUID, RFC del emisor y del receptor, fecha, total, forma de pago y conceptos, directamente del archivo, sin IA.
- **PDF e imágenes:** se extrae el texto, o se interpreta la imagen con la API de Claude para identificar el tipo de documento (factura, ticket, nota, recibo, estado de cuenta) y sugerir proveedor, fecha y monto. Si hay XML y PDF del mismo comprobante, se comparan y se avisa de diferencias.
- Todo lo leído por IA se presenta como **sugerencia editable**. El empleado lo confirma y Finanzas hace la validación final. Nunca se acepta un comprobante automáticamente por IA.
- Quien se equivoque en el tipo de documento debe poder corregirlo (por ejemplo, alguien que comprueba con un documento que no es comprobante fiscal).

### Validaciones sobre CFDI

- El RFC receptor debe coincidir con el de la empresa. Si no, alerta crítica.
- El UUID no puede repetirse en ninguna solicitud. Si se repite, alerta crítica.
- Estatus ante el SAT (vigente o cancelado). Si es cancelado, alerta crítica. La consulta se hace al agregar el comprobante y se repite antes del cierre de la revisión.
- Pago en efectivo sobre el límite configurado: se marca como no deducible.
- Las alertas **informan, no bloquean**. Finanzas decide si acepta o rechaza.

### Deducibilidad estimada

La solicitud muestra una estimación de monto deducible y no deducible, calculada con los parámetros de la sección 3. Es una referencia, no un cálculo fiscal definitivo, y así debe decirlo la pantalla.

Para gastos sin CFDI, el monto considerado se limita al tope diario multiplicado por los días del viaje (nacional o extranjero según el ámbito).

### Resumen de importes

En cada solicitud con comprobantes se muestra: anticipo o monto solicitado, total capturado, total aceptado y saldo.

## 8. Bitácora

- Cada acción relevante genera un registro: solicitud creada, aprobada, devuelta, rechazada, anticipo entregado, comprobantes agregados, comprobación enviada, comprobante aceptado o rechazado, revisión concluida, reintegro recibido, reembolso pagado, recordatorio enviado, cambio de parámetros, cambio de usuario o rol.
- Cada registro guarda fecha y hora, usuario, acción y comentario.
- **No se puede editar ni borrar**, ni siquiera por el administrador. Solo se agregan registros.
- Se muestra en el expediente de cada solicitud, en orden cronológico.

## 9. Pantallas

Ver `prototipo.html`. Resumen:

- **Selector de bandeja por rol** con filtros y contadores (por ejemplo, Finanzas: por autorizar, por entregar, por revisar, pagos y reintegros, reembolsos).
- **Lista y expediente:** en computadora, lista a la izquierda y expediente a la derecha. En celular se alternan.
- **Expediente:** estado, línea de pasos, datos del viaje, ruta de autorización, siguiente paso, acciones del rol, comprobantes y bitácora.
- **Nueva solicitud:** formulario con total y ruta de aprobación en vivo.
- **Alertas:** lista ordenada por gravedad.
- **Parámetros** (solo administrador).
- Debe funcionar bien en pantallas de celular y tener modo claro y oscuro.

## 10. Seguridad, privacidad y cumplimiento

- Inicio de sesión con la cuenta corporativa (Microsoft o Google, según lo que use la empresa). La app no guarda contraseñas.
- Comunicación cifrada (HTTPS). Archivos en almacenamiento privado, sin enlaces públicos.
- Respaldos automáticos de la base de datos y de los archivos.
- Permisos verificados en el servidor, no solo en la pantalla.
- Se manejan datos personales de empleados y datos financieros. La aplicación debe cumplir con la **Ley Federal de Protección de Datos Personales en Posesión de los Particulares**: aviso de privacidad visible, recolección limitada a lo necesario y acceso restringido por rol. La revisión legal del aviso de privacidad la hace el área jurídica de la empresa.
- Los CFDI deben conservarse el tiempo que exige la normativa fiscal. El plazo exacto lo confirma el contador y debe ser un parámetro.
- La clave de la API de Claude y cualquier credencial se guardan como variables de entorno, nunca en el código.
- Antes de abrir a todos los usuarios, Sistemas revisa la seguridad y el acceso a los datos.

## 11. Fuera de alcance por ahora

- Bloqueo de nuevos anticipos por comprobaciones pendientes.
- Descuentos por nómina.
- Integración con sistema contable (el registro contable se hace manualmente por ahora).
- Aplicación móvil nativa (se usará la web).
- Pagos automáticos desde el sistema (Finanzas registra el pago hecho por fuera).
- Tarifas por destino o por nivel de puesto.
- Multimoneda con tipo de cambio (los gastos en el extranjero se capturan en MXN por ahora).

## 12. Plan de construcción por etapas

Cada etapa termina con pruebas y mi validación antes de seguir.

1. **Base.** Proyecto, base de datos, inicio de sesión corporativo, usuarios, roles, jerarquía de jefes, parámetros y bitácora.
2. **Solicitudes y autorizaciones.** Crear, aprobar, devolver y rechazar. Ruta según el umbral. Bandejas por rol.
3. **Anticipos.** Entrega del anticipo, plazo de comprobación en días hábiles, alertas en pantalla.
4. **Comprobantes.** Subida de archivos, lectura de XML, validaciones, revisión por Finanzas, cierre con reintegro o reembolso por excedente.
5. **Lectura de PDF e imágenes con IA.** Clasificación del documento y datos sugeridos.
6. **Reembolsos independientes y reportes.** Reembolsos sin anticipo, reportes por área, persona y periodo, exportación a Excel.
7. **Notificaciones por correo** y piloto con 5 a 8 personas.
8. **Revisión de seguridad** y apertura a los 50 usuarios.

## 13. Preguntas abiertas

Resuélvelas conmigo antes de construir la etapa correspondiente.

1. ¿Cuántos niveles de jerarquía hay y qué pasa cuando el jefe directo no está disponible (suplencias o delegación)?
2. ¿El umbral se evalúa sobre el monto solicitado, sobre el monto total del viaje o sobre el monto acumulado por persona? En este borrador se evalúa sobre el monto solicitado de cada solicitud.
3. ¿El anticipo se entrega siempre completo o puede entregarse en parcialidades?
4. ¿Los gastos del extranjero se comprueban con CFDI o con documentos del país? ¿Cómo se maneja el tipo de cambio?
5. ¿Cuánto tiempo debe pasar un reintegro pendiente antes de escalarlo, y a quién?
6. ¿Quién es el contador o responsable que valida los topes fiscales y con qué frecuencia se revisan?
7. ¿Se necesita aviso a Finanzas cuando se rechaza un comprobante, o basta con la bitácora?
8. ¿Qué datos de empleados (RFC, cuenta bancaria) se almacenan y quién los captura?
9. ¿Cuál es el tiempo de conservación de comprobantes y bitácora que exige la política interna?
10. ¿Qué proveedor de correo y de inicio de sesión usa la empresa (Microsoft 365 o Google Workspace)?

## 14. Criterios de aceptación (ejemplos)

Estos casos deben pasar como pruebas.

- Un anticipo de $12,600 aprobado por el jefe directo no pasa por Finanzas.
- Un anticipo de $64,000 aprobado por el jefe directo pasa a `Por autorizar · Finanzas`. Si el administrador cambia el umbral a $70,000 después, la solicitud ya aprobada por el jefe conserva su ruta.
- Un viaje con regreso el viernes tiene límite de comprobación el martes (sábado y domingo no cuentan). Si el lunes es día inhábil configurado, el límite es el miércoles.
- Con anticipo de $18,500 y $18,000 aceptados, queda un reintegro de $500.
- Con anticipo de $9,000 y $11,000 aceptados, se cierra el anticipo y se crea un reembolso de $2,000 ligado al anticipo de origen.
- Un CFDI con UUID ya registrado en otra solicitud muestra alerta crítica de duplicado, pero no impide que Finanzas decida.
- Un pago en efectivo de $2,500 con CFDI se marca como no deducible.
- Un empleado no puede ver solicitudes de otro empleado, y nadie puede aprobar su propia solicitud.
- La bitácora no ofrece ninguna forma de editar o borrar registros.
- Un comprobante enviado como PDF sin XML muestra "falta el XML".
