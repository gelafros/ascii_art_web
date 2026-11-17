# ASCII-Art-Web

[![Go](https://img.shields.io/badge/Go-1.25+-00ADD8?logo=go&logoColor=white)](https://go.dev/)
[![CI](https://github.com/gelafros/ascii_art_web/actions/workflows/ci.yml/badge.svg)](https://github.com/gelafros/ascii_art_web/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## Description

ASCII-Art-Web is a web application that converts text into ASCII art using different banner styles. Built with Go and standard library only, it provides a simple web interface to generate artistic text representations in three styles: standard, shadow, and thinkertoy.

## Authors

- [gelafros](https://github.com/gelafros)
- [skamprog](https://github.com/gelafros/ascii_art_web)
- [emanola](https://github.com/gelafros/ascii_art_web)

## Usage

### Prerequisites

- **Go**: 1.25 or higher (standard library only, zero external dependencies)
- **Docker** (optional, for containerized deployment)

### Installation & Running Locally

1. Clone the repository and navigate to the project directory:

```bash
git clone https://github.com/gelafros/ascii_art_web.git
cd ascii_art_web
```

2. Run the HTTP server:

```bash
go run .
```

If your environment is missing a C compiler (`gcc`) and `go run .` fails with a cgo error, run:

```bash
CGO_ENABLED=0 go run .
```

3. Open your browser and visit `http://localhost:8080`.
4. Enter text, select a banner style (`standard`, `shadow`, or `thinkertoy`), optionally choose a color and target substring, and click **Generate ASCII Art**.

### Docker

You can build and run the project using the included [`docker.sh`](docker.sh) helper script or plain Docker commands from the project root:

```bash
# Using the helper script
./docker.sh build
./docker.sh run

# Or using plain Docker commands
docker image build -t ascii-art-web-docker .
docker container run -d -p 8080:8080 --name dockerize ascii-art-web-docker
```

Useful follow-up commands:

```bash
docker images
docker ps -a
docker exec -it dockerize /bin/bash
docker image inspect ascii-art-web-docker --format '{{json .Config.Labels}}'
docker stop dockerize
docker rm dockerize
```

## Testing

Run the full unit test suite across all packages (handlers, banner loader, parser, color converter, and renderer):

```bash
go test -v ./...
```

Run static analysis (`go vet`):

```bash
go vet ./...
```

### Features

- Web-based GUI for ASCII art generation
- Single Page Application (SPA) experience using JavaScript `fetch`
- JSON response mode on `POST /ascii-art` via `Accept: application/json`
- Server-backed export downloads on `POST /export`
- Live typing (debounced auto-generation)
- Three banner styles: standard, shadow, thinkertoy
- Real-time result display on the same page
- Color picker and substring coloring support
- Dark/Light mode canvas toggle
- Download generated art as `.txt`, `.html`, or `.json`
- Explicit `Content-Type`, `Content-Length`, and `Content-Disposition` headers on generated responses and exports
- Proper HTTP status code handling (200, 400, 404, 500)
- Strict ASCII range validation (32-126) for secure input

### Security Measures

- **DoS Protection**: Strict 1000-character payload limit on the backend to prevent CPU and memory exhaustion.
- **Slowloris Mitigation**: Custom `http.Server` configured with strict 10-second read/write timeouts.
- **Path Traversal Prevention**: Hardcoded whitelisting for banner file selection (blocks `../../` injections).
- **XSS Prevention**: Safe HTML escaping using Go's `html/template` package.

## Implementation Details

### Algorithm

1. **HTTP Server Setup**: Server listens on port 8080 with three endpoints
    - `GET /`: Serves the main HTML page with form
    - `POST /ascii-art`: Processes form data and returns HTML or JSON (content negotiation)
    - `POST /export`: Returns downloadable `.txt`, `.html`, or `.json` files

2. **Request Processing**:
    - Parse form data (text input and banner selection)
    - Validate banner type (standard, shadow, thinkertoy)
    - Validate input length and printable ASCII characters
    - Return 400 Bad Request for invalid input

3. **ASCII Art Generation**:
    - Load selected banner file from `banners/` directory
    - Each banner contains 8 lines per character (ASCII 32-126)
    - Calculate character position: `(ASCII_code - 32) * 9 + 1 + rowNumber`
    - Render each line by concatenating character art horizontally
    - Handle newlines (`\n`) by processing text in segments

4. **Response Handling**:
    - `Accept: application/json` → JSON payload with `ascii_art`, `ascii_art_html`, and `error`
    - Other `Accept` values → HTML template response for browser rendering
    - `POST /export` → attachment download with `.txt`, `.html`, or `.json`
    - HTML and JSON responses set explicit `Content-Length` headers
    - Export responses set `Content-Type`, `Content-Length`, and `Content-Disposition`
    - 200 OK: Successful generation
    - 400 Bad Request: Invalid input or banner
    - 404 Not Found: Invalid route or missing template/banner
    - 500 Internal Server Error: Rendering or internal server errors

### JSON Usage Example

```bash
curl -X POST "http://localhost:8080/ascii-art" \
  -H "Accept: application/json" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "text=Hello" \
  --data-urlencode "banner=standard" \
  --data-urlencode "color=#8b5cf6" \
  --data-urlencode "substr=He"
```

Example JSON fields:
- `input_text`, `banner`, `color`, `substr`
- `ascii_art` (plain text)
- `ascii_art_html` (HTML with optional span coloring)
- `error` (empty on success)

### Export Usage Example

```bash
curl -X POST "http://localhost:8080/export" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "text=Hello" \
  --data-urlencode "banner=standard" \
  --data-urlencode "format=html" \
  -OJ
```

Supported export formats:
- `txt` → plain text ASCII art
- `html` → downloadable HTML document with the rendered output
- `json` → structured JSON payload containing both `ascii_art` and `ascii_art_html`

### Download Headers Explained

- `Content-Type` tells the client what kind of data the server is sending. Examples here include `text/plain`, `text/html`, and `application/json`.
- `Content-Length` tells the client the exact response size in bytes. This helps browsers and tools know how much data to expect.
- `Content-Disposition` tells the client that the response should be downloaded as a file and also provides the filename, such as `ascii-art.txt`.

For exports, the project now uses all three headers together so auditors can verify the file type, byte size, and download filename from the HTTP response itself.

### Project Structure

```
ascii-art-web/
├── templates/           # HTML templates
│   └── index.html
├── static/              # CSS and frontend JS assets
│   ├── style.css
│   ├── layout.css
│   └── app.js
├── banners/            # Banner files
│   ├── standard.txt
│   ├── shadow.txt
│   └── thinkertoy.txt
├── asciiart/           # Public library
│   └── asciiart.go
├── internal/           # Internal packages
│   ├── banner/         # Banner file loading and normalization
│   ├── color/          # ANSI, Hex, RGB, and HSL color parsing
│   ├── generator/      # Shared plain/HTML generation orchestration
│   ├── parser/         # Input argument parsing
│   └── render/         # ASCII art and HTML rendering
├── docs/               # Architecture and golden test specifications
│   ├── architecture.MD
│   └── golden_tests.md
├── Dockerfile          # Multi-stage Docker build
├── docker.sh           # Docker helper script
├── main.go             # Server entry point
├── main_test.go        # HTTP handler unit tests
└── go.mod              # Go module
```

### Technical Stack

- Language: Go (standard library only)
- Server: net/http
- Templating: html/template
- Port: 8080
