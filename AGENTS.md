# AGENTS.md

## Contexto

Lee `SPEC.md` antes de cualquier cambio. Es el contrato del proyecto; si el código y el SPEC se contradicen, gana el SPEC (y avísame del conflicto).

## Stack

- **Frontend:** React 19 + TypeScript + Vite + Tailwind v4 + shadcn/ui + supabase-js
- **Datos:** Supabase (Postgres + Auth + Realtime + PostgREST)
- **Worker:** Python 3.12 en `worker/` (solo recordatorios por email)
- **Deploy:** Vercel (`web/`) + Supabase Cloud

No hay backend propio. No hay FastAPI. No hay Next.js. **No agregues un servidor intermedio.**

## Estructura

```
web/                        # app React (Vercel root directory)
  src/
    components/ui/          # shadcn (código propio, se puede editar)
    features/<feature>/
      components/
      hooks/
      api/
    lib/supabase.ts
    lib/database.types.ts   # generado, no editar a mano
worker/                     # Python
supabase/migrations/        # SQL versionado
scripts/                    # test_rls.py, etc.
```

## Comandos

```bash
# dentro de web/
pnpm dev
pnpm build
npx tsc --noEmit

# raíz del repo
python scripts/test_rls.py
npx supabase gen types typescript --linked > web/src/lib/database.types.ts
```

Antes de decir que algo está terminado: **corre `npx tsc --noEmit` y reporta el resultado.**

## Reglas duras

1. **Nunca** modifiques `.env`, `.env.local` ni secrets.
2. **Todo SQL va en `supabase/migrations/`.** Nunca queries sueltas dentro de componentes.
3. **No toques las policies RLS** sin mostrarme el diff y esperar aprobación.
4. **No instales dependencias** sin preguntar primero.
5. **Prohibido `any`.** TypeScript estricto.
6. Componentes de máximo **200 líneas**. Si crece, se divide.
7. La lógica de datos va en **hooks**, nunca en el JSX.
8. En realtime: actualiza el estado local, **no refetchees la lista completa**.
9. Nada de `console.log` en código commiteado.
10. **No inventes tablas ni columnas.** El esquema está en `SPEC.md` y en las migraciones.

## Flujo de trabajo

- **Si el cambio toca más de 3 archivos:** dame el plan primero y espera mi aprobación. No escribas código todavía.
- **Un cambio lógico = un commit.** Conventional commits (`feat:`, `fix:`, `chore:`, `refactor:`).
- **Si encuentras un problema de diseño** en el SPEC, señálalo. No lo resuelvas por tu cuenta.
- **Al terminar una tarea**, lista qué archivos tocaste y qué comando usé para verificarlo.

## Criterios de calidad

- TypeScript sin errores, sin `any`, sin `@ts-ignore`.
- Sin dependencias nuevas sin justificación.
- Manejo de errores visible al usuario (toast), nunca silencioso.
- Estados de carga con skeleton, no spinners de pantalla completa.
- Responsive: debe funcionar a 375px de ancho.
