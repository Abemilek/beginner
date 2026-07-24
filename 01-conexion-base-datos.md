# 01 — Conexión a la base de datos

## Problema

`appsettings.json` solo tiene `Logging` y `AllowedHosts`. No hay `ConnectionStrings`, y no existe ninguna clase que abra una `SqlConnection`. Sin esto, ningún método de `WebApi.Implementation` puede tocar SQL Server, así que es el primer bloqueante para todo lo demás.

## Solución

1. Agregar la cadena de conexión en `appsettings.json` / `appsettings.Development.json`.
2. Crear una clase `ConexionBD` en `WebApi.Implementation` que lea esa cadena vía `IConfiguration` y entregue conexiones nuevas cuando se le pidan.
3. Registrarla en el contenedor de dependencias (`Program.cs`) para poder inyectarla en cualquier servicio.

No hay cambio de tablas acá, es pura configuración y plomería de acceso a datos.

## Paso a paso

### 1. `WebApi/appsettings.Development.json`

Reemplazá el contenido completo por:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "BD_CosechaClima": "Server=localhost;Database=BD_CosechaClima;User Id=sa;Password=TU_PASSWORD_LOCAL;TrustServerCertificate=True;"
  }
}
```

**Explicación:**
- `"ConnectionStrings"` es la sección estándar que ASP.NET Core sabe leer con `IConfiguration.GetConnectionString(...)`.
- `"BD_CosechaClima"` es el nombre que le vamos a poner nosotros a esa cadena; tiene que coincidir exactamente con el string que usemos después en el código.
- `Server=localhost` — apunta a tu SQL Server local (o Docker). En producción esto va a cambiar por el servidor de Azure SQL.
- `Database=BD_CosechaClima` — coincide con el `CREATE DATABASE` de tu script.
- `User Id` / `Password` — credenciales de SQL Server (o quitalas y usá `Trusted_Connection=True` si usás autenticación de Windows).
- `TrustServerCertificate=True` — necesario en desarrollo local porque SQL Server suele usar un certificado autofirmado; el driver rechaza la conexión sin esto.

> Esta cadena va en `appsettings.Development.json` (no se sube con datos reales a git) y en producción se sobreescribe con una variable de entorno o el `appsettings.json` de Azure. Agregá `appsettings.Development.json` al `.gitignore` si tiene contraseñas reales, o usá `dotnet user-secrets` en vez de escribir la contraseña en el archivo.

### 2. Crear `Services/WebApi.Implementation/Conexion/ConexionBD.cs`

```csharp
using Microsoft.Data.SqlClient;
using Microsoft.Extensions.Configuration;

namespace WebApi.Implementation.Conexion;

public class ConexionBD
{
    private readonly string _cadena;

    public ConexionBD(IConfiguration configuracion)
    {
        _cadena = configuracion.GetConnectionString("BD_CosechaClima")
            ?? throw new InvalidOperationException(
                "No se encontro la cadena de conexion 'BD_CosechaClima' en appsettings.json");
    }

    public SqlConnection CrearConexion()
    {
        return new SqlConnection(_cadena);
    }
}
```

**Explicación línea por línea:**
- `using Microsoft.Data.SqlClient;` — importa el driver de SQL Server que ya tenés referenciado en el `.csproj` de Implementation.
- `using Microsoft.Extensions.Configuration;` — trae la interfaz `IConfiguration`, que es como ASP.NET Core expone todo lo que hay en `appsettings.json` al resto de la app.
- `namespace WebApi.Implementation.Conexion;` — carpeta/namespace nuevo, para que quede claro que es infraestructura de acceso a datos, no lógica de negocio.
- `private readonly string _cadena;` — guardamos la cadena de conexión una sola vez cuando se crea la clase; `readonly` evita que alguien la reasigne por error después.
- `public ConexionBD(IConfiguration configuracion)` — esto es **inyección de dependencias**: no instanciamos `ConexionBD` nosotros mismos, se lo pedimos al contenedor de ASP.NET Core y él nos entrega la configuración automáticamente.
- `_cadena = configuracion.GetConnectionString("BD_CosechaClima")` — busca en `appsettings.json` la cadena con ese nombre exacto.
- `?? throw new InvalidOperationException(...)` — si el nombre está mal escrito o falta la sección, preferís que la app truene al arrancar con un mensaje claro, en vez de fallar más adelante con un error críptico de "cadena de conexión vacía" en medio de una consulta.
- `public SqlConnection CrearConexion()` — método público que cualquier servicio (`UsuarioService`, `ParcelaService`, etc.) va a llamar cada vez que necesite hablar con la base. Se crea una conexión **nueva** cada vez (no se comparte una sola conexión global) porque `SqlConnection` no es thread-safe y ADO.NET está diseñado para abrir/usar/cerrar rápido (connection pooling se encarga de la eficiencia por debajo).

### 3. Registrar el servicio en `WebApi/Program.cs`

Agregá esta línea junto a las demás de `builder.Services`:

```csharp
using WebApi.Implementation.Conexion;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddScoped<ConexionBD>();
```

**Explicación:**
- `using WebApi.Implementation.Conexion;` — para que `Program.cs` conozca la clase nueva.
- `builder.Services.AddScoped<ConexionBD>();` — registra `ConexionBD` en el contenedor de DI con ciclo de vida **scoped**: se crea una instancia por cada petición HTTP que llega, y se reutiliza dentro de esa misma petición si varios servicios la piden. Es el ciclo de vida correcto para algo ligado a una conexión de datos (ni singleton, que viviría para siempre y podría filtrar conexiones muertas; ni transient, que crearía una instancia nueva en cada inyección sin necesidad).

### 4. Cómo se usa desde un servicio (ejemplo, para cuando implementes `UsuarioService`)

```csharp
using Microsoft.Data.SqlClient;
using WebApi.Implementation.Conexion;
using WebApi.Interface;
using WebApi.Models;

namespace WebApi.Implementation;

public class UsuarioService : IUsuarioService
{
    private readonly ConexionBD _conexionBD;

    public UsuarioService(ConexionBD conexionBD)
    {
        _conexionBD = conexionBD;
    }

    public async Task<Usuario?> ObtenerPorTelefono(string telefono)
    {
        using var conexion = _conexionBD.CrearConexion();
        using var comando = new SqlCommand(
            "SELECT Id, Nombre, Telefono, PinHash, PinSalt, FechaRegistro, Activo " +
            "FROM Usuarios WHERE Telefono = @Telefono", conexion);
        comando.Parameters.AddWithValue("@Telefono", telefono);

        await conexion.OpenAsync();
        using var lector = await comando.ExecuteReaderAsync();

        if (await lector.ReadAsync())
        {
            return new Usuario
            {
                Id = lector.GetInt32(0),
                Nombre = lector.GetString(1),
                Telefono = lector.GetString(2),
                PinHash = lector.GetString(3),
                PinSalt = lector.GetString(4),
                FechaRegistro = lector.GetDateTime(5),
                Activo = lector.GetBoolean(6)
            };
        }
        return null;
    }
}
```

**Explicación:**
- `using var conexion = _conexionBD.CrearConexion();` — el `using` asegura que la conexión se cierre (y vuelva al pool) automáticamente al salir del método, incluso si hay una excepción.
- `new SqlCommand("...", conexion)` — el comando SQL, atado a esa conexión específica.
- `comando.Parameters.AddWithValue("@Telefono", telefono)` — **esto es obligatorio, no opcional**: usar parámetros en vez de concatenar el string evita inyección SQL. Nunca hagas `"...WHERE Telefono = '" + telefono + "'"`.
- `await conexion.OpenAsync();` — abre la conexión física a SQL Server, de forma asíncrona para no bloquear el hilo mientras espera.
- `ExecuteReaderAsync()` — ejecuta el `SELECT` y devuelve un cursor (`SqlDataReader`) para ir leyendo filas.
- `lector.GetInt32(0)`, `GetString(1)`, etc. — leen columna por columna **en el orden exacto** en que las pediste en el `SELECT`. Si cambiás el orden del `SELECT` sin actualizar los índices, te vas a traer datos cruzados sin que compile te avise — es el riesgo principal de ADO.NET crudo, tené cuidado acá.

## Checklist

- [ ] `appsettings.Development.json` tiene la cadena de conexión y corre `dotnet build` sin errores.
- [ ] `ConexionBD.cs` compila dentro de `WebApi.Implementation`.
- [ ] `Program.cs` registra `ConexionBD` como scoped.
- [ ] Podés levantar la API (`dotnet run` en `WebApi/`) sin que truene al arrancar.
