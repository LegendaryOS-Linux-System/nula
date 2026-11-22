# nula

Menedżer pakietów i **łącznik H# ↔ Z#** (opcjonalnie Hacker Lang / Hacker Lang++), napisany w 100% w Z#.
Dla Z# i Hacker Lang++ to „cargo”: zamiast `Cargo.toml` jest `nula.hk`, ale **bibliotek się w nim nie deklaruje** — nula wykrywa je w kodzie, instaluje i zapisuje `nula.lock`.

```
projekt/
├── src/           kod H#
├── mod/           kod Z#
├── source-code/   opcjonalnie: Hacker Lang / Hacker Lang++
├── build/         <nazwa> (binarka) · libs/ (biblioteki) · cache/ (reszta)
├── nula.hk
└── nula.lock
```

Kod nuli: `mod/` (Z#), `website/` (HackerScript, `mode:website` + `Bit.hk`), `nula.hk`.

## Szybki start
```bash
nula new narzedzie --lang h#,z#     # H# i Z# razem; --lang z# | h# | hlpp | hl (po przecinku)
cd narzedzie
nula build --release                # zależności → most → h# / z# / hlpp
nula run
```
Zakres: czysty H# ma już `bit` — nula odmówi (`--force` wymusza). Sens ma dla Z#, Hacker Lang(++) albo H# razem z nimi.

## Skąd nula zna zależności
| Język | Import w kodzie | Zależność |
|---|---|---|
| H# | `use "bit -> mold" from "mold"` | pakiet `mold` z indeksu |
| H# | `use "github.com/u/repo" from "r"` | git `https://github.com/u/repo` |
| H# | `use "std -> …"`, `python -> …` | wbudowane / tylko raport |
| Z# | `import Colors.Palette;` | pakiet `Colors` — o ile to nie `Std`, `Workspace`, wbudowany moduł ani własny plik/katalog/`namespace` z `mod/` |
| HL/HL++ | `# <bit/nazwa:^1.2>`, `# <github/u/repo:v2>` | pakiet / git (aliasy `virus/`, `community/`; `main/`, `std/` wbudowane) |

Źródła: `[pins]` w `nula.hk` → indeksy (`$NULA_INDEX`, `[registry] index`, na końcu indeks bit.io — te same biblioteki co w `bit`) → adres z kodu. Pobieranie przez `git`/`curl` (Std.Net w Z# nie ma TLS). Wspólny store: `~/.hackeros/nula/store/<nazwa>/<commit12>/`; w projekcie `build/libs/<nazwa>/current` to link — taki układ czytają kompilator H# i `hlpp` (przez `BIT_HOME`), więc `use "bit -> x"` i `# <bit/x>` działają bez zmian.

`[pins]` tylko **zawęża**: `-> colors => ^1.2`, `-> x => git https://github.com/u/x v2`, `-> y => path ../y`, `-> z => rev <commit>`.

## nula.hk
Gramatyka jak w hk-parser (bit): `!` komentarz całej linii, `[sekcja]`, `-> klucz => wartość`, `-->` zagnieżdżenie, `[a, b]`.
```
[package]   name, version, description, license, lang
[build]     main (h# | z#), profile (release | debug), backend (Z#: c | llvm | gcc)
[layout]    src, mod, source-code, build          (domyślnie jak wyżej)
[bridge]    transport (auto | static | process)
[toolchain] h# => 0.9                             (pobierz h#, gdy go brak — jak bit)
[registry]  index => [url, ...]
[hooks]     pre-build, post-build
[pins]      zawężenia wersji
```

## Most H# ↔ Z#
Funkcję oznaczasz komentarzem nad nią (żaden język nie widzi zmiany składni):
```
;; nula:export                                   # nula:export
pub fn add(a: int, b: int) -> int is             func greet(string $who): string { … }   (namespace w pliku)
```
Typy: `int`, `float`, `bool`, `string`, `void`. nula zapisuje IDL (`build/cache/bridge/bridge.json`) i generuje:
- **H# → Z#**: `import Workspace.HBridge;` → `HBridge.add(2, 3)`. Same typy skalarne → **static** (`h# --emit lib` → `libhside.a`, `extern "hside"`, `[extern.hside]` w wygenerowanym `Z#.toml`). Są stringi → **process** (`build/libs/hside <fn> <arg>…`).
- **Z# → H#**: `mod nula_zbridge` w H# (plik `src/nula_zbridge.h#`, w `.gitignore`), zawsze **process** (`build/libs/zside`) — Z# nie emituje jeszcze biblioteki linkowalnej.

`nula bridge --check` pokazuje, co zostanie wyeksportowane. Wygenerowane `Z#.toml` leży w `build/cache/z/` (członkowie workspace = `namespace` z plików `mod/`).

## Polecenia
`new init install update outdated tree deps pin search build run check test clean bridge script doctor version` — `nula help`.

## Budowanie nuli
Pierwszy raz (nuli jeszcze nie ma): `z# build --release -o nula` w tym katalogu (używa `Z#.toml`), potem `nula build`.
Testy: `nula test` (tests/*.zsharp, wyniki złote w `.out`).

## Uczciwie: stan
- **Nic z tego nie zostało uruchomione.** Stage0 Z# nie było dostępne w czasie pisania; kod powstał z `docs/SPEC.md`, `STDLIB.md`, `wrapper/main.zsharp` i stdlib. Pierwszy `z# check` pokaże literówki i miejsca, w których Z# jeszcze nie ma funkcji (to też zamierzony test Z#). Pliki `.out` w `tests/` policzono ręcznie.
- Założenia do sprawdzenia: `extern` wewnątrz pliku z `namespace`; `\$` w napisach; `Os.walk`/`Os.list` zwracają to, co opisuje STDLIB.md.
- **Warstwy generowane po stronie H#** (`hside_main.h#`, `nula_zbridge.h#`) używają `env::args`, `env::get`, `proc::shell`, `conv::str_to_int/f64`, `str::replace/trim` — nazwy z README H#, ale sygnatur nie sprawdzono. `nula check` zgłosi to przez `h# check`.
- Transport process: parametry `float` po stronie H# jeszcze nieobsługiwane; w IDL tylko funkcje top-level.
- Pozostałe ograniczenia: jedna kopia pakietu w drzewie (konflikt zakresów = błąd), brak podpisów, `nula pin` przy istniejącym `[pins]` przepisuje `nula.hk` bez komentarzy, `outdated` patrzy tylko na tagi semver.
- Przeglądarkowa strona (`website/`) nie była budowana ani otwierana.
