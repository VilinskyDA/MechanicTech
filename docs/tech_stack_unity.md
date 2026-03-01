# Технический стек: Unity — архитектура и технологии

> Документ фиксирует выбор технологий, архитектурные решения и технические контракты для реализации проекта на Unity.  
> Служит ориентиром при принятии технических решений и при онбординге новых участников команды.  
> Версия: 0.1 (соответствует Slice 0–1 из roadmap GDD).

---

## 1. Обоснование выбора Unity

| Фактор | Почему Unity |
|---|---|
| Язык команды | C# (средний уровень) — нативный язык Unity, нет FFI-моста |
| Вычислительная нагрузка | Тривиальна для Mono/IL2CPP; Burst — страховка, не необходимость |
| VFX-пайплайн | Shader Graph + VFX Graph + DecalProjector — оптимально для физических эффектов |
| Профилирование | Profiler + Frame Debugger + Memory Profiler — лучший инструментарий для малой команды |
| Прецеденты жанра | Sprocket (танковый конструктор + баллистика, 1 разработчик), Besiege, From the Depths |
| Лицензия | Бесплатно до $200K/год; Runtime Fee отменена |

---

## 2. Версии и конфигурация движка

### 2.1 Версия Unity

- **Unity 6.0 LTS** (минимум) или **Unity 6.3 LTS** (при доступности).  
  LTS выбирается намеренно: стабильность важнее новых функций на ранних этапах.

### 2.2 Render Pipeline

**HDRP (High Definition Render Pipeline)** — основной выбор.

Обоснование:
- Нативная поддержка `DecalProjector` — необходима для пробоин, следов ожогов, абляции.
- Полная интеграция `VFX Graph` (GPU-частицы) — необходима для физических эффектов высокого качества.
- Физически корректный `Bloom` — для emission-шейдеров нагрева металла и энергетического оружия.
- `Volumetric Fog / Lighting` — дым, пылевые эффекты от разрушения укрытий.
- Целевая платформа — PC Windows, RTX 3060 / RX 6600; HDRP комфортно работает на этом железе.

Минимальные системные требования для разработки: GPU с поддержкой DX12 / Vulkan, 16 GB RAM.

### 2.3 Скриптинг и компиляция

| Backend | Когда используется |
|---|---|
| **Mono** | В Editor (Play Mode) — быстрая итерация |
| **IL2CPP** | Финальные билды — ~15–30% прирост производительности vs Mono |
| **Burst Compiler** | Горячие пути симуляции (см. раздел 4) |

**.NET Standard 2.1** — целевой профиль. Использование `System.Numerics` для SIMD-векторов в тех горячих путях, которые не покрыты Burst.

---

## 3. Структура проекта

```
Assets/
├── _Project/                    # Весь код проекта (нижнее подчёркивание — наверх в Project window)
│   ├── Scripts/
│   │   ├── Simulation/          # Ядро симуляции — чистый C#, без зависимостей от Unity API
│   │   │   ├── Core/            # Impact Packet, Carrier State, F_layer, H transfer
│   │   │   ├── Materials/       # MaterialDefinition, LayerDefinition, библиотека
│   │   │   ├── Tables/          # Lookup-таблицы и интерполяторы
│   │   │   ├── Validator/       # Валидатор сборки, граф деталей
│   │   │   └── Tests/           # Unit и golden-тесты (запускаются без движка)
│   │   ├── Constructor/         # Конструктор оружия/брони/снаряжения
│   │   │   ├── Graph/           # Граф деталей (Part, Component, Node, Connection)
│   │   │   ├── UI/              # MonoBehaviour и ViewModel для UI конструктора
│   │   │   ├── Gizmos/          # 3D-манипуляторы, snap, attachment highlights
│   │   │   └── Preview/         # Превью деталей, heatmap overlay
│   │   ├── Combat/              # Пошаговый бой
│   │   │   ├── Grid/            # Тайловая карта, A*, LOS
│   │   │   ├── Units/           # Бойцы, AP, ранения, состояние
│   │   │   ├── Actions/         # Очередь действий, turn manager
│   │   │   ├── AI/              # ИИ противника (отдельный документ)
│   │   │   └── Report/          # Post-battle отчёт, аналитика
│   │   ├── World/               # Окружение, барьеры, тайлы карты
│   │   └── Shared/              # Общие типы, утилиты, расширения
│   ├── Data/                    # ScriptableObject-данные
│   │   ├── Materials/           # MaterialDefinition assets
│   │   ├── Parts/               # Определения деталей
│   │   └── Configs/             # Игровые конфиги
│   ├── VFX/
│   │   ├── Graphs/              # VFX Graph (.vfx файлы)
│   │   └── Shaders/             # ShaderGraph (.shadergraph), HLSL includes
│   ├── Art/
│   │   ├── Models/
│   │   ├── Textures/
│   │   └── Materials/           # Material instances
│   └── Scenes/
│       ├── Constructor.unity
│       ├── Combat.unity
│       └── MainMenu.unity
├── Packages/                    # Package Manager манифест
└── Tests/
    ├── EditMode/                # Тесты симуляционного ядра (без запуска сцены)
    └── PlayMode/                # Интеграционные тесты
```

**Ключевой принцип:** `Simulation/` — полностью изолированный слой без зависимостей от `UnityEngine.*`. Это обеспечивает:
- Запуск всех unit/golden-тестов в Edit Mode без загрузки сцены.
- Возможность тестирования симуляции из командной строки (CI/CD).
- Теоретическую переносимость ядра на другой движок.

---

## 4. Симуляционное ядро (C# / Burst)

### 4.1 Принципы

Всё, что описано в GDD как «тяжёлые расчёты» и «lookup/интерполяция», реализуется здесь. Ядро **не знает** о Unity, MonoBehaviour, GameObject, сценах.

### 4.2 Структуры данных

Все структуры — **blittable value types** (`struct`), совместимые с `NativeArray<T>` и Burst.

```csharp
// Флаги активных каналов угроз в пакете (bitfield)
[Flags]
public enum ThreatChannels : byte
{
    None    = 0,
    Kinetic = 1 << 0,   // Кинетика/проникновение
    Thermal = 1 << 1,   // Термо-воздействие
    // EM, Blast — будущие каналы
}

// Impact Packet v0.1 — immutable record в единицах SI
// Контракт: GDD §9.2, §13.1, §13.2
// Все поля blittable — совместимо с NativeArray<T> и Burst
[StructLayout(LayoutKind.Sequential)]
public struct ImpactPacket
{
    // ── Заголовок (GDD §9.2) ──────────────────────────────────────────
    public int            SchemaVersion;      // = 1 для v0.1
    public uint           PacketId;           // Уникальный ID пакета
    public uint           SourceId;           // ID источника (оружие/носитель)
    public int            Step;               // Номер тактического хода
    public ThreatChannels ActiveChannels;     // Какие каналы содержат данные
    public uint           TargetZoneId;       // 0 = неизвестна (опц.)
    public uint           RngSeedRef;         // Seed для детерминированной стохастики

    // ── Геометрия контакта (GDD §9.2) ────────────────────────────────
    public float3         ContactPoint;       // [м]
    public float3         ContactNormal;      // Нормаль поверхности (единичный вектор)
    public float          ContactArea;        // [м²]
    public float          IncidenceAngle;     // [рад] — угол между вектором скорости и нормалью

    // ── Кинетический канал (GDD §13.1) ───────────────────────────────
    public float3         VelocityVec;        // v_vec [м/с]
    public float          EffectiveMass;      // m_eff [кг]
    public float          EquivDiameter;      // d_eq [м]
    public float          KineticEnergy;      // E [Дж]
    public float3         MomentumVec;        // p_vec [Н·с]
    public uint           ShapeId;            // Идентификатор формы носителя
    public uint           PenetratorMaterialId; // ID материала носителя (опц., 0 = не задан)
    public float          Yaw;               // [рад] опц. — отклонение носителя от оси
    public float          Spin;              // [рад/с] опц. — вращение носителя

    // ── Термо-канал (GDD §13.2) — опц. в v0.1 ────────────────────────
    // Активен только если (ActiveChannels & Thermal) != 0
    public float          HeatFlux;          // q_flux [Вт/м²]  (или q_dot [Вт] если SpotArea=0)
    public float          SpotArea;          // [м²]
    public float          ExposureDt;        // [с]
    public float          SourceTemp;        // T_source [К]
    public uint           SpectrumModeId;    // опц. — идентификатор спектра/режима излучения

    public const int SCHEMA_VERSION = 1;
}

// Carrier State — состояние носителя между контактами
// Контракт: GDD §10
// Для пули: масса/форма/спин. Для луча: мощность/расходимость.
[StructLayout(LayoutKind.Sequential)]
public struct CarrierState
{
    public uint    PropagationProfileId; // Определяет, какой оператор G применяется
    public float3  Position;             // [м]
    public float3  Direction;            // Единичный вектор направления
    public float   Speed;               // [м/с]
    public float   KineticEnergy;       // [Дж]
    public float   Mass;                // [кг]
    public float   Diameter;            // d_eq [м]
    public float   Spin;               // [рад/с] — вращение носителя (для пули)
    public float   BeamDivergence;      // [рад] — расходимость (для луча)
    public float   BeamPower;           // [Вт] — мощность (для луча)
    public uint    MaterialId;
    public bool    IsActive;            // false = носитель остановлен/поглощён
}

// Результат взаимодействия с барьером/слоем
// Контракт: GDD §9.4, §13.3
[StructLayout(LayoutKind.Sequential)]
public struct LayerEffects
{
    // ── Для барьеров/брони (GDD §13.3) ───────────────────────────────
    public PenetrationMode Mode;          // Stop | Penetrate | Ricochet
    public float  ResidualSpeed;          // v_out [м/с]
    public float  ResidualEnergy;         // E_out [Дж]
    public float3 ResidualMomentum;       // p_out [Н·с]
    public float  OutAngle;              // angle_out [рад]
    public float  DamageDelta;           // [0..1] — прирост повреждения слоя
    public float  HoleDiameter;          // [м], 0 = нет пробоя
    public uint   SpallProfileId;        // Профиль осколков (0 = нет); опц. v0.1

    // ── Для цели (GDD §13.3) ─────────────────────────────────────────
    public float  PenetrationDepth;      // [м] — глубина пробития в материале
    public float  EnergyDeposited;       // [Дж]
    public float  ImpulseTransmitted;    // [Н·с]
    public float  BluntTraumaIndex;      // Безразмерный индекс тупой травмы
    public float  ThermalDose;           // [Дж/м²] — суммарная термическая доза
    public float  DeltaTemperature;      // delta_T [К] — локальный прирост температуры

    // ── Вторичные носители (GDD §9.4) ────────────────────────────────
    // Генерируются при фрагментации; обрабатываются движком как новые CarrierState
    public bool   HasSecondaryCarriers;  // true = в буфере есть вторичные носители
}
```

### 4.3 Операторы F_layer, G и H

Три ключевых оператора из GDD фиксируются как C#-интерфейсы. Это обеспечивает:
- единый контракт независимо от реализации (формула / lookup / Burst Job);
- замену реализации без изменения кода потребителей;
- тестируемость через mock-реализации.

```csharp
// F_layer: оператор слоя защиты/барьера
// Контракт: GDD §9.3
// F_layer(packet, state, env) -> packet_residual + effects
public interface ILayerOperator
{
    // Применяет слой к входящему пакету
    // state — текущее состояние слоя (нагрев/повреждение/износ)
    // env   — параметры среды (температура, давление, влажность)
    LayerEffects Evaluate(
        in ImpactPacket    packet,
        in LayerState      state,
        in EnvironmentData env);
}

// G: оператор распространения носителя в среде
// Контракт: GDD §10
// G(state, medium_segment, env) -> state'
public interface IPropagationOperator
{
    // Обновляет CarrierState после прохождения отрезка среды
    // segment — описание отрезка (длина, материал среды)
    CarrierState Propagate(
        in CarrierState    state,
        in MediumSegment   segment,
        in EnvironmentData env);
}

// H: transfer function барьера (высокоуровневый оператор над стеком слоёв)
// Контракт: GDD §11.3
// H(material, channel, incident_params, state, env) -> residual_params + effects
public interface ITransferFunction
{
    LayerEffects Evaluate(
        in MaterialDefinition material,
        ThreatChannels        channel,
        in ImpactPacket       incidentParams,
        in BarrierState       state,
        in EnvironmentData    env);
}

// Параметры среды — передаются в F_layer, G, H
public struct EnvironmentData
{
    public float AmbientTemperature; // [К]
    public float AmbientPressure;    // [Па]
    public float Humidity;           // [0..1]
    public uint  MediumMaterialId;   // Материал среды (воздух, вакуум, вода)
}

// Отрезок среды для оператора G
public struct MediumSegment
{
    public float Length;         // [м]
    public uint  MediumMaterialId;
    public float Density;        // [кг/м³] — может отличаться от стандартного (воздух на высоте)
}

// Текущее состояние слоя (передаётся в F_layer)
// Контракт: GDD §9.3 (state включает нагрев/повреждение/износ)
public struct LayerState
{
    public float Damage;      // [0..1]
    public float Temperature; // [К]
    public float WearFactor;  // [0..1] — износ
}

// Состояние барьера целиком (передаётся в H)
// Контракт: GDD §11.1
public struct BarrierState
{
    public float OverallDamage;    // Агрегированное состояние барьера [0..1]
    public float Temperature;     // [К]
}
```

**Барьер как структура данных:**

```csharp
// BarrierInstance — физический барьер окружения
// Контракт: GDD §11.1, §11.2
public class BarrierInstance
{
    public uint              BarrierId;
    public BarrierShapeData  Shape;          // Геометрия (AABB / mesh / capsule)
    public LayerDefinition[] Layers;         // Стек слоёв (снаружи → внутрь)
    public BarrierState      CurrentState;   // Повреждение, температура

    // Вычисляет эффективную толщину для данного направления атаки (GDD §11.2)
    public float GetEffectiveThickness(int layerIndex, float3 incidentDirection)
    {
        float cosAngle = math.abs(math.dot(incidentDirection, Shape.Normal));
        return cosAngle > 1e-4f
            ? Layers[layerIndex].Thickness / cosAngle  // t / cos(θ)
            : float.PositiveInfinity;                  // Скольжение
    }

    // Режим контакта носителя с барьером
    public ContactMode GetContactMode(float3 incidentDirection)
    {
        float dot = math.dot(incidentDirection, Shape.Normal);
        if (dot < -0.95f) return ContactMode.Entry;     // Почти перпендикулярно
        if (dot >  0.95f) return ContactMode.Exit;
        return ContactMode.Glancing;                    // Скользящий контакт
    }
}

public enum ContactMode { Entry, Exit, Glancing }
```

### 4.4 Lookup-таблицы и интерполяция

```csharp
// H_penetration: v_out / stop
// Контракт: GDD §12.5 — H_penetration(material, thickness, angle, v_in, m, d, shape)
// Трёхмерная lookup-таблица — линейный layout для cache efficiency
public struct PenetrationTable
{
    public NativeArray<float> Data;   // [nThickness × nAngle × nVelocity]
    public float   ThicknessMin, ThicknessMax;
    public float   AngleMin,    AngleMax;
    public float   VelocityMin, VelocityMax;
    public int     NThickness, NAngle, NVelocity;
    public TableHeader Header;

    // Трилинейная интерполяция — вызывается из Burst Job
    [BurstCompile]
    public float Sample(float thickness, float angle, float velocity) { ... }
}

// H_ricochet: P(ricochet), angle_out, v_out
// Контракт: GDD §12.5 — H_ricochet(material, angle, v_in, surface_state)
public struct RicochetTable
{
    public NativeArray<float3> Data;    // [nAngle × nVelocity] → (P_ricochet, angle_out, v_out)
    public float   AngleMin,    AngleMax;
    public float   VelocityMin, VelocityMax;
    public int     NAngle, NVelocity;
    public TableHeader Header;

    // Билинейная интерполяция; surface_state влияет через multiplier (damage degradation)
    [BurstCompile]
    public float3 Sample(float angle, float velocity, float surfaceDamage) { ... }
}

// G_drag: dv/dx или lookup v(x) для распространения носителя
// Контракт: GDD §12.5 — G_drag(profile_id, v, medium_params) -> dv/dx
public struct DragTable
{
    public NativeArray<float> Data;    // [nVelocity] → dv/dx [м/с / м]
    public float   VelocityMin, VelocityMax;
    public int     NVelocity;
    public uint    ProfileId;          // Соответствует PropagationProfileId в CarrierState
    public TableHeader Header;

    [BurstCompile]
    public float SampleDvDx(float velocity, float mediumDensityRatio) { ... }
}
```

Политика clamp/extrapolation — часть версионированной модели. Фиксируется в golden-тестах.

### 4.5 Burst Jobs

Горячие пути оборачиваются в `IJobParallelFor` для автоматической параллелизации и Burst-компиляции:

```csharp
[BurstCompile(CompileSynchronously = true, OptimizeFor = OptimizeFor.Performance)]
public struct PenetrationBatchJob : IJobParallelFor
{
    [ReadOnly]  public NativeArray<ImpactPacket>    Packets;
    [ReadOnly]  public NativeArray<LayerDefinition> Layers;
    [ReadOnly]  public PenetrationTable             HTable;
    [WriteOnly] public NativeArray<LayerEffects>    Results;

    public void Execute(int index)
    {
        ref readonly var packet = ref Packets.ElementAt(index);
        // Lookup + триlinear интерполяция
        Results[index] = ComputeLayerResponse(packet, Layers[index], HTable);
    }
}
```

**Когда Burst нужен, когда нет:**

| Задача | Нагрузка | Burst нужен? |
|---|---|---|
| Lookup H_penetration (50 снарядов/ход) | < 0.01 мс | Нет, Mono справляется |
| Тепловые профили 200 зон | < 0.01 мс | Нет |
| A* для 20 юнитов (20×40 тайлов) | < 0.1 мс | Нет |
| Предрасчёт таблиц при валидации (1M точек) | ~500 мс → ~10 мс | **Да** |
| Batch-симуляция для балансировки | > 10K итераций | **Да** |

### 4.6 Детерминизм

- Все gameplay-расчёты — на CPU.
- Стохастика через детерминированный `uint` seed (xorshift32 или PCG32).
- Floating-point порядок операций фиксируется через `[BurstCompile(FloatMode = FloatMode.Strict)]` для воспроизводимых расчётов.
- Golden-тесты фиксируют бит-точные результаты для ключевых пайплайнов.

```csharp
// Детерминированный RNG — не System.Random, не Unity.Random
public struct DeterministicRng
{
    private uint _state;
    public DeterministicRng(uint seed) { _state = seed == 0 ? 1u : seed; }

    public uint NextUInt()   // xorshift32
    {
        _state ^= _state << 13;
        _state ^= _state >> 17;
        _state ^= _state << 5;
        return _state;
    }

    public float NextFloat01() => (NextUInt() >> 8) * (1f / 16777216f);
}
```

---

## 5. Конструктор (граф деталей)

### 5.1 Граф деталей

Сборка — `Dictionary<uint, PartInstance>` + `List<Connection>`. Граф обходится при валидации и рендеринге.

```csharp
// Part — физический объект конструктора
public class PartInstance
{
    public uint           InstanceId;
    public PartDefinition Definition;     // ScriptableObject с данными
    public List<IComponent> Components;   // Функциональная начинка
    public List<NodeState>  Nodes;        // Узлы крепления (состояние: free/connected)
    public float3         LocalPosition;
    public quaternion     LocalRotation;
    public DamageState    Damage;         // Текущее состояние повреждений
}

// Connection — ребро графа
public readonly struct Connection
{
    public readonly uint   PartA, PartB;
    public readonly uint   NodeA, NodeB;
    public readonly ConnectionType Type;
}

// Граф сборки
public class AssemblyGraph
{
    private readonly Dictionary<uint, PartInstance> _parts = new();
    private readonly List<Connection>               _connections = new();

    // O(V+E) обход для валидации
    public ValidationResult Validate(IValidator[] validators) { ... }

    // Предрасчёт производных параметров → WeaponProfile
    public WeaponProfile Precompute(MaterialLibrary materials, TableCache tables) { ... }
}
```

### 5.2 ScriptableObject-данные

`PartDefinition` — ScriptableObject, описывает статические параметры детали.

```csharp
[CreateAssetMenu(menuName = "MechanicTech/PartDefinition")]
public class PartDefinition : ScriptableObject
{
    [Header("Identity")]
    public string    PartId;
    public string    DisplayName;
    public PartRole[] Roles;

    [Header("Physical")]
    public float     Mass;         // [кг]
    public float3    Dimensions;   // [м]
    public uint      MaterialId;   // Ссылка в MaterialLibrary

    [Header("Nodes")]
    public NodeDefinition[] Nodes;

    [Header("Components")]
    public ComponentDescriptor[] ComponentDescriptors;

    [Header("LOD Parameters")]
    public int       MinTechLevel;         // Открывается при этом техуровне
    public ParameterGroup[] ParameterGroups; // Вкладки LOD (см. GDD §5.1)

    [Header("Compatibility")]
    public CompatibilityRule[] Rules;      // Правила совместимости с соседями
}
```

`MaterialDefinition` — ScriptableObject. Соответствует контракту MaterialDefinition v0.1 из GDD §12.2.  
Материал хранит **параметры и законы**, «решения» для боя — в предрасчитанных таблицах.

```csharp
[CreateAssetMenu(menuName = "MechanicTech/MaterialDefinition")]
public class MaterialDefinition : ScriptableObject
{
    // ── Служебные (GDD §12.2) ─────────────────────────────────────────
    [Header("Identity")]
    public uint     MaterialId;
    public string   DisplayName;
    public string   Family;           // "metal" | "polymer" | "composite" | "ceramic" | ...
    public string[] Tags;             // Произвольные теги для фильтрации и поиска
    public string[] References;       // Ссылки на источники данных (опц.)
    public int      SchemaVersion = 1;

    // ── Механика SI (GDD §12.2 — rho, E, nu, sigma_y, sigma_u, H, K_IC, eta) ──
    [Header("Mechanical (SI)")]
    public float   Density;           // rho [кг/м³]
    public float   YoungModulus;      // E [ГПа]
    public float   PoissonRatio;      // nu [безразм.]
    public float   YieldStrength;     // sigma_y [МПа] (если применимо)
    public float   UltimateStrength;  // sigma_u [МПа] (опц.)
    public float   HardnessVickers;   // H [HV] — с типом шкалы
    public float   FractureToughness; // K_IC [МПа·√м] (опц.)
    public float   DynamicViscosity;  // eta [Па·с] (опц. — для жидкостей/расплавов)

    // ── Термо SI (GDD §12.2 — c_p, k, T_deg, T_m, L_m, epsilon) ────────
    [Header("Thermal (SI)")]
    public float   SpecificHeat;         // c_p [Дж/(кг·К)]
    public float   ThermalConductivity;  // k [Вт/(м·К)]
    public float   DegradationTemp;      // T_deg [К] — для полимеров/композитов
    public float   MeltingPoint;         // T_m [К] (если применимо)
    public float   LatentHeatMelting;    // L_m [Дж/кг] (опц.)
    public float   ThermalEmissivity;    // epsilon [0..1] (опц.)

    // ── Контакт (GDD §12.2 — mu) ──────────────────────────────────────
    [Header("Contact")]
    public float   FrictionCoeff;        // mu [безразм.] — средний профиль трения

    // ── Электромагнитные (GDD §12.2 — sigma_e, eps_r, E_break, mu_r) ──
    // Активны только при наличии EM-канала угроз (ThreatModel v0.x+)
    [Header("Electromagnetic (SI, optional)")]
    public float   ElectricalConductivity; // sigma_e [См/м] (опц.)
    public float   RelPermittivity;        // eps_r [безразм.] (опц.)
    public float   DielectricBreakdown;    // E_break [МВ/м] (опц.)
    public float   RelPermeability;        // mu_r [безразм.] (опц.)

    // ── Калиброванные коэффициенты модели (GDD §12.2 — model-specific) ─
    // Явно помечены как model-specific и версионируются вместе с ModelVersion
    [Header("Model-specific Coefficients (versioned)")]
    public float   PenCoeff;      // R_pen — коэффициент модели пробития
    public float   ImpactCoeff;   // C_imp — коэффициент модели удара
    public float   AblationCoeff; // C_ablation — коэффициент абляции (термо)
    public int     ModelVersion;  // При изменении → пересборка всех таблиц материала
}
```

`LayerDefinition` — описание одного слоя барьера/брони. Контракт: GDD §12.3.

```csharp
// LayerDefinition v0.1 — один слой барьера или брони
public class LayerDefinition
{
    public uint   MaterialId;         // Ссылка в MaterialLibrary
    public float  Thickness;          // t [м]
    public float3 Orientation;        // Опц. — анизотропия/ориентация волокон (для композитов)
    public float  Porosity;           // [0..1] опц. — пористость (для пористых/сетчатых материалов)
    public uint   InterfaceLayerId;   // Опц. — ID клея/подложки между слоями (0 = нет)

    // Текущее состояние слоя — изменяется в процессе боя (GDD §12.3)
    public float  Damage;             // [0..1] — накопленное повреждение
    public float  Temperature;        // [К] — текущая температура слоя
}
```

**Операторы законов материала (GDD §12.4)** — хуки для вычислений в симуляционном ядре:

```csharp
// M_mech: деформация / пластичность / разрушение
public interface IMechLaw
{
    float ComputeYieldResponse(float stress, float strainRate, float temperature);
    float ComputeFailureStress(MaterialDefinition mat, float temperature);
}

// M_therm: нагрев / теплоперенос / абляция
public interface IThermalLaw
{
    float ComputeHeatDeposition(in ImpactPacket packet, in LayerDefinition layer);
    float ComputeAblationRate(float heatFlux, float surfaceTemp, MaterialDefinition mat);
}

// M_wave: упрощённое распространение импульса/ударных волн
public interface IWaveLaw
{
    float ComputeShockImpedance(MaterialDefinition mat, float velocity);
    float ComputeSpallThreshold(MaterialDefinition mat);
}

// M_surface: контакт / трение (используется в модели рикошета)
public interface ISurfaceLaw
{
    float ComputeFrictionForce(float normalForce, MaterialDefinition mat, float velocity);
}

// M_em: экранирование / затухание / пробой (будущие каналы угроз)
public interface IEMLaw
{
    float ComputeAttenuation(float frequency, MaterialDefinition mat, float thickness);
}
```

### 5.3 Рендеринг конструктора

```
AssemblyGraph
    └── ConstructorRenderer (MonoBehaviour)
            ├── Instantiate Part GameObject (per PartInstance)
            │       └── MeshFilter + MeshRenderer (Part mesh)
            ├── Update transform per frame (drag/drop)
            ├── OnValidate → run AssemblyGraph.Validate() → highlight errors
            └── HeatmapOverlay (Compute Shader или vertex color pass)
```

Оптимизация рендеринга конструктора:
- SRP Batcher включён по умолчанию (HDRP). Детали с одинаковым Material Variant объединяются в пакеты.
- GPU Instancing для повторяющихся подчастей (болты, крепёж, одинаковые модули).
- `Mesh.CombineMeshes` при финализации сборки для перехода в бой — один меш вместо N объектов.

---

## 6. Тактический бой

### 6.1 Тайловая карта

```csharp
// Тайл карты — struct для cache-friendly хранения
public struct TileData
{
    public TileType   Type;           // Floor, Wall, Cover, Void
    public ushort     Height;         // [мм], фиксированная точка
    public byte       Passability;    // Bitflags: N/S/E/W
    public byte       CoverQuality;   // 0–255 → 0.0–1.0 нормализованно
    public uint       BarrierInstanceId; // 0 = нет барьера
    public bool       IsVisible;      // Кэш видимости (инвалидируется при изменении)
}

// Карта — обёртка над NativeArray<TileData>
public class BattleGrid
{
    private NativeArray<TileData> _tiles;
    public int Width  { get; }
    public int Height { get; }

    public ref TileData GetTile(int x, int z)    // ref — без копирования
        => ref _tiles.ElementAt(z * Width + x);

    public int2 WorldToGrid(float3 worldPos) { ... }
    public float3 GridToWorld(int2 gridPos)  { ... }
}
```

### 6.2 Pathfinding

A* реализуется собственный (не сторонний, для полного контроля над детерминизмом и стоимостью AP).

```csharp
// Burst-совместимый A* на NativeArray
[BurstCompile]
public struct AStarJob : IJob
{
    [ReadOnly]  public NativeArray<TileData>  Grid;
    [ReadOnly]  public int2  Start, Goal;
    [ReadOnly]  public int   GridWidth;
    [WriteOnly] public NativeList<int2> Path;   // Результат

    // NativeMinHeap — кастомная priority queue на NativeArray
    private NativeMinHeap<AStarNode>  _openSet;
    private NativeHashMap<int2, int2> _cameFrom;
    private NativeHashMap<int2, float> _gScore;

    public void Execute() { /* A* без heap-аллокаций */ }
}
```

Стоимость тайла для A* включает базовую стоимость перемещения + штраф от массы снаряжения (передаётся через параметры Unit).

### 6.3 Line-of-Sight

Алгоритм Брезенхэма по тайлам с учётом высоты объектов. Кэш LOS — `NativeHashMap<int4, bool>` (sourceX, sourceZ, targetX, targetZ → видимость). Инвалидируется только при перемещении юнита или изменении тайла.

```csharp
[BurstCompile]
public struct LOSJob : IJobParallelFor
{
    [ReadOnly]  public NativeArray<int4>     Queries;  // (sx,sz,tx,tz)[]
    [ReadOnly]  public NativeArray<TileData> Grid;
    [ReadOnly]  public int GridWidth;
    [WriteOnly] public NativeArray<bool>     Results;

    public void Execute(int i) { /* Bresenham по тайлам */ }
}
```

### 6.4 AP-система и очередь действий

```csharp
// ── Параметры бойца (постоянные — хранятся вне боя) ──────────────────
// Контракт: GDD §14.2 — индивидуальные параметры, нет жёстких классов
public struct UnitBaseStats
{
    // Физические
    public float Strength;     // Грузоподъёмность без штрафа, компенсация отдачи
    public float Endurance;    // Базовый AP-бюджет, скорость набора усталости
    public float Agility;      // Скорость действий, уклонение

    // Навыковые
    public float Accuracy;     // Влияет на разброс (помимо оружия)
    public float Reaction;     // Overwatch-триггер, штраф на первое действие при сюрпризе
    public float Perception;   // Дальность обнаружения, эффективность сенсоров

    // Ментальные (GDD §14.2)
    public float StressResistance;  // Устойчивость к панике/страху под давлением
    public float Concentration;     // Влияет на сложные действия (снайперский выстрел, взлом)

    // Прогрессия — параметры растут с опытом (GDD §14.2)
    public int   TotalShots;       // Счётчик для прокачки Accuracy
    public float TotalCarriedMass; // Счётчик для прокачки Strength
}

// Зоны здоровья по частям тела — контракт: GDD §14.4
public struct HealthZones
{
    public float Head;        // [0..1] — критично: смерть/оглушение при 0
    public float Torso;       // Основной пул HP; ранение → штраф Endurance
    public float ArmLeft;
    public float ArmRight;    // Ранение → штраф Accuracy, скорость перезарядки
    public float LegLeft;
    public float LegRight;    // Ранение → штраф AP на перемещение
}

// Состояние бойца в бою
// Контракт: GDD §15.3, §15.4, §14.4
public struct UnitCombatState
{
    public uint   UnitId;

    // AP-экономика (GDD §15.4)
    public int    CurrentAP;
    public int    MaxAP;              // Из Endurance + снаряжение (WeaponProfile)
    public float  FatigueFactor;      // [0..1] — растёт от массы + действий

    // Позиция и стойка
    public int2   GridPosition;
    public byte   Stance;            // Stand=0, Crouch=1, Prone=2
    // Смена стойки стоит AP (GDD §15.4)

    // Здоровье и ранения (GDD §14.4)
    public HealthZones HP;           // Текущий HP по зонам

    // Ранения — каждое накладывает штраф на параметры
    public byte   WoundFlags;        // Bitflags: HeadWound, TorsoWound, ArmLWound, ...
    public float  AccuracyPenalty;   // Суммарный штраф от ранений рук
    public float  MovementAPPenalty; // Суммарный штраф от ранений ног

    // Видимость и сенсоры (GDD §15.5)
    public float  EffectiveVisionRange;  // [тайлы] — из Perception + оборудование
    public bool   HasThermalVision;      // Из сенсорного оборудования

    // Overwatch (GDD §15.3, §15.4)
    public bool   HasOverwatch;
    public int    OverwatchAPReserved;
    public float3 OverwatchDirection;    // Направление / сектор overwatch
}

// WeaponProfile — результат валидации сборки оружия, передаётся в бой
// Содержит только производные параметры — тяжёлые расчёты уже выполнены
public struct WeaponProfile
{
    // Основные боевые параметры
    public float MuzzleVelocity;     // [м/с]
    public float ProjectileMass;     // [кг]
    public float ProjectileDiameter; // [м]
    public uint  ProjectileShapeId;
    public uint  ProjectileMaterialId;

    // AP-стоимости (GDD §15.4)
    public int   APCostFire;         // Базовая стоимость выстрела
    public int   APCostReload;
    public int   APCostAim;          // Прицеливание (опц. действие)

    // Нагрузка на бойца
    public float TotalMass;          // [кг] — вся сборка с боезапасом
    public float RecoilImpulse;      // [Н·с] — отдача (влияет на Accuracy, AP-восстановление)
    public float HeatGenPerShot;     // [Дж] — тепловыделение

    // Ограничения работоспособности
    public float MaxHeatBeforeJam;   // [К] — перегрев → отказ (GDD §16.2)
    public float DegradationPerShot; // [0..1] — деградация ствола за выстрел
    public int   RoundsInMagazine;

    // Энергосистема (GDD §16.2 — энергобюджет)
    public float EnergyPerShot;      // [Дж] — 0 = кинетика без питания
    public float MaxEnergyCapacity;  // [Дж] — ёмкость накопителя

    // Производные — предрасчитаны валидатором
    public uint  AssemblyHash;       // Для инвалидации кэша при пересборке
    public int   ModelVersion;       // Версия модели, при которой сгенерирован профиль
}
```

```csharp
// Действие — Command pattern для undo/replay
// AP-стоимость зависит от параметров бойца и профиля снаряжения (GDD §15.4)
public interface ICombatAction
{
    int          APCost(in UnitCombatState state, in WeaponProfile profile);
    ActionResult Execute(ref CombatWorldState world, DeterministicRng rng);
    void         Undo(ref CombatWorldState world); // Для replay/golden-тестов
}
```

---

### 6.6 Процедурная генерация тактических карт

> **Принцип:** тактические сцены **не создаются вручную**. Карта формируется программно на этапе загрузки миссии на основе параметров операции. Ручная работа с Unity-сценами ограничена разработкой конструктора и UI.

### Параметры генерации

Каждая миссия задаётся `MissionDescriptor` — структурой с параметрами, которые драйвят генератор:

```csharp
// Контракт: GDD §15.2, §15.7
[Serializable]
public class MissionDescriptor
{
    // Тип операции — определяет структуру карты и условия победы (GDD §15.7)
    public MissionType Type;
    // Assault       — штурм укреплённой точки
    // Infiltration  — зачистка базы, коридоры и помещения
    // OpenField     — открытая пересечённая местность
    // Mixed         — переход открытое → закрытое (GDD §15.2)
    // Extraction    — эвакуация через контролируемую зону
    // Defense       — защита объекта/персонажа

    // Местность и среда
    public BiomeType    Biome;           // Desert, Arctic, Urban, Station, Ship, ...
    public float        UrbanDensity;    // [0..1] — плотность застройки
    public float        ElevationRange;  // [м] — перепад высот
    public bool         IsIndoor;        // true = станция/корабль (коридорная геометрия)

    // Масштаб (GDD §15.2)
    public int2         GridSize;        // Размер тайловой карты [тайлы]
    public MapScale     Scale;           // Small | Medium | Large — влияет на дистанции

    // Условия победы (GDD §15.7)
    public VictoryCondition[] WinConditions;
    public int                TurnLimit;   // 0 = без лимита

    // Силы противника — задают угрозы, под которые подбирается снаряжение
    public EnemyGroupDescriptor[] EnemyGroups;

    // Seed — детерминированная генерация, воспроизводимые миссии
    public uint         GenerationSeed;
}
```

### Архитектура генератора

```
MissionDescriptor
    ↓
MissionLoader (MonoBehaviour, запускается при загрузке сцены)
    ├── TerrainGenerator        — базовый рельеф (heightmap по Biome + ElevationRange)
    ├── StructureGenerator      — здания/укрытия по UrbanDensity и MissionType
    ├── BarrierInstanceFactory  — создаёт BarrierInstance из шаблонов материалов
    ├── SpawnPointAllocator     — размещает точки входа игрока и противника
    ├── ObjectiveMarker         — расставляет цели миссии (захват, защита, эвакуация)
    └── BattleGrid.Build()      — финализирует NativeArray<TileData>
```

**Вся процедурная геометрия генерируется в runtime через Mesh API** — нет предсозданных тайловых префабов для самих карт. Барьеры строятся из пула `BarrierPrefabLibrary` (материал + геометрия), который задаётся в `BiomeConfig` ScriptableObject.

```csharp
public class MissionLoader : MonoBehaviour
{
    [SerializeField] private BiomeConfigLibrary _biomeConfigs;
    [SerializeField] private BarrierPrefabLibrary _barrierPrefabs;
    [SerializeField] private MaterialLibrary _materialLibrary;

    // Вызывается Addressables при загрузке сцены Combat.unity
    public async UniTask LoadMission(MissionDescriptor descriptor)
    {
        var rng = new DeterministicRng(descriptor.GenerationSeed);
        var biome = _biomeConfigs.Get(descriptor.Biome);

        // 1. Рельеф
        var heightmap = TerrainGenerator.Generate(descriptor, biome, rng);

        // 2. Постройки/укрытия
        var structures = StructureGenerator.Place(descriptor, heightmap, biome, rng);

        // 3. Создать BarrierInstance для каждой структуры
        var barriers = BarrierInstanceFactory.Build(structures, _barrierPrefabs,
                                                     _materialLibrary);
        // 4. Finalise grid
        _battleGrid = BattleGrid.Build(heightmap, barriers, descriptor.GridSize);

        // 5. Spawn points, objectives
        SpawnPointAllocator.Place(descriptor, _battleGrid, rng);
    }
}
```

**Сцены в проекте:**

```
Scenes/
├── Constructor.unity   — конструктор оружия/снаряжения (статичная сцена)
├── Combat.unity        — пустая «оболочка» боя; карта формируется MissionLoader
└── MainMenu.unity      — главное меню
```

`Combat.unity` содержит только постоянные компоненты: камеру, UI-слой, менеджеры систем. Вся тактическая геометрия — процедурная.

---

### 6.8 Барьеры окружения в бою

Барьеры в бою — те же `F_layer` и `H` transfer functions, что и в симуляционном ядре. Физический механизм унифицирован:

```
Carrier State
    ↓  (пуля летит к укрытию)
Impact Packet (генерируется при пересечении геометрии барьера)
    ↓
F_layer(packet, barrierState, env)  →  LayerEffects
    ↓
Carrier State'  (остаточные параметры) или Stop
    ↓  (пуля продолжает путь)
Impact Packet → цель
```

Барьер деградирует: `DamageDelta` аккумулируется в `BarrierInstance.DamageState`, влияет на последующий `H`. Это реализует «деградирующие укрытия» из GDD §15.6.

---

## 7. GPU Compute

### 7.1 Где применяется

GPU Compute используется **только там, где задача достаточно параллельна и нет требования к детерминизму результатов между GPU**:

| Задача | Движок | Когда |
|---|---|---|
| Предрасчёт H_penetration surface (100³+ точек) | Compute Shader | При валидации сборки (офлайн, не realtime) |
| Heatmap overlay в конструкторе | Compute Shader | Каждый кадр при изменении параметров |
| GPU-частицы VFX | VFX Graph (GPU) | Realtime, только визуализация |

Все gameplay-affecting расчёты (Impact Packet, A*, LOS, AI) — **только CPU** для детерминизма.

### 7.2 Структура Compute Shader для предрасчёта

```hlsl
// Assets/_Project/VFX/Shaders/PenetrationTableGen.compute
#pragma kernel GeneratePenetrationSurface

RWStructuredBuffer<float> ResultTable;  // [nThick × nAngle × nVel]

cbuffer Params {
    uint   NThickness, NAngle, NVelocity;
    float  ThickMin, ThickMax;
    float  AngleMin, AngleMax;
    float  VelMin, VelMax;
    float  PenCoeff;    // R_pen материала
    float  Density;
    int    ModelVersion;
}

[numthreads(8, 8, 4)]   // 256 потоков — оптимально для большинства GPU
void GeneratePenetrationSurface(uint3 id : SV_DispatchThreadID)
{
    if (id.x >= NThickness || id.y >= NAngle || id.z >= NVelocity) return;

    float t  = ThickMin + id.x * (ThickMax - ThickMin) / (NThickness - 1);
    float a  = AngleMin + id.y * (AngleMax - AngleMin) / (NAngle - 1);
    float v  = VelMin   + id.z * (VelMax   - VelMin)   / (NVelocity - 1);

    // Упрощённая модель пробития — заменяется реальной моделью материала
    float vResidual = ComputeResidualVelocity(v, t, a, PenCoeff, Density);

    uint idx = id.x * NAngle * NVelocity + id.y * NVelocity + id.z;
    ResultTable[idx] = vResidual;
}
```

```csharp
// Диспатч из C# при валидации сборки
public class TableGenerator
{
    private ComputeShader _shader;

    public NativeArray<float> GeneratePenetrationTable(
        MaterialDefinition mat,
        TableParams p)
    {
        int totalEntries = p.NThickness * p.NAngle * p.NVelocity;
        var buffer = new ComputeBuffer(totalEntries, sizeof(float));

        int kernel = _shader.FindKernel("GeneratePenetrationSurface");
        _shader.SetBuffer(kernel, "ResultTable", buffer);
        _shader.SetInt("NThickness", p.NThickness);
        // ... SetFloat для остальных параметров

        _shader.Dispatch(kernel,
            Mathf.CeilToInt(p.NThickness / 8f),
            Mathf.CeilToInt(p.NAngle    / 8f),
            Mathf.CeilToInt(p.NVelocity / 4f));

        // Readback → NativeArray для передачи в симуляционное ядро
        var result = new NativeArray<float>(totalEntries, Allocator.Persistent);
        buffer.GetData(result.ToArray()); // Один readback — нет per-frame overhead
        buffer.Dispose();
        return result;
    }
}
```

---

## 8. VFX-пайплайн

### 8.1 Архитектура

VFX-слой полностью отделён от симуляционного ядра. Ядро возвращает `LayerEffects`, Combat-слой транслирует их в `VFXEvent` и передаёт в `VFXDirector`.

```
CombatEngine → LayerEffects
    ↓
VFXEventTranslator
    ↓  (mapping: режим × материал × энергия → параметры VFX)
VFXEvent { type, position, normal, intensity, materialTag, ... }
    ↓
VFXDirector (MonoBehaviour)
    ├── PenetrationVFX.vfx     (пробоина + брызги металла)
    ├── RicochetVFX.vfx        (искры + трассер)
    ├── ThermalVFX.vfx         (нагрев + абляция)
    └── ImpactDecalSystem      (пробоины на поверхностях)
```

### 8.2 Шейдеры (Shader Graph / HLSL)

**Heat Glow Material** — температурный emission на основе параметра `_Temperature [К]`:

```hlsl
// Кривая: 0→чёрный, 800К→тёмно-красный, 1200К→оранжевый, 1600К→белый
float3 TemperatureToEmission(float tempK)
{
    float t = saturate((tempK - 300.0) / 1600.0);   // Нормализованная температура
    // Blackbody approximation (Planck)
    float3 color = lerp(float3(0.2, 0.0, 0.0),     // Тёмно-красный
                   lerp(float3(1.0, 0.4, 0.0),      // Оранжевый
                        float3(1.0, 0.95, 0.8),      // Белый
                        saturate((t - 0.5) * 2.0)),
                   saturate(t * 2.0));
    float intensity = pow(max(0.0, t), 2.5) * 50.0; // Нелинейный bloom-ready
    return color * intensity;
}
```

Параметр `_Temperature` передаётся из симуляции через `Material.SetFloat()` — один float per part per frame, нулевой overhead.

**Surface Damage Material** — смешение текстур по маске повреждений:

```hlsl
// Damage: 0=intact, 1=destroyed
float damage = _DamageState; // из LayerDefinition.damage

float3 baseAlbedo   = SAMPLE_TEXTURE2D(_BaseMap,    ...);
float3 damagedAlbedo = SAMPLE_TEXTURE2D(_DamageMap, ...);
float  damageMask   = SAMPLE_TEXTURE2D(_DamageMask, ...).r;

float3 finalAlbedo = lerp(baseAlbedo, damagedAlbedo, damage * damageMask);
float  finalRoughness = lerp(_Roughness, _DamageRoughness, damage * damageMask);
```

**Heatmap Overlay** — для конструктора, визуализация нагрузок:

```hlsl
// Передаётся как float[] через MaterialPropertyBlock (без создания новых Material)
uniform float _HeatmapValues[64];     // Значения по зонам детали
uniform float4 _HeatmapColors[5];     // [синий, голубой, зелёный, жёлтый, красный]

float GetZoneValue(float2 uv)
{
    // UV-маска зоны → индекс → значение
    int zoneIdx = (int)SAMPLE_TEXTURE2D(_ZoneMask, ...).r * 63;
    return _HeatmapValues[zoneIdx];
}

float4 HeatmapColor(float v)  // v: [0..1]
{
    // 5-цветный gradient
    float idx = saturate(v) * 4.0;
    return lerp(_HeatmapColors[(int)idx], _HeatmapColors[(int)idx + 1], frac(idx));
}
```

### 8.3 DecalProjector (пробоины)

Декали управляются через пул. Параметры пробоины (диаметр, нормаль, материал цели) поступают из `LayerEffects.HoleDiameter` и `ContactNormal`.

```csharp
public class ImpactDecalPool
{
    private readonly Stack<DecalProjector> _pool;
    private const int MAX_DECALS = 128;    // Ring buffer — старые перезаписываются

    public void SpawnBulletHole(float3 position, quaternion rotation,
                                float holeDiameter, uint targetMaterialId)
    {
        var decal = _pool.Count > 0 ? _pool.Pop() : CreateNewDecal();

        // Материал декали определяется targetMaterialId
        decal.material    = _materialRegistry[targetMaterialId];
        decal.size        = new Vector3(holeDiameter * 3f,  // Decal больше пробоины
                                        holeDiameter * 3f,
                                        0.05f);
        decal.transform.SetPositionAndRotation(position, rotation);
        decal.gameObject.SetActive(true);
    }
}
```

---

## 9. Управление памятью и GC

### 9.1 Правила работы с GC

В gameplay-цикле (горячий путь — Per Turn, Per Action) соблюдаются следующие правила:

| Запрещено | Причина | Альтернатива |
|---|---|---|
| `new T()` для reference types | GC-аллокация | Object pool / struct |
| LINQ в hot path | Скрытые аллокации | Явные циклы |
| `string` конкатенация | Аллокации | `StringBuilder` / предаллоцированный буфер |
| `Instantiate()` без пула | GC + Load | `ObjectPool<T>` |
| `List<T>.Add()` сверх capacity | Resize = GC | `Capacity = maxExpected` при старте |
| Boxing value types | GC | Generic constraints / struct interfaces |

### 9.2 Архитектура памяти для горячего пути

```csharp
// Все данные хода — pre-allocated, нет new() в процессе хода
public class TurnMemoryArena : IDisposable
{
    // NativeArray — unmanaged heap, невидимы для GC
    public NativeArray<ImpactPacket>    ImpactPackets;
    public NativeArray<CarrierState>    Carriers;
    public NativeArray<LayerEffects>    Effects;
    public NativeArray<TileData>        GridSnapshot;
    public NativeArray<UnitCombatState> UnitStates;

    // Managed: только там, где неизбежно
    private readonly List<VFXEvent>     _vfxEvents;  // Capacity = maxExpected

    public TurnMemoryArena(int maxCarriers, int maxEffects, int gridSize)
    {
        ImpactPackets = new NativeArray<ImpactPacket>(maxCarriers, Allocator.Persistent);
        // ...
        _vfxEvents = new List<VFXEvent>(maxCarriers * 3); // Pre-allocate
    }

    public void Dispose()
    {
        ImpactPackets.Dispose();
        Carriers.Dispose();
        Effects.Dispose();
        GridSnapshot.Dispose();
        UnitStates.Dispose();
    }
}
```

### 9.3 Incremental GC

Unity 6 — Incremental GC включён по умолчанию. Ручная настройка через `GarbageCollector.GCMode = GCMode.Incremental` не требуется, но monitored через Unity Profiler (цель: < 0.3 мс GC per frame в бою).

---

## 10. Тестирование

### 10.1 Стратегия тестирования

| Уровень | Инструмент | Что тестируется |
|---|---|---|
| **Unit** | NUnit (Unity Test Framework, EditMode) | Функции симуляционного ядра без движка |
| **Golden** | NUnit + бинарные снимки | Детерминизм: фиксированный seed → точный результат |
| **Integration (PlayMode)** | Unity Test Framework, PlayMode | Полный пайплайн сборка→валидация→бой |
| **Performance** | Unity Performance Testing Package | Регрессии производительности Burst Jobs |

### 10.2 Golden-тесты

```csharp
[TestFixture, Category("Golden")]
public class PenetrationGoldenTests
{
    [Test]
    public void SteelPlate_10mm_45deg_800mps_ExpectedResidual()
    {
        // Arrange
        var packet = new ImpactPacket {
            VelocityVec    = new float3(0, 0, 800f),
            EffectiveMass  = 0.010f,   // 10 г
            EquivDiameter  = 0.009f,   // 9 мм
            IncidenceAngle = 0.785f,   // 45°
            RngSeed        = 42u
        };
        var layer = new LayerDefinition {
            MaterialId = MaterialIds.Steel30CrMnSi,
            Thickness  = 0.010f        // 10 мм
        };
        var hTable = TableCache.GetOrCreate(layer.MaterialId, ModelVersion.V01);

        // Act
        var effects = PenetrationModel.Evaluate(packet, layer, hTable);

        // Assert — золотое значение зафиксировано при первом прохождении
        Assert.AreEqual(PenetrationMode.Penetrate,  effects.Mode);
        Assert.AreEqual(0.7234f, effects.ResidualSpeed,   delta: 1e-4f);
        Assert.AreEqual(0.0091f, effects.HoleDiameter,    delta: 1e-4f);
    }
}
```

**Политика:** при изменении модели материала или таблиц — все golden-тесты пересчитываются и фиксируются заново (коммит с тегом `[golden-update]`). Это явное подтверждение, что модель изменилась намеренно.

### 10.3 Performance-тесты

```csharp
[Test, Performance]
public void BurstPenetrationBatch_1000Packets_Under1ms()
{
    var arena = new TurnMemoryArena(1000, 1000, 800);
    FillTestData(arena);

    Measure.Method(() =>
    {
        var job = new PenetrationBatchJob {
            Packets = arena.ImpactPackets,
            Results = arena.Effects,
            HTable  = TableCache.Get(MaterialIds.Steel30CrMnSi)
        };
        job.Schedule(1000, 64).Complete();
    })
    .WarmupCount(3)
    .MeasurementCount(10)
    .Run();

    // Критерий: 1000 пакетов < 1 мс (с Burst)
}
```

---

## 11. Сериализация и версионирование данных

### 11.1 Форматы данных

| Тип данных | Формат | Обоснование |
|---|---|---|
| MaterialDefinition, PartDefinition | ScriptableObject | Редактирование в Editor, Inspector-friendly |
| Lookup-таблицы (H_penetration, G_drag) | Бинарный float[] + JSON-заголовок | Компактно, быстрая загрузка, версионируется |
| Сохранения сборок | JSON (JsonUtility / Newtonsoft.Json) | Читаемость для отладки |
| Replay / golden-тесты | Бинарный snapshot | Точность до бита |
| Post-battle отчёт | JSON | Читаемость, возможна выгрузка |

### 11.2 Версионирование таблиц

Каждая lookup-таблица хранит `{material_id, model_version, table_hash}` в заголовке. При загрузке `TableCache` проверяет совпадение `model_version` с текущей версией модели. Несоответствие — автоматическая регенерация таблицы.

```csharp
public struct TableHeader
{
    public uint   MaterialId;
    public int    ModelVersion;  // Должен совпадать с MaterialDefinition.ModelVersion
    public uint   TableHash;     // CRC32 данных таблицы — для golden-тестов
    public int    NThickness, NAngle, NVelocity;
    public float  ThickMin, ThickMax, AngleMin, AngleMax, VelMin, VelMax;
}
```

---

## 12. Пайплайн сборки и CI

### 12.1 Минимальный CI (GitHub Actions)

```yaml
# .github/workflows/tests.yml
name: Simulation Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: game-ci/unity-test-runner@v3
        with:
          unityVersion: 6000.0.xx
          testMode: editmode          # Только симуляционное ядро — быстро
          customParameters: -category Simulation,Golden
```

EditMode-тесты симуляционного ядра не требуют запуска Unity Player — выполняются за секунды.

---

## 13. Технический долг и ограничения v0.1

| Ограничение | Описание | Когда решать |
|---|---|---|
| Моно-хрупкость Burst | При изменении struct → перекомпиляция; ошибки Burst бывают неочевидны | По мере роста числа Jobs |
| GC в Managed C# | При нарушении правил (раздел 9.1) → спайки | Профилировать с Slice 1 |
| Cross-GPU детерминизм Compute Shaders | Таблицы, сгенерированные на разных GPU, могут отличаться | Добавить CPU-фоллбэк и тест сравнения |
| HDRP минимальные требования | DirectX 12 / Vulkan обязателен | Фиксировано системными требованиями |
| Отсутствие Level Editor | Тактические карты создаются вручную через Scene | Slice 2–3: тулинг на ImGui или Unity EditorWindow |

---

## 14. Список технологий (итог)

| Технология | Назначение | Версия |
|---|---|---|
| Unity 6.0+ LTS | Движок | 6000.x |
| HDRP | Render Pipeline | Bundled |
| C# (.NET Standard 2.1) | Основной язык | — |
| Burst Compiler | Нативная компиляция горячих путей | 1.8+ |
| Unity.Mathematics | SIMD-совместимые типы (float3, int2, ...) | 1.3+ |
| Unity.Collections (NativeArray) | Unmanaged-коллекции без GC | 2.x |
| Unity.Jobs | Многопоточность для Burst Jobs | 0.70+ |
| Shader Graph (HDRP) | Heat glow, damage blending, heatmap | Bundled |
| VFX Graph | GPU-частицы: искры, трассеры, взрывы | Bundled |
| DecalProjector (HDRP) | Пробоины, следы ожогов | Bundled |
| Compute Shaders (HLSL/DX12) | Предрасчёт response surfaces | DX12 |
| Unity Test Framework | Unit, golden, PlayMode тесты | 1.4+ |
| Unity Performance Testing | Регрессии производительности | 3.x |
| NUnit | Assert API для тестов | 3.x (bundled) |
| JsonUtility / Newtonsoft.Json | Сериализация сборок, отчётов | — |
