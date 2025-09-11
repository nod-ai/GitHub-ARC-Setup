# escape=`
FROM mcr.microsoft.com/windows/servercore:ltsc2022

ARG RUNNER_OS=win
ARG RUNNER_ARCH=x64
ARG RUNNER_VERSION
ARG RUNNER_CONTAINER_HOOKS_VERSION=0.6.1

WORKDIR C:/home/runner

################################################################################
# Use PowerShell for initial installs
################################################################################
SHELL ["powershell", "-Command", "$ErrorActionPreference='Stop';$ProgressPreference='SilentlyContinue';"]

################################################################################
# Install GitHub Actions Runner
################################################################################
RUN Invoke-WebRequest -Uri "https://github.com/actions/runner/releases/download/v${env:RUNNER_VERSION}/actions-runner-${env:RUNNER_OS}-${env:RUNNER_ARCH}-${env:RUNNER_VERSION}.zip" -OutFile actions-runner.zip; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory('actions-runner.zip', $PWD); `
    Remove-Item -Force actions-runner.zip

################################################################################
# Install Chocolatey + tools (Git, Python, CMake, Docker CLI, etc.)
################################################################################
RUN Set-ExecutionPolicy Bypass -Scope Process -Force; `
    [Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor 3072; `
    Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1')); `
    choco feature enable -n allowGlobalConfirmation; `
    choco install -y git.install gh powershell-core azure-cli docker-cli docker-compose cmake python --installargs '"ADD_CMAKE_TO_PATH=System"'

# Correctly append Git & Python to PATH (handles spaces)
RUN setx /M PATH ('{0};{1};{2}' -f $Env:PATH, 'C:\Program Files\Git\bin', 'C:\Python313')

# Enable long paths in Windows
RUN New-ItemProperty -Path "HKLM:\\SYSTEM\\CurrentControlSet\\Control\\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

################################################################################
# Switch to CMD for VS Build Tools
################################################################################
SHELL ["cmd", "/S", "/C"]

# Download VS Build Tools installer
ADD https://aka.ms/vs/17/release/vs_buildtools.exe C:\TEMP\vs_buildtools.exe

# Install VS Build Tools (all workloads needed)
RUN (start /w C:\TEMP\vs_buildtools.exe --quiet --wait --norestart --nocache `
    --add Microsoft.VisualStudio.Workload.AzureBuildTools `
    --add Microsoft.VisualStudio.Workload.VCTools `
    --add Microsoft.VisualStudio.Workload.ManagedDesktop `
    --add Microsoft.VisualStudio.Workload.DesktopDevelopmentWithC++ `
    --add Microsoft.VisualStudio.Workload.CoreEditor `
    --add Microsoft.VisualStudio.Component.ATL `
    || IF "%ERRORLEVEL%"=="3010" EXIT 0) `
    && del /q C:\TEMP\vs_buildtools.exe

# Add VS Build Tools to PATH
RUN setx /M PATH "%PATH%;%ProgramFiles(x86)%\Microsoft Visual Studio\2022\BuildTools\Common7\Tools;%ProgramFiles(x86)%\Microsoft Visual Studio\2022\BuildTools\MSBuild\Current\Bin"

################################################################################
# Switch back to PowerShell for Python certs and vcpkg
################################################################################
SHELL ["powershell", "-Command"]

# Configure Python certificates
RUN python -m pip install --upgrade certifi setuptools; `
    setx SSL_CERT_FILE "C:\Users\ContainerAdministrator\AppData\Local\Programs\Python\Python310\Lib\site-packages\certifi\cacert.pem"; `
    certutil -generateSSTFromWU certs.sst; `
    certutil -addstore -f ROOT certs.sst; `
    del certs.sst

# --- vcpkg installation ---
# Clone vcpkg
RUN git clone https://github.com/microsoft/vcpkg.git C:\vcpkg

# Bootstrap vcpkg (disable telemetry)
RUN C:\vcpkg\bootstrap-vcpkg.bat -disableMetrics

# Add vcpkg to PATH
SHELL ["powershell", "-Command", "$ErrorActionPreference='Stop';$ProgressPreference='SilentlyContinue';"]
RUN $env:PATH += ';C:\vcpkg'; setx /M PATH $env:PATH

# Initialize VS Build Tools environment and install packages
RUN powershell -NoProfile -Command `
    "$ErrorActionPreference = 'Stop'; `
    $ProgressPreference = 'SilentlyContinue'; `
    cmd.exe /c '\"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat\" && vcpkg install zlib:x64-windows --clean-after-build'"


################################################################################
# Entrypoint: VS Dev prompt + PowerShell
################################################################################
ENTRYPOINT ["cmd.exe", "/S", "/C", "%ProgramFiles(x86)%\\Microsoft Visual Studio\\2022\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat && powershell.exe -NoLogo -ExecutionPolicy Bypass"]
