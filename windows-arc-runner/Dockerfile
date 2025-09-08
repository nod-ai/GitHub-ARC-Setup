# escape=`

FROM mcr.microsoft.com/dotnet/framework/sdk:4.8-windowsservercore-ltsc2022
ARG RUNNER_OS=win
ARG RUNNER_ARCH=x64
ARG RUNNER_VERSION
ARG RUNNER_CONTAINER_HOOKS_VERSION=0.6.1

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';$ProgressPreference='silentlyContinue';"]

WORKDIR C:/home/runner

# Make sure PATH includes runner home
RUN setx /M PATH "$($Env:PATH);C:/home/runner"

###############################################################################################
# Install GitHub Actions Runner
###############################################################################################
RUN Invoke-WebRequest -Uri "https://github.com/actions/runner/releases/download/v${env:RUNNER_VERSION}/actions-runner-${env:RUNNER_OS}-${env:RUNNER_ARCH}-${env:RUNNER_VERSION}.zip" -OutFile actions-runner.zip ; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem ; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory('actions-runner.zip', $PWD) ; `
    Remove-Item -Path actions-runner.zip -Force

###############################################################################################
# Install Chocolatey + common tools
###############################################################################################
RUN Set-ExecutionPolicy Bypass -Scope Process -Force ; `
    [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072 ; `
    Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1')) ; `
    choco feature enable -n allowGlobalConfirmation ; `
    choco install -y git.install gh powershell-core azure-cli docker-cli docker-compose

# Add Git Bash to PATH
RUN setx /M PATH "$($Env:PATH);C:\Program Files\Git\bin"

# Enable long paths
RUN New-ItemProperty -Path "HKLM:\\SYSTEM\\CurrentControlSet\\Control\\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

###############################################################################################
# Install Visual Studio 2022 Build Tools + workloads
###############################################################################################
SHELL ["cmd", "/S", "/C"]

ADD https://aka.ms/vs/17/release/vs_buildtools.exe C:\\TEMP\\vs_buildtools.exe

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

RUN del C:\\TEMP\\vs_buildtools.exe

# Add MSBuild + VS tools to PATH
RUN setx /M PATH "%PATH%;C:\\BuildTools\\Common7\\Tools;C:\\BuildTools\\MSBuild\\Current\\Bin"

###############################################################################################
# Back to PowerShell for rest
###############################################################################################
SHELL ["powershell", "-Command"]

# Install CMake and Python
RUN choco install cmake --version=3.31.0 -y --no-progress --installargs '"ADD_CMAKE_TO_PATH=System"'
RUN choco install python -y --no-progress

# Configure Python SSL certs
RUN python -m pip install --upgrade certifi setuptools ; `
    setx SSL_CERT_FILE "C:\\Users\\ContainerAdministrator\\AppData\\Local\\Programs\\Python\\Python310\\Lib\\site-packages\\certifi\\cacert.pem" ; `
    certutil -generateSSTFromWU certs.sst ; `
    certutil -addstore -f ROOT certs.sst ; `
    del certs.sst

###############################################################################################
# Install vcpkg
###############################################################################################
RUN git clone https://github.com/microsoft/vcpkg.git C:\\vcpkg ; `
    cd C:\\vcpkg ; `
    .\\bootstrap-vcpkg.bat ; `
    setx /M PATH "$($Env:PATH);C:\\vcpkg" ; `
    .\\vcpkg install zlib:x64-windows --clean-after-build ; `
    .\\vcpkg remove zlib:x64-windows

###############################################################################################
# Entrypoint: open VS dev prompt then PowerShell
###############################################################################################
ENTRYPOINT ["cmd.exe", "/S", "/C", "C:\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat && powershell.exe -NoLogo -ExecutionPolicy Bypass"]
