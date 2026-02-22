# Mac 터미널 환경 설치 가이드 (WezTerm + Zellij + 문서뷰어)

기존 `mac-setup-guide.md`(Alacritty + Zellij)를 바탕으로,  
macOS에서 창/보안(Gatekeeper) 이슈를 줄인 `WezTerm + Zellij` 조합으로 구성한 설치 가이드입니다.

핵심 목표:
- Zellij 기존 Alt 단축키 유지 (`Alt+n`, `Alt+v`, `Alt+Enter`, `Alt+\`, `Alt+1~5` 등)
- WezTerm에서 macOS Option(⌥) 키를 Alt로 안정 전달
- 문서 확인용 Markdown 뷰어(`glow`) 같이 사용

---

## 1. 사전 준비

```bash
# Homebrew (이미 설치되어 있으면 생략)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

---

## 2. 패키지 설치

```bash
# 폰트 + 터미널 + 멀티플렉서 + 문서뷰어
brew install --cask font-jetbrains-mono-nerd-font
brew install --cask wezterm
brew install zellij glow
```

---

## 3. WezTerm 설정

설정 파일: `~/.wezterm.lua`

```lua
local wezterm = require 'wezterm'
local config = wezterm.config_builder()

-- WezTerm 실행 시 Zellij 세션(main) 자동 연결
config.default_prog = { '/opt/homebrew/bin/zellij', 'attach', '--create', 'main' }

-- 창/렌더링
config.window_decorations = 'INTEGRATED_BUTTONS|RESIZE'
config.window_background_opacity = 0.95
config.macos_window_background_blur = 20
config.native_macos_fullscreen_mode = true
config.enable_tab_bar = false

-- 폰트
config.font = wezterm.font('JetBrainsMono Nerd Font')
config.font_size = 13.0
config.line_height = 1.05

-- 터미널 동작
config.term = 'xterm-256color'
config.scrollback_lines = 0 -- 스크롤은 Zellij 중심으로 사용
config.use_ime = true

-- macOS Option(⌥)을 Alt로 전달 (Zellij Alt 단축키 호환 핵심)
config.send_composed_key_when_left_alt_is_pressed = false
config.send_composed_key_when_right_alt_is_pressed = false

-- Mac 관례 키
config.keys = {
  { key = 'N', mods = 'CMD|SHIFT', action = wezterm.action.SpawnWindow },
  { key = 'V', mods = 'CMD', action = wezterm.action.PasteFrom 'Clipboard' },
}

return config
```

---

## 4. Zellij 설정 (기존 단축키 유지 버전)

설정 파일: `~/.config/zellij/config.kdl`

```kdl
// Zellij config — Catppuccin Mocha
// ~/.config/zellij/config.kdl

keybinds clear-defaults=false {
    shared_except "locked" {
        // Alt+방향키: 패널 이동
        bind "Alt Left" { MoveFocus "Left"; }
        bind "Alt Right" { MoveFocus "Right"; }
        bind "Alt Up" { MoveFocus "Up"; }
        bind "Alt Down" { MoveFocus "Down"; }

        // Alt+숫자: 탭 전환
        bind "Alt 1" { GoToTab 1; SwitchToMode "Normal"; }
        bind "Alt 2" { GoToTab 2; SwitchToMode "Normal"; }
        bind "Alt 3" { GoToTab 3; SwitchToMode "Normal"; }
        bind "Alt 4" { GoToTab 4; SwitchToMode "Normal"; }
        bind "Alt 5" { GoToTab 5; SwitchToMode "Normal"; }

        // 분할 (기존 스타일 유지)
        bind "Alt \\" { NewPane "Right"; }
        bind "Alt Enter" { NewPane "Down"; }
        bind "Alt n" { NewPane "Right"; }
        bind "Alt v" { NewPane "Down"; }

        // 탭/패널
        bind "Alt t" { NewTab; }
        bind "Alt w" { CloseFocus; }
        bind "Alt f" { ToggleFloatingPanes; }
        bind "Alt z" { ToggleFocusFullscreen; }
        bind "Alt e" { ToggleFloatingPanes; }

        // 리사이즈
        bind "Ctrl Alt Left" { Resize "Left"; }
        bind "Ctrl Alt Right" { Resize "Right"; }
        bind "Ctrl Alt Up" { Resize "Up"; }
        bind "Ctrl Alt Down" { Resize "Down"; }
        bind "Alt -" { Resize "Decrease"; }
        bind "Alt =" { Resize "Increase"; }
    }
}

theme "catppuccin-mocha"

default_layout "compact"
default_mode "normal"
mouse_mode true
advanced_mouse_actions true
scroll_buffer_size 50000
copy_on_select true
scrollback_editor "/usr/bin/less"
pane_frames true
auto_layout true
session_serialization true
serialize_pane_viewport true

ui {
    pane_frames {
        rounded_corners true
        hide_session_name true
    }
}
```

개발 레이아웃 파일: `~/.config/zellij/layouts/dev.kdl`

```kdl
// 개발용 레이아웃
// 실행: zellij --layout dev
layout {
    tab name="code" focus=true {
        pane size="70%" {
            // 메인 코딩 패널
        }
        pane split_direction="vertical" size="30%" {
            pane name="server" {
                // dev server (npm run dev 등)
            }
            pane name="shell" {
                // 일반 쉘 (git, 명령어)
            }
        }
    }
    tab name="git" {
        pane
    }
}
```

---

## 5. 문서뷰어(glow) 사용

`glow`는 Markdown 문서를 터미널에서 보기 좋게 렌더링합니다.

```bash
# 파일 보기
glow README.md

# 페이지 모드(스크롤/검색)
glow -p mac-setup-guide-wezterm.md
```

원하면 단축 alias 추가:

```bash
echo "alias mdv='glow -p'" >> ~/.zshrc
source ~/.zshrc
```

사용 예:

```bash
mdv mac-setup-guide-wezterm.md
```

---

## 6. 설치 후 검증

```bash
# 버전 확인
wezterm --version
zellij --version
glow --version

# WezTerm 실행 (자동으로 zellij attach 되는지 확인)
open -a WezTerm
```

Zellij 키 테스트:
- `Alt+t` 새 탭
- `Alt+\` 오른쪽 분할
- `Alt+Enter` 아래 분할
- `Alt+n` 오른쪽 분할 (기존 습관용)
- `Alt+v` 아래 분할 (기존 습관용)
- `Alt+1~5` 탭 이동
- `Alt+w` 패널 닫기

---

## 7. 자주 발생하는 문제와 해결

### 1) Option(⌥)+단축키가 안 먹음
- 입력 소스를 `ABC`로 변경 후 테스트
- `~/.wezterm.lua`에서 아래 값이 `false`인지 확인
  - `send_composed_key_when_left_alt_is_pressed`
  - `send_composed_key_when_right_alt_is_pressed`

### 2) WezTerm 실행은 되는데 Zellij 자동 연결이 안 됨
- `/opt/homebrew/bin/zellij` 경로 확인:
  - `which zellij`
- Intel Mac인 경우 경로가 `/usr/local/bin/zellij`일 수 있으니 `default_prog` 경로 수정

### 3) 폰트가 적용되지 않음
- `JetBrainsMono Nerd Font` 이름 정확히 확인
- 앱 완전 종료 후 재실행

---

## 요약

| 항목 | 설치 명령 |
|------|-----------|
| 폰트 | `brew install --cask font-jetbrains-mono-nerd-font` |
| WezTerm | `brew install --cask wezterm` |
| Zellij | `brew install zellij` |
| 문서뷰어 | `brew install glow` |
| 설정 | `~/.wezterm.lua`, `~/.config/zellij/config.kdl`, `~/.config/zellij/layouts/dev.kdl` |

핵심 단축키:
- `Alt + n`: 오른쪽 분할
- `Alt + v`: 아래 분할
- `Alt + \`: 오른쪽 분할
- `Alt + Enter`: 아래 분할
- `Alt + t`: 새 탭
- `Alt + 1~5`: 탭 이동

