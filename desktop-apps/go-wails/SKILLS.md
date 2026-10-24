---
name: wails
description: >
  Expert skill for building native desktop applications in Go with Wails v3. Use whenever a Go
  desktop app, webview app, or GUI is being built, reviewed, or debugged — any mention of Wails v3,
  wails3, application.New, services, frontend bindings, or a frontend talking to a Go backend via
  Wails v3.
---

# Wails v3: Native Desktop Apps in Go

Wails v3 builds native desktop applications with a Go backend and a web frontend rendered in the platform webview (WebKit on macOS/Linux, WebView2 on Windows). No bundled Chromium — binaries remain lightweight (megabytes, not hundreds).

## Wails v3 Core Architecture

### Canonical main.go

```go
package main

import (
	"embed"
	"log"

	"[github.com/wailsapp/wails/v3/pkg/application](https://github.com/wailsapp/wails/v3/pkg/application)"
)

//go:embed all:frontend/dist
var assets embed.FS

func main() {
	app := application.New(application.Options{
		Name: "myapp",
		Assets: application.AssetOptions{
			Handler: application.AssetFileServerFS(assets),
		},
		Services: []application.Service{
			application.NewService(NewGreetService()),
		},
	})

	app.Window.NewWithOptions(application.WebviewWindowOptions{
		Title:  "myapp",
		Width:  1024,
		Height: 768,
	})

	if err := app.Run(); err != nil {
		log.Fatal(err)
	}
}

```

* **Manager API Style**: Use modern manager references (`app.Window.NewWithOptions`, `app.Event.On`, `app.Menu.New()`, `app.SystemTray.New()`).
* **Explicit Lifecycle**: Wails v3 separates application creation, window creation, and execution phases. Avoid implicit context passing from startup hooks; use direct application and window methods instead.

---

### Services: The v3 Binding Model

Services replace older global bindings. A service is a plain struct where exported methods are exposed to the frontend:

```go
type NoteService struct {
	app *application.App
	db  *sql.DB
	mu  sync.RWMutex
}

func NewNoteService(app *application.App, db *sql.DB) *NoteService {
	return &NoteService{app: app, db: db}
}

func (s *NoteService) ServiceStartup(ctx context.Context, opts application.ServiceOptions) error {
	// Lifecycle hook: initialize resources, migrations, background tasks governed by ctx
	return nil 
}

func (s *NoteService) ServiceShutdown() error {
	return s.db.Close() // Services shut down in reverse registration order
}

func (s *NoteService) Save(n Note) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	// Business logic...
	return nil
}

```

* **Singletons**: Services are singletons shared across all application windows. Protect mutable state with synchronization primitives (`sync.RWMutex`).
* **Dependencies**: Inject dependencies via standard Go constructors (`application.NewService(NewNoteService(app, db))`) — no external DI framework required.
* **Thin Adapter Rule**: Keep services light. Core domain logic should live in independent, Wails-free Go packages to ensure testability as pure Go code.

---

### Bindings, Events, and Multi-Window Management

* **Frontend Bindings**: Generate or update bindings using `wails3 generate bindings` (handled automatically during `wails3 dev`). Import directly from `frontend/bindings/<full-go-import-path>/<service>`.
* **Eventing System**:
* Go to JS / JS to Go: Use `app.Event.Emit("name", data)` and `app.Event.On("name", handler)`.
* Compile-time safe events: Leverage typed constant name generation via `application.RegisterEvent`.


* **Lifecycle Hooks**: Intercept app/window states cleanly. For instance, implement "confirm before close" logic via window hooks:
```go
window.RegisterHook(events.Common.WindowClosing, func(e *application.WindowEvent) {
    // Perform checks or cancel close action
    // e.Cancel()
})

```


* **Multi-Window Support**: First-class capability. Create, manage, and close independent `WebviewWindow` instances dynamically at runtime.

---

## Build System & Best Practices

* **Taskfile Orchestration**: v3 utilizes `Taskfile.yml` (go-task) for transparent build pipelines, manifest generation, and packaging (`wails3 package` for `.app`/dmg, NSIS, AppImage/deb).
* **Input Validation**: Treat the frontend as untrusted input. Validate parameters in bound service methods rigorously; never construct file system paths or execute shell commands directly from raw frontend string parameters.
* **Error Handling**: Always return errors from bound service methods. Wails maps these automatically to rejected JavaScript Promises with the respective error strings.
* **Goroutine & UI Responsiveness**: Long-running background operations inside bound methods must run asynchronously (using goroutines) and report progress back to the UI via `app.Event.Emit` rather than locking up the JS Promise or blocking the UI thread.
* **Security & Secrets**: Never expose credentials or secret keys in frontend assets (`frontend/dist`), as client bundles can be inspected by users. Keep all sensitive operations strictly on the Go backend.
