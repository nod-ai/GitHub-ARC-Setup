# escape=`

FROM mcr.microsoft.com/windows/servercore:ltsc2022
ARG RUNNER_OS=win
ARG RUNNER_ARCH=x64
ARG RUNNER_VERSION
ARG RUNNER_CONTAINER_HOOKS_VERSION=0.6.1

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';$ProgressPreference='silentlyContinue';"]

WORKDIR /home/runner

SHELL ["cmd", "/C"]

RUN setx /M PATH "%PATH%;C:/home/runner"

RUN echo $Env:PATH

SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';$ProgressPreference='silentlyContinue';"]

RUN `
  ###############################################################################################
  #   Install Actions Runner
  #   You must always install the runner, and you want the latest version to avoid the restrictions
  #   applied to out-of-date runners.
  #   https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/autoscaling-with-self-hosted-runners#:~:text=Warning,-Any%20updates%20released
  ###############################################################################################
  Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v${env:RUNNER_VERSION}/actions-runner-${env:RUNNER_OS}-${env:RUNNER_ARCH}-${env:RUNNER_VERSION}.zip -OutFile actions-runner.zip;`
  Add-Type -AssemblyName System.IO.Compression.FileSystem;`
  [System.IO.Compression.ZipFile]::ExtractToDirectory('actions-runner.zip', $PWD);`
  Remove-Item -Path actions-runner.zip -Force;`
  ###############################################################################################
  #   Install Runner Container Hooks
  #   While it is possible to include these hooks, Windows runners can't use these today. 
  #   GitHub documents that you must use Linux runners for Docker container actions, job containers,
  #   or service containers.
  #   See also https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#jobsjob_idservices
  #   and https://github.com/actions/runner/issues/904
  ###############################################################################################
  # Invoke-WebRequest -OutFile runner-container-hooks.zip -Uri https://github.com/actions/runner-container-hooks/releases/download/v${env:RUNNER_CONTAINER_HOOKS_VERSION}/actions-runner-hooks-k8s-${env:RUNNER_CONTAINER_HOOKS_VERSION}.zip;`
  # [System.IO.Compression.ZipFile]::ExtractToDirectory('runner-container-hooks.zip', (Join-Path -Path $PWD -ChildPath 'k8s'));`
  # Remove-Item -Path runner-container-hooks.zip -Force;`
  ###############################################################################################
  #   Install Git Using Choco
  #   Runners should have access to the latest version of Git and Git LFS, which we can
  #   install using Choco. This also makes a Bash shell available on Windows for scripting.
  #   You may want to include other tools and script engines as well.
  ###############################################################################################
  Set-ExecutionPolicy Bypass -Scope Process -Force;`
  [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072;`
  Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'));`
  choco install -y git.install gh powershell-core azure-cli;`
  choco feature enable -n allowGlobalConfirmation;`
  ###############################################################################################
  #   Install Docker CLI Using Choco
  #   It's important to know that Windows doesn't support nested containers, so you can't
  #   use a Docker-in-Docker on Windows. That frequently limits the value of having the
  #   Docker CLI available on your images.
  ###############################################################################################
  choco install docker-cli docker-compose -force;

# Append Git\bin to $PATH so bash.exe can be used.
RUN setx /M PATH $($Env:PATH + ';C:\Program Files\Git\bin')

RUN New-ItemProperty -Path "HKLM:\\SYSTEM\\CurrentControlSet\\Control\\FileSystem" -Name "LongPathsEnabled" -Value 1 -PropertyType DWORD -Force

# Download vswhere.exe from the official GitHub releases
RUN Invoke-WebRequest -Uri "https://github.com/microsoft/vswhere/releases/download/2.8.4/vswhere.exe" -OutFile "vswhere.exe" 

RUN .\vswhere.exe
RUN vswhere
# Set the entrypoint to cmd.exe so you can run vswhere

# Restore the default Windows shell for correct batch processing.
SHELL ["cmd", "/S", "/C"]

RUN `
    # Download the Build Tools bootstrapper.
    curl -SL --output vs_buildtools.exe https://aka.ms/vs/17/release/vs_buildtools.exe `
    `
    # Install Build Tools with the Microsoft.VisualStudio.Workload.AzureBuildTools workload, excluding workloads and components with known issues.
    && (start /w vs_buildtools.exe --quiet --wait --norestart --nocache `
        --installPath "%ProgramFiles(x86)%\Microsoft Visual Studio\2022\BuildTools" `
        --add Microsoft.VisualStudio.Workload.AzureBuildTools `
        --remove Microsoft.VisualStudio.Component.Windows10SDK.10240 `
        --remove Microsoft.VisualStudio.Component.Windows10SDK.10586 `
        --remove Microsoft.VisualStudio.Component.Windows10SDK.14393 `
        --remove Microsoft.VisualStudio.Component.Windows81SDK `
        || IF "%ERRORLEVEL%"=="3010" EXIT 0) `
    `
    # Cleanup
    && del /q vs_buildtools.exe

# Define the entry point for the docker container.
# This entry point starts the developer command prompt and launches the PowerShell shell.
RUN choco install cmake --version=3.31.0 -y --no-progress --installargs '"ADD_CMAKE_TO_PATH=System"'
RUN choco install python --version=3.13.7 -y --no-progress
RUN python -m pip install --upgrade certifi
RUN setx SSL_CERT_FILE "C:\Users\ContainerAdministrator\AppData\Local\Programs\Python\Python310\Lib\site-packages\certifi\cacert.pem"
RUN certutil -generateSSTFromWU certs.sst
RUN certutil -addstore -f ROOT certs.sst
RUN del certs.sst
RUN choco install visualstudio2022-workload-vctools "--add Microsoft.VisualStudio.Component.ATL" -y --no-progress
RUN python -m pip install setuptools
RUN choco install visualstudio2022buildtools -y
RUN choco install visualstudio2022community --package-parameters "--add Microsoft.VisualStudio.Workload.CoreEditor --add Microsoft.VisualStudio.Workload.ManagedDesktop --add Microsoft.VisualStudio.Workload.DesktopDevelopmentWithC++" -y
RUN choco install visualstudio2022-workload-nativedesktop -y
RUN git clone https://github.com/microsoft/vcpkg.git C:\vcpkg `
    && cd C:\vcpkg `
    && bootstrap-vcpkg.bat `
    && SETX /M PATH "C:\vcpkg;%PATH%" `
    && vcpkg install zlib:x64-windows --clean-after-build `
    && vcpkg remove zlib:x64-windows

ENTRYPOINT ["C:\\Program Files (x86)\\Microsoft Visual Studio\\2022\\BuildTools\\VC\\Auxiliary\\Build\\vcvars64.bat", "&&", "powershell.exe", "-NoLogo", "-ExecutionPolicy", "Bypass"]