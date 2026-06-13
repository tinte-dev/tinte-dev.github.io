# tinte Developer Guide 📚

이 문서는 Python 및 Go 개발자를 위한 `tinte` 라이브러리의 종합 개발자 가이드 및 API 레퍼런스입니다. 

---

## 🚀 1. 설치 가이드 (Installation)

### Python
Python 3.10 이상을 지원합니다.
```bash
pip install tinte
```

### Go
Go 1.21 이상을 지원합니다.
```bash
go get github.com/tinte-dev/tinte
```

---

## 🔄 2. 기본 흐름 제어 (Session & Flow Control)

모든 CLI 마법사는 세션의 시작(`intro`)과 종료(`outro`) 사이에서 진행됩니다. 중간에 작업 대기가 필요할 경우 `spinner`를 사용하고, 사용자가 `Ctrl+C`나 `Esc`로 이탈할 경우 중단 상태(`cancel`)를 일관되게 처리해야 합니다.

### Python 세션 제어

```python
import tinte
import sys
import time

try:
    # 1. 세션 시작
    tinte.intro("프로젝트 생성기")
    
    # 2. 긴 프로세스는 스피너로 연출
    with tinte.spinner("프로젝트 템플릿 다운로드 중...") as s:
        time.sleep(1.0)
        s.update("템플릿 압축 해제 중...")
        time.sleep(1.0)
        
    # 3. 완료 출력
    tinte.outro("성공적으로 완료되었습니다! 🎉")
    
except KeyboardInterrupt:
    # 사용자가 Ctrl+C를 입력했을 때 세로 가이드라인을 깔끔하게 마무리하고 정리합니다.
    tinte.cancel("작업이 사용자에 의해 취소되었습니다.")
    sys.exit(1)
```

### Go 세션 제어

```go
package main

import (
	"time"
	"github.com/tinte-dev/tinte"
)

func main() {
	// 1. 세션 시작
	tinte.Intro("Go Project Scaffold")

	// 2. 스피너 초기화 및 시작
	spin := tinte.NewSpinner("Downloading components...")
	spin.Start()
	time.Sleep(1500 * time.Millisecond)
	
	// 스피너 텍스트 업데이트
	spin.Update("Compiling files...")
	time.Sleep(1000 * time.Millisecond)
	
	// 스피너 종료 (성공 여부에 따라 마크 변경)
	spin.Stop("Download completed!", true)

	// 3. 완료 출력
	tinte.Outro("Scaffold complete!")
}
```

---

## 🙋 3. 7가지 대화형 프롬프트 (Interactive Prompts API)

모든 프롬프트는 입력을 검증하기 위한 `validate` 콜백을 지원하며, 사용자가 취소했을 경우 에러 상태를 리턴합니다.

### 1. text (기본 텍스트 입력)
* **Python**: `tinte.text(message: str, placeholder: str = "", validate: Callable[[str], str | None] = None)`
* **Go**: `tinte.Text(options TextOptions)`

```python
# Python 예제
name = tinte.text(
    message="프로젝트 이름을 입력하세요:",
    placeholder="my-awesome-app",
    validate=lambda v: "이름은 필수 입력입니다." if not v.strip() else None
)
```

```go
// Go 예제
name, err := tinte.Text(tinte.TextOptions{
    Message:     "Enter project name:",
    Placeholder: "my-awesome-app",
    Validate: func(s string) string {
        if len(s) == 0 {
            return "Project name cannot be empty"
        }
        return ""
    },
})
if err == tinte.ErrCancelled {
    tinte.Cancel("Scaffold cancelled.")
    return
}
```

---

### 2. password (비밀번호 입력)
* **Python**: `tinte.password(message: str, placeholder: str = "", validate: Callable[[str], str | None] = None)`
* **Go**: `tinte.Password(options PasswordOptions)`

```python
# Python 예제
password = tinte.password(
    message="데이터베이스 비밀번호 설정:",
    placeholder="최소 6자 이상",
    validate=lambda v: "비밀번호가 너무 짧습니다." if len(v) < 6 else None
)
```

```go
// Go 예제
dbPass, err := tinte.Password(tinte.PasswordOptions{
    Message:     "Database password:",
    Placeholder: "Min 6 chars",
    Validate: func(s string) string {
        if len(s) < 6 {
            return "Password too short"
        }
        return ""
    },
})
```

---

### 3. select (단일 선택)
* **Python**: `tinte.select(message: str, options: list[tinte.Option])`
* **Go**: `tinte.Select(options SelectOptions)`

```python
# Python 예제
framework = tinte.select(
    message="사용할 프레임워크를 선택하세요:",
    options=[
        tinte.Option("fastapi", "FastAPI", hint="빠른 API 서버 구축"),
        tinte.Option("django", "Django", hint="풀스택 배터리 내장"),
        tinte.Option("flask", "Flask", hint="미니멀 마이크로프레임워크"),
    ]
)
```

```go
// Go 예제
env, err := tinte.Select(tinte.SelectOptions{
    Message: "Select deploy target:",
    Options: []tinte.Option{
        {Value: "staging", Label: "Staging Server", Hint: "Internal testing"},
        {Value: "production", Label: "Production", Hint: "Live customer site"},
    },
})
```

---

### 4. multiselect (다중 선택)
* **Python**: `tinte.multiselect(message: str, options: list[tinte.Option])`
* **Go**: `tinte.MultiSelect(options MultiSelectOptions)`

```python
# Python 예제
tools = tinte.multiselect(
    message="설치할 개발 도구를 모두 선택하세요:",
    options=[
        tinte.Option("linter", "Ruff (Linter)"),
        tinte.Option("formatter", "Black (Formatter)"),
        tinte.Option("tester", "PyTest (Testing Framework)"),
    ]
)
```

```go
// Go 예제
features, err := tinte.MultiSelect(tinte.MultiSelectOptions{
    Message: "Select tools to integrate:",
    Options: []tinte.Option{
        {Value: "docker", Label: "Docker Compose", Hint: "Container config"},
        {Value: "actions", Label: "GitHub Actions", Hint: "CI/CD template"},
    },
    Required: true,
})
```

---

### 5. autocomplete (자동 완성 퍼지 검색)
옵션이 많을 때 실시간으로 키워드를 입력하여 일치하는 항목을 찾는 대화형 선택 메뉴입니다.
* **Python**: `tinte.autocomplete(message: str, options: list[tinte.Option], placeholder: str = "")`
* **Go**: `tinte.Autocomplete(options AutocompleteOptions)`

```python
# Python 예제
pkg = tinte.autocomplete(
    message="설치할 npm 패키지 이름 검색:",
    options=[
        tinte.Option("lodash", "lodash", "유틸리티 벨트"),
        tinte.Option("axios", "axios", "HTTP 클라이언트"),
        tinte.Option("express", "express", "웹 프레임워크"),
    ],
    placeholder="입력 시 실시간 검색..."
)
```

---

### 6. path (파일 시스템 경로 선택)
탭 키(`Tab`)를 눌러 시스템 파일 및 디렉토리 경로를 순환 및 자동 완성하는 경로 입력창입니다.
* **Python**: `tinte.path(message: str, placeholder: str = "", only_directories: bool = False)`
* **Go**: `tinte.Path(options PathOptions)`

```python
# Python 예제
dest_dir = tinte.path(
    message="백업을 저장할 디렉토리를 지정하세요:",
    placeholder="./backups",
    only_directories=True
)
```

---

### 7. confirm (예/아니오)
* **Python**: `tinte.confirm(message: str, initial: bool = True)`
* **Go**: `tinte.Confirm(options ConfirmOptions)`

```python
# Python 예제
is_ready = tinte.confirm("지금 배포를 진행하시겠습니까?", initial=True)
```

```go
// Go 예제
proceed, err := tinte.Confirm(tinte.ConfirmOptions{
    Message: "Proceed with database migration?",
    Initial: true,
})
```

---

## 🎨 4. 9가지 레이아웃 컴포넌트 (Visual Displays API)

비대화형(또는 정적/인터랙티브 뷰) 데이터 표시 컴포넌트 목록입니다.

### 1. table (표 형식 정렬)
* **Python**: `tinte.table(headers: list[str], rows: list[list[str]], border: str = "rounded", align: dict = None)`
* **Go**: `tinte.Table(headers []string, rows [][]string, border string, align map[string]string, showRowSeparators bool)`
* **지원 테두리 스타일**: `rounded`(둥근 박스), `sharp`(네모 박스), `double`(두 줄 박스), `minimal`(상하 구분선만)

```python
# Python 예제
tinte.table(
    headers=["Package", "Version", "License"],
    rows=[
        ["tinte", "v1.0.0", "MIT"],
        ["colorama", "v0.4.6", "BSD"],
    ],
    border="rounded"
)
```

---

### 2. tree (JSON/딕셔너리 트리 뷰어)
딕셔너리 구조를 가지런히 트리 모양으로 들여쓰기 렌더링합니다. `interactive=True`로 실행 시 터미널에서 위/아래 방향키와 스페이스바로 직접 노드를 펼치거나 닫을 수 있습니다.
* **Python**: `tinte.tree(data: dict, title: str = "", expand_depth: int = 1, interactive: bool = False)`
* **Go**: `tinte.Tree(data map[string]interface{}, title string, expandDepth int, interactive bool)`

```python
# Python 예제
config_data = {
    "database": {
        "host": "localhost",
        "port": 5432,
        "tables": ["users", "sessions"]
    },
    "debug_mode": True
}
tinte.tree(config_data, title="Configuration", expand_depth=2)
```

---

### 3. accordion (본문 접기/펼치기)
여러 구획의 텍스트가 있을 때, 특정 항목만 접거나 펼칠 수 있는 아코디언 컴포넌트입니다.
* **Python**: `tinte.accordion(sections: list[tinte.Section], default_open: list[int] = None, interactive: bool = False)`
* **Go**: `tinte.Accordion(sections []Section, defaultOpen []int, interactive bool, submitOnEnter bool)`

```python
# Python 예제
tinte.accordion(
    sections=[
        tinte.Section("데이터베이스 백업 방법", "1. CLI 로그인\n2. pg_dump 실행\n3. 파일 저장"),
        tinte.Section("롤백 가이드", "1. 컨테이너 일시 정지\n2. 백업 복원\n3. 헬스체크 확인")
    ],
    default_open=[0]
)
```

---

### 4. panel (경고 및 팁 상자)
중요 메시지나 안내 사항을 특정 형태의 외곽선 테두리로 둘러싸는 패널입니다.
* **Python**: `tinte.panel(content: str, title: str = "", border: str = "rounded")`
* **Go**: `tinte.Panel(content string, title string, border string, width int)`

```python
# Python 예제
tinte.panel(
    content="로컬 서버가 8080 포트에서 실행되었습니다.\nURL: http://localhost:8080",
    title="알림",
    border="double"
)
```

---

### 5. diff (코드 및 텍스트 차이 뷰어)
텍스트 라인 간 차이점을 직관적인 Unified Diff 형태로 빨갛고 초록색으로 나누어 보여줍니다.
* **Python**: `tinte.diff(old: str, new: str, title: str = "")`
* **Go**: `tinte.Diff(old string, new string, title string)`

```python
# Python 예제
tinte.diff("version = 1.0", "version = 1.1\nchannel = stable", "pyproject.toml 변경 사항")
```

---

### 6. syntax (구문 강조 뷰어)
코드 파일 내용 등을 구문 분석(Syntax Highlighting)하여 출력합니다.
* **Python**: `tinte.syntax(code: str, language: str, line_numbers: bool = True)`
* **Go**: `tinte.Syntax(code string, language string, lineNumbers bool)`

```python
# Python 예제
js_code = "const greet = () => console.log('hello tinte');"
tinte.syntax(js_code, "js", line_numbers=True)
```

---

## 🎨 5. 테마 적용 및 커스텀 (Themes)

### 빌드인 테마 전환

```python
# Python 예제
tinte.set_theme(tinte.OCEAN) # OCEAN, NEON, MONO, PASTEL 지원
```

```go
// Go 예제
tinte.SetTheme(tinte.OceanTheme) // OceanTheme, NeonTheme, MonoTheme, PastelTheme 지원
```

### 커스텀 테마 정의
개발자만의 시그니처 배포판 CLI 룩앤필을 위한 독자적인 테마를 색상 해시코드로 간단히 직접 생성할 수 있습니다.

#### Python 커스텀 테마
```python
import tinte

my_brand_theme = tinte.Theme(
    name="MyBrand",
    primary="#1FF0FF",   # 브랜드 컬러 (메인)
    secondary="#FF00A0", # 강조 컬러
    success="#00FF66",
    error="#FF3300",
    warning="#FFFF33",
    info="#00CCFF",
    muted="#666666",
    text="#FFFFFF",
    title_style="bold"
)

tinte.set_theme(my_brand_theme)
```

#### Go 커스텀 테마
```go
package main

import "github.com/tinte-dev/tinte"

var myBrandTheme = tinte.Theme{
	Name:       "MyBrand",
	Primary:    "#1FF0FF",
	Secondary:  "#FF00A0",
	Success:    "#00FF66",
	Error:      "#FF3300",
	Warning:    "#FFFF33",
	Info:       "#00CCFF",
	Muted:      "#666666",
	Text:       "#FFFFFF",
	TitleStyle: "bold",
}

func main() {
	tinte.SetTheme(myBrandTheme)
	tinte.Intro("Welcome to Custom Theme!")
}
```
