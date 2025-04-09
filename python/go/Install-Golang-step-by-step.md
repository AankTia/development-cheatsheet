# ✅ Step-by-Step: Install Golang

---

## 🐧 For **Linux (Ubuntu/Debian)**

### Step 1: Remove old versions (if any)
```bash
sudo rm -rf /usr/local/go
```

### Step 2: Download the latest Go version
```bash
wget https://go.dev/dl/go1.22.1.linux-amd64.tar.gz
```

> 💡 Check [https://go.dev/dl/](https://go.dev/dl/) for the latest version.

### Step 3: Extract and install
```bash
sudo tar -C /usr/local -xzf go1.22.1.linux-amd64.tar.gz
```

### Step 4: Set up Go environment variables
Add to your `~/.bashrc`, `~/.zshrc`, or `~/.profile`:
```bash
export PATH=$PATH:/usr/local/go/bin
```

Then apply:
```bash
source ~/.bashrc
```

### Step 5: Verify installation
```bash
go version
```

---

## 🍏 For **macOS**

### Option 1: Using Homebrew (Recommended)
```bash
brew install go
```

### Option 2: Manual Install

1. Download the `.pkg` installer from [https://go.dev/dl/](https://go.dev/dl/)
2. Run the installer.
3. Verify:
   ```bash
   go version
   ```

---

## 🪟 For **Windows**

### Step 1: Download Installer
- Go to [https://go.dev/dl/](https://go.dev/dl/)
- Download the `.msi` file for Windows.

### Step 2: Run the Installer
- Follow the wizard instructions.

### Step 3: Verify in Command Prompt
```cmd
go version
```

---

## 🧪 Test Your Installation

Create a test Go file:
```bash
mkdir ~/go-test && cd ~/go-test
nano hello.go
```

Paste this:
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

Then run:
```bash
go run hello.go
```

You should see:
```
Hello, Go!
```

---

Want help setting up a Go workspace (`GOPATH`, modules, etc.) or building your first CLI or web app in Go?