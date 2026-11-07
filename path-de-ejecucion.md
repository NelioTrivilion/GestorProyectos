# Path Definitivo — Gestor de Proyectos

> Stack cerrado. Construcción por fases con gate de verificación.
> Esquema SQL y políticas RLS: `plan-gestor-proyectos.md`

**Stack:** React + Vite + Tailwind + shadcn/ui · Supabase · Python (worker) · GitHub Actions · Vercel

**Tiempo estimado:** 3–4 semanas full-time · 6–8 semanas a 3h/día.

---

## Mapa de fases

| Fase | Qué | Tiempo | Gate |
|---|---|---|---|
| **F0** | Terreno: herramientas, repo, SPEC.md | 0.5 d | Todo instalado, repo creado |
| **F0.5** | Calentamiento: Vite, Tailwind, shadcn | 1 d | Sabes por qué existe cada archivo |
| **F1** | Supabase: esquema + RLS + seed | 2 d | 7 tests de RLS pasan |
| **F2** | Auth + shell de la app | 1.5 d | Sesión persiste al recargar |
| **F3** | Proyectos + miembros | 2.5 d | Un tercero no ve el proyecto |
| **F4** | Fases + tareas + progreso | 3 d | 4/5 = 80%, consistente |
| **F5** | Realtime + Kanban | 2 d | Se actualiza en <1s entre navegadores |
| **F6** | Worker Python + emails | 1.5 d | Llega el correo y no se duplica |
| **F7** | Deploy + PWA + cierre | 2 d | Un desconocido entiende la app en 60s |

---

## Las 3 herramientas nuevas: qué aprender y qué ignorar

No hagas cursos de 40 horas. Aprende esto y arranca — el resto se aprende just-in-time.

### Vite — 2 horas

**Aprende:** qué es un bundler, el dev server con hot reload, `npm run build`, y por qué las variables de entorno del frontend deben empezar con `VITE_`.

**Clave:** todo lo que empieza con `VITE_` **se hornea en el bundle público**. Por eso la `anon key` de Supabase puede ir ahí (es pública por diseño) y la `service_role` jamás.

**Ignora:** plugins custom, SSR, proxy.

### Tailwind v4 — 4 horas

**Aprende:** clases utilitarias, prefijos responsive (`md:`, `lg:`), estados (`hover:`, `focus-visible:`), dark mode por clase, y `@theme` para definir tus tokens (colores, radios) en CSS.

**Clave:** en v4 ya no hay `tailwind.config.js` — la configuración es CSS-first, en tu `index.css` con `@theme`.

**Bonus:** `cn()` (clsx + tailwind-merge) para combinar clases condicionales sin conflictos.

**Ignora:** plugins, animaciones custom, `@apply` (evítalo, rompe la utilidad de Tailwind).

### shadcn/ui — 3 horas

**Aprende:** **no es una librería que instalas, es código que copias a tu repo.** Cuando corres `npx shadcn@latest add button`, el componente aparece en `src/components/ui/button.tsx` y es tuyo: lo puedes editar. Por debajo usa Radix (accesibilidad de verdad).

**Clave:** por eso se ve bien y no genérico — no es un theme, es código que controlas.

**Ignora:** blocks premium, themes avanzados.

### React / TypeScript

Asumo que viste React en la carrera. Repaso: componentes, props, `useState`, `useEffect`, listas con `key`, eventos, y tipar props con `interface`.

**Si nunca tocaste React:** suma 1 semana antes de F2. No intentes aprender React construyendo la app de verdad.

---

## F0 — Terreno (0.5 día)

### Tareas

1. Instalar y verificar:
   ```bash
   node -v          # 22+
   pnpm -v          # 9+
   python --version # 3.12+
   git --version
   npm i -g supabase
   supabase --version
   ```
2. Crear repo + estructura:
   ```
   gestor-proyectos/
     web/                # React (Vercel root = web/)
     worker/             # Python
     supabase/
       migrations/
     scripts/            # test_rls.py, etc.
     SPEC.md
     AGENTS.md
     README.md
     .env.example
     .gitignore
   ```
3. Escribir `SPEC.md` — modelo de datos, matriz de permisos, estados, criterios de aceptación.
4. Escribir `AGENTS.md` — stack, comandos, convenciones, prohibiciones.
5. Cuentas: Supabase (proyecto **dev**), GitHub, Vercel (conecta después), Resend (después).

### Decisiones a cerrar en SPEC.md

| # | Decisión | Valor |
|---|---|---|
| 1 | Roles | Global: `admin`/`leader`/`member` · Por proyecto: `leader`/`member` |
| 2 | Quién crea proyectos | `leader` y `admin` |
| 3 | Progreso | `done/total` sin pesos |
| 4 | Estados | `planning → active → completed`, + `paused`, `archived` |
| 5 | Fechas | tipo `date`, sin hora. Cron a las 13:00 UTC |
| 6 | Fuera de v1 | Gantt, chat, adjuntos, push, i18n, multi-tenant |

> **Gate:** `node -v`, `pnpm -v`, `supabase --version` responden. Repo con estructura. SPEC.md y AGENTS.md escritos.

---

## F0.5 — Calentamiento (1 día)

Un mini-proyecto desechable para tocar las 3 herramientas sin la presión de la app real.

### Tareas

1. `pnpm create vite@latest calentamiento --template react-ts`
2. Instalar Tailwind v4, configurar el `@theme` con tus tokens (fondo, superficie, acento violeta).
3. `npx shadcn@latest init` + `npx shadcn@latest add button card input`
4. Construir una pantalla estática: header, 2 cards con un anillo de progreso simulado, un botón shadcn, dark mode.
5. `pnpm run build` y abrir `dist/` para entender qué produce Vite.

### Ejercicio mental (el gate real)

Abre el proyecto y responde sin mirar Google:

- ¿Qué hace `vite.config.ts`?
- ¿Por qué el CSS está en `index.css` y no en un archivo por componente?
- ¿Dónde vive `button.tsx` y quién lo mantiene?
- ¿Qué pasa si pongo `SUPABASE_KEY` en `.env` sin el prefijo `VITE_`?

> **Gate:** respondes las 4. Si no, no sigas — vas a pelear con la app real por razones equivocadas.

**Borra el proyecto al terminar.** No lo mezcles con el repo principal.

---

## F1 — Supabase: esquema + RLS (2 días)

### Tareas

1. `supabase init` en el repo, `supabase link` al proyecto **dev**.
2. Migraciones versionadas — nunca SQL pegado en el dashboard:
   ```
   supabase/migrations/
     0001_schema.sql      -- profiles, projects, project_members, phases, tasks, reminders_log
     0002_functions.sql   -- is_project_member, is_project_leader, task_project, handle_new_user
     0003_views.sql       -- project_progress (con security_invoker = true)
     0004_rls.sql         -- enable RLS + policies + create_project_with_leader (RPC)
     0005_seed.sql        -- 3 usuarios, 3 proyectos, tareas con fechas RELATIVAS
   ```
3. Índices:
   ```sql
   create index on tasks (phase_id);
   create index on tasks (assignee_id) where not is_done;
   create index on tasks (due_date)   where not is_done;
   create index on phases (project_id);
   create index on project_members (user_id);
   ```
4. Seed con `current_date + 2`, `+ 5`, `- 1` — nunca fechas fijas.
5. `scripts/test_rls.py` con los 7 casos de la matriz.

### El caso que rompe todo

Al crear un proyecto, el creador debe entrar en `project_members` como `leader` **en la misma transacción**. Si no, las RLS le impiden ver su propio proyecto. Va en la RPC `create_project_with_leader`, no en dos llamadas del frontend.

> **Gate:** los 7 tests de RLS pasan. Un usuario no puede leer un proyecto al que no pertenece.
> **Prompt:** *"Lee SPEC.md y AGENTS.md. Crea las migraciones 0001–0005 en supabase/migrations/. No generes frontend. Muéstrame primero el SQL de las policies de 0004 para revisarlo antes de aplicar."*

---

## F2 — Auth + shell (1.5 días)

### Tareas

1. App React en `web/` + Tailwind + shadcn con los tokens del design system.
2. `src/lib/supabase.ts` + `.env.local`.
3. `AuthProvider`: `getSession()` inicial + `onAuthStateChange`.
4. `ProtectedRoute`, `/login`, `/register` (shadcn Card + Input + Button).
5. Layout: sidebar (Dashboard / Mis tareas / Proyectos) + topbar con avatar y logout.
6. `supabase gen types typescript --linked > web/src/lib/database.types.ts`

### El bug #1 de este stack

La sesión tiene **3 estados**: `loading | authenticated | anonymous`. Si tratas `loading` como `anonymous`, la app te expulsa al login en cada recarga.

> **Gate:** registro → entro → veo mi nombre → recargo y sigo dentro → logout.
> **Prompt:** *"Implementa auth con supabase-js. AuthProvider con estados loading/authenticated/anonymous, ProtectedRoute, páginas login y register con shadcn. Criterio: recargar no expulsa al usuario."*

---

## F3 — Proyectos + miembros (2.5 días)

### Tareas

1. Grid de proyectos: card con color, anillo de progreso, semáforo de salud, avatares apilados.
2. Crear/editar proyecto (React Hook Form + Zod) → llama la RPC.
3. Modal de miembros: buscar en `profiles`, asignar rol, quitar.
4. Detalle de proyecto con tabs: Fases / Miembros.
5. Query contra `project_progress`.

### Cuidado

El modal de miembros necesita leer `profiles` de gente que no es miembro de tus proyectos. Si cierras la policy de `profiles` a "solo miembros", el buscador no encuentra a nadie. Deja `profiles` legible por cualquier autenticado.

> **Gate:** creo proyecto → asigno 2 personas → cada una lo ve → un tercero no.
> **Prompt:** *"Grid de proyectos con anillo de progreso desde project_progress y semáforo. Si no hay proyectos, empty state. Criterio: tsc pasa y el progreso es 0 en proyectos sin tareas, no NaN."*

---

## F4 — Fases, tareas y progreso (3 días)

### Tareas

1. CRUD de fases con `position` y reordenamiento.
2. CRUD de tareas: título, descripción, asignado, fecha, fase.
3. Toggle completado vía RPC `toggle_task` (setea `is_done`, `completed_at`, `completed_by`).
4. Progreso por fase y por proyecto.
5. Vista "Mis tareas" con filtros: pendientes / atrasadas / esta semana.
6. Skeletons y empty states.

### Casos borde

- Proyecto sin tareas → `0`, no `NaN`.
- Desmarcar → limpiar `completed_at` y `completed_by`.
- El toggle no confía en el cliente: la RPC valida que la tarea sea tuya o que seas líder.

> **Gate:** marco 4 de 5 → 80%. Desmarco una → 60%. Recargo → 60%.
> **Prompt:** *"CRUD de tareas con toggle vía RPC toggle_task. Incluye filtro 'mis tareas'. Criterio: tsc pasa, no hay NaN, y desmarcar limpia completed_at."*

---

## F5 — Realtime + Kanban (2 días)

### Tareas

1. Suscripción a `postgres_changes` en `tasks` filtrada por proyecto.
2. Kanban con dnd-kit: columnas = fases, tarjetas = tareas.
3. Optimistic update al arrastrar (mover de fase = cambiar `phase_id`).
4. Rollback si el update falla.

> **Gate:** dos navegadores con usuarios distintos; marco una tarea en uno y el otro actualiza sin recargar en <1s.
> **Prompt:** *"Suscríbete a postgres_changes de tasks filtrando por project_id y actualiza el estado local. No refetchees la lista completa en cada evento."*

---

## F6 — Worker Python + emails (1.5 días)

### Tareas

1. Cuenta Resend + `RESEND_API_KEY` (prueba con `onboarding@resend.dev`).
2. Módulo:
   ```
   worker/
     main.py        # orquestador + --dry-run
     queries.py     # Supabase con service_role
     classify.py    # due_3d | due_1d | overdue
     emails.py      # cliente Resend
     templates.py   # HTML con estilos inline
   ```
3. `--dry-run` que imprime qué enviaría. **Prueba así primero.**
4. Idempotencia con `reminders_log` (+ unique constraint como red de seguridad).
5. `.github/workflows/reminders.yml` con `schedule` y `workflow_dispatch`.

### Gotchas

- `service_role` solo en GitHub Secrets.
- Estilos **inline** en el HTML del correo; Gmail ignora `<style>` del `<head>`.
- El worker corre en UTC: compara contra `date.today()` en UTC o fuerza la zona.

> **Gate:** disparo manual → llega. Disparo otra vez → no duplica.
> **Prompt:** *"Implementa worker/main.py con --dry-run. No envíes nada real hasta que yo lo pida. Usa reminders_log para no duplicar."*

---

## F7 — Deploy + cierre (2 días)

### Tareas

1. Proyecto Supabase **prod** + migraciones con `supabase db push`.
2. Vercel: root directory `web/`, `VITE_SUPABASE_URL` + `VITE_SUPABASE_ANON_KEY`.
3. PWA: manifest + iconos + service worker (vite-plugin-pwa). Instalable desde el navegador, ventana propia, sin barra de URL.
4. Usuario demo con proyecto "Ejemplo" y 2 tareas por vencer.
5. README: diagrama Mermaid, GIF del kanban, tabla de permisos, decisiones y por qué.
6. Video de 90s.
7. Limpieza: `grep -r "service_role" web/dist/` debe estar vacío. Lighthouse > 90.

> **Gate:** un desconocido entra a la URL, prueba el demo y entiende la app en <60s.

---

## Cómo pilotar el agente en cada fase

### `AGENTS.md` (lo lee OpenCode automáticamente)

```markdown
## Stack
React 19 + TS + Vite + Tailwind v4 + shadcn/ui + supabase-js
Supabase (Postgres, Auth, Realtime). Worker Python en worker/.

## Comandos
- dev:   pnpm dev (dentro de web/)
- types: npx supabase gen types typescript --linked > web/src/lib/database.types.ts
- test:  python scripts/test_rls.py

## Reglas
- Nunca modifiques .env / .env.local ni secrets.
- Todo SQL en supabase/migrations/. Nunca queries sueltas en componentes.
- No toques las policies RLS sin mostrar el diff primero.
- No instales dependencias sin preguntar.
- Componentes < 200 líneas. Datos en hooks (src/features/<x>/hooks), no en JSX.
- TypeScript estricto: prohibido `any`.
```

### Reglas de operación

1. **Un prompt = una feature + criterio de aceptación verificable.**
2. **Pide plan antes del código** si el cambio toca más de ~3 archivos.
3. **Commit antes de cada prompt grande.** `git reset --hard` es tu undo real.
4. **Verifica con comandos:** `tsc --noEmit`, el script de RLS, abrir la app. No con "se ve bien".
5. **Las RLS las revisas tú.** Es el contrato de seguridad.
6. **Sesión nueva por fase.** El contexto viejo hace que "arregle" lo que ya funcionaba.
7. **MCPs activos: solo `context7` y `supabase` (apuntando a dev).** Los demás son ruido en el contexto.
