# SPEC — Gestor de Proyectos

> Este archivo es el contrato. Si el código y este documento se contradicen, gana el documento.

---

## 1. Qué es

App web donde un líder de proyecto asigna trabajo a personas, lo divide en fases y minifases, y cada persona marca su avance. El progreso se ve en tiempo real y llegan recordatorios por correo antes de cada vencimiento.

**Usuario objetivo del demo:** un líder con 3 personas y 2 proyectos activos.

---

## 2. Alcance

### v1 incluye
- Auth email/password (registro, login, logout, sesión persistente)
- CRUD de proyectos
- Asignación de miembros con rol dentro del proyecto
- Fases ordenables y tareas dentro de cada fase
- Progreso calculado automáticamente
- Actualización en tiempo real entre usuarios
- Kanban con drag & drop
- Recordatorios por correo (3 días, 1 día, atrasadas)
- PWA instalable
- Usuario demo público con datos

### v1 NO incluye
Gantt · dependencias entre tareas · chat/comentarios · adjuntos · notificaciones push · multi-tenant · i18n · exportar PDF · sub-tareas anidadas · etiquetas · time tracking.

Si algo no está en la lista de "incluye", **no se construye** sin actualizar este documento primero.

---

## 3. Roles y permisos

### Roles globales (`profiles.global_role`)
`admin` · `leader` · `member`

### Roles por proyecto (`project_members.role`)
`leader` · `member`

### Matriz

| Acción | admin | leader del proyecto | member del proyecto | ajeno |
|---|---|---|---|---|
| Ver proyecto | ✅ | ✅ | ✅ | ❌ |
| Crear proyecto | ✅ | ✅ | ✅¹ | — |
| Editar proyecto | ✅ | ✅ | ❌ | ❌ |
| Gestionar miembros | ✅ | ✅ | ❌ | ❌ |
| Crear/editar fases | ✅ | ✅ | ❌ | ❌ |
| Crear/editar tareas | ✅ | ✅ | ❌ | ❌ |
| Marcar tarea como hecha | ✅ | ✅ | solo la suya | ❌ |
| Ver perfil (nombre, avatar) | ✅ | ✅ | ✅ | — |

¹ Cualquier usuario autenticado puede crear un proyecto y queda automáticamente como `leader` de ese proyecto. No es un privilegio especial: es lo que hace que el demo funcione.

**`admin` no es lo mismo que `leader`.** Un `leader` solo administra sus proyectos. `admin` ve todo, y existe para el rol de "jefe" del demo.

---

## 4. Entidades

### profiles
`id` (uuid, = auth.users.id) · `full_name` · `email` · `global_role` · `avatar_url` · `created_at`

### projects
`id` · `name` · `description` · `status` · `color` · `start_date` · `due_date` · `created_by` · `created_at`

### project_members
`project_id` · `user_id` · `role` · `joined_at` — PK compuesta `(project_id, user_id)`

### phases
`id` · `project_id` · `name` · `position` · `due_date` · `created_at`

### tasks
`id` · `phase_id` · `title` · `description` · `assignee_id` (nullable) · `due_date` · `is_done` · `completed_at` · `completed_by` · `position` · `created_at`

### reminders_log
`id` · `task_id` · `user_id` · `kind` · `sent_at` — UNIQUE `(task_id, user_id, kind)`

### Vista project_progress
`project_id` · `total_tasks` · `done_tasks` · `progress` (0–100) · `overdue_tasks`

---

## 5. Reglas de negocio

### Progreso
```
progress = round(100 * done_tasks / total_tasks)
```
- Si `total_tasks = 0` → `progress = 0` (**nunca NaN, nunca error**)
- Se calcula en la vista, **nunca se almacena como columna**
- Las tareas sin asignar **sí cuentan** para el total

### Salud del proyecto
| Condición | Estado | Color |
|---|---|---|
| `overdue_tasks > 0` | en riesgo | rojo |
| `progress >= 60` y faltan < 7 días para `due_date` | atención | amarillo |
| resto | en curso | verde |

### Estados del proyecto
```
planning → active → completed
              ↕
           paused
    (cualquiera) → archived
```
- Un proyecto `completed` o `archived` es de **solo lectura** en la UI (no se crean ni editan tareas).
- `archived` es la forma de "borrar" en v1. No hay delete.

### Completar una tarea
- Solo por RPC `toggle_task(task_id)`, nunca por update directo desde el cliente.
- Al marcar: `is_done = true`, `completed_at = now()`, `completed_by = auth.uid()`.
- Al desmarcar: `is_done = false`, `completed_at = null`, `completed_by = null`.
- La RPC valida: el usuario es el `assignee_id` **o** es `leader` del proyecto.

### Crear un proyecto
- Solo por RPC `create_project_with_leader(...)`: inserta el proyecto **y** el `project_members` del creador como `leader` en **una sola transacción**.
- Sin esto, el creador no puede ver su propio proyecto por las RLS.

### Recordatorios
- `due_3d`: la tarea vence en 3 días o menos
- `due_1d`: vence hoy o mañana
- `overdue`: ya venció y sigue sin completarse
- Máximo **un correo por** `(tarea, usuario, tipo)` — garantizado por el UNIQUE de `reminders_log`
- Solo se envían a tareas **asignadas** y **no completadas**
- El cron corre a las 13:00 UTC (≈ 9:00 Caracas)

### Fechas
- Todas las fechas de negocio son tipo `date` (sin hora), hora local del usuario.
- `created_at`, `completed_at` son `timestamptz`.

---

## 6. Pantallas y criterios de aceptación

### `/login` y `/register`
- [ ] Error de credenciales se muestra sin recargar
- [ ] Al entrar redirige a `/dashboard`
- [ ] Si ya hay sesión, no se puede volver al login

### `/dashboard`
- [ ] Cards de proyectos con anillo de progreso, semáforo y avatares
- [ ] Contadores: proyectos activos, mis tareas pendientes, tareas atrasadas
- [ ] Sin proyectos → empty state con CTA

### `/projects`
- [ ] Grid de todos los proyectos donde el usuario es miembro
- [ ] Filtro por estado
- [ ] Botón "Nuevo proyecto" (abre modal)

### `/projects/:id`
- [ ] Header: nombre, estado, fechas, miembros, progreso total
- [ ] Tabs: Fases (lista + kanban) · Miembros
- [ ] Un `member` no ve botones de crear/editar
- [ ] Proyecto `completed` → todo en solo lectura

### `/my-tasks`
- [ ] Lista de tareas asignadas al usuario, agrupadas por proyecto
- [ ] Filtros: pendientes / atrasadas / esta semana
- [ ] Marcar hecha actualiza la lista sin recargar

---

## 7. Convenciones

- **DB:** `snake_case`, tablas en plural, PK `id uuid`, FKs `<tabla_singular>_id`
- **TypeScript:** `camelCase`, tipos generados desde Supabase (no escritos a mano)
- **Componentes:** máximo 200 líneas; si crece, se divide
- **Datos:** en hooks (`src/features/<feature>/hooks/`), nunca queries en el JSX
- **Errores:** toast de shadcn, nunca `alert()`
- **Carga:** skeletons, nunca spinner a pantalla completa
- **Prohibido:** `any`, `console.log` en commits, `service_role` en el frontend

---

## 8. Definición de terminado

La v1 está terminada cuando:

1. Un usuario nuevo se registra, crea un proyecto, invita a 2 personas y les asigna tareas.
2. Esas personas entran, ven solo lo suyo y marcan tareas.
3. El progreso se actualiza en la pantalla del líder sin recargar.
4. Un tercero no puede ver el proyecto (RLS).
5. Llega un correo de vencimiento y no se duplica.
6. La app se instala como PWA y se ve correctamente en móvil.
7. Un desconocido con el link del demo entiende qué hace en menos de 60 segundos.
