# Plan general — CosechaClima Backend

Este es el índice de los 6 README que resuelven, uno por uno, los problemas detectados en la revisión. Cada archivo tiene: el problema puntual, la solución, los cambios en base de datos (si aplica), el código completo a pegar, y la explicación línea por línea.

## Orden recomendado de implementación

No los hagas en cualquier orden — hay dependencias reales entre ellos:

1. **01-conexion-base-datos.md** — Sin esto, ningún servicio puede hablar con SQL Server. Es la base de todo lo demás.
2. **02-motor-decisiones.md** — El corazón del proyecto: calcula el semáforo. Depende del punto 1.
3. **03-seed-reglas-decision.md** — Sin datos en `ReglasDecision`, el motor del punto 2 corre pero siempre devuelve "sin regla encontrada". Depende del punto 1.
4. **04-cliente-nasa-power.md** — Alimenta de datos climáticos reales al motor. Depende del punto 1 y complementa el punto 2.
5. **05-autenticacion.md** — Login + JWT. Es independiente de los anteriores, podés hacerlo en paralelo si alguien más del equipo lo toma.
6. **06-actualizar-readme.md** — Se hace al final, cuando el código ya refleja lo que vas a documentar.

## Resumen de cambios en base de datos

Sí, va a haber cambios. Puntualmente:

| Cambio | Tabla | Motivo | README |
|---|---|---|---|
| Nueva tabla | `Alertas` | Guardar el resultado calculado del semáforo por parcela/día (hoy no existe, solo existe `BitacoraCampo` que es *después* de que el usuario actúa) | 02 |
| Nueva columna | `ConnectionStrings` en `appsettings.json` (no es tabla, es configuración) | Sin esto no hay conexión a SQL Server | 01 |
| Poblar datos | `ReglasDecision` | Está creada pero vacía; sin las 90 filas el motor no tiene nada que consultar | 03 |
| Nueva sección | `Jwt` en `appsettings.json` | Configuración de emisión de tokens | 05 |

Ningún cambio rompe lo que ya tenés (modelos, `UmbralConfiguracion`, `BitacoraCampo`, catálogos). Todo es aditivo.

## Antes de empezar

Trabajá en una rama nueva (`feature/motor-decisiones`, etc.) y no directo en `main`, sobre todo porque van a tocar el mismo repo varias personas del equipo. Corré `dotnet build` desde la raíz del `.sln` después de cada README para confirmar que compila antes de pasar al siguiente.
