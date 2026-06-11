# Windows Builder Image für SOP native:windows CI-Job
#
# Enthält:
#   - VS Build Tools 2022 (Workload VCTools + cl.exe)
#   - GraalVM Community Edition 21 (native-image, java, javac)
#   - Apache Maven 3.9.6
#
# Bauen:
#   docker build -t gitlab-registry.adesso-group.com/julian.paul/sop/windows-builder:latest .
#
# Hinweis: VS Build Tools Bootstrapper läuft als Child-Prozess weiter, nachdem
# Start-Process zurückgekehrt ist. Der RUN-Layer wartet daher explizit auf
# den echten Installer-Prozess (Setup.exe).

FROM mcr.microsoft.com/windows/servercore:ltsc2022

SHELL ["powershell", "-NoProfile", "-ExecutionPolicy", "Bypass", "-Command", \
       "$ErrorActionPreference='Stop'; $ProgressPreference='SilentlyContinue';"]

# ---------------------------------------------------------------------------
# 1. VS Build Tools 2022 — VC++ Workload (cl.exe, link.exe, libs)
# ---------------------------------------------------------------------------
RUN Write-Host '--- Downloading VS Build Tools ---'; \
    Invoke-WebRequest \
      -Uri 'https://aka.ms/vs/17/release/vs_buildtools.exe' \
      -OutFile 'C:\vs_buildtools.exe' \
      -UseBasicParsing; \
    \
    Write-Host '--- Starting VS Build Tools installer ---'; \
    $proc = Start-Process 'C:\vs_buildtools.exe' \
      -ArgumentList \
        '--quiet', '--wait', '--norestart', '--nocache', \
        '--add', 'Microsoft.VisualStudio.Workload.VCTools', \
        '--includeRecommended' \
      -PassThru -Wait; \
    \
    Write-Host "vs_buildtools.exe ExitCode: $($proc.ExitCode)"; \
    if ($proc.ExitCode -ne 0 -and $proc.ExitCode -ne 3010) { \
      throw "VS Build Tools installer failed with exit code $($proc.ExitCode)"; \
    } \
    \
    # Bootstrapper wartet nicht immer auf alle Child-Prozesse.
    # Warten bis Setup.exe vollständig abgeschlossen ist.
    $timeout = 600; $elapsed = 0; $interval = 10; \
    do { \
      Start-Sleep -Seconds $interval; \
      $elapsed += $interval; \
      $running = Get-Process -Name 'Setup', 'vs_installer', 'VSIXInstaller' \
                   -ErrorAction SilentlyContinue; \
    } while ($running -and $elapsed -lt $timeout); \
    \
    Write-Host '--- VS Build Tools install complete ---'; \
    Remove-Item 'C:\vs_buildtools.exe' -Force

# ---------------------------------------------------------------------------
# 2. GraalVM Community Edition 21
# ---------------------------------------------------------------------------
ARG GRAALVM_VERSION=21.0.2

RUN Write-Host "--- Downloading GraalVM CE $env:GRAALVM_VERSION ---"; \
    $url = "https://github.com/graalvm/graalvm-ce-builds/releases/download/jdk-$env:GRAALVM_VERSION/graalvm-community-jdk-$($env:GRAALVM_VERSION)_windows-x64_bin.zip"; \
    Invoke-WebRequest -Uri $url -OutFile 'C:\graalvm.zip' -UseBasicParsing; \
    \
    Write-Host '--- Extracting GraalVM ---'; \
    Expand-Archive -Path 'C:\graalvm.zip' -DestinationPath 'C:\graalvm-tmp' -Force; \
    $extracted = (Get-ChildItem 'C:\graalvm-tmp' | Select-Object -First 1).FullName; \
    Move-Item $extracted 'C:\graalvm'; \
    Remove-Item 'C:\graalvm.zip', 'C:\graalvm-tmp' -Recurse -Force; \
    Write-Host '--- GraalVM installed at C:\graalvm ---'

# ---------------------------------------------------------------------------
# 3. Apache Maven 3.9.6
# ---------------------------------------------------------------------------
ARG MAVEN_VERSION=3.9.6

RUN Write-Host "--- Downloading Maven $env:MAVEN_VERSION ---"; \
    $url = "https://archive.apache.org/dist/maven/maven-3/$env:MAVEN_VERSION/binaries/apache-maven-$($env:MAVEN_VERSION)-bin.zip"; \
    Invoke-WebRequest -Uri $url -OutFile 'C:\maven.zip' -UseBasicParsing; \
    \
    Write-Host '--- Extracting Maven ---'; \
    Expand-Archive -Path 'C:\maven.zip' -DestinationPath 'C:\' -Force; \
    Remove-Item 'C:\maven.zip' -Force; \
    Write-Host "--- Maven installed at C:\apache-maven-$env:MAVEN_VERSION ---"

# ---------------------------------------------------------------------------
# 4. Umgebungsvariablen dauerhaft im Machine-Scope setzen
# ---------------------------------------------------------------------------
ARG MAVEN_VERSION=3.9.6

RUN Write-Host '--- Configuring PATH ---'; \
    $msvcBase = @( \
      'C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC', \
      'C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC' \
    ) | Where-Object { Test-Path $_ } | Select-Object -First 1; \
    \
    if (-not $msvcBase) { \
      Get-ChildItem 'C:\Program Files*' -Recurse -Filter 'cl.exe' -ErrorAction SilentlyContinue | \
        Select-Object -First 5 | ForEach-Object { Write-Host "Found cl.exe at: $($_.FullName)" }; \
      throw 'MSVC directory not found — VS Build Tools installation may have failed'; \
    } \
    \
    $msvcVer = (Get-ChildItem $msvcBase | Sort-Object Name -Descending | Select-Object -First 1).FullName; \
    $clPath   = "$msvcVer\bin\Hostx64\x64"; \
    Write-Host "cl.exe path: $clPath"; \
    \
    [System.Environment]::SetEnvironmentVariable('JAVA_HOME', 'C:\graalvm', 'Machine'); \
    $syspath = [System.Environment]::GetEnvironmentVariable('PATH', 'Machine'); \
    [System.Environment]::SetEnvironmentVariable( \
      'PATH', \
      "C:\graalvm\bin;C:\apache-maven-$env:MAVEN_VERSION\bin;$clPath;$syspath", \
      'Machine' \
    ); \
    Write-Host '--- Environment configured ---'

# ---------------------------------------------------------------------------
# 5. Smoke-Test
# ---------------------------------------------------------------------------
RUN Write-Host '--- Smoke tests ---'; \
    native-image --version; \
    mvn --version; \
    cl 2>&1 | Select-Object -First 1; \
    Write-Host '--- All tools verified ---'
