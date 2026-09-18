# HeroesWeb — Scaffolding Inverso (Database First)

Aplicación ASP.NET Core Razor Pages que demuestra el flujo **Database First** con Entity Framework Core: la base de datos se crea primero en SQL Server, y el código C# se genera desde ella con `dotnet ef dbcontext scaffold` + `dotnet aspnet-codegenerator`.

## Requisitos

- [.NET 8 SDK](https://dotnet.microsoft.com/download)
- SQL Server o SQL Server Express (`.\SQLEXPRESS`)
- SQL Server Management Studio (opcional, para crear la BD)

## 1 — Crear la base de datos

Ejecuta el siguiente script en SQL Server Management Studio (o `sqlcmd`):

```sql
CREATE DATABASE HeroesDb;
GO

USE HeroesDb;
GO

CREATE TABLE Heroes (
    Id               INT IDENTITY(1,1) PRIMARY KEY,
    Nombre           NVARCHAR(100) NOT NULL,
    Ciudad           NVARCHAR(100) NOT NULL,
    IdentidadSecreta NVARCHAR(100) NULL
);

CREATE TABLE SuperPoderes (
    Id          INT IDENTITY(1,1) PRIMARY KEY,
    Nombre      NVARCHAR(100) NOT NULL,
    Descripcion NVARCHAR(250) NULL,
    HeroeId     INT NOT NULL,
    CONSTRAINT FK_SuperPoderes_Heroes
        FOREIGN KEY (HeroeId) REFERENCES Heroes(Id)
        ON DELETE CASCADE
);

-- Datos de ejemplo
INSERT INTO Heroes (Nombre, Ciudad, IdentidadSecreta) VALUES
('Flash',   'Central City', 'Barry Allen'),
('Batman',  'Gotham City',  'Bruce Wayne'),
('Superman','Metrópolis',   'Clark Kent');

INSERT INTO SuperPoderes (Nombre, Descripcion, HeroeId) VALUES
('Super velocidad', 'Corre más rápido que la luz', 1),
('Viaje en el tiempo', 'Puede viajar al pasado y futuro', 1),
('Batarang', 'Arma de precisión', 2),
('Vuelo', 'Puede volar a gran altitud', 3);
```

## 2 — Clonar y configurar

```bash
git clone https://github.com/F4SG/Scaffolding-inverso.git
cd Scaffolding-inverso
```

Verifica que la cadena de conexión en `appsettings.json` apunte a tu instancia:

```json
"ConnectionStrings": {
  "HeroesDb": "Server=localhost\\SQLEXPRESS;Database=HeroesDb;Trusted_Connection=True;TrustServerCertificate=True;"
}
```

Cambia `localhost\\SQLEXPRESS` por el nombre de tu servidor si es diferente.

## 3 — Ejecutar

```bash
dotnet run
```

Abre el navegador en `https://localhost:xxxx` (el puerto se muestra en la consola).

## Estructura del proyecto

```
HeroesWeb/
├── Data/
│   └── HeroesContext.cs          # DbContext generado con scaffold
├── Models/
│   ├── Heroes.cs                 # Modelo generado con scaffold
│   └── SuperPoderes.cs           # Modelo generado con scaffold
├── Pages/
│   ├── Heroes/                   # CRUD de Héroes (generado con codegenerator)
│   └── SuperPoderes/             # CRUD de SuperPoderes (generado con codegenerator)
├── appsettings.json
└── Program.cs
```

## Comandos usados (para referencia)

```bash
# 1. Generar modelos y DbContext desde la BD existente
dotnet ef dbcontext scaffold \
  "Server=localhost\SQLEXPRESS;Database=HeroesDb;Trusted_Connection=True;TrustServerCertificate=True;" \
  Microsoft.EntityFrameworkCore.SqlServer \
  --output-dir Models \
  --context-dir Data \
  --context HeroesContext \
  --data-annotations --force

# 2. Generar páginas CRUD para cada entidad
dotnet aspnet-codegenerator razorpage \
  -m Heroes -dc HeroesContext \
  --relativeFolderPath Pages/Heroes \
  --referenceScriptLibraries --force

dotnet aspnet-codegenerator razorpage \
  -m SuperPoderes -dc HeroesContext \
  --relativeFolderPath Pages/SuperPoderes \
  --referenceScriptLibraries --force
```

---

## 🐛 Correcciones de errores (16/09/2026)

Durante la sesión de depuración se identificaron y corrigieron **4 errores** en [`Program.cs`](Program.cs) y [`Data/HeroesContext.cs`](Data/HeroesContext.cs) que impedían que la aplicación arrancara correctamente.

---

### ❌ Error 1 — Servicios registrados después de `builder.Build()`

**Archivo:** `Program.cs`  
**Causa:** En ASP.NET Core, el contenedor de dependencias se construye cuando se llama a `builder.Build()`. Cualquier servicio registrado **después** de esa llamada es ignorado silenciosamente, lo que provoca errores en tiempo de ejecución.

El código original hacía:

```csharp
// ❌ INCORRECTO
var app = builder.Build();               // ← contenedor ya construido

builder.Services.AddDefaultIdentity<IdentityUser>(...);   // ← ignorado
builder.Services.AddRazorPages(options => { ... });        // ← ignorado
```

**Corrección:** todos los servicios se movieron antes de `builder.Build()`:

```csharp
// ✅ CORRECTO
builder.Services.AddDefaultIdentity<ApplicationUser>(...).AddEntityFrameworkStores<HeroesContext>();
builder.Services.AddRazorPages(options =>
{
    options.Conventions.AuthorizeFolder("/Heroes");
});

var app = builder.Build();  // ← siempre al final del registro de servicios
```

---

### ❌ Error 2 — `AddRazorPages()` duplicado sin configuración

**Archivo:** `Program.cs`  
**Causa:** Había una primera llamada a `builder.Services.AddRazorPages()` sin opciones, y luego una segunda (después de `Build()`) con la convención de autorización `AuthorizeFolder("/Heroes")`. Al estar la segunda después de `Build()`, la autorización nunca se aplicaba.

**Corrección:** se eliminó la primera llamada sin opciones y se consolidó todo en una sola llamada correctamente posicionada:

```csharp
// ✅ Una única llamada, antes de Build(), con la autorización configurada
builder.Services.AddRazorPages(options =>
{
    options.Conventions.AuthorizeFolder("/Heroes");
});
```

---

### ❌ Error 3 — Faltaba `app.UseAuthentication()` en el pipeline

**Archivo:** `Program.cs`  
**Causa:** ASP.NET Core Identity requiere que `UseAuthentication()` esté en el pipeline HTTP **antes** de `UseAuthorization()`. Sin ello, aunque el usuario inicie sesión, nunca sería reconocido como autenticado y todas las rutas protegidas lo redirigirían al login indefinidamente.

```csharp
// ❌ Antes — solo UseAuthorization sin UseAuthentication
app.UseAuthorization();
```

**Corrección:**

```csharp
// ✅ Orden correcto
app.UseAuthentication();   // ← añadido
app.UseAuthorization();
```

---

### ❌ Error 4 — Inconsistencia de tipo de usuario en Identity (`IdentityUser` vs `ApplicationUser`)

**Archivos:** `Program.cs` y `Data/HeroesContext.cs`  
**Causa:** Las páginas de Identity scaffoldeadas (Login, Register, Manage, etc.) inyectan servicios genéricos tipados con `ApplicationUser` (la clase definida en `Data/ApplicationUser.cs`). Sin embargo, los registros de servicios y el `DbContext` usaban el tipo base `IdentityUser`. Esto causaba el error en tiempo de ejecución:

```
InvalidOperationException: Unable to resolve service for type
'Microsoft.AspNetCore.Identity.SignInManager`1[HeroesWeb.Data.ApplicationUser]'
```

**Corrección en `Program.cs`:**

```csharp
// ❌ Antes
builder.Services.AddDefaultIdentity<IdentityUser>(...)

// ✅ Después
builder.Services.AddDefaultIdentity<ApplicationUser>(...)
```

**Corrección en `HeroesContext.cs`:**

```csharp
// ❌ Antes
public partial class HeroesContext : IdentityDbContext<IdentityUser>

// ✅ Después
public partial class HeroesContext : IdentityDbContext<ApplicationUser>
```

---

### Resultado final

| # | Error | Archivo | Estado |
|---|-------|---------|--------|
| 1 | Servicios registrados después de `Build()` | `Program.cs` | ✅ Corregido |
| 2 | `AddRazorPages()` duplicado sin configuración | `Program.cs` | ✅ Corregido |
| 3 | Faltaba `UseAuthentication()` en el pipeline | `Program.cs` | ✅ Corregido |
| 4 | Tipo de usuario incorrecto (`IdentityUser` vs `ApplicationUser`) | `Program.cs` + `HeroesContext.cs` | ✅ Corregido |

```
dotnet build → Compilación correcta. 0 Errores. 7 Advertencias.
```
