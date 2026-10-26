# Requerimientos — Frontend Mobile (Flutter)

## Vista Cliente

### Tarjetas
- Listar tarjetas de sellos activas del cliente
- Tarjeta visual con sellos rellenos/vacíos y animación pop al sumar sello
- Detalle de tarjeta: historial de sellos y recompensas obtenidas
- Estado "¡Lista!" cuando se completa la meta

### Código de cliente
- Pantalla/sheet con código único derivado de teléfono (ej: V-78901C)
- Mostrar nombre y teléfono junto al código

### Reservar
- Selector encadenado: negocio → servicio/profesional o cancha → día (7 próximos) → hora disponible
- Mostrar advertencia si el cliente tiene inasistencias y se requerirá abono
- Mostrar nota si la reserva suma sello en algún programa
- Lista de reservas próximas y historial

### Premios
- Listar premios: disponibles, canjeados, vencidos
- Sheet con código de 6 chars para presentar en el negocio
- Fecha de vencimiento visible

### Explorar
- Listado de negocios con sus programas activos
- Unirse a un programa (primera interacción)
- Acceso rápido a reservar desde explorar

### Notificaciones (in-app)
- Ícono de campana con badge de no leídas
- Lista de avisos: recordatorio de reserva, premio desbloqueado, "te falta 1", "hace un mes no vienes", confirmación de reserva, cancelación

## Vista Admin (negocio)

### Agenda
- Navegador de días (semana actual)
- Vista por recurso (profesional o cancha)
- Por cada reserva: nombre cliente, servicio, precio, estado
- Acciones: Atendida / No asistió / Cancelar / Confirmar abono

### Registrar sellos
- Buscar cliente por nombre, teléfono o código
- Seleccionar programa activo del negocio
- Sumar sello manual (clientes sin reserva)
- Preview de tarjeta del cliente en tiempo real
- Canjear premio por código de 6 chars

### Programas
- Listado con toggle activo/inactivo
- Crear/editar programa: nombre, meta, unidad, tipo de premio, valor, texto cliente, días de vencimiento, reinicio
- Preview de tarjeta en tiempo real al editar
- Eliminar programa (con confirmación)

### Clientes
- Listado de clientes del negocio con progreso por programa
- Conteo de reservas e inasistencias
- Acción "perdonar inasistencias"

### Resumen / Estadísticas
- Clientes con tarjeta activa
- Sellos entregados
- Premios por canjear y canjeados
- Reservas próximos 7 días
- Porcentaje de inasistencias
- Avisos automáticos enviados
- Lista de clientes a punto de completar tarjeta (faltan ≤2 sellos)

### Configuración del negocio
- Crear negocio: nombre, dirección, categoría
- Configurar booking: horario apertura/cierre, recordatorio (horas antes), umbral de abono (N inasistencias)
- Gestionar profesionales o canchas (agregar/quitar)
- Gestionar servicios (peluquería): nombre, duración, precio
- Gestionar precios cancha: diurno, nocturno, hora inicio nocturno
- Vincular programas de fidelización a reservas

## UX general
- Dark mode (sistema + toggle manual)
- Bottom sheets para formularios y confirmaciones
- Toasts/snackbars para feedback rápido
- Selector de negocio activo en modo admin
- Selector de cliente simulado en modo cliente (demo)
