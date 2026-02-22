# AI 실행용: Windows 터미널 환경 설치 (WezTerm + Yazi + Glow)

> **이 문서는 AI가 위에서 아래로 순서대로 실행하는 플레이북입니다.**
> 모든 명령은 PowerShell(관리자)에서 실행합니다.
> 각 STEP의 `[CHECK]`가 통과해야 다음 STEP으로 진행하세요.

## 전제 조건

- OS: Windows 10 (1709+) 또는 Windows 11
- 셸: PowerShell (관리자 권한으로 실행)
- WSL 불필요 — 순수 Windows만 사용

---

## STEP 1: winget 동작 확인

```powershell
winget --version
```

**[CHECK]** 버전 문자열이 출력되면 통과. 오류 시:

```powershell
# Microsoft Store에서 "앱 설치 관리자" 업데이트
# 또는 수동 설치: https://github.com/microsoft/winget-cli/releases
```

---

## STEP 2: 핵심 패키지 설치

```powershell
winget install --id Microsoft.PowerShell -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id Git.Git -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id wez.wezterm -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id sxyazi.yazi -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id charmbracelet.glow -e --source winget --accept-package-agreements --accept-source-agreements
```

**[CHECK]** 설치 후 **PowerShell을 완전히 닫고 관리자 권한으로 다시 열어** PATH를 갱신한 뒤 확인:

```powershell
pwsh --version
git --version
wezterm --version
yazi --version
glow --version
```

5개 모두 버전이 출력되면 통과. 특정 명령이 안 되면 터미널을 한 번 더 재시작.

---

## STEP 3: Yazi 의존성 설치

```powershell
winget install --id Gyan.FFmpeg -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id 7zip.7zip -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id jqlang.jq -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id oschwartz10612.Poppler -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id sharkdp.fd -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id BurntSushi.ripgrep.MSVC -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id junegunn.fzf -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id ajeetdsouza.zoxide -e --source winget --accept-package-agreements --accept-source-agreements
winget install --id ImageMagick.ImageMagick -e --source winget --accept-package-agreements --accept-source-agreements
```

**[CHECK]** 오류 없이 완료되면 통과. "이미 설치됨" 메시지도 정상.

---

## STEP 4: Nerd Font 설치

```powershell
winget install --id JanDeDobbeleer.OhMyPosh -e --source winget --accept-package-agreements --accept-source-agreements
oh-my-posh font install JetBrainsMono
```

**[CHECK]**

```powershell
[System.Drawing.Text.InstalledFontCollection]::new().Families | Where-Object { $_.Name -like '*JetBrains*' }
```

`JetBrainsMono Nerd Font` 계열 이름이 하나 이상 출력되면 통과.

폴백 — 위 명령 실패 시 수동 설치:
1. https://www.nerdfonts.com/font-downloads 에서 JetBrainsMono 다운로드
2. ZIP 해제 → `.ttf` 전체 선택 → 우클릭 → 모든 사용자용으로 설치

---

## STEP 5: YAZI_FILE_ONE 환경 변수 설정

> 의존: STEP 2 (Git 설치 완료)

```powershell
$gitPath = (Get-Command git).Source | Split-Path | Split-Path
$filePath = Join-Path $gitPath "usr\bin\file.exe"

if (Test-Path $filePath) {
    setx YAZI_FILE_ONE $filePath
    $env:YAZI_FILE_ONE = $filePath
    Write-Host "OK: YAZI_FILE_ONE = $filePath"
} else {
    Write-Host "FAIL: file.exe not found at $filePath"
}
```

**[CHECK]**

```powershell
Test-Path $env:YAZI_FILE_ONE
```

`True` 출력 시 통과.

---

## STEP 6: WezTerm 설정 파일 생성

> 의존: STEP 2 (WezTerm 설치), STEP 4 (폰트 설치)
> 파일 경로: `%USERPROFILE%\.wezterm.lua`

```powershell
@'
local wezterm = require 'wezterm'
local act = wezterm.action
local config = wezterm.config_builder()

-- 셸
config.default_prog = { 'pwsh.exe', '-NoLogo' }

-- UI
config.window_decorations = 'INTEGRATED_BUTTONS|RESIZE'
config.window_background_opacity = 0.95
config.enable_tab_bar = true

-- 폰트
config.font = wezterm.font_with_fallback({
  'JetBrainsMono Nerd Font',
  'Cascadia Mono',
})
config.font_size = 12.5

-- Alt+e: yazi pane 토글
wezterm.on('toggle-yazi-pane', function(window, pane)
  local tab = window:active_tab()
  if not tab then return end

  for _, info in ipairs(tab:panes_with_info()) do
    local proc = info.pane:get_foreground_process_name()
    if proc and proc:lower():find('yazi') then
      if info.is_active then
        window:perform_action(act.CloseCurrentPane { confirm = false }, info.pane)
      else
        window:perform_action(act.ActivatePaneByIndex(info.index), pane)
      end
      return
    end
  end

  window:perform_action(
    act.SplitPane {
      direction = 'Right',
      size = { Percent = 35 },
      command = { args = { 'yazi' } },
    },
    pane
  )
end)

config.keys = {
  { key = 'n', mods = 'ALT', action = act.SplitPane { direction = 'Right', size = { Percent = 50 } } },
  { key = '\\', mods = 'ALT', action = act.SplitPane { direction = 'Right', size = { Percent = 50 } } },
  { key = 'v', mods = 'ALT', action = act.SplitPane { direction = 'Down',  size = { Percent = 50 } } },
  { key = 'Enter', mods = 'ALT', action = act.SplitPane { direction = 'Down', size = { Percent = 50 } } },

  { key = 't', mods = 'ALT', action = act.SpawnTab 'CurrentPaneDomain' },
  { key = '1', mods = 'ALT', action = act.ActivateTab(0) },
  { key = '2', mods = 'ALT', action = act.ActivateTab(1) },
  { key = '3', mods = 'ALT', action = act.ActivateTab(2) },
  { key = '4', mods = 'ALT', action = act.ActivateTab(3) },
  { key = '5', mods = 'ALT', action = act.ActivateTab(4) },

  { key = 'w', mods = 'ALT', action = act.CloseCurrentPane { confirm = false } },
  { key = 'z', mods = 'ALT', action = act.TogglePaneZoomState },
  { key = 'f', mods = 'ALT', action = act.PaneSelect },
  { key = 'e', mods = 'ALT', action = act.EmitEvent 'toggle-yazi-pane' },

  { key = 'LeftArrow',  mods = 'ALT', action = act.ActivatePaneDirection 'Left' },
  { key = 'RightArrow', mods = 'ALT', action = act.ActivatePaneDirection 'Right' },
  { key = 'UpArrow',    mods = 'ALT', action = act.ActivatePaneDirection 'Up' },
  { key = 'DownArrow',  mods = 'ALT', action = act.ActivatePaneDirection 'Down' },

  { key = 'LeftArrow',  mods = 'ALT|SHIFT', action = act.AdjustPaneSize { 'Left', 3 } },
  { key = 'RightArrow', mods = 'ALT|SHIFT', action = act.AdjustPaneSize { 'Right', 3 } },
  { key = 'UpArrow',    mods = 'ALT|SHIFT', action = act.AdjustPaneSize { 'Up', 2 } },
  { key = 'DownArrow',  mods = 'ALT|SHIFT', action = act.AdjustPaneSize { 'Down', 2 } },
}

return config
'@ | Set-Content -Path "$HOME\.wezterm.lua" -Encoding UTF8
```

**[CHECK]**

```powershell
Test-Path "$HOME\.wezterm.lua"
```

`True` 출력 시 통과.

---

## STEP 7: PowerShell 프로필에 함수 등록

> 의존: STEP 2 (yazi, glow 설치)

```powershell
if (!(Test-Path $PROFILE)) { New-Item -Path $PROFILE -ItemType File -Force | Out-Null }

@'

# === Claude Code 단축 명령 ===
# cc = Claude Code를 자동 실행 모드로 실행
# 사용법: cc "이 프로젝트 설명해줘"
function cc { claude --dangerously-skip-permissions @args }

# === Yazi cd 연동 ===
function y {
  $tmp = New-TemporaryFile
  yazi @args --cwd-file="$tmp"
  if (Test-Path $tmp) {
    $cwd = Get-Content -Path $tmp -Raw
    if ($cwd -and (Test-Path -LiteralPath $cwd)) {
      Set-Location -LiteralPath $cwd
    }
    Remove-Item -Path $tmp -Force -ErrorAction SilentlyContinue
  }
}
Set-Alias -Name yy -Value y

# === Markdown 뷰어 ===
function mdv { param([string]$File='README.md') glow -p $File }
'@ | Add-Content -Path $PROFILE

. $PROFILE
```

**[CHECK]**

```powershell
Get-Command cc
Get-Command y
Get-Command mdv
```

세 함수 모두 `Function` 타입으로 출력되면 통과.

---

## STEP 8: Yazi Glow 미리보기 설정 (piper 플러그인)

> 의존: STEP 2 (yazi, glow 설치)
> Yazi에서 `.md` 파일에 커서를 올리면 오른쪽 미리보기 패널에 Glow 렌더링 결과가 표시됩니다.
> 공식 권장 방식인 piper.yazi 플러그인을 사용합니다.

### 8-1. piper 플러그인 설치

```powershell
ya pkg add yazi-rs/plugins:piper
```

**[CHECK]**

```powershell
Test-Path "$env:APPDATA\yazi\plugins\piper.yazi\main.lua"
```

`True` 출력 시 통과.

폴백 — `ya` 명령이 안 되면 수동 설치:

```powershell
$piperDir = "$env:APPDATA\yazi\plugins\piper.yazi"
if (!(Test-Path $piperDir)) {
    git clone https://github.com/yazi-rs/plugins.git "$env:TEMP\yazi-plugins"
    Copy-Item -Path "$env:TEMP\yazi-plugins\piper.yazi" -Destination $piperDir -Recurse -Force
    Remove-Item -Path "$env:TEMP\yazi-plugins" -Recurse -Force
}
```

### 8-2. yazi.toml에 Glow 프리뷰어 등록

```powershell
$yaziConfigDir = "$env:APPDATA\yazi\config"
if (!(Test-Path $yaziConfigDir)) {
    New-Item -Path $yaziConfigDir -ItemType Directory -Force | Out-Null
}

@'
[[plugin.prepend_previewers]]
url = "*.md"
run = 'piper -- CLICOLOR_FORCE=1 glow -w=$w -s=dark "$1"'
'@ | Set-Content -Path "$yaziConfigDir\yazi.toml" -Encoding UTF8
```

**[CHECK]**

```powershell
Test-Path "$env:APPDATA\yazi\config\yazi.toml"
Get-Content "$env:APPDATA\yazi\config\yazi.toml"
```

`piper` 와 `glow` 가 포함된 내용이 출력되면 통과.

---

## STEP 9: 최종 검증 (전체)

모든 STEP 완료 후 아래를 한 번에 실행:

```powershell
Write-Host "`n=== 버전 확인 ==="
pwsh --version
git --version
wezterm --version
yazi --version
glow --version

Write-Host "`n=== 환경 변수 ==="
Write-Host "YAZI_FILE_ONE = $env:YAZI_FILE_ONE"
Test-Path $env:YAZI_FILE_ONE

Write-Host "`n=== 설정 파일 ==="
Test-Path "$HOME\.wezterm.lua"
Test-Path $PROFILE
Test-Path "$env:APPDATA\yazi\config\yazi.toml"
Test-Path "$env:APPDATA\yazi\plugins\piper.yazi\main.lua"

Write-Host "`n=== 함수 등록 ==="
Get-Command cc | Select-Object Name, CommandType
Get-Command y | Select-Object Name, CommandType
Get-Command mdv | Select-Object Name, CommandType

Write-Host "`n=== 폰트 ==="
[System.Drawing.Text.InstalledFontCollection]::new().Families |
  Where-Object { $_.Name -like '*JetBrains*' } |
  Select-Object -First 1 -ExpandProperty Name
```

**[CHECK]** 모든 항목이 정상 출력되면 설치 완료.

항목별 기대값:

| 항목 | 기대 출력 |
|------|-----------|
| pwsh --version | `PowerShell 7.x.x` |
| git --version | `git version 2.x.x` |
| wezterm --version | `wezterm x.x.x` |
| yazi --version | `Yazi x.x.x` |
| glow --version | `glow version x.x.x` |
| YAZI_FILE_ONE | `C:\Program Files\Git\usr\bin\file.exe` (경로) |
| Test-Path (4개) | 모두 `True` |
| cc, y, mdv | `CommandType: Function` |
| 폰트 | `JetBrainsMono Nerd Font` 계열 |

---

## STEP 10: WezTerm 실행 검증 (설정 오류 자동 수정)

> 의존: STEP 6 (WezTerm 설정 파일), STEP 9 (최종 검증 통과)
> 이 STEP에서 WezTerm을 실제로 실행하고, Lua 설정 파싱 오류가 있으면 자동으로 수정합니다.

### 10-1. 설정 파일 문법 검증

```powershell
# WezTerm에 내장된 Lua 파서로 설정 파일 검증
$result = wezterm --config-file "$HOME\.wezterm.lua" ls-fonts 2>&1
$errors = $result | Select-String -Pattern "error|Error|ERROR|failed|Failed|FAILED|parse|Parse"

if ($errors) {
    Write-Host "FAIL: .wezterm.lua 설정에 오류가 있습니다:" -ForegroundColor Red
    $errors | ForEach-Object { Write-Host $_.Line -ForegroundColor Yellow }
} else {
    Write-Host "OK: .wezterm.lua 설정 문법 정상" -ForegroundColor Green
}
```

**[CHECK]** `OK` 출력 시 통과.

오류 발생 시 — AI는 아래 순서로 수정:

1. 오류 메시지에서 줄 번호 확인 (예: `line 42`)
2. `$HOME\.wezterm.lua` 해당 줄 읽기
3. Lua 문법에 맞게 수정 (빠진 콤마, 괄호, 따옴표 등)
4. 저장 후 10-1 재실행
5. 최대 3회 반복. 3회 실패 시 STEP 6의 원본 코드를 다시 Set-Content로 덮어쓰기

### 10-2. WezTerm 실행 및 프로세스 확인

```powershell
# WezTerm 실행 (백그라운드)
Start-Process wezterm

# 3초 대기 후 프로세스 확인
Start-Sleep -Seconds 3
$proc = Get-Process -Name "wezterm*" -ErrorAction SilentlyContinue

if ($proc) {
    Write-Host "OK: WezTerm이 정상 실행 중입니다 (PID: $($proc.Id))" -ForegroundColor Green
} else {
    Write-Host "FAIL: WezTerm 프로세스를 찾을 수 없습니다" -ForegroundColor Red
}
```

**[CHECK]** `OK` 출력 시 통과.

실패 시 — AI는 아래를 순서대로 확인:

1. `wezterm --version` 재확인 → 안 되면 STEP 2 재실행
2. `Test-Path "$HOME\.wezterm.lua"` → `False`면 STEP 6 재실행
3. 이벤트 로그 확인:
   ```powershell
   Get-WinEvent -LogName Application -MaxEvents 5 |
     Where-Object { $_.Message -like '*wezterm*' } |
     Select-Object TimeCreated, Message
   ```

### 10-3. WezTerm 내부에서 도구 동작 확인

```powershell
# WezTerm CLI로 새 pane을 열어서 yazi, glow 실행 테스트
# (WezTerm이 실행 중이어야 동작)

# yazi 실행 테스트
$yaziTest = & yazi --version 2>&1
if ($yaziTest -match 'Yazi') {
    Write-Host "OK: yazi 정상" -ForegroundColor Green
} else {
    Write-Host "FAIL: yazi 실행 불가" -ForegroundColor Red
}

# glow 실행 테스트
$glowTest = & glow --version 2>&1
if ($glowTest -match 'glow') {
    Write-Host "OK: glow 정상" -ForegroundColor Green
} else {
    Write-Host "FAIL: glow 실행 불가" -ForegroundColor Red
}

# PowerShell 프로필 함수 테스트
$ccTest = Get-Command cc -ErrorAction SilentlyContinue
$yyTest = Get-Command yy -ErrorAction SilentlyContinue
$mdvTest = Get-Command mdv -ErrorAction SilentlyContinue

if ($ccTest -and $yyTest -and $mdvTest) {
    Write-Host "OK: cc, yy, mdv 함수 모두 등록됨" -ForegroundColor Green
} else {
    Write-Host "FAIL: 누락된 함수 있음. '. `$PROFILE' 실행 필요" -ForegroundColor Red
    . $PROFILE
    Write-Host "프로필 재로드 완료. 다시 확인하세요." -ForegroundColor Yellow
}
```

**[CHECK]** 3개 모두 `OK` 출력 시 통과.

### 10-4. 전체 실행 검증 요약

```powershell
Write-Host "`n========================================" -ForegroundColor Cyan
Write-Host "  STEP 10 실행 검증 결과" -ForegroundColor Cyan
Write-Host "========================================" -ForegroundColor Cyan

$allPass = $true

# 설정 파일
$luaCheck = wezterm --config-file "$HOME\.wezterm.lua" ls-fonts 2>&1 |
  Select-String -Pattern "error|Error|ERROR|failed|Failed|FAILED"
if ($luaCheck) { Write-Host "[FAIL] .wezterm.lua 설정 오류" -ForegroundColor Red; $allPass = $false }
else { Write-Host "[PASS] .wezterm.lua 설정 정상" -ForegroundColor Green }

# 프로세스
$proc = Get-Process -Name "wezterm*" -ErrorAction SilentlyContinue
if ($proc) { Write-Host "[PASS] WezTerm 실행 중 (PID: $($proc.Id))" -ForegroundColor Green }
else { Write-Host "[FAIL] WezTerm 미실행" -ForegroundColor Red; $allPass = $false }

# 도구
foreach ($cmd in @('yazi','glow')) {
    $v = & $cmd --version 2>&1
    if ($v -match $cmd) { Write-Host "[PASS] $cmd 정상" -ForegroundColor Green }
    else { Write-Host "[FAIL] $cmd 실행 불가" -ForegroundColor Red; $allPass = $false }
}

# 함수
foreach ($fn in @('cc','yy','mdv')) {
    if (Get-Command $fn -ErrorAction SilentlyContinue) { Write-Host "[PASS] $fn 함수 등록됨" -ForegroundColor Green }
    else { Write-Host "[FAIL] $fn 함수 없음" -ForegroundColor Red; $allPass = $false }
}

Write-Host "========================================" -ForegroundColor Cyan
if ($allPass) {
    Write-Host "  ALL PASS — 설치가 정상 완료되었습니다!" -ForegroundColor Green
} else {
    Write-Host "  일부 항목 실패 — 위 FAIL 항목을 확인하세요" -ForegroundColor Red
}
Write-Host "========================================`n" -ForegroundColor Cyan
```

**[CHECK]** `ALL PASS` 출력 시 통과. FAIL 항목이 있으면 해당 STEP으로 돌아가서 재실행.

---

## STEP 11: 설치 완료 안내 (AI가 사용자에게 전달)

> 이 STEP은 명령 실행이 아닙니다.
> STEP 10까지 모두 통과한 뒤, 아래 내용을 **사용자에게 그대로 안내**하세요.

---

사용자에게 전달할 메시지:

```
설치가 모두 완료되었습니다!

터미널에서 바로 쓸 수 있는 명령어가 등록되었습니다:

  cc   → Claude Code 실행 (AI가 자동으로 작업을 수행합니다)
  yy   → Yazi 파일 탐색기 열기
  mdv  → Markdown 문서 보기 (예: mdv README.md)


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  폴더 바로가기 등록 안내
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

자주 가는 폴더를 세 글자 단축 명령으로 등록해 드릴 수 있습니다.
원하시는 폴더 경로를 말씀해 주시면 바로 만들어 드리겠습니다.

예를 들어 이렇게 요청하시면 됩니다:
  "바탕화면의 project 폴더를 pro로 등록해줘"
  "다운로드 폴더를 dwn으로 등록해줘"
  "D드라이브 작업폴더를 wrk로 등록해줘"


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  실전 사용 흐름 (이것만 기억하세요!)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

등록이 끝나면 이렇게 두 번만 입력하면 AI가 바로 작업합니다:

  1단계: 폴더로 이동
  ┌─────────────────────────┐
  │ pro  (엔터)             │  ← 바탕화면/project 폴더로 이동!
  └─────────────────────────┘

  2단계: AI에게 작업 지시
  ┌─────────────────────────────────────────┐
  │ cc "이 프로젝트 구조 설명해줘"  (엔터)  │  ← AI가 즉시 작업 시작!
  └─────────────────────────────────────────┘

  이게 끝입니다. 두 줄이면 AI가 해당 폴더에서 작업합니다.


  다른 사용 예시:

  pro → cc "README.md 한국어로 번역해줘"
       AI가 project 폴더의 README를 찾아서 번역합니다.

  pro → cc "여기 있는 코드 분석해줘"
       AI가 project 폴더의 모든 파일을 읽고 분석합니다.

  pro → cc "테스트 코드 작성해줘"
       AI가 project 폴더의 코드를 보고 테스트를 만듭니다.

  dwn → cc "여기 있는 csv 파일 정리해줘"
       AI가 다운로드 폴더의 csv를 찾아서 정리합니다.


  요약:
  ┌──────────────────────────────────────────┐
  │  {폴더단축키}  (엔터)  →  폴더 이동      │
  │  cc "할 일"    (엔터)  →  AI가 작업 시작  │
  └──────────────────────────────────────────┘

원하시는 폴더를 말씀해 주세요! 바로 등록해 드리겠습니다.
```

---

AI가 사용자 요청을 받았을 때 등록하는 방법:

```powershell
# 예시: 사용자가 "프로젝트 폴더 C:\Users\홍길동\Projects를 prj로 등록해줘" 라고 하면
# 아래 형태로 PowerShell 프로필에 추가
Add-Content -Path $PROFILE -Value "function prj { Set-Location 'C:\Users\홍길동\Projects' }"
. $PROFILE
```

등록 패턴 (AI용 템플릿):

```powershell
# function {3글자축약} { Set-Location '{사용자가 지정한 경로}' }
# 경로에 공백이 있으면 반드시 작은따옴표로 감싸기

# 자주 쓰는 예시
# function prj { Set-Location 'C:\Users\사용자\Projects' }
# function dsk { Set-Location "$HOME\Desktop" }
# function dwn { Set-Location "$HOME\Downloads" }
# function doc { Set-Location "$HOME\Documents" }
```

**[CHECK]** 안내 메시지를 사용자에게 전달했으면 통과.

---

## 설치 완료 — 이하는 사용자 참고용 (AI 실행 불필요)

---

### 등록된 명령어 요약

| 명령어 | 동작 | 예시 |
|--------|------|------|
| `cc` | Claude Code 자동 실행 | `cc "이 코드 리뷰해줘"` |
| `cc "이 문서 읽고 설치해줘"` | AI에게 작업 지시 | 파일 경로와 함께 사용 |
| `yy` | Yazi 파일 탐색기 | 종료 시 해당 폴더로 이동 |
| `mdv` | Markdown 문서 보기 | `mdv README.md` |
| `prj` 등 (사용자 등록) | 폴더 바로가기 | 세 글자로 즉시 이동 |

### 사용 방법

1. **WezTerm 실행** → PowerShell 7이 자동으로 열림
2. `yy` → Yazi 파일 탐색기 (종료 시 해당 디렉터리로 이동)
3. `Alt+e` → 오른쪽에 Yazi 사이드바 토글
4. `Alt+n` / `Alt+v` → pane 분할
5. `mdv 파일명.md` → Markdown 문서 보기

추천 화면 배치:

```
┌──────────────────────┬─────────────────────────┐
│                      │ Yazi (Alt+e)            │
│                      ├────────────┬────────────┤
│   코드 / 셸 작업     │  파일 목록  │  미리보기   │
│   (메인 pane)        │            │            │
│                      │  README.md │ # 프로젝트  │
│                      │  docs/     │ 설명 텍스트 │
│                      │  src/      │ 가 Glow로   │
│                      │            │ 렌더링됨    │
└──────────────────────┴────────────┴────────────┘
                        ↑ 커서 올리면  ↑ 자동 미리보기
```

### 단축키

| 키 | 동작 |
|----|------|
| `Alt+n` | 오른쪽 분할 |
| `Alt+\` | 오른쪽 분할 |
| `Alt+v` | 아래 분할 |
| `Alt+Enter` | 아래 분할 |
| `Alt+t` | 새 탭 |
| `Alt+1~5` | 탭 이동 |
| `Alt+w` | 현재 pane 닫기 |
| `Alt+z` | pane 확대/복귀 |
| `Alt+f` | pane 선택 |
| `Alt+e` | Yazi pane 토글 |
| `Alt+방향키` | pane 이동 |
| `Alt+Shift+방향키` | pane 리사이즈 |

### 트러블슈팅

| 증상 | 해결 |
|------|------|
| winget 안 됨 | Windows 10 1709+ 필요. Microsoft Store에서 "앱 설치 관리자" 업데이트 |
| 명령어가 안 먹힘 | PowerShell 완전히 닫고 다시 열기 (PATH 갱신) |
| 폰트 아이콘 깨짐 (□) | STEP 4 재실행. WezTerm 완전 종료 후 재실행 |
| 파일 타입 감지 이상 | `Test-Path $env:YAZI_FILE_ONE` 확인. `False`면 STEP 5 재실행 |
| `Alt+키` 안 먹힘 | 입력기를 영문(ENG)으로 전환 후 테스트 |
| `Alt+e`가 yazi 못 찾음 | `yazi --version` 확인. 안 되면 WezTerm 재시작 |
| glow 없다고 나옴 | `winget install --id charmbracelet.glow -e --source winget` |

### 공식 문서

| 도구 | 링크 |
|------|------|
| WezTerm 설치 | https://wezterm.org/install/windows.html |
| WezTerm 키바인딩 | https://wezterm.org/config/lua/keyassignment/SplitPane.html |
| Yazi 설치 (Windows) | https://yazi-rs.github.io/docs/installation/ |
| Yazi 설정 | https://yazi-rs.github.io/docs/configuration/yazi/ |
| Glow | https://github.com/charmbracelet/glow |
| Nerd Fonts | https://www.nerdfonts.com/font-downloads |
