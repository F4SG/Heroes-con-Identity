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

