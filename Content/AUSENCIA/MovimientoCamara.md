# Movimiento y cámara — revisión del 11 de noviembre de 2026

## Estado de entrega

**Implementación parcial.** Se han ajustado y guardado los parámetros indicados abajo mediante la API Python del editor UE 5.6.1. La integración por velocidad real descrita en esta guía está pendiente de realizar en Blueprint. No se ha probado en Play In Editor.

## Diagnóstico comprobado en el proyecto

- Personaje: `/Game/FirstPerson/Blueprints/BP_FirstPersonCharacter`.
- Enhanced Input: `IA_Move.Triggered → Move`; X controla desplazamiento lateral e Y avance. `Move` utiliza `Add Movement Input` con los vectores Right/Forward del actor.
- `LeftShift.Pressed → Set MaxWalkSpeed 350`; `Released → Set MaxWalkSpeed 200`. El valor inicial era 150: caminar cambiaba de velocidad después de usar Shift.
- Ratón: `IA_MouseLook.Triggered → Aim`, con ambos ejes multiplicados por 0,3. Se conserva. Hay nodos antiguos desconectados; no se han eliminado.
- Cámara: FOV 75°, `Use Pawn Control Rotation` activado; el actor usa la rotación Yaw del controlador. Se conserva la jerarquía y el transform existente de la cámara.
- `FirstPersonMesh.Hidden in Game=True`. La malla heredada tiene otro valor; no se han cambiado las mallas.
- Los eventos de salto y llamadas Jump/StopJumping inspeccionados están desconectados. No se han reconectado.
- `IA_Move.Started → Client Start Camera Shake(CS_Pasos, Scale=1, CameraLocal)`.
- `IA_Move.Completed → Stop All Instances Of Camera Shake(CS_Pasos, Immediately=True)`.
- `CS_Pasos` deriva de CameraShakeBase y usa WaveOscillatorCameraShakePattern con Duration=1 s. Esto explica que la oscilación termine aunque se mantenga el movimiento: Started no vuelve a dispararse cada segundo.
- El sistema actual depende de teclas, no de velocidad; tampoco distingue caminar de caminar rápido. La parada inmediata impide usar el Blend Out.

## Cambios ya aplicados

En `BP_FirstPersonCharacter → Character Movement`:

| Parámetro | Antes | Ahora | Finalidad |
|---|---:|---:|---|
| Max Walk Speed | 150 | 200 cm/s | Coincidir con el valor actual al soltar Shift |
| Max Acceleration | 2048 | 650 cm/s² | Arranque con peso y respuesta controlada |
| Braking Deceleration Walking | 2048 | 850 cm/s² | Frenada corta, perceptible |
| Ground Friction | 8 | 5 | Moderar correcciones de dirección |
| Use Separate Braking Friction | False | True | Separar giro y frenada |
| Braking Friction | 0 | 0 | Frenar mediante desaceleración constante |
| Braking Friction Factor | 2 | 1 | Multiplicador explícito, sin efecto mientras la fricción sea cero |

En `CS_Pasos → Class Defaults`:

| Parámetro | Antes | Ahora |
|---|---:|---:|
| Single Instance | False | True |
| Root Shake Pattern / Duration | 1 | 0 (indefinido) |
| Blend In Time | 0,2 | 0,25 s |
| Blend Out Time | 0,2 | 0,30 s |
| X Amplitude / Frequency | 1 / 1 | 0 / 0,8 |
| Y Amplitude / Frequency | 1 / 1 | 0,25 cm / 0,8 Hz |
| Z Amplitude / Frequency | 1,5 / 2 | 0,65 cm / 1,6 Hz |
| Initial Offset Type, X/Y/Z | Random | Zero |

Rotation Amplitude Multiplier sigue en 0 y FOV Amplitude en 0. Se evita añadir balanceo angular antes de probar comodidad. Los valores son un punto de partida propio, no valores extraídos de P.T. o LUTO.

**Limitaciones actuales:** Shift todavía pone 350; la parada sigue siendo inmediata; contra una pared sigue habiendo pasos. Cambiar Duration no resuelve estas tres cuestiones. Las conexiones siguientes completan la solución.

## Integración pendiente: reutilizar CS_Pasos con un componente

Elegimos un único Camera Shake continuo, controlado por velocidad real. No añadir otro head bob que modifique el transform de cámara: eso duplicaría oscilaciones. WaveOscillator mantiene su fase al cambiar el multiplicador de frecuencia, y el Camera Manager aplica un desplazamiento temporal sin acumularlo sobre el transform del componente.

### 1. Crear el componente y sus variables

En `/Game/AUSENCIA/Blueprints/Characters`, crear Blueprint Class → Actor Component, nombre `BPC_MovimientoCamara`. Activar Start with Tick Enabled, Tick Interval=0. Añadirlo a `BP_FirstPersonCharacter`. No cambiar GameMode, personaje activo ni mapas.

Variables públicas, Instance Editable, categoría `AUSENCIA|Movimiento`:

| Variable | Tipo | Inicial |
|---|---|---:|
| WalkSpeed | Float | 200 |
| FastWalkSpeed | Float | 260 |
| SpeedInterp | Float | 6 |

Categoría `AUSENCIA|Camara`:

| Variable | Tipo | Inicial |
|---|---|---:|
| BobIntensity | Float | 1 |
| BobBlendSpeed | Float | 8 |
| FrequencyInterp | Float | 6 |
| StopThreshold | Float | 3 |

Variables privadas de estado:

| Variable | Tipo | Inicial |
|---|---|---|
| OwnerCharacter | Character Object Reference | None |
| Movement | Character Movement Component Object Reference | None |
| Controller | Player Controller Object Reference | None |
| CameraManager | Player Camera Manager Object Reference | None |
| PasosInstance | Camera Shake Base Object Reference | None |
| PasosPattern | Wave Oscillator Camera Shake Pattern Object Reference | None |
| FastRequested | Boolean | False |
| CurrentMaxSpeed | Float | 200 |
| CurrentScale | Float | 0 |
| CurrentFrequency | Float | 0 |
| Initialized | Boolean | False |

### 2. Inicializar una única instancia

Crear función `TryInitialize`, sin argumentos. Llamarla desde BeginPlay y desde Tick mientras Initialized=False. Esto permite esperar a que exista posesión; no usar Delay fijo.

Conexiones blancas en este orden:

1. Entrada → Branch(Initialized). True termina; False → `Get Owner → Cast To Character` → Set OwnerCharacter.
2. `OwnerCharacter → Is Locally Controlled` a otro Branch. False termina. True → `OwnerCharacter.Get Controller → Cast To PlayerController` → Set Controller.
3. Desde Controller obtener Player Camera Manager → `Is Valid` → Set CameraManager. Invalid termina y permite reintentar en Tick.
4. `OwnerCharacter.Get Character Movement` → Set Movement. Set CurrentMaxSpeed=WalkSpeed; `Set Max Walk Speed` con Target=Movement y valor CurrentMaxSpeed.
5. `Start Camera Shake` **del CameraManager**, no Client Start Camera Shake. Target=CameraManager, Shake Class=CS_Pasos, Scale=1, Play Space=Camera Local. Return Value → Set PasosInstance.
6. Comprobar Is Valid(PasosInstance). Desde la instancia → `Get Root Shake Pattern` → `Cast To WaveOscillatorCameraShakePattern` → Set PasosPattern.
7. `Set Shake Scale` con Target=PasosInstance, valor 0 → Set CurrentScale=0 → Set CurrentFrequency=0 → Set Location Frequency Multiplier(Target=PasosPattern, valor 0) → Set Initialized=True.

Si falla el cast del patrón, detener PasosInstance con Stop Camera Shake(Immediately=True), dejar Initialized=False y revisar que CS_Pasos siga usando WaveOscillator. No dejar instancias huérfanas.

### 3. Conectar Shift sin dos escritores de velocidad

En el Event Graph del personaje, conservar el evento Left Shift conectado actualmente. Desconectar sus dos cables a Set Max Walk Speed; conservar esos nodos desconectados hasta verificar la migración.

- Left Shift.Pressed → Set FastRequested=True; Target: referencia al componente BPC_MovimientoCamara.
- Left Shift.Released → Set FastRequested=False; mismo Target.

Hacer FastRequested accesible desde el personaje (desmarcar Private) o crear función pública `SetFastRequested(Requested: Boolean)` que asigne la variable. Usar una de esas alternativas, no ambas.

En Tick del componente, si `Controller.Is Input Key Down(Left Shift)` es False, asignar FastRequested=False antes de calcular la velocidad. Esto evita conservar el estado rápido tras perder el foco. Mantener los nodos existentes de IA_Move y Aim sin cambios.

### 4. Actualización de velocidad y cámara

Event Tick proporciona Delta Seconds. Usar Sequence si ya hay otra lógica. Si Initialized=False, llamar TryInitialize; continuar únicamente si termina inicializado. Comprobar referencias válidas antes de usarlas.

Crear función `UpdateMovementCamera(DeltaSeconds: Float)` y ejecutar las operaciones siguientes en el orden indicado; cada asignación es un nodo Set conectado por ejecución a la siguiente. Las fórmulas se construyen con nodos Float, no con expresiones de texto pegadas.

**Velocidad máxima:**

1. `Select Float`: A=FastWalkSpeed, B=WalkSpeed, Pick A=FastRequested → TargetSpeed.
2. `FInterp To`: Current=CurrentMaxSpeed, Target=TargetSpeed, Delta Time=DeltaSeconds, Interp Speed=SpeedInterp → Set CurrentMaxSpeed.
3. `Set Max Walk Speed`: Target=Movement, valor=CurrentMaxSpeed.

La aceleración y frenada físicas siguen perteneciendo a Character Movement; no interpolar manualmente la posición ni la velocidad del actor.

**Estado real:**

4. `OwnerCharacter.Get Velocity → Vector Length XY` → Speed (Float local).
5. `Movement.Is Moving On Ground AND Speed > StopThreshold AND OwnerCharacter.Is Locally Controlled` → Moving (Boolean local).
6. `Map Range Clamped`: Value=Speed, In Range A=WalkSpeed, B=FastWalkSpeed, Out A=0, B=1 → FastAlpha.
7. `Speed / Max(WalkSpeed,1) → Clamp(0,1)` → SpeedWeight.
8. `Lerp(Float)`: A=1, B=1,15, Alpha=FastAlpha. Multiplicar resultado por SpeedWeight y por Clamp(BobIntensity,0,2).
9. `Select Float`: A=resultado anterior, B=0, Pick A=Moving → TargetScale.
10. `FInterp To`: Current=CurrentScale, Target=TargetScale, Delta Time=DeltaSeconds, Interp Speed=BobBlendSpeed → Set CurrentScale.
11. Branch: `TargetScale == 0 AND CurrentScale < 0,001`. True → Set CurrentScale=0; False mantiene el valor. Volver a la misma continuación de la función para ambas ramas.
12. `Set Shake Scale`: Target=PasosInstance, valor=CurrentScale.

**Ritmo:**

13. `Speed / Max(WalkSpeed,1) → Clamp(0,1,3)` → Ratio. En el nodo Clamp, Min=0 y Max=1,3.
14. Select Float: A=Ratio, B=0, Pick A=Moving → TargetFrequency.
15. FInterp To: Current=CurrentFrequency, Target=TargetFrequency, Delta Time=DeltaSeconds, Interp Speed=FrequencyInterp → Set CurrentFrequency.
16. `Set Location Frequency Multiplier`: Target=PasosPattern, valor=CurrentFrequency.

No reiniciar el shake en Tick. La misma instancia permanece activa con escala cero en reposo. A 200 cm/s el eje vertical tiene 1,6 Hz; a 260 cm/s alcanza 2,08 Hz. Contra una pared Speed cae y TargetScale pasa a cero. Al caer también se desvanece. El umbral elimina vibraciones por velocidades residuales; el ajuste explícito a cero elimina oscilación residual permanente.

Para una pausa explícita o cambio de posesión, limpiar FastRequested y desvanecer la escala. Si el componente pierde su controlador local, detener su instancia y limpiar Initialized y las referencias; la siguiente posesión inicializará una nueva. En este juego de un jugador no hay que crear shakes para personajes remotos.

### 5. Desconectar el disparo antiguo y limpiar al terminar

En `BP_FirstPersonCharacter → Event Graph`, desconectar solamente:

- IA_Move.Started → Client Start Camera Shake(CS_Pasos).
- IA_Move.Completed → Stop All Instances Of Camera Shake(CS_Pasos).

Conservar **IA_Move.Triggered → Move** y ambos ejes. La instancia pertenece ahora al componente. No utilizar Stop All Camera Shakes, pues afectaría a futuros efectos ajenos a pasos.

En `BPC_MovimientoCamara → Event EndPlay`: Is Valid(CameraManager) → Is Valid(PasosInstance) → Stop Camera Shake(Target=CameraManager, Shake=PasosInstance, Immediately=True). Limpiar referencias. Aquí la parada inmediata corresponde a la destrucción del personaje; la parada normal ya se suaviza con CurrentScale.

Si solo se desea una mejora provisional antes de crear el componente, desmarcar Immediately en el Stop All Instances existente permite usar el Blend Out aplicado. Esa mejora provisional sigue dependiendo del input y no resuelve la pared ni el ritmo con Shift.

### 6. FOV, ratón y comodidad

Conservar inicialmente FOV=75 y los multiplicadores del ratón=0,3. Promover ambos valores 0,3 a una única variable Float `MouseSensitivity`, categoría `AUSENCIA|Camara`, y conectar su Get al pin B de ambos Multiply anteriores a Aim. No multiplicar los deltas del ratón por Delta Seconds.

No añadir FOV dinámico ni sway angular en esta iteración: la cámara ya tendrá peso mediante desplazamiento sutil y frenada. Probar FOV entre 75 y 85 según pantalla y distancia, y sensibilidad alrededor del valor actual según DPI. No cambiar globalmente DefaultInput.ini para compensar una sensibilidad que ya se escala en Blueprint. BobIntensity=0 debe permitir desactivar el efecto sin alterar el movimiento.

## Comprobación manual en PIE

1. Compilar componente, personaje y CS_Pasos; guardar solo esos recursos. Comprobar que no hay errores de compilación ni Accessed None.
2. Iniciar PIE en el mapa habitual. Estar quieto 15 s: cámara estable, sin brazos; Space y entrada de salto no producen salto.
3. Caminar 20 s: pasos continuos más allá del primer segundo. Soltar: frenada breve y vuelta suave al reposo, sin desplazamiento acumulado.
4. Mantener W y pulsar/soltar Shift varias veces: transición 200↔260, sin salto brusco, frecuencia e intensidad progresivas. Antes de completar esta guía, Shift sigue dando 350.
5. Mantener W contra una pared 5 s: la oscilación debe desaparecer. Deslizarse lateralmente contra ella: responder a la velocidad real.
6. Probar diagonales, marcha atrás y cambios de dirección; comprobar que la diagonal no supera el límite. Caer desde un escalón: pasos se desvanecen mientras no hay suelo.
7. Perder y recuperar foco manteniendo Shift; al soltar debe volver a caminar. Repetir inicio y fin de PIE para descartar instancias duplicadas.
8. Comparar 30, 60 y 120 FPS, y BobIntensity=0/1. Durante depuración se puede mostrar Speed, CurrentScale y CurrentFrequency; retirar esos mensajes al terminar.

## Verificación realizada y Git

Se invocó Compile Blueprint para ambos recursos y el editor confirmó su guardado. Se leyeron de nuevo y verificaron los parámetros. No se hizo una prueba jugable; la percepción, las colisiones y la integración pendiente requieren los pasos anteriores.

Rama observada: main. Había modificaciones anteriores en el personaje, configuraciones, mapa y otros recursos; CS_Pasos estaba sin seguimiento. LFS ya configura uasset/umap como lockable. La primera consulta normal de status falló por acceso denegado a `.git/lfs/tmp`; una consulta con filtro desactivado solo para ese comando puede mostrar falsos positivos en archivos LFS y no se usa como prueba de contenido modificado.

Una comparación SHA-256 de 277 archivos existentes antes y después confirmó cambios únicamente en BP_FirstPersonCharacter.uasset y CS_Pasos.uasset. No se hicieron commits, push, pull, cambios de rama ni cambios a LFS. Coordinar la edición de estos dos binarios con el equipo: no se pueden fusionar como texto.

Copias previas exactas, exportaciones y scripts de auditoría están en `Saved/AusenciaAudit/`; las copias en `before/` conservan los cambios locales previos. Son auxiliares locales, no nuevos recursos de juego. No restaurarlos sobre un editor con cambios posteriores sin revisarlos.

Nuevos documentos: `AGENTS.md` en raíz y esta guía en `Content/AUSENCIA/MovimientoCamara.md`. El componente descrito **no se ha creado todavía**.

Referencias técnicas: cabeceras y fuentes locales de UE 5.6 de CameraShakeBase y WaveOscillatorCameraShakePattern; [Camera Shakes, Epic](https://dev.epicgames.com/documentation/unreal-engine/camera-shakes-in-unreal-engine) y [FInterp To, Epic](https://dev.epicgames.com/documentation/unreal-engine/BlueprintAPI/Math/Interpolation/FInterpTo). Los valores de diseño son propuestas para AUSENCIA, no recomendaciones numéricas de esas páginas.
