# escape=`
FROM mcr.microsoft.com/windows/servercore:ltsc2022
ARG RUNNER_OS=win
ARG RUNNER_ARCH=x64
ARG RUNNER_VERSION
ARG RUNNER_CONTAINER_HOOKS_VERSION=0.6.1

WORKDIR C:/home/runner

# PowerShell as default shell
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';$ProgressPreference='SilentlyContinue';"]

################################################################################
# Install GitHub Actions Runner
################################################################################
RUN Invoke-WebRequest -Uri "https://github.com/actions/runner/releases/download/v${env:RUNNER_VERSION}/actions-runner-${env:RUNNER_OS}-${env:RUNNER_ARCH}-${env:RUNNER_VERSION}.zip" -OutFile actions-runner.zip; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory('actions-runner.zip', $PWD); `
    Remove-Item -Force actions-runner.zip

################################################################################
# Install Chocolatey + common tools
################################################################################
RUN Set-ExecutionPolicy Bypass -Scope Process -Force; `
    [Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor 3072; `
    Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1')); `
    choco feature enable -n allowGlobalConfirmation; `
    choco install -y git.install gh powershell-core azure-cli docker-cli docker-compose cmake python --installargs '"ADD_CMAKE_TO_PATH=System"'; `
    refreshenv

# Add Git Bash to PATH
RUN setx /M PATH "$($Env:PATH);C:\Program Files\Git\bin"

# Enable long paths
RUN New-ItemProperty -Path "HKLM:\\SYSTEM\\CurrentControlSet\\Control\\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

################################################################################
# Switch to cmd for VS installer
################################################################################
SHELL ["cmd", "/S", "/C"]

# Download VS Build Tools
ADD https://aka.ms/vs/17/release/vs_buildtools.exe C:\\TEMP\\vs_buildtools.exe

# Install VS Build Tools with workloads (NativeDesktop, VCTools, ManagedDesktop, ATL, CoreEditor)
RUN C:\\TEMP\\vs_buildtools.exe --quiet --wait --norestart --nocache ^
    --installPath "C:\\BuildTools" ^
    --add Microsoft.VisualStudio.Workload.AzureBuildTools ^
    --add Microsoft.VisualStudio.Workload.VCTools ^
    --add Microsoft.VisualStudio.Workload.ManagedDesktop ^
    --add Microsoft.VisualStudio.Workload.DesktopDevelopmentWithC++ ^
    --add Microsoft.VisualStudio.Workload.CoreEditor ^
    --add Microsoft.VisualStudio.Component.ATL ^
    --remove Microsoft.VisualStudio.Component.Windows10SDK.10240 ^
    --remove Microsoft.VisualStudio.Component.Windows10SDK.10586 ^
    --remove Microsoft.VisualStudio.Component.Windows10SDK.14393 ^
    --remove Microsoft.VisualStudio.Component.Windows81SDK ^
 || IF "%ERRORLEVEL%"=="3010" EXIT 0

# Clean up installer
RUN del C:\\TEMP\\vs_buildtools.exe

# Add Build Tools to PATH
RUN setx /M PATH "%PATH%;C:\\BuildTools\\Common7\\Tools;C:\\BuildTools\\MSBuild\\Current\\Bin"

################################################################################
# Back to PowerShell for Python, certs, vcpkg
################################################################################
SHELL ["powershell", "-Command"]

# Upgrade Python packages & configure certs
RUN python -m pip install --upgrade certifi setuptools; `
    setx SSL_CERT_FILE "C:\\Users\\ContainerAdministrator\\AppData\\Local\\Programs\\Python\\Python310\\Lib\\site-packages\\certifi\\cacert.pem"; `
    certutil -generateSSTFromWU certs.sst; `
    certutil -addstore -f ROOT certs.sst; `
    del certs.sst

# Install vcpkg
RUN git clone https://github.com/microsoft/vcpkg.git C:\\vcpkg; `
    C:\\vcpkg\\bootstrap-vcpkg.bat; `
    setx /M PATH "$($Env:PATH);C:\\vcpkg"; `
    C:\\vcpkg\\vcpkg.exe install zlib:x64-windows --clean-after-build; `
    C:\\vcpkg\\vcpkg.exe remove zlib:x64-windows

################################################################################
# Entrypoint: VS Dev prompt + PowerShell
################################################################################
ENTRYPOINT ["cmd.exe", "/S", "/C", "C:\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat && powershell.exe -NoLogo -ExecutionPolicy Bypass"]
