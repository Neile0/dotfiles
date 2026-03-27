# dotfiles

Terminal and editor configuration using the **Kelvin Marsala** theme.

| Colour | Hex |
|---|---|
| Marsala (primary) | `#964F4C` |
| Black Beauty | `#27272A` |
| Cloud Dancer | `#F0EEE9` |

---

## Contents

| File | Destination |
|---|---|
| `ghostty/config` | `~/.config/ghostty/config` |
| `ghostty/themes/kelvin-systems` | `~/.config/ghostty/themes/kelvin-systems` |
| `starship/starship.toml` | `~/.config/starship.toml` |
| `zsh/.zshrc` | `~/.zshrc` |
| `windows/terminal/kelvin-systems.json` | Merge into Windows Terminal `settings.json` |
| `windows/powershell/Microsoft.PowerShell_profile.ps1` | `$PROFILE` |

---

## macOS setup

**Prerequisites:** [Ghostty](https://ghostty.org), [Starship](https://starship.rs) (`brew install starship`), a [Nerd Font](https://www.nerdfonts.com/)

```bash
git clone https://github.com/<your-username>/dotfiles.git ~/dotfiles
cd ~/dotfiles

mkdir -p ~/.config/ghostty/themes
cp ghostty/config ~/.config/ghostty/config
cp ghostty/themes/kelvin-systems ~/.config/ghostty/themes/kelvin-systems

cp starship/starship.toml ~/.config/starship.toml

cp ~/.zshrc ~/.zshrc.bak 2>/dev/null; cp zsh/.zshrc ~/.zshrc
source ~/.zshrc

touch ~/.hushlogin
```

---

## Windows setup

**Prerequisites:** [Windows Terminal](https://aka.ms/terminal), [PowerShell 7](https://github.com/PowerShell/PowerShell/releases) (`winget install Microsoft.PowerShell`), [Starship](https://starship.rs) (`winget install Starship.Starship`), a [Nerd Font](https://www.nerdfonts.com/)

```powershell
git clone https://github.com/<your-username>/dotfiles.git $HOME\dotfiles
cd $HOME\dotfiles

New-Item -ItemType Directory -Force "$HOME\.config"
Copy-Item starship\starship.toml "$HOME\.config\starship.toml"

New-Item -ItemType Directory -Force (Split-Path $PROFILE)
Copy-Item windows\powershell\Microsoft.PowerShell_profile.ps1 $PROFILE
. $PROFILE
```

**Windows Terminal colour scheme:** open `Ctrl+Shift+,` and add the contents of `windows/terminal/kelvin-systems.json` into the `schemes` array, then set `"colorScheme": "Kelvin Systems"` in your profile defaults.

---

## Updating

```bash
cd ~/dotfiles
cp ~/.config/ghostty/config ghostty/config
cp ~/.config/starship.toml starship/starship.toml
cp ~/.zshrc zsh/.zshrc
git add -A && git commit -m "update configs" && git push
```
