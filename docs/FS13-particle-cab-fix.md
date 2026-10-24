# FS13 – kurz z hederu w kabinie kombajnu (`threshingParticleSystem`)

Pytanie: **czy da się zablokować cząstki, żeby nie przenikały przez przednią szybę – np. niewidzialną kostką / kolizjerem w kombajnie?**

## 1. Krótka odpowiedź

**Nie – kostka z kolizją (rigid body) tego nie zatrzyma.**

Cząstki w FS13 to *sprite particle system* (GIANTS Engine 5). Ich symulacja nie ma **żadnej** fizyki kolizji:

- `<ParticleSystem>` w i3d 1.6 ma tylko: `rate`, `lifespan`, `maxCount`, `speed`, `speedRandom`, `tangentSpeed`, `normalSpeed`, `emitterShapeVelocityScale`, `spriteScaleX/Y`, `…Gain`, `blendFactor`, `blendIn/OutFactor`, rotację, atlas tekstury.
- Jedyne „pola sił” to dzieci `<Gravity>` i `<Air>` – nie ma odpowiednika collision shape, collision maski, raycasta ani „blockera”.
- `<Dynamic>` (węzeł emitera) nie ma atrybutów `collision` / `collisionMask` – te dotyczą wyłącznie brył fizycznych (`TransformGroup`/`Shape` z Rigid Body). Nawet gdybyś je ustawił, solver fizyki ich nie widzi, bo cząstki nie są bryłami.
- To nie jest ograniczenie „FS13 tylko” – sprite particles w FS15/17/19/22/25 też nie kolidują z geometrią (tam po prostu doszły *soft particles*, czyli wygaszanie przy styku z geometrią **na podstawie bufora głębi**, a nie fizyka).

Więc: kolizjer = 0 efektu. Cząstka i tak przeleci przez szybę, dach i błotnik.

## 2. Skąd w ogóle bierze się efekt „kurz w kabinie”

To dwie niezależne rzeczy, często nakładające się:

1. **Cząstki naprawdę wlatują w objętość kabiny** – emiter siedzi na hederze, `lifespan="5000"` ms przy `speed="0.001"` (≈1 m/s) daje ~5 m drogi – czyli słupek kurzu spokojnie sięga poziomu szyby i wpada do środka. Zero kolizji = zero zatrzymania.
2. **Szyba nie zasłania cząstek na ekranie** – szyby w FS13 są materiałem alpha-blended, który **nie zapisuje głębi**. Cząstki rysowane są więc na wierzchu szyby, nawet gdy „w rzeczywistości” są tuż za nią. Z zewnątrz tego nie widać, z kabiny widać brud na całej szybie.

Punkt 2 jest powodem, dla którego „niewidzialna kostka z depth-write” też nie zadziała: w FS13 nie ma w materiale przełącznika *depth write* dla przezroczystych materiałów, `nonRenderable="true"` nic nie rysuje (więc nic nie zapisuje do bufora głębi), a zwykły opaque box byłby po prostu widoczną bryłą. (W nowszych wersjach robi się to shaderem „invisible occluder”, w FS13 tego nie ma.)

## 3. Co da się wdrożyć (3 warianty, można łączyć)

### A. Tuning samego efektu (zero kodu, najszybsze)

W `threshingCutterMiddleParticleSystem.i3d`:

```xml
<ParticleSystem ... lifespan="5000" speed="0.001" normalSpeed="1" tangentSpeed="0.2" emitterShapeVelocityScale="0.85" ...>
  <Gravity force="0 -0.002 0"/>
</ParticleSystem>
```

- `lifespan` 5000 → **1500–2500** – słupek kurzu robi się krótszy i nie dolatuje do szyby.
- `Gravity force` → np. `0 -0.01 -0.002` – kurz opada zamiast wisieć i cofać się w stronę kabiny.
- `tangentSpeed` – zmniejsz, żeby cząstki nie „uciekały” do tyłu w stronę kabiny.
- W XML hederu przesuń/obróć węzeł: `<threshingParticleSystem position="…" rotation="…"/>` – np. lekko do przodu i w dół (`rotation` wokół X), żeby strugi szły przed heder, a nie pod szybę.

To zwykle wystarcza, żeby problem przestał być widoczny, ale nie jest „inteligentne” – działa tak samo dla każdego widoku.

### B. Ograniczenie czasu życia cząstek zależnie od odległości kamery (skrypt, polecane)

Dokładnie ta sama technika, której GIANTS używa w `Combine.lua` dla cząstek z rury wyładowczej:

```lua
local lifespan = math.min(v.originalLifespan, (self.pipeUnloadingDistance + self.pipeParticleSystemExtraDistance) / v.speed)
setParticleSystemLifespan(v.geometry, lifespan, true)
```

Czyli: **skoro nie można cząstki zatrzymać, to można jej skrócić życie, żeby zginęła zanim doleci do kamery.** Efekt:

- widok z zewnątrz (kamera daleko) → pełny kurz, nic się nie zmienia,
- widok z kabiny (kamera ~2–4 m od hedera) → cząstki żyją ułamek oryginalnego czasu i giną przy podajniku, nigdy nie dochodzą do szyby.

Implementacja jest gotowa w tym repo: **`vehicles/specializations/CabParticleFix.lua`** (pisana w konwencji FS13, UTF-16LE jak reszta plików gry).

### C. Twarde wyłączenie efektu przy kamerze z kabiny

W `CabParticleFix` jest opcja `onlyWhenCameraInside="true"` – skrypt sprawdza, czy aktywna kamera to kamera `isInside="true"` kombajnu (`getCamera()` vs `vehicle.cameras[i].cameraNode`, wzorzec z `Steerable.lua`/`VehicleCamera.lua`) i tylko wtedy dławi efekt. Wtedy np. podczas jazdy w kabinie kurz praktycznie znika, a z zewnątrz zostaje nietknięty.

## 4. Instalacja wariantu B/C

### Plik

`vehicles/specializations/CabParticleFix.lua` – jest już w repo (skopiuj do `dataS/scripts/vehicles/specializations/` w instalce gry albo do folderu moda).

### Konfiguracja w XML hederu (pod `<vehicle>`)

```xml
<vehicle>
    <!-- ... -->
    <cabParticleFix activationRadius="6" margin="0.5" minFactor="0.05"
                    smoothing="4" onlyWhenCameraInside="false" enabled="true"/>

    <threshingParticleSystem position="1.2 0 0" rotation="0 0 0"
                             file="$data/vehicles/particleSystems/threshingCutterMiddleParticleSystem.i3d"/>
</vehicle>
```

| atrybut | domyślnie | znaczenie |
|---|---|---|
| `activationRadius` | `6` | od tej odległości kamery (m) cząstki żyją pełny `lifespan` |
| `margin` | `0.5` | bufor przy kamerze, w którym cząstki są niemal od razu usuwane |
| `minFactor` | `0.05` | minimalny ułamek oryginalnego `lifespan` (5%) |
| `smoothing` | `4` | płynność przejścia (0 = natychmiastowe, 1–10 = łagodne) |
| `onlyWhenCameraInside` | `false` | `true` = tylko przy kamerze z kabiny |
| `enabled` | `true` | wyłącznik całości |
| `assumedLifespan` | `5000` | fallback, gdy `lifespanInfinite` |

Opcjonalnie dodatkowe emitery (np. z kombajnu) po indeksie węzła z `<i3dMappings>`/drzewa i3d:

```xml
<cabParticleFix activationRadius="5">
    <extraParticleSystem index="0|3|1"/>
</cabParticleFix>
```

### Podłączenie – dwie drogi

**Droga 1 (mod, bez ruszania plików gry – polecane):** w głównym skrypcie moda:

```lua
source("CabParticleFix.lua")   -- ścieżka w Twoim modzie
CabParticleFix.installModHook()
```

`installModHook()` robi `Utils.overwrittenFunction` na `Cutter.update` / `Cutter.delete`, więc nie trzeba nic rejestrować w typach pojazdów. Skrypt działa per-heder i tylko po stronie klienta (efekt czysto graficzny – **bez synchronizacji sieciowej, bez wpływu na savegame**).

**Droga 2 (edycja `dataS` – jeśli i tak grzebiesz w źródłach):** w `dataS/scripts/vehicles/Vehicle.lua` dopisz `source("dataS/scripts/vehicles/specializations/CabParticleFix.lua")` obok istniejącego `source(...VehicleSetBeaconLightEvent.lua)`, a w definicji typu dodaj specjalizację:

```xml
<type name="cutter" className="Vehicle" filename="dataS/scripts/vehicles/Vehicle.lua">
    <specialization name="attachable" />
    <specialization name="cutter" />
    <specialization name="cabParticleFix" />
    <specialization name="mountable" />
</type>
```

(zawsze **po** `cutter` – wtedy `threshingParticleSystems` są już wczytane; jeśli nazwa specjalizacji nie zostanie rozpoznana przez loader typów, użyj Drogi 1).

## 5. Jak sprawdzić, że działa

1. Uruchom grę, wsiądź do kombajnu z hederem, włącz koszenie.
2. Widok z kabiny (`isInside`) – kurz powinien ginąć przy podajniku / przed szybą.
3. Przełącz na kamerę zewnętrzną – słupek kurzu wraca do pełnej długości.
4. `log.txt`: brak błędów typu `attempt to call global 'setParticleSystemLifespan'` (funkcja istnieje w FS13 – używa jej `Combine.lua`) i `CabParticleFix: mod hook installed` przy Drodze 1.

## 6. Czego NIE warto robić

- **Niewidzialna kostka z Rigid Body / collisionMask** – solver fizyki nie dotyka cząstek.
- **`nonRenderable` occluder** – nie rysuje się, więc nie zapisuje głębi, więc nie zasłania.
- **Przestawianie `clipDistance` / `distanceBlending`** – `clipDistance` to dystans rysowania całego systemu, `distanceBlending` w FS13 nie robi soft-particle fade (można sprawdzić eksperymentalnie, ale nie licz na efekt).
- **Ukrywanie szyby / kabiny** – to psuje model, a kurz dalej „jest” w środku.

## 7. Jeśli chcesz iść dalej

Ten sam mechanizm da się rozszerzyć na inne efekty (kurz z kół `driveGroundParticleSystems`, wydech, oprysk): wystarczy dodać ich nody przez `<extraParticleSystem index="…"/>` – logika ograniczania `lifespan` jest niezależna od tego, czyj to pojazd.
