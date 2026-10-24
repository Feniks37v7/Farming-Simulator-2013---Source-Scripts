# FS13 – kurz z hederu w kabinie kombajnu (`threshingParticleSystem`)

Pytanie wyjściowe: *czy da się zablokować cząstki, żeby nie przenikały przez przednią szybę – np. niewidzialną kostką/kolizjerem w kombajnie?*

Pytanie drugie (po teście): *script nie pomógł, cząstki dalej przenikają – jest lepsze rozwiązanie?*

---

## 1. Dlaczego kolizjer / kostka nie zadziała (i nie zadziała nigdy)

Cząstki w FS13 to *sprite particle system* (GIANTS Engine 5). W formacie i3d 1.6 `<ParticleSystem>` ma wyłącznie parametry emisji:

`rate`, `lifespan`, `lifespanInfinite`, `maxCount`, `speed`, `speedRandom`, `tangentSpeed`, `normalSpeed`, `emitterShapeVelocityScale`, `spriteScaleX/Y`, `spriteScaleX/YGain`, `blendFactor`, `blendIn/OutFactor`, rotacja, `depthSort`, atlas tekstury — plus dzieci `<Gravity>` i `<Air>`.

**Brak**: collision shape, collision maski, raycasta, „blockera”, soft particles. Węzeł `<Dynamic>` też nie ma `collision`/`collisionMask` (te atrybuty są tylko dla brył Rigid Body i solver fizyki w ogóle nie widzi cząstek).

To nie ograniczenie FS13 – w FS15/17/19/22/25 też nie ma fizyki cząstek; tam doszło jedynie *soft particles* (wygaszanie przy geometrii na podstawie bufora głębi), czego engine 5 nie ma.

**Wniosek: jedyne, czym można sterować, to CZAS ŻYCIA cząstki (a więc i długość jej lotu) oraz kierunek emisji.**

## 2. Dlaczego wersja 1.0 skryptu nie pomogła

Wersja 1.0 liczyła nowy `lifespan` z **odległości kamery** od emitera:

```lua
alpha = (distance - margin) / (activationRadius - margin)   -- 0..1
factor = lerp(minFactor, 1, alpha)
```

Przy kamerze z kabiny dystans wynosi ~3 m, `activationRadius=6`, `margin=0.5`, więc:

```
alpha  = (3 - 0.5) / (6 - 0.5) = 0.45
factor = 0.05 + 0.95 * 0.45    = 0.48
lifespan = 5000 * 0.48         = ~2400 ms  ->  ~2.4 m lotu
```

A szyba kombajnu jest ~1.5–2.5 m od emitera. **Czyli chmura dalej dochodziła do szyby** — skrypt działał, tylko za słabo. Do tego dochodziło `smoothing`, które dodatkowo opóźniało wygaszenie.

## 3. Wersja 2.0 – lifespan liczony z geometrii, nie z kamery

Nowa reguła jest deterministyczna i niezależna od widoku:

```
droga cząstki [m] = (lifespan [ms] / 1000) * speed [m/s]
```

`speed` bierzemy z gry (`getParticleSystemAverageSpeed` – tej samej funkcji używa `Combine.lua` dla cząstek z rury), a Ty podajesz **maksymalny dystans lotu**:

```lua
maxLifespan = 1000 * maxTravelDistance / speed
targetLifespan = clamp(maxLifespan, minLifespan, originalLifespan)
```

Dla oryginalnego pliku (`normalSpeed=1`, `lifespan=5000`) i `maxTravelDistance="1.2"`:

```
targetLifespan = 1000 * 1.2 / 1 = 1200 ms   ->  cząstka ginie 1.2 m od emitera
```

Cząstka **nie ma jak** dolecieć do szyby — ginie wcześniej, w każdej kamerze, u każdego gracza, w multi identycznie. Zero zależności od tego, gdzie stoi kamera, więc zero migotania.

Dodatkowo:

- `hideWhenInsideCab="true"` – tryb „gwarancja”: gdy aktywna jest kamera `isInside` kombajnu, `lifespan` spada do `minLifespan` (150 ms), czyli efekt praktycznie znika. Twarda siatka bezpieczeństwa, jeśli nawet 1.2 m to za dużo dla Twojego modelu.
- Skrypt nie walczy z `Cutter:update` o stan emisji (`Utils.setEmittingState`) – zmienia tylko lifespan, więc nie kłóci się z logiką hederu.

Plik: **`vehicles/specializations/CabParticleFix.lua`** (UTF-16LE, jak reszta skryptów FS13).

### Konfiguracja (XML hederu, pod `<vehicle>`)

```xml
<vehicle>
    <cabParticleFix maxTravelDistance="1.2" minLifespan="150"
                    hideWhenInsideCab="false" debug="true"/>

    <threshingParticleSystem position="1.2 0 0" rotation="0 0 0"
                             file="$data/vehicles/particleSystems/threshingCutterMiddleParticleSystem.i3d"/>
</vehicle>
```

| atrybut | domyślnie | znaczenie |
|---|---|---|
| `maxTravelDistance` | `1.2` | maks. droga cząstki w metrach (promień chmury, który nie może dojść do szyby) |
| `minLifespan` | `150` | dolna granica lifespanu (ms) |
| `speedOverride` | `0` | ręcznie podana prędkość w m/s; `0` = auto z gry |
| `hideWhenInsideCab` | `false` | `true` = przy kamerze z kabiny efekt gaśnie do `minLifespan` |
| `debug` | `false` | `true` = wypisze w `log.txt`, ile ms → ile ms i przy jakiej prędkości |
| `enabled` | `true` | wyłącznik całości |
| `assumedLifespan` | `5000` | fallback, gdy `lifespanInfinite` |

Podłączenie (mod, bez ruszania `dataS`):

```lua
source("scripts/CabParticleFix.lua")
CabParticleFix.installModHook()   -- patchuje Cutter.update / Cutter.delete
```

albo `modDesc.xml`: `<extraSourceFile filename="scripts/CabParticleFix.lua"/>` + wywołanie w głównym skrypcie.

**Diagnostyka:** ustaw `debug="true"` i sprawdź `log.txt`. Powinno pojawić się:

```
CabParticleFix v2.0: mod hook installed
CabParticleFix: '.../twojHeder.xml' lifespan 5000 ms -> 1200 ms (speed 1.000 m/s, maxTravel 1.20 m)
```

Brak drugiej linii = skrypt nie widzi `threshingParticleSystems` (np. heder nie ma specjalizacji `cutter`) albo hook nie został wywołany.

## 4. Lepsze niż skrypt: poprawiony plik efektu (0 linii Lua)

Skoro i tak trzeba skrócić lot, można to zrobić raz w i3d — bez żadnego skryptu i bez kosztu na klatkę:

**`vehicles/particleSystems/threshingCutterParticleSystem_cabFriendly.i3d`** (jest w repo):

```xml
<ParticleSystem name="emitter1Ref" dynamicId="1" type="sprite"
    rate="0.02" lifespan="1200" maxCount="40"
    speed="0.001" speedRandom="0" tangentSpeed="0.15" normalSpeed="1"
    emitterShapeVelocityScale="1"
    spriteScaleX="1.2" spriteScaleY="1.2" spriteScaleXGain="0.0001" spriteScaleYGain="0.0001"
    blendFactor="0.25" blendInFactor="0.2" blendOutFactor="0.85"
    randomInitRotation="true" deltaRotateMin="-0.0015708" deltaRotateMax="0.0015708">
  <Gravity force="0 -0.02 0"/>
</ParticleSystem>
```

Co zmieniono względem oryginału i dlaczego:

| parametr | oryginał | nowy | efekt |
|---|---|---|---|
| `lifespan` | `5000` | `1200` | lot 5 m → 1.2 m (nie dochodzi do szyby) |
| `maxCount` | `80` | `40` | mniej sprite'ów = mniej „ściany kurzu” przy kabinie, tańsze |
| `rate` | `0.015` | `0.02` | gęstość chmury zostaje podobna przy krótszym życiu |
| `emitterShapeVelocityScale` | `0.85` | `1` | cząstki nie zostają w tyle za kombajnem, czyli **nie dryfują do tyłu w stronę kabiny** |
| `blendOutFactor` | `0.8` | `0.85` | łagodniejsze zniknięcie, mniej widoczny „koniec” chmury |
| `Gravity` | `0 -0.002 0` | `0 -0.02 0` | delikatne opadanie zamiast wiszenia |

W hederze:

```xml
<threshingParticleSystem position="1.2 0 0" rotation="0 0 0"
                         file="particleSystems/threshingCutterParticleSystem_cabFriendly.i3d"/>
```

(Jeśli plik leży w folderze moda, ścieżka bez `$data`. W pliku są odwołania `$data/shaders/particleSystemShader.xml` i `$data/vehicles/particleSystems/threshingParticle.png` — sprawdź w swojej instalce, czy shader nazywa się dokładnie tak; jeśli leży gdzie indziej, popraw `<File>`.)

## 5. Najlepszy efekt wizualny: zmień KIERUNEK emisji

To najczęściej pomijany, a najskuteczniejszy trik. Kierunek lotu cząstek **nie** wynika z `normalSpeed` w świecie, tylko z **lokalnej osi Y płaszczyzny emitera** (`n="0 1 0"` w `pPlaneShape2`). Domyślnie chmura leci „w górę” hederu, czyli dokładnie w stronę szyby.

Dwie zmiany w XML hederu (albo w GE, na węźle emitera):

```xml
<!-- kurz leci do przodu i w dół, z dala od kombajnu -->
<threshingParticleSystem position="1.2 -0.3 0" rotation="70 0 0"
                         file="particleSystems/threshingCutterParticleSystem_cabFriendly.i3d"/>
```

- `rotation` wokół X przechyla płaszczyznę → wektor emisji przestaje być pionowy i idzie **przed heder / w bok**, a nie pod szybę.
- `position` w dół (~-0.3 m) zabiera punkt startu spod poziomu, na którym zaczyna się szkło.
- wartości dobierz testowo w GIANTS Editorze (podgląd dynamiki) lub w grze – zależy od orientacji korzenia hederu.

Po przechyleniu emitera możesz nawet wrócić do dłuższego `lifespan` (np. 2500–3000 ms) — kurz będzie leciał **przed** maszynę i nie będzie miał jak wejść do kabiny, a wizualnie będzie bujniejszy niż wariant z samym skróconym życiem.

## 6. Kolejność wdrażania (od najskuteczniejszego)

1. **Kierunek emisji** (`rotation`/`position` w XML hederu) – największa różnica, zero kodu.
2. **Poprawiony i3d** (`threshingCutterParticleSystem_cabFriendly.i3d`) – deterministycznie krótki lot.
3. **Skrypt v2** (`CabParticleFix`) – gdy chcesz mieć to pod kontrolą z XML i/lub twardą gwarancję (`hideWhenInsideCab`).
4. Wszystkie trzy można łączyć; nie kolidują ze sobą.

## 7. Czego nie warto robić

- **Niewidzialna kostka z Rigid Body / collisionMask** – solver fizyki nie dotyka cząstek.
- **`nonRenderable` occluder** – nie rysuje się, więc nie zapisuje głębi, więc nie zasłania.
- **`clipDistance` / `distanceBlending`** – `clipDistance` to dystans rysowania całego systemu; `distanceBlending` pojawił się w późniejszych silnikach (FS17+) i w FS13 nie daje soft-particle fade.
- **Ukrywanie szyby** – psuje model, a kurz dalej „jest” w środku.
- **Regulacja tylko odległością kamery** (wersja 1.0) – daje zbyt łagodne wartości i migocze przy ruchu kamery.

## 8. Sprawdzenie po wdrożeniu

1. `log.txt` – brak błędów, przy `debug="true"` linia `lifespan ... -> ...`.
2. Widok z kabiny – kurz ginie przy hederze, nie ma smug na szybie.
3. Widok zewnętrzny – chmura nadal widoczna (krótsza) nad hederem.
4. Test w multi – efekt jest kliencki, więc każdy gracz widzi to samo, a serwer nie jest obciążony.
