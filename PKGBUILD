# Maintainer: Maxime Gauduin <alucryd@archlinux.org>
# Maintainer: George Sofianos <george@sofianos.dev>
# Contributor: Michele Balistreri <michele@bitgamma.com>

pkgbase=lemonade
pkgname=(
  lemonade-server
  lemonade-desktop
)
pkgdesc='Lemonade helps users discover and run local AI apps by serving optimized LLMs right from their own GPUs and NPUs'
pkgver=10.10.0
pkgrel=1
arch=(x86_64)
url=https://github.com/lemonade-sdk/lemonade
license=(Apache-2.0)
makedepends=(
  cargo
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
  git+${url}.git#tag=v${pkgver}
  lemonade-sysusers.patch
  lemonade-web-app.patch
)
b2sums=('b88e9ec2f2aaf27855b589144f1ba2040142328abaa0654db5d81ac85971d8e2e40eb904f8fdaa9978609280c544b2a289e7394bd2352e830d8048c987a4edb9'
        '5b8174e0d88a825eb69e96f82140bae27a081665b472efaac0920829963f7790b9a1b9c9f5d787901ff4278eb31dc389e3e09dfad61fda490c7ceea355c51e14'
        '6df13873099bfd038e9bcbe7b61f7447d529846c3854babd5b8a36a3ccc50d2b625fe353e1118b1f690c77af85e0dc2946ef0f45c197e686dbb85fd878a22582')

prepare() {
  cd lemonade
  patch -Np1 -i ../lemonade-sysusers.patch
  patch -Np1 -i ../lemonade-web-app.patch

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
  npm run tauri build -- --no-bundle
}

package_lemonade-server() {
  depends=(
    curl
    glibc
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
  )
  backup=(etc/lemonade/conf.d/zz-secrets.conf)

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
