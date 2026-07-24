# 02 — Motor de decisiones (el semáforo)

## Problema

Es tu Objetivo General y RF-04/RF-05: cruzar 4 variables (evento climático, cultivo, etapa fenológica, tipo de suelo) contra `ReglasDecision` y devolver semáforo + 3 acciones. Hoy no existe: no hay tabla para guardar el resultado del día, no hay servicio que haga el cruce, no hay endpoint.

Importante: esto **depende del README 01** (necesitás `ConexionBD` ya funcionando).

## Solución — resumen del flujo

```
GET /api/motor/semaforo?parcelaId=5
        │
        ▼
1. Buscar la parcela (cultivo, etapa fenológica, tipo de suelo)
2. Buscar los umbrales configurados por el usuario (UmbralConfiguracion)
3. Buscar el dato climático más reciente de esa parcela (DatosClimaticos)
4. Comparar el dato climático contra los umbrales → determinar el EventoClimaticoId activo
5. Buscar en ReglasDecision la fila que matchea (Evento, Cultivo, Etapa, Suelo)
6. Guardar/actualizar la Alerta del día en la tabla nueva "Alertas"
7. Devolver el semáforo (verde/amarillo/rojo) + las 3 acciones
```

## Cambio en base de datos

Agregá esto a un script nuevo `Scripts/02-Alertas.sql` (no lo mezcles con `BD-CosechaClima.sql`, así el historial de cambios queda claro):

```sql
USE BD_CosechaClima;
GO

CREATE TABLE Alertas (
    Id INT PRIMARY KEY IDENTITY(1,1),
    UsuarioId INT NOT NULL,
    ParcelaId INT NOT NULL,
    Fecha DATE NOT NULL,
    EventoClimaticoId INT NOT NULL,
    NivelRiesgo NVARCHAR(20) NOT NULL,
    Accion1 NVARCHAR(500) NOT NULL,
    Accion2 NVARCHAR(500) NOT NULL,
    Accion3 NVARCHAR(500) NOT NULL,
    DescripcionAlerta NVARCHAR(500) NOT NULL,
    FechaGeneracion DATETIME DEFAULT GETDATE(),
    CONSTRAINT FK_Alerta_Usuario FOREIGN KEY (UsuarioId) REFERENCES Usuarios(Id),
    CONSTRAINT FK_Alerta_Parcela FOREIGN KEY (ParcelaId) REFERENCES Parcelas(Id),
    CONSTRAINT FK_Alerta_Evento FOREIGN KEY (EventoClimaticoId) REFERENCES EventoClimatico(Id),
    CONSTRAINT UQ_Alerta_Parcela_Fecha UNIQUE (ParcelaId, Fecha)
);
GO
```

**Explicación:**
- Es una tabla separada de `BitacoraCampo` a propósito: `Alertas` es lo que el **motor calculó** ese día (puede regenerarse); `BitacoraCampo` es lo que el **usuario hizo** con esa alerta (marcar acciones completadas, notas). Mezclarlas te obligaría a recalcular el semáforo cada vez que alguien marca una acción como hecha.
- `CONSTRAINT UQ_Alerta_Parcela_Fecha UNIQUE (ParcelaId, Fecha)` — evita tener dos alertas distintas para la misma parcela el mismo día; si el motor se vuelve a correr ese día, tenés que hacer `UPDATE` en vez de `INSERT` (lo maneja el código de abajo).

## Paso a paso del código

### 1. Modelo — `WebApi.Models/Alerta.cs`

```csharp
namespace WebApi.Models;

public class Alerta {
    public int Id { get; set; }
    public int UsuarioId { get; set; }
    public int ParcelaId { get; set; }
    public DateTime Fecha { get; set; }
    public int EventoClimaticoId { get; set; }
    public string NivelRiesgo { get; set; } = string.Empty;
    public string Accion1 { get; set; } = string.Empty;
    public string Accion2 { get; set; } = string.Empty;
    public string Accion3 { get; set; } = string.Empty;
    public string DescripcionAlerta { get; set; } = string.Empty;
    public DateTime FechaGeneracion { get; set; } = DateTime.Now;
}
```

Este archivo sigue exactamente el mismo estilo que tus otros modelos (`BitacoraCampo.cs`, `ReglaDecision.cs`), así que no hace falta explicar línea por línea — son propiedades espejo de las columnas SQL.

### 2. DTO — `WebApi/Dto/SemaforoDto.cs`

```csharp
namespace WebApi.Dto;

public class SemaforoDto {
    public string NivelRiesgo { get; set; } = string.Empty;
    public string DescripcionAlerta { get; set; } = string.Empty;
    public List<string> Acciones { get; set; } = new();
    public DateTime Fecha { get; set; }
}
```

**Explicación:**
- Este es el objeto que **realmente** le devolvés a la app Flutter — no le devolvés la entidad `Alerta` completa (que tiene `UsuarioId`, `Id` interno, etc., datos que el cliente no necesita). Separar DTO de modelo es buena práctica: el modelo representa la tabla, el DTO representa lo que expone la API.
- `List<string> Acciones` — juntamos `Accion1/2/3` en una sola lista, más cómodo de recorrer del lado de Flutter (`for each accion in acciones`) que tres campos sueltos.

### 3. Interfaces

`Services/WebApi.Interface/IMotorDecisionesService.cs`:

```csharp
using WebApi.Models;

namespace WebApi.Interface;

public interface IMotorDecisionesService {
    Task<Alerta> CalcularSemaforo(int parcelaId);
}
```

`Services/WebApi.Interface/IAlertaService.cs`:

```csharp
using WebApi.Models;

namespace WebApi.Interface;

public interface IAlertaService {
    Task<int> GuardarOActualizar(Alerta alerta);
    Task<Alerta?> ObtenerPorParcelaYFecha(int parcelaId, DateTime fecha);
}
```

**Explicación:** separamos "calcular" (lógica de negocio, motor) de "guardar/leer" (acceso a datos, alertas) para respetar la misma arquitectura en capas que ya tenías — el motor no debería saber cómo se guarda en SQL, solo pedirle a `IAlertaService` que lo haga.

### 4. Implementación — `AlertaService.cs`

`Services/WebApi.Implementation/AlertaService.cs`:

```csharp
using Microsoft.Data.SqlClient;
using WebApi.Implementation.Conexion;
using WebApi.Interface;
using WebApi.Models;

namespace WebApi.Implementation;

public class AlertaService : IAlertaService
{
    private readonly ConexionBD _conexionBD;

    public AlertaService(ConexionBD conexionBD)
    {
        _conexionBD = conexionBD;
    }

    public async Task<Alerta?> ObtenerPorParcelaYFecha(int parcelaId, DateTime fecha)
    {
        using var conexion = _conexionBD.CrearConexion();
        using var comando = new SqlCommand(
            "SELECT Id, UsuarioId, ParcelaId, Fecha, EventoClimaticoId, NivelRiesgo, " +
            "Accion1, Accion2, Accion3, DescripcionAlerta, FechaGeneracion " +
            "FROM Alertas WHERE ParcelaId = @ParcelaId AND Fecha = @Fecha", conexion);
        comando.Parameters.AddWithValue("@ParcelaId", parcelaId);
        comando.Parameters.AddWithValue("@Fecha", fecha.Date);

        await conexion.OpenAsync();
        using var lector = await comando.ExecuteReaderAsync();

        if (await lector.ReadAsync())
        {
            return new Alerta
            {
                Id = lector.GetInt32(0),
                UsuarioId = lector.GetInt32(1),
                ParcelaId = lector.GetInt32(2),
                Fecha = lector.GetDateTime(3),
                EventoClimaticoId = lector.GetInt32(4),
                NivelRiesgo = lector.GetString(5),
                Accion1 = lector.GetString(6),
                Accion2 = lector.GetString(7),
                Accion3 = lector.GetString(8),
                DescripcionAlerta = lector.GetString(9),
                FechaGeneracion = lector.GetDateTime(10)
            };
        }
        return null;
    }

    public async Task<int> GuardarOActualizar(Alerta alerta)
    {
        var existente = await ObtenerPorParcelaYFecha(alerta.ParcelaId, alerta.Fecha);

        using var conexion = _conexionBD.CrearConexion();
        await conexion.OpenAsync();

        if (existente is null)
        {
            using var comandoInsert = new SqlCommand(
                "INSERT INTO Alertas (UsuarioId, ParcelaId, Fecha, EventoClimaticoId, NivelRiesgo, " +
                "Accion1, Accion2, Accion3, DescripcionAlerta) " +
                "OUTPUT INSERTED.Id " +
                "VALUES (@UsuarioId, @ParcelaId, @Fecha, @EventoClimaticoId, @NivelRiesgo, " +
                "@Accion1, @Accion2, @Accion3, @DescripcionAlerta)", conexion);

            AgregarParametros(comandoInsert, alerta);
            var nuevoId = (int)await comandoInsert.ExecuteScalarAsync()!;
            return nuevoId;
        }
        else
        {
            using var comandoUpdate = new SqlCommand(
                "UPDATE Alertas SET EventoClimaticoId = @EventoClimaticoId, NivelRiesgo = @NivelRiesgo, " +
                "Accion1 = @Accion1, Accion2 = @Accion2, Accion3 = @Accion3, " +
                "DescripcionAlerta = @DescripcionAlerta, FechaGeneracion = GETDATE() " +
                "WHERE Id = @Id", conexion);

            AgregarParametros(comandoUpdate, alerta);
            comandoUpdate.Parameters.AddWithValue("@Id", existente.Id);
            await comandoUpdate.ExecuteNonQueryAsync();
            return existente.Id;
        }
    }

    private static void AgregarParametros(SqlCommand comando, Alerta alerta)
    {
        comando.Parameters.AddWithValue("@UsuarioId", alerta.UsuarioId);
        comando.Parameters.AddWithValue("@ParcelaId", alerta.ParcelaId);
        comando.Parameters.AddWithValue("@Fecha", alerta.Fecha.Date);
        comando.Parameters.AddWithValue("@EventoClimaticoId", alerta.EventoClimaticoId);
        comando.Parameters.AddWithValue("@NivelRiesgo", alerta.NivelRiesgo);
        comando.Parameters.AddWithValue("@Accion1", alerta.Accion1);
        comando.Parameters.AddWithValue("@Accion2", alerta.Accion2);
        comando.Parameters.AddWithValue("@Accion3", alerta.Accion3);
        comando.Parameters.AddWithValue("@DescripcionAlerta", alerta.DescripcionAlerta);
    }
}
```

**Explicación de las partes nuevas (lo demás ya lo viste en el README 01):**
- `ObtenerPorParcelaYFecha` primero, dentro de `GuardarOActualizar` — así sabemos si hay que hacer `INSERT` o `UPDATE`, respetando la restricción `UNIQUE (ParcelaId, Fecha)` que pusimos en la tabla.
- `OUTPUT INSERTED.Id` — le pide a SQL Server que devuelva el `Id` recién generado por el `IDENTITY`, así no hace falta un segundo `SELECT @@IDENTITY` separado.
- `ExecuteScalarAsync()` — se usa cuando el resultado esperado es **un solo valor** (acá, el id), a diferencia de `ExecuteReaderAsync()` que es para múltiples filas/columnas.
- `AgregarParametros` como método privado aparte — evita repetir 8 líneas de `Parameters.AddWithValue` dos veces (en el `INSERT` y en el `UPDATE`); es el único parámetro que cambia es si mandás también `@Id` (solo en el `UPDATE`).

### 5. Implementación — `MotorDecisionesService.cs`

Este es el archivo más importante del proyecto. Asumo que ya tenés (o vas a tener) `ParcelaService`, `UmbralConfiguracionService`, `DatosClimaticoService` y `ReglaDecisionService` funcionando con el mismo patrón ADO.NET del README 01 — este servicio los combina.

```csharp
using WebApi.Interface;
using WebApi.Models;

namespace WebApi.Implementation;

public class MotorDecisionesService : IMotorDecisionesService
{
    private readonly IParcelaService _parcelaService;
    private readonly IUmbralConfiguracionService _umbralService;
    private readonly IDatosClimaticoService _datosClimaticoService;
    private readonly IReglaDecisionService _reglaDecisionService;
    private readonly IAlertaService _alertaService;

    public MotorDecisionesService(
        IParcelaService parcelaService,
        IUmbralConfiguracionService umbralService,
        IDatosClimaticoService datosClimaticoService,
        IReglaDecisionService reglaDecisionService,
        IAlertaService alertaService)
    {
        _parcelaService = parcelaService;
        _umbralService = umbralService;
        _datosClimaticoService = datosClimaticoService;
        _reglaDecisionService = reglaDecisionService;
        _alertaService = alertaService;
    }

    public async Task<Alerta> CalcularSemaforo(int parcelaId)
    {
        var parcela = await _parcelaService.ObtenerPorId(parcelaId)
            ?? throw new InvalidOperationException($"No existe la parcela {parcelaId}");

        if (parcela.EtapaFenologicaId is null)
            throw new InvalidOperationException("La parcela no tiene etapa fenologica asignada");

        var umbrales = await _umbralService.ObtenerPorUsuario(parcela.UsuarioId)
            ?? throw new InvalidOperationException("El usuario no tiene umbrales configurados");

        var ultimosDatos = await _datosClimaticoService.ObtenerUltimosDatos(parcelaId, dias: 1);
        var datoHoy = ultimosDatos.FirstOrDefault()
            ?? throw new InvalidOperationException("No hay datos climaticos disponibles para esta parcela");

        int eventoClimaticoId = DeterminarEventoActivo(datoHoy, umbrales);

        var reglas = await _reglaDecisionService.ObtenerTodas();
        var regla = reglas.FirstOrDefault(r =>
            r.EventoClimaticoId == eventoClimaticoId &&
            r.CultivoId == parcela.CultivoId &&
            r.EtapaFenologicaId == parcela.EtapaFenologicaId &&
            r.TipoSueloId == parcela.TipoSueloId);

        if (regla is null)
            throw new InvalidOperationException(
                $"No existe una regla para Evento={eventoClimaticoId}, Cultivo={parcela.CultivoId}, " +
                $"Etapa={parcela.EtapaFenologicaId}, Suelo={parcela.TipoSueloId}. Revisar seed de ReglasDecision.");

        var alerta = new Alerta
        {
            UsuarioId = parcela.UsuarioId,
            ParcelaId = parcela.Id,
            Fecha = DateTime.Today,
            EventoClimaticoId = eventoClimaticoId,
            NivelRiesgo = regla.NivelRiesgo,
            Accion1 = regla.Accion1,
            Accion2 = regla.Accion2,
            Accion3 = regla.Accion3,
            DescripcionAlerta = regla.DescripcionAlerta
        };

        alerta.Id = await _alertaService.GuardarOActualizar(alerta);
        return alerta;
    }

    private static int DeterminarEventoActivo(DatosClimaticos dato, UmbralConfiguracion umbrales)
    {
        // El orden importa: se evalua de mas a menos severo.
        // IDs segun el INSERT de EventoClimatico en BD-CosechaClima.sql:
        // 1 = Lluvia intensa, 2 = Canicula, 3 = Viento fuerte,
        // 4 = Temperatura extrema, 5 = Riesgo de helada

        if (dato.TemperaturaMin is not null && dato.TemperaturaMin <= 2)
            return 5; // Riesgo de helada

        if (dato.Precipitacion is not null && dato.Precipitacion >= umbrales.LluviaIntensaMm)
            return 1; // Lluvia intensa

        if (dato.VientoVelocidad is not null && dato.VientoVelocidad >= umbrales.VientoFuerteKmh)
            return 3; // Viento fuerte

        if (dato.TemperaturaMax is not null && dato.TemperaturaMax >= 35)
            return 4; // Temperatura extrema

        // Sin dato puntual de "dias sin lluvia acumulados" todavia (requiere
        // revisar varios dias seguidos, no un solo registro) -- por ahora
        // se marca canicula si la precipitacion es 0. Ajustar cuando se
        // implemente el conteo de dias consecutivos usando CaniculaDias.
        if (dato.Precipitacion is not null && dato.Precipitacion == 0)
            return 2; // Canicula

        throw new InvalidOperationException("Ningun evento climatico supera los umbrales configurados");
    }
}
```

**Explicación:**
- El constructor recibe **5 interfaces**, no clases concretas — esto es el patrón de inyección de dependencias que ya usa el resto de tu proyecto (`WebApi.Interface`). El motor no sabe ni le importa si detrás hay ADO.NET, Dapper o lo que sea; solo conoce el contrato.
- `?? throw new InvalidOperationException(...)` aparece varias veces — es el patrón de "falla rápido y con mensaje claro" en vez de dejar que un `NullReferenceException` genérico explote más abajo sin decir por qué.
- `DeterminarEventoActivo` es la función que traduce **datos crudos** (temperatura, lluvia, viento) en **un evento categórico** (uno de los 5 `EventoClimatico`). Este es el paso que tu informe no detalla del todo — tuve que inferir un orden de severidad razonable (helada > lluvia intensa > viento > temperatura extrema > canícula), pero **esto lo tiene que revisar y ajustar el técnico del INTA** que mencionás en la sección 7.3.3 del informe; los números de umbral (35°C, 2°C) son placeholders razonables, no verdades agronómicas.
- El comentario sobre "canícula" es honesto sobre una limitación real: canícula es un evento que depende de **varios días seguidos sin lluvia** (tenés el campo `CaniculaDias` en `UmbralConfiguracion` para esto), no de un solo registro de `DatosClimaticos`. Implementarlo bien requiere consultar los últimos N días, no solo "hoy". Te dejo la versión simple funcionando y anotada para que no se te olvide que falta esa mejora — no la escondas, es justo el tipo de cosa que un jurado técnico pregunta.
- `reglas.FirstOrDefault(r => ...)` — esto usa LINQ sobre la lista completa de reglas en memoria (la trae `ObtenerTodas()`) en vez de hacer una consulta SQL con 4 condiciones en el `WHERE`. Es más simple de leer y, con máximo 90 filas, el costo de traerlas todas a memoria es insignificante. Si el árbol creciera a miles de reglas, ahí sí convendría mover el filtro a SQL.

### 6. Controlador — `WebApi/Controllers/MotorDecisionesController.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using WebApi.Dto;
using WebApi.Interface;

namespace WebApi.Controllers;

[ApiController]
[Route("api/motor")]
public class MotorDecisionesController : ControllerBase
{
    private readonly IMotorDecisionesService _motorDecisionesService;

    public MotorDecisionesController(IMotorDecisionesService motorDecisionesService)
    {
        _motorDecisionesService = motorDecisionesService;
    }

    [HttpGet("semaforo")]
    public async Task<ActionResult<SemaforoDto>> ObtenerSemaforo([FromQuery] int parcelaId)
    {
        try
        {
            var alerta = await _motorDecisionesService.CalcularSemaforo(parcelaId);

            var dto = new SemaforoDto
            {
                NivelRiesgo = alerta.NivelRiesgo,
                DescripcionAlerta = alerta.DescripcionAlerta,
                Acciones = new List<string> { alerta.Accion1, alerta.Accion2, alerta.Accion3 },
                Fecha = alerta.Fecha
            };

            return Ok(dto);
        }
        catch (InvalidOperationException ex)
        {
            return NotFound(new { mensaje = ex.Message });
        }
    }
}
```

**Explicación:**
- `[ApiController]` — habilita validación automática de parámetros y respuestas estandarizadas de error (400) sin que tengas que escribirlas a mano.
- `[Route("api/motor")]` + `[HttpGet("semaforo")]` — arma la ruta final `GET /api/motor/semaforo`, tal como aparece en tu propio README raíz.
- `[FromQuery] int parcelaId` — lee `?parcelaId=5` de la URL. Tu informe usa `usuarioId` en el ejemplo de la tabla de endpoints, pero el motor necesita saber de **qué parcela** específica (un usuario puede tener varias parcelas con distintos cultivos), así que usé `parcelaId`. Decidilo con tu equipo y sé consistente en toda la API.
- `try/catch (InvalidOperationException ex)` — atrapa los errores de negocio que lanzamos en `MotorDecisionesService` (parcela sin etapa, sin umbrales, sin regla, etc.) y los convierte en un `404` con mensaje legible, en vez de un `500` genérico que no le dice nada útil a quien está probando la API desde Swagger.

### 7. Registrar todo en `Program.cs`

```csharp
builder.Services.AddScoped<ConexionBD>();
builder.Services.AddScoped<IParcelaService, ParcelaService>();
builder.Services.AddScoped<IUmbralConfiguracionService, UmbralConfiguracionService>();
builder.Services.AddScoped<IDatosClimaticoService, DatosClimaticoService>();
builder.Services.AddScoped<IReglaDecisionService, ReglaDecisionService>();
builder.Services.AddScoped<IAlertaService, AlertaService>();
builder.Services.AddScoped<IMotorDecisionesService, MotorDecisionesService>();
```

**Explicación:** cada línea le dice al contenedor de DI "cuando alguien pida la interfaz de la izquierda, entregale una instancia de la clase de la derecha". Sin este registro, `MotorDecisionesController` va a tronar al arrancar con un error de "no se pudo resolver el servicio". Esto asume que ya implementaste `ParcelaService`, `UmbralConfiguracionService` y `DatosClimaticoService` (son CRUD estándar con el mismo patrón del README 01 — si querés, te armo esos tres en un README aparte).

## Checklist

- [ ] Corriste el script `02-Alertas.sql` contra tu base local.
- [ ] `Alerta.cs`, `SemaforoDto.cs` compilan.
- [ ] `IMotorDecisionesService`, `IAlertaService` agregadas.
- [ ] `AlertaService` y `MotorDecisionesService` implementados y registrados en `Program.cs`.
- [ ] Desde Swagger, `GET /api/motor/semaforo?parcelaId=1` responde (aunque sea con 404 "no hay reglas" si todavía no corriste el README 03).
