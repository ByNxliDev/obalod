# Guía de scripting de Obalod Studio

Los scripts de Studio se escriben en C# y se compilan al iniciar **Play** con
el ScriptHost de .NET. El runtime actual es local: ejecuta los `LocalScript`
habilitados como código .NET en la máquina donde corre Studio.

## Crear y ejecutar un script

1. En el Explorer, selecciona un objeto compatible o la carpeta de scripts del
   jugador y elige **Insert LocalScript**.
2. Abre el `LocalScript`, escribe una clase pública concreta que herede de
   `PlayerScript` e implemente `OnUpdate`.
3. Confirma que **Enabled** esté activado y presiona **Play**.
4. Consulta `Console.WriteLine(...)` y los errores de compilación/ejecución en
   **Output**.

Todos los `LocalScript` habilitados con código se compilan juntos; cada clase
de script debe tener un nombre único. Se ejecuta `OnStart` en la primera
actualización, luego `OnUpdate` en cada actualización y `OnStop` al terminar
el runtime. El contexto del jugador y del mundo se actualiza entre
actualizaciones.

El host agrega automáticamente los `using` comunes de .NET y
`Obalod.Studio.PlayerServices`; para usar `PlayerScript` o `Color3`, importa
`Obalod.Studio.Scripting`. No hace falta declarar un namespace:

```csharp
using Obalod.Studio.Scripting;

public sealed class MiScript : PlayerScript
{
    public override void OnUpdate(PlayerScriptContext context)
    {
        Console.WriteLine($"Tiempo de actualización: {context.DeltaSeconds}");
    }
}
```

Los errores de compilación o ejecución se muestran en Output y detienen el
Play Test. Actualmente `ServerScript` no se ejecuta.

## Ciclo de vida y contexto

`PlayerScript` tiene tres puntos de entrada:

```csharp
public virtual void OnStart(PlayerScriptContext context) { }
public abstract void OnUpdate(PlayerScriptContext context);
public virtual void OnStop(PlayerScriptContext context) { }
```

`PlayerScriptContext` expone:

| Miembro | Descripción |
| --- | --- |
| `DeltaSeconds` | Tiempo transcurrido usado por esta actualización. |
| `Input` | Entrada del jugador: `CameraYawDegrees`, `WalkSpeed`, `Forward`, `Backward`, `Left`, `Right`, `Jump` y `Sprint`. |
| `Character` | Estado del personaje para esta actualización. |
| `Workspace` | Vista del mundo, equivalente a `workspace`. |
| `GlobalLight` | Servicio de iluminación global de esta actualización. |
| `TouchedParts` | Nombres de las partes que están tocando al personaje. |
| `Command` | Comando de movimiento: `MoveX`, `MoveZ`, `WalkSpeed` y `Jump`. |

## Referencias a Player y al personaje

Puedes usar la fachada estática `Player` o su alias en minúscula `player`.
Ambos ofrecen:

- `Health`, `MaxHealth`, `Name` y `Speed` (también existen `health`,
  `maxHealth` y `speed` en minúsculas).
- `X`, `Y`, `Z`, `IsGrounded`, `IsSprinting` y `Character`.
- `Touch`, un objeto dinámico que permite consultar contacto por nombre.
- `TakeDamage(amount)`, `Heal(amount)`, `Kill()` y `Jump()`.

La salud está limitada a `0..MaxHealth`; `MaxHealth` a `1..1,000,000` y
`Speed` a `0..100`. `Jump()` solicita saltar si el personaje está en el suelo.
`X`, `Y` y `Z` son la posición observada, no una orden para teletransportar.
Para controlar el movimiento, usa `context.Command`.

`Player.Touch.NombreDeParte` consulta el contacto por nombre, sin distinguir
mayúsculas/minúsculas. También puedes llamar
`Player.Touch.IsTouching("NombreDeParte")`. El contacto se calcula con las
cajas alineadas a los ejes del personaje y de la parte; solo se consideran
partes con `CanTouch` habilitado.

```csharp
using Obalod.Studio.Scripting;

public sealed class Hazard : PlayerScript
{
    public override void OnUpdate(PlayerScriptContext context)
    {
        if (Player.Touch.Lava)
            Player.TakeDamage(10.0f * context.DeltaSeconds);
    }
}
```

## Referencias a partes del mundo

`workspace` y `context.Workspace` son vistas del mismo servicio. La búsqueda
por nombre no distingue mayúsculas/minúsculas:

```csharp
var part = workspace.Find("MovingPlatform");
var samePart = context.Workspace.Find("MovingPlatform");
var allParts = workspace.GetParts();
var dynamicReference = workspace.MovingPlatform;
```

`Find` y el acceso dinámico devuelven `null` si no hay una parte con ese
nombre. Si existen nombres duplicados, la búsqueda devuelve la primera parte
recibida del mundo. `GetParts()` devuelve la colección de partes disponibles.
Actualmente la vista incluye objetos `Part` y `Spawn`, no objetos del
personaje, carpetas ni scripts.

Cada `WorkspacePart` expone las propiedades siguientes. Las asignaciones se
envían al motor y se aplican durante Play:

| Propiedad | Tipo | Notas |
| --- | --- | --- |
| `Id`, `Name` | `string` | `Id` es de solo lectura; `Name` se puede cambiar. |
| `X`, `Y`, `Z` | `float` | Posición por eje. |
| `Position` | `Vector3` | Lee o asigna `X`, `Y`, `Z` en conjunto. |
| `Rotation` | `Vector3` | Rotación por eje, en grados. |
| `Size` | `Vector3` | Tamaño; cada dimensión debe ser mayor que cero. |
| `Color` | `Color3` | Color de la parte. |
| `Transparency` | `float` | Rango `0..1`. |
| `Mass` | `float` | Rango `0.001..1,000,000`. |
| `CanCollide`, `CanTouch`, `Anchored`, `WeldToParent` | `bool` | Ajustes de física y contacto. |
| `IsTouched` | `bool` | Indica si el personaje toca esta parte en esta actualización. |

Ejemplo: cambiar una pieza a un color aleatorio al empezar cada contacto:

```csharp
using Obalod.Studio.Scripting;

public sealed class RandomColorOnTouch : PlayerScript
{
    private bool wasTouching;

    public override void OnUpdate(PlayerScriptContext context)
    {
        var part = workspace.Find("Part1");
        if (part is null)
            return;

        if (part.IsTouched && !wasTouching)
            part.Color = Color3.FromHex($"#{Random.Shared.Next(0x1000000):X6}");

        wasTouching = part.IsTouched;
    }
}
```

Las propiedades se leen desde la instantánea de la actualización actual y los
cambios se aplican en el siguiente ciclo del motor. Al detener Play, las
modificaciones del mundo se descartan; no se guardan en el proyecto.
Actualmente los scripts no pueden crear ni eliminar objetos.

## Vector3 y colores

`Vector3` contiene `X`, `Y` y `Z` de tipo `float`:

```csharp
part.Position = new Vector3(5.0f, 2.0f, -3.0f);
part.Size = new Vector3(4.0f, 1.0f, 4.0f);
```

`Color3` contiene canales normalizados entre `0` y `1`. Para generar colores
es más sencillo usar `FromRGB` con canales enteros `0..255` o `FromHex` con
seis dígitos hexadecimales, con `#` opcional:

```csharp
part.Color = Color3.FromRGB(255, 32, 96);
part.Color = Color3.FromHex("#FF2060");
```

Por ejemplo, `Color3.FromHex("11AACC")` también es válido. `FromHex` requiere
exactamente seis dígitos; lanza `FormatException` para un valor no válido.

## Controlar GlobalLight

El servicio de iluminación está disponible mediante la fachada estática
`GlobalLight` o como `context.GlobalLight`. Ambas exponen estas propiedades
de lectura y escritura:

| Propiedad | Tipo | Valores permitidos |
| --- | --- | --- |
| `LightStyle` | `string` | `"Legacy"` o `"Advanced"`. |
| `Saturation` | `float` | `0..2`. |
| `Brightness` | `float` | `0..4`. |
| `GlobalShadows` | `bool` | Activar/desactivar sombras globales. |
| `TimeOfDay` | `float` | `0..23.99`. |
| `DayNightCycle` | `bool` | Activar/desactivar el ciclo día/noche. |
| `CycleDurationMinutes` | `float` | `1..1440` minutos por ciclo. |
| `SkyEffect` | `string` | `"Clear"`, `"Soft"`, `"Hazy"`, `"Storm"` o `"Sunset"`. |
| `FogEnabled` | `bool` | Activar/desactivar la niebla. |
| `FogColor` | `Color3` | Color de la niebla. |
| `FogStart` | `float` | Distancia inicial, `0..100000`. |
| `FogEnd` | `float` | Distancia final, `1..100000`, debe ser mayor que `FogStart`. |

Ejemplo:

```csharp
using Obalod.Studio.Scripting;

public sealed class SunsetLighting : PlayerScript
{
    public override void OnStart(PlayerScriptContext context)
    {
        GlobalLight.LightStyle = "Advanced";
        GlobalLight.Saturation = 0.8f;
        GlobalLight.Brightness = 1.4f;
        GlobalLight.GlobalShadows = true;
        GlobalLight.TimeOfDay = 18.5f;
        GlobalLight.DayNightCycle = false;
        GlobalLight.SkyEffect = "Sunset";
        GlobalLight.FogEnabled = true;
        GlobalLight.FogColor = Color3.FromHex("#F0A070");
        GlobalLight.FogStart = 30.0f;
        GlobalLight.FogEnd = 900.0f;
    }

    public override void OnUpdate(PlayerScriptContext context) { }
}
```

Los cambios se aplican al render durante Play y se restauran al detenerlo; no
se guardan como cambios permanentes del proyecto. Los valores deben respetar
los rangos anteriores y `FogEnd` debe ser mayor que `FogStart`. Las seis
direcciones `skyboxUrls` de la configuración no están expuestas a scripts.

## Movimiento y scripts combinados

`context.Command` permite devolver el movimiento calculado para el ciclo
actual. `MoveX` y `MoveZ` deben estar entre `-1` y `1`; `WalkSpeed` entre
`0` y `100`. Por ejemplo, un script de movimiento puede basarse en la entrada:

```csharp
using Obalod.Studio.Scripting;

public sealed class SimpleMovement : PlayerScript
{
    public override void OnUpdate(PlayerScriptContext context)
    {
        context.Command.MoveX =
            (context.Input.Right ? 1.0f : 0.0f)
            - (context.Input.Left ? 1.0f : 0.0f);
        context.Command.MoveZ =
            (context.Input.Backward ? 1.0f : 0.0f)
            - (context.Input.Forward ? 1.0f : 0.0f);
        context.Command.WalkSpeed = 10.0f;
        context.Command.Jump = context.Input.Jump && context.Character.Grounded;
    }
}
```

Las clases cargadas se ejecutan dentro del mismo runtime y comparten el
contexto de esa actualización. El script de movimiento predeterminado también
es un `LocalScript`; si escribes uno propio, deshabilita el predeterminado
para evitar que ambos escriban el comando de movimiento.

## Límites y seguridad

- Los scripts se ejecutan localmente con los permisos del usuario de Studio.
  C# no está aislado en un sandbox de seguridad: ejecuta solo scripts de
  confianza.
- `ServerScript`, replicación multijugador y ejecución independiente de
  scripts del mundo aún no están implementados. Un script bajo un objeto del
  mundo sigue compartiendo el ciclo de vida del jugador.
- El runtime ofrece instantáneas de partes y permite modificar las
  propiedades documentadas, pero no crear/eliminar objetos ni acceder a una
  API de red o a un sistema de eventos de contacto.
- Los scripts se actualizan junto con el ciclo del motor; evita trabajo
  prolongado en `OnUpdate`.
