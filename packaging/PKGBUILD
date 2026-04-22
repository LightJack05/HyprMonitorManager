# Maintainer: Elias Paitz
pkgname=hyprmonitormanager
pkgver=1.0.0
pkgrel=1
pkgdesc="Monitor configuration manager for Hyprland"
arch=('any')
url="https://github.com/LightJack05/HyprMonitorManager"
license=('MIT')
depends=(
    'hyprland'
    'jq'
    'nmap'        # provides ncat
    'zenity'
    'nwg-displays'
    'libnotify'   # provides notify-send
)

source=("$pkgname::git+$url.git")
sha256sums=('SKIP')

package() {
    cd "$pkgname"
    install -Dm755 hyprmonitormanager "$pkgdir/usr/bin/hyprmonitormanager"
    install -Dm644 settings.conf.example \
        "$pkgdir/usr/share/$pkgname/settings.conf.example"
    install -Dm644 LICENSE "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
}
