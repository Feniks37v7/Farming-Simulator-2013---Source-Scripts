# glowShader.xml – FS13 (wariant „slider”)

Port shadera **`glowShader.xml` z Farming Simulator 25** (`<CustomShader version="5">`)
do formatu **Farming Simulator 2013** (`<CustomShader version="2">` – GIANTS Engine 5.0).

Zgodnie z zamówieniem przeniesiony został **tylko wariant `slider`**. Pozostałe warianty
z FS25 (`sineFloatingAnimation`, `multiType`, `staticLight`, `blink`, `loadingCircle`,
`loadingBar`, `fresnel`, `billboard`, `pulseNoise`, `customEmissiveMap`…) nie są obsługiwane –
format `version="2"` nie ma w ogóle mechanizmu `<Variations>`, więc jeden plik = jeden wariant.

---

## 1. Co robi shader

Dokładnie to samo, co wariant `slider` w FS25:

```glsl
// FS25
float mask = float(In.vs.albedoMapTexCoord2.x <=
                   (object.sliderPos.x + frac(object.sliderPos.y * cTime_s)));
globals.gAlpha            = mask;
globals.gEmissiveColor.w *= mask;
```

Świecenie (glow) widać tylko na tej części mesha, której współrzędna „suwaka” jest
**mniejsza lub równa** `sliderPos.x + frac(sliderPos.y * czas)`. Reszta paska jest zgaszona
(czarna / przezroczysta). Efekt: pasek świetlny / wskaźnik „napełnia się” wraz ze wzrostem
`sliderPos.x`, a przy `sliderPos.y > 0` sam się przewija w czasie.

### Mapowanie FS25 → FS13

| FS25 | FS13 (ten plik) | Uwagi |
|---|---|---|
| `<CustomShader version="5">` | `<CustomShader version="2" classRequirement="">` | format FS13 |
| `<Parameters>` na poziomie pliku | `<Parameters>` wewnątrz `<LodLevel startDistance="0">` | tak jest w shaderach FS13 |
| `sliderPos` typu `float2` (x = pozycja 0..1, y = prędkość przewijania) | `sliderPos` typu `float4` (x, y – jak wyżej, z/w – rezerwa) | w FS13 sprawdzony jest tylko `float4` |
| `lightControl` (float, intensywność) | `lightControl` typu `float4` (x = intensywność) | jak wyżej |
| `cTime_s` | `time.y` | tak liczy czas shader `uvScrollShader.xml` z FS13 |
| `In.vs.albedoMapTexCoord2.x` (drugi zestaw UV) | `getTexCoords(In, ALBEDOMAP_TEXCOORD).x` | FS13 nie ma drugiego zestawu UV – szczegóły w pkt 4 |
| `globals.gAlpha`, `globals.gEmissiveColor.w` | maska mnożona przez `oColor` w `FINAL_POS_FS` | FS13 udostępnia w tej pozycji zmienną `oColor` |
| wariant `slider` z `<Variations>` | cały plik = wariant `slider` | FS13 nie ma wariantów |

Wykorzystane pozycje wstrzyknięć (`MATERIALINFO`, `VS_OUTPUT`, `POST_SET_TEXCOORDS_VS`,
`FINAL_POS_FS`) oraz zmienne (`oColor`, `time.y`, `getTexCoords(...)`, `ALPHA_BLENDED`)
to konstrukcje występujące w shaderach FS13 (`uvScrollShader.xml`, `scrollUVShader.xml`,
`emissiveBillboardShader.xml`, `vehicleShader.xml`, mody FS13).

---

## 2. Instalacja

Plik wrzucamy do moda (np. `twojMod/shaders/glowShader.xml`) albo do `data/shaders/` w grze,
a następnie podpinamy go w pliku `.i3d`:

```xml
<Files>
    <!-- ... -->
    <File fileId="100" filename="shaders/glowShader.xml" relativePath="true"/>
</Files>

<Materials>
    <!-- ... -->
    <Material name="glowSlider" materialId="50" ambientColor="1 1 1" alphaBlending="true" customShaderId="100">
        <!-- tekstura swiecenia (obowiazkowo) -->
        <Emissivemap fileId="72" wrap="false"/>
        <!-- wartosci startowe parametrow -->
        <CustomParameter name="lightControl" value="1 0 0 0"/>
        <CustomParameter name="sliderPos"    value="0 0 0 0"/>
    </Material>
</Materials>
```

* `customShaderId` = `fileId` wpisu z shaderem,
* `<Emissivemap>` jest potrzebna, bo to ona daje efekt świecenia (glow),
* `alphaBlending="true"` – dla pasków świetlnych/overlayów (wtedy zgaszona część jest
  w pełni przezroczysta); dla materiału nieprzezroczystego zgaszona część będzie czarna,
* `<CustomParameter>` w FS13 jest obsługiwany (przykład z FS13: `alphaScale`, `exhaustingSystem`).

---

## 3. Sterowanie z Lua (FS13)

FS13 ma funkcję `setShaderParameter(node, "nazwa", x, y, z, w, false)` – wywołania można
znaleźć m.in. w `BaleLoader.lua`, `Combine.lua`, `SowingMachine.lua`, `Trailer.lua`
(„`uvScrollSpeed`”, „`scrollPosition`”, „`partScale`”).

```lua
-- w load() specjalizacji:
self.glowNode = Utils.indexToObject(self.components,
                   getXMLString(xmlFile, "vehicle.glowIndicator#node"));

-- w update()/updateTick():
local fillLevel = 0.0; -- np. poziom paliwa / nawozu, 0..1
setShaderParameter(self.glowNode, "sliderPos", fillLevel, 0, 0, 0, false);
setShaderParameter(self.glowNode, "lightControl", 1, 0, 0, 0, false);  -- 0 = zgaszone
```

* `sliderPos.x` = pozycja suwaka `0..1` (np. poziom paliwa),
* `sliderPos.y` = prędkość automatycznego przewijania (np. `0.25`); można ją ustawić raz,
  a „animacją” zajmie się shader,
* `lightControl.x` = `1` normalnie, `0` – światło zgaszone (np. gdy silnik nie pracuje),
  wartości pomiędzy przygaszają efekt.

---

## 4. Wymagania mesha i dostosowanie osi „suwaka”

W FS25 maska liczona jest z **drugiego zestawu UV** (`uv1`), które w FS13 praktycznie nie
występuje (meshe FS13 mają jeden zestaw UV). Dlatego w tym porcie osią suwaka jest
**współrzędna X podstawowego UV** – mesh powinien mieć tak rozłożone UV, żeby `u` biegło
wzdłuż paska (0 na początku, 1 na końcu).

Jeśli chcesz użyć innej osi, zmieniasz jedną linię w `POST_SET_TEXCOORDS_VS`:

```glsl
float2 mSliderUV = getTexCoords(In, ALBEDOMAP_TEXCOORD);
Out.mSliderCoord = mSliderUV.x;        // <- tu np. mSliderUV.y
```

Wersja „pozycyjna” (gdy UV nie da się tak rozłożyć – oś obiektu, np. X):

```glsl
Out.mSliderCoord = saturate((In.position.x - 0.0) / 1.0);   // zakres dopasuj do mesha
```

---

## 5. Świadome różnice względem FS25

| Temat | FS25 | Ten port FS13 |
|---|---|---|
| Drugi UV | maska z `uv1` | współrzędna z `uv0` (patrz pkt 4) |
| `lightControl` | skaluje **tylko** emisję (diffuse bez zmian) | skaluje cały kolor końcowy (`oColor`) – dla pasków świetlnych wizualnie to samo; przy `lightControl.x = 0` gaśnie również kolor bazowy |
| Maska | emisja (`gEmissiveColor.w`) + alpha | `oColor.rgb` + `oColor.a` w `FINAL_POS_FS` |
| Warianty | `<Variations>` | brak mechanizmu – plik obsługuje wyłącznie `slider` |
| Krawędź maski | porównanie `<=` na piksel | porównanie `<=` na interpolantach (efekt identyczny przy liniowym UV) |

Jeśli zależy Ci na tym, żeby w strefie „zgaszonej” kolor bazowy (diffuse) **pozostał widoczny**,
przenieś blok z `FINAL_POS_FS` do `EMISSIVE_FS` (pozycja ta istnieje w FS13 – używa jej np.
shader `DynamicExhaustingSystemShader.xml` z MoreRealistic DLC, operując na `oColor`).

---

## 6. Rozwiązywanie problemów

1. **`Failed to create vertex/fragment shader` / błąd shadera w log.txt**
   Silnik FS13 nie zna którejś pozycji lub nazwy zmiennej. Najpierw sprawdź dwie linie
   na początku bloku `FINAL_POS_FS`:

   ```glsl
   #define GS_SLIDER_COORD     In.vs.mSliderCoord
   #define GS_SLIDER_THRESHOLD In.vs.mSliderThreshold
   ```

   i w razie potrzeby zmień prefiks `In.vs.` na `In.` (albo odwrotnie). Jeśli mimo to
   shader się nie kompiluje – prześlij mi swój folder `data/shaders/` z FS13
   (albo sam plik bazowego shadera z pozycją `EMISSIVE_FS`), dopasuję nazwy 1:1.

2. **Shader działa, ale nic nie świeci**
   * materiał musi mieć `<Emissivemap>` – bez niej nie ma czego maskować,
   * sprawdź `sliderPos.x` (0 = zgaszone, 1 = pełne wypełnienie),
   * sprawdź `lightControl.x > 0`.

3. **Świeci cały pasek niezależnie od `sliderPos`**
   Znaczy to, że maska nie dochodzi do pixel shadera – patrz punkt 1;
   ewentualnie UV mesha nie ma rozkładu 0..1 wzdłuż paska (patrz pkt 4).

4. **Zgaszona część jest czarna, a chciałbym przezroczystą**
   Ustaw `alphaBlending="true"` na materiale.

---

## 7. Pliki

* `glowShader.xml` – przeniesiony shader (wariant `slider`) w formacie FS13 `version="2"`.
* `glowShader_README.md` – ten opis.
