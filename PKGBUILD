# Maintainer: Maxime Gauduin <alucryd@archlinux.org>
# Maintainer: Christian Heusel <gromit@archlinux.org>
# Contributor: George Sofianos <george@sofianos.dev>
# Contributor: Michele Balistreri <michele@bitgamma.com>
# Contributor: Caleb Maclennan <caleb@alerque.com>

pkgbase=lemonade
pkgname=(
  lemonade-server
  lemonade-desktop
)
pkgdesc='Lemonade helps users discover and run local AI apps by serving optimized LLMs right from their own GPUs and NPUs'
pkgver=2026.40.0
pkgrel=1
_tag=candidate-v${pkgver}
arch=(x86_64)
url=https://github.com/lemonade-sdk/lemonade
license=(Apache-2.0)
options=(!debug)
makedepends=(
  cargo
  cargo-tauri
  cli11
  cmake
  cpp-httplib
  gdk-pixbuf2
  git
  gtk3
  libcap
  libdrm
  libwebsockets
  mbedtls
  ninja
  nlohmann-json
  nodejs-lts-krypton
  npm
  systemd-libs
  unzip
  webkit2gtk-4.1
  zstd
)
source=(
  git+${url}.git#tag=${_tag}
  lemonade-sysusers.patch
  lemonade-web-app.patch
)
b2sums=('64a2c730886b667c7cea2fa3e27513ffdcb036a1d4aabcaa1624074a12ee7d5d6ac5420bda50071a90b46c0deb102d832cd41f7e5f6c4f13ebca03873b0cd3d7'
        '5b8174e0d88a825eb69e96f82140bae27a081665b472efaac0920829963f7790b9a1b9c9f5d787901ff4278eb31dc389e3e09dfad61fda490c7ceea355c51e14'
        '229a2de3961618b00fffbaf0fdec40b121a5b900542099d82c27b486c6368ab11237128da3e4d0ceeef1c71e37325857667a37459dd7f82572be1f86cf57eae8')

prepare() {
  cd lemonade
  patch -Np1 -i ../lemonade-sysusers.patch
  patch -Np1 -i ../lemonade-web-app.patch

  printf '%s\n' "${pkgver}" > .version

  cd src/app
  export RUSTUP_TOOLCHAIN=stable
  npm ci --ignore-scripts
}

build() {
  local cmake_options=(
    -S lemonade
    -B build
    -G Ninja
    -D CMAKE_BUILD_TYPE=None
    -D CMAKE_INSTALL_PREFIX=/usr
    -W no-dev
  )
  cmake "${cmake_options[@]}"
  cmake --build build

  cd lemonade/src/app
  export RUSTUP_TOOLCHAIN=stable
  cargo tauri build --no-bundle
}

package_lemonade-server() {
  depends=(
    curl
    glibc
    jq
    libcap
    libgcc
    libstdc++
    libwebsockets
    mbedtls
    systemd-libs
    unzip
    zstd
  )
  optdepends=(
    'fastflowlm: FLM support'
    'llama-cpp: Use system llama.cpp'
    'xdg-utils: Open the web app when no Chromium-based browser is available'
  )
  backup=(etc/default/lemond)

  DESTDIR="${pkgdir}" cmake --install build
}

package_lemonade-desktop() {
  depends=(
    cairo
    gdk-pixbuf2
    glib2
    glibc
    gtk3
    libgcc
    libsoup3
    webkit2gtk-4.1
  )

  cd lemonade
  install -Dm 755 src/app/src-tauri/target/release/lemonade-app -t "${pkgdir}/usr/bin/"
  install -Dm 644 data/lemonade-app.desktop -t "${pkgdir}/usr/share/applications/"
  install -Dm 644 src/app/assets/logo.svg "${pkgdir}/usr/share/pixmaps/lemonade-app.svg"
}
