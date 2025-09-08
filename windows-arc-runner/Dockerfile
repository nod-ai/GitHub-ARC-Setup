# escape=`

FROM mcr.microsoft.com/vs2022/buildtools:17.13.1-windowsservercore-ltsc2022

ARG RUNNER_OS=win
ARG RUNNER_ARCH=x64
ARG RUNNER_VERSION
ARG RUNNER_CONTAINER_HOOKS_VERSION=0.6.1

# Use PowerShell as default shell
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';$ProgressPreference='silentlyContinue';"]

WORKDIR /home/runner

# Ensure PATH is updated for runner
RUN setx /M PATH "%PATH%;C:/home/runner"

###############################################################################################
#   Install Actions Runner
###############################################################################################
RUN Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v${env:RUNNER_VERSION}/actions-runner-${env:RUNNER_OS}-${env:RUNNER_ARCH}-${env:RUNNER_VERSION}.zip -OutFile actions-runner.zip; `
    Add-Type -AssemblyName System.IO.Compression.FileSystem; `
    [System.IO.Compression.ZipFile]::ExtractToDirectory('actions-runner.zip', $PWD); `
    Remove-Item -Path actions-runner.zip -Force;

###############################################################################################
#   Install Git, GH CLI, PowerShell Core, Azure CLI
###############################################################################################
RUN Set-ExecutionPolicy Bypass -Scope Process -Force; `
    [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; `
    Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1')); `
    choco feature enable -n allowGlobalConfirmation; `
    choco install -y git.install gh powershell-core azure-cli docker-cli docker-compose cmake --version=3.31.0; `
    choco install python -y --no-progress

# Add Git to PATH for bash.exe
RUN setx /M PATH "$($Env:PATH);C:\Program Files\Git\bin"

# Enable long paths
RUN New-ItemProperty -Path "HKLM:\\SYSTEM\\CurrentControlSet\\Control\\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

###############################################################################################
#   Python SSL certs setup
###############################################################################################
RUN python -m pip install --upgrade certifi setuptools; `
    setx SSL_CERT_FILE "C:\Users\ContainerAdministrator\AppData\Local\Programs\Python\Python310\Lib\site-packages\certifi\cacert.pem"; `
    certutil -generateSSTFromWU certs.sst; `
    certutil -addstore -f ROOT certs.sst; `
    del certs.sst

###############################################################################################
#   Install vcpkg and test basic package
###############################################################################################
RUN git clone https://github.com/microsoft/vcpkg.git C:\vcpkg; `
    cd C:\vcpkg; `
    bootstrap-vcpkg.bat; `
    setx /M PATH "C:\vcpkg;$Env:PATH"; `
    vcpkg install zlib:x64-windows --clean-after-build; `
    vcpkg remove zlib:x64-windows

###############################################################################################
#   Entrypoint: Developer command prompt with PowerShell
###############################################################################################
ENTRYPOINT ["C:\\Program Files (x86)\\Microsoft Visual Studio\\2022\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat", "&&", "powershell.exe", "-NoLogo", "-ExecutionPolicy", "Bypass"]
