# Compilar Zink/OSMesa para ARM64 (ARMv8.0-A) — o renderizador do PojavLauncher

Este pacote reproduz, de forma modernizada, o build oficial do
**PojavLauncherTeam/osmesa-zink-builder** (repo hoje arquivado) que gera o
**OSMesa com Zink** — o renderizador que o PojavLauncher usa para rodar o
Minecraft Java (OpenGL) em cima de **Vulkan** no Android ARM64.

## O que e Zink aqui
- **Zink** = driver Gallium do Mesa que traduz **OpenGL -> Vulkan**.
- **OSMesa** = off-screen Mesa; o Pojav carrega `libOSMesa_8.so` e faz o GL do
  jogo sair por Zink -> Vulkan do aparelho. Por isso `-Dgallium-drivers=zink`,
  `-Dosmesa=true`, `-Dvulkan-drivers=` (vazio: Zink usa o Vulkan do sistema,
  nao compila ICD).
- O patch do Pojav troca o carregamento do Vulkan por um ponteiro passado em
  runtime: `screen->loader_lib = (void*) strtoul(getenv("VULKAN_PTR"), ...)`.
  Ou seja, o Pojav injeta o handle do `libvulkan` via a variavel de ambiente
  **`VULKAN_PTR`** — isso nao e um bug, e o jeito que o loader funciona dentro
  do app.

## Arquivos
| Arquivo | Para que |
|---|---|
| `.github/workflows/build-zink-aarch64.yml` | Workflow que compila no GitHub Actions |
| `build-crossfile.in` | Cross-file do meson (NDK aarch64, `-march=armv8-a`) |

O `mesa-zink.diff` **real** e baixado pelo workflow direto do repo do Pojav
(nao vem copiado aqui para garantir que e o patch original, nao inventado).

## ARMv8.0-A ("v8.0")
O cross-file usa `aarch64-linux-android24-clang` com **`-march=armv8-a`**
(baseline ARMv8.0-A, sem exigir extensoes mais novas). Isso maximiza a
compatibilidade com aparelhos antigos — inclusive a sua PowerVR Rogue.

## Como rodar
1. Suba estes arquivos num repositorio seu (mantendo os caminhos).
2. GitHub -> **Actions** -> “Build Zink/OSMesa ARM64” -> **Run workflow**.
3. Baixe o artifact **`libzink_aarch64`** (contem `lib/libOSMesa_8.so`).
4. No Pojav, substitua o renderizador OSMesa/Zink pelo `.so` gerado.

### Rodar no Termux (em vez do CI)
```bash
pkg install meson ninja ccache clang python git
pip install mako
# instale o Android NDK e exporte ANDROID_NDK_ROOT
envsubst < build-crossfile.in > build-crossfile
git clone --depth 1 --branch mesa-23.3.6 https://gitlab.freedesktop.org/mesa/mesa.git
cd mesa
curl -fsSL -o mesa-zink.diff https://raw.githubusercontent.com/PojavLauncherTeam/osmesa-zink-builder/master/mesa-zink.diff
git apply --reject --whitespace=fix mesa-zink.diff || true
meson setup build-android --prefix=/tmp/zink --cross-file ../build-crossfile \
  -Dplatforms=android -Dplatform-sdk-version=25 -Dandroid-stub=true \
  -Dllvm=disabled -Dxlib-lease=disabled -Degl=disabled -Dgbm=disabled \
  -Dglx=disabled -Dopengl=true -Dosmesa=true -Dvulkan-drivers= \
  -Dgallium-drivers=zink -Dshared-glapi=false -Dandroid-libbacktrace=disabled \
  -Dbuildtype=release
ninja -C build-android install
```

## Proveniencia
| Item | Origem |
|---|---|
| Recipe / cross-file / diff | `github.com/PojavLauncherTeam/osmesa-zink-builder` (branch `master`, arquivado em 2025-06-20) |
| Mesa | `gitlab.freedesktop.org/mesa/mesa`, tag `mesa-23.3.6` (ajustavel em `MESA_REF`) |

## Observacoes / cuidados
- **Nao foi compilado aqui** (este ambiente nao tem NDK/meson prontos) — a
  compilacao roda no GitHub Actions ou no seu Termux; valide no hardware.
- O patch foi escrito para a era do **Mesa 23.x**. Se voce apontar `MESA_REF`
  para master/24.x, alguns hunks caem em `.rej` e e preciso reaplica-los a mao
  (a logica do `VULKAN_PTR`, o OSMesa-sem-softpipe e o `android_stub` sao os
  pontos que mais mudam entre versoes).
- `-Dvulkan-drivers=` fica **vazio de proposito**: Zink nao e um ICD; ele usa
  o `libvulkan` do aparelho. Para a sua PowerVR, o que importa e ter um Vulkan
  1.1+ funcional no dispositivo.
- Saida esperada: `/tmp/zink/lib/libOSMesa_8.so` (ELF `AArch64`).
