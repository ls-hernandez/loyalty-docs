# Requerimientos — Frontend Web (Angular)

> Panel de administración web para negocios. Misma funcionalidad que la vista Admin del app móvil Flutter, optimizada para pantalla grande.

## Stack
- Angular (versión a definir)
- Consume la misma REST API del backend Java/Spring

## Módulos previstos

### Dashboard / Resumen
- Métricas clave: clientes, sellos, premios, reservas, inasistencias
- Lista de clientes a punto de completar tarjeta

### Agenda
- Vista calendario semanal/diaria por recurso (profesional o cancha)
- Acciones por reserva: atendida, no asistió, cancelar, confirmar abono
- Crear nueva reserva desde el panel

### Sellos y canje
- Buscar cliente y sumar sello manual
- Validar y canjear premio por código

### Programas de fidelización
- CRUD completo de programas
- Toggle activo/inactivo
- Preview de tarjeta

### Clientes
- Listado con progreso por programa
- Historial de reservas e inasistencias
- Perdonar inasistencias

### Configuración
- Gestión de negocios (multi-negocio)
- Config de booking: horarios, recursos, servicios, precios
- Vinculación programas ↔ reservas

## Notas
- Este frontend se desarrolla en una fase posterior al mobile y backend
- Reutiliza todos los endpoints existentes del backend sin cambios
