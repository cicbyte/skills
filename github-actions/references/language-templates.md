# GitHub Actions 语言 CI 模板

## 目录

1. [Node.js](#1-nodejs)
2. [Python](#2-python)
3. [Go](#3-go)
4. [Rust](#4-rust)
5. [Java (Maven)](#5-java-maven)
6. [Java (Gradle)](#6-java-gradle)
7. [.NET](#7-net)
8. [Ruby](#8-ruby)
9. [PowerShell](#9-powershell)
10. [Swift](#10-swift)

---

## 1. Node.js

```yaml
name: Node.js CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18.x, 20.x, 22.x]
      fail-fast: false
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'
      - run: npm ci
      - run: npm run build --if-present
      - run: npm test
```

### pnpm 支持

```yaml
- uses: pnpm/action-setup@v4
  with:
    version: 9
- uses: actions/setup-node@v4
  with:
    node-version: '22.x'
    cache: 'pnpm'
```

### 私有注册表

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '22.x'
    registry-url: https://registry.npmjs.org
- run: npm ci
  env:
    NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## 2. Python

```yaml
name: Python CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
      fail-fast: false
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      - run: pip install -r requirements.txt
      - run: pip install ruff pytest pytest-cov
      - run: ruff check --output-format=github
      - run: pytest --junitxml=junit/test-results.xml --cov=com --cov-report=xml
      - uses: actions/upload-artifact@v4
        if: ${{ always() }}
        with:
          name: pytest-results-${{ matrix.python-version }}
          path: junit/test-results.xml
```

---

## 3. Go

```yaml
name: Go CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        go-version: ['1.21', '1.22', '1.23']
      fail-fast: false
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
      - run: go get .
      - run: go build -v ./...
      - run: go test -v ./...
```

---

## 4. Rust

```yaml
name: Rust CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
      - run: cargo build --release
      - run: cargo test

  release:
    needs: build
    if: startsWith(github.ref, 'refs/tags/')
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v5
      - uses: dtolnay/rust-toolchain@stable
      - run: cargo build --release
      - run: gh release create ${{ github.ref_name }} --generate-notes ./target/release/my-app
        env:
          GH_TOKEN: ${{ github.token }}
```

---

## 5. Java (Maven)

```yaml
name: Java CI (Maven)
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        java-version: ['17', '21']
      fail-fast: false
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'
          cache: maven
      - run: mvn --batch-mode --update-snapshots verify
```

### 发布到 GitHub Packages

```yaml
on:
  release:
    types: [created]

permissions:
  contents: read
  packages: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
          server-id: github
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN
      - run: mvn --batch-mode deploy
        env:
          GITHUB_ACTOR: ${{ github.actor }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 6. Java (Gradle)

```yaml
name: Java CI (Gradle)
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - uses: gradle/actions/setup-gradle@v4
      - run: ./gradlew build
      - uses: actions/upload-artifact@v4
        with:
          name: package
          path: build/libs
```

---

## 7. .NET

```yaml
name: .NET CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        dotnet-version: ['6.0.x', '8.0.x']
      fail-fast: false
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ matrix.dotnet-version }}
      - run: dotnet restore
      - run: dotnet build --no-restore
      - run: dotnet test --no-build --verbosity normal
```

---

## 8. Ruby

```yaml
name: Ruby CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        ruby-version: ['3.1', '3.2', '3.3']
      fail-fast: false
    steps:
      - uses: actions/checkout@v5
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: ${{ matrix.ruby-version }}
          bundler-cache: true
      - run: bundle install
      - run: bundle exec rake test
```

### RuboCop 代码检查

```yaml
- run: bundle exec rubocop -f github
# -f github 输出格式会在 PR 中内联显示 Lint 错误
```

---

## 9. PowerShell

```yaml
name: PowerShell CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: actions/cache@v4
        id: cache
        with:
          path: ~/.local/share/powershell/Modules
          key: ${{ runner.os }}-Pester-PSScriptAnalyzer
      - name: Install modules
        if: steps.cache.outputs.cache-hit != 'true'
        shell: pwsh
        run: |
          Set-PSRepository PSGallery -InstallationPolicy Trusted
          Install-Module Pester, PSScriptAnalyzer -ErrorAction Stop
      - name: Run PSScriptAnalyzer
        shell: pwsh
        run: Invoke-ScriptAnalyzer -Path . -Recurse
      - name: Run Pester tests
        shell: pwsh
        run: Invoke-Pester -Path ./tests -Output Detailed
```

---

## 10. Swift

```yaml
name: Swift CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        swift-version: ['5.9', '5.10']
      fail-fast: false
    steps:
      - uses: swift-actions/setup-swift@v2
        with:
          swift-version: ${{ matrix.swift-version }}
      - uses: actions/checkout@v5
      - run: swift build
      - run: swift test
```
