---
layout: single
title:  "Setting Up IIS Docker on Mac"
date:   2024-11-12 12:00:00 +0000
categories: devops
tagline: "It's like fitting a square peg in a round port"
toc: true
toc_sticky: true
header:
  overlay_image: /assets/images/shane-rounce-1ZZ96uESRJQ-unsplash.jpg
  overlay_filter: 0.7
  caption: "Photo by [Shane Rounce](https://unsplash.com/@shanerounce) on [Unsplash](https://unsplash.com/photos/black-electrical-tower-1ZZ96uESRJQ)"
---

When working on a legacy application that requires IIS, setting up a development environment on a Mac can present a few challenges. My team faced this recently while making updates to a long-standing app that runs on IIS. Since Docker doesn’t support IIS natively on Mac, we had to find a workaround. Here’s how we set up a local IIS environment using **Docker**, **VirtualBox**, and **Vagrant**.

## Why Run IIS on a Mac?

In modern development, cross-platform flexibility is essential. However, some applications still rely on Windows-only environments, like IIS, which presents a challenge when the primary development machine is a Mac. Using Docker to create a consistent, replicable environment is ideal for this, but we quickly realized that getting IIS to run on a Mac requires some extra steps.

## The First Roadblock: No Native IIS on Mac

Our initial approach was to create a Docker container directly on macOS, but we quickly discovered that IIS Docker images won’t run natively on a Mac. Docker for Mac doesn’t support Windows containers due to the need for a Windows kernel. So, we had to create a virtual machine running Windows.

## Setting Up a Virtual Machine with VirtualBox and Vagrant

To emulate a Windows environment on macOS, we used [VirtualBox](https://www.virtualbox.org/) to create a virtual machine. While this can be done manually, setting it up with [Vagrant](https://www.vagrantup.com/) simplifies the process significantly. Vagrant scripts automate the download and setup of a Windows VM, complete with Docker installed. This allows us to create a Windows-based Docker environment quickly and consistently.

1.	Install Vagrant and VirtualBox – Ensure both Vagrant and VirtualBox are installed on your Mac. 
2.  Use the script from [this repo](https://github.com/StefanScherer/windows-docker-machine) to set up the VM. It will automate the creation and configuration of your Windows VM with Docker. In our case, we would need a Windows Server 2019:
```bash
$ git clone https://github.com/StefanScherer/windows-docker-machine
$ cd windows-docker-machine
$ vagrant up --provider virtualbox 2019-box
```

If you are running it for the first time, it would take a while to download the image and set up the VM.

## Switching Docker Context to the Windows VM

Once the Windows VM is up and running with Docker, the next step is to tell Docker on your Mac to switch context to this Windows environment. This way, any Docker commands you run will be executed within the Windows VM rather than on your Mac directly.
```bash
$ docker context use 2019-box
```

## File System Mapping and Source Code Management

In this setup, the VM maps the Mac’s home directory (e.g., `/Users/FooBar`) to a corresponding directory on Windows (`C:\Users\FooBar`). This is useful for accessing files across the Mac and the Windows VM.

However, **we encountered an issue where mounted volumes weren’t recognized by IIS**, which meant it couldn’t serve files directly from the mounted directory. To work around this, we took a two-step approach:

### Initial Setup in the Dockerfile

In the Dockerfile, we delete the default IIS sample website files in `C:\inetpub\wwwroot`, and then copy the initial source files to this directory.
```Dockerfile
# delete existing content in wwwroot
RUN Remove-Item -Recursive C:\inetpub\wwwroot\*
# copy initial source files
WORKDIR /inetpub/wwwroot
COPY src/ .
```

### Ongoing File Sync with ROBOCOPY

To keep the source code updated without relying on live mounts, we set up a scheduled task that periodically runs `ROBOCOPY`([The robust file and folder copy](https://ss64.com/nt/robocopy.html)). This command copies any changes from the mounted source directory (e.g., `C:\website`) to `C:\inetpub\wwwroot`, making the latest version of the code available to IIS.

Here is the PowerShell script that sets up the scheduled task:
```powershell
$action = New-ScheduledTaskAction -WorkingDirectory "c:\windows\system32" -Execute "Robocopy.exe" -Argument "C:\WEBSITE C:\INETPUB\\WWWROOT /S /XF .DS_Store /XD .git"
$trigger = New-ScheduledTaskTrigger -At (Get-Date) -Once -RepetitionInterval (New-TimeSpan -Minutes 1) -RepetitionDuration (New-TimeSpan -Days 1) 
$principal = New-ScheduledTaskPrincipal -UserId "NT AUTHORITY\SYSTEM" -LogonType "ServiceAccount" -RunLevel Highest
$settings = New-ScheduledTaskSettingsSet
$task = New-ScheduledTask -Action $action -Principal $principal -Trigger $trigger 
Register-ScheduledTask -TaskName "Sync Website Files" -InputObject $task -Force
```
The `-RepetitionInterval` and `-RepetitionDuration` options specify how often you need updates. Here we set it to every minute.

The `/XF` and `/XD` arguments are used to exclude certain files and directories from being copied.

Save the script to a file (e.g. `register_task.ps1`), and then run it in the Dockerfile:
```Dockerfile
COPY register_task.ps1 .
RUN ./register_task.ps1
```

Finally, remember to mount your source directory so that it is visible to the script, in `compose.yaml`:
```yaml
  volumes:
    - c:\users\foobar\website:c:\website:rw
```

To sum up, here’s how the source code moves from your Mac to the IIS server in the Docker container, step by step:
1.	Mac to VM: The source code directory on the Mac, e.g., `/Users/FooBar/website`, is mounted into the VM as `C:\Users\FooBar\website`. This mapping enables the VM to access files stored on the Mac, making any changes available on both systems.
2.	VM to Docker Container: In the `compose.yaml` file, `C:\Users\FooBar\website` is mounted as a volume in the container. This mount points to a directory inside the container, such as `C:\website`, which serves as the source directory in the container’s Windows filesystem.
3.	File Sync to IIS Root: Since IIS doesn’t directly recognize files in mounted volumes, a scheduled ROBOCOPY task is set up to copy the content of `C:\website` into `C:\inetpub\wwwroot`, where IIS can serve the files.

This layered file-mapping setup allows the development code to move seamlessly from the Mac through to the IIS server, updating automatically with ROBOCOPY syncs.

## Installing and Enabling Features

Most of the IIS features, which can be installed and enabled through PowerShell commands, can be included in `Dockerfile`. For example:
```Dockerfile
SHELL ["powershell", "-NoProfile", "-Command"]

# install server side include
RUN Install-WindowsFeature -Name Web-Includes

# enable *.html for server side include
RUN New-WebHandler -Name "SSI-html" -Path "*.html" -Verb "*" -Modules "ServerSideIncludeModule" -ResourceType "File" -RequiredAccess "Script"
RUN Add-WebConfiguration "/system.webServer/handlers/@accessPolicy" -Value "Script"

# add webp MIME type
RUN Add-WebConfigurationProperty -PSPath 'MACHINE/WEBROOT/APPHOST' -Filter "system.webServer/staticContent" -Name "." -Value @{ fileExtension='.webp'; mimeType='image/webp' }
```

One exception is the **URL rewrite module**. For some reason it can't be installed through `Dockerfile`. So all we did is to download the module first in the `Dockerfile`:
```Dockerfile
# download the URL rewrite module
ADD https://download.microsoft.com/download/1/2/8/128E2E22-C1B9-44A4-BE2A-5859ED1D4592/rewrite_amd64_en-US.msi /install/rewrite_amd64.msi
```
And then run the install *after* the container is up, *manually*:
```bash
$ docker compose up -d
$ docker exec website-docker-iis-1 powershell -NoProfile -Command "msiexec.exe /i C:\\install\\rewrite_amd64.msi /passive"
```

## Testing

Note that the address of your IIS site isn’t `localhost`, but rather the IP of the Windows VM hosting IIS, which is the same IP that Docker uses to communication between different contexts. To find this IP, you can run the following command:
```bash
$ docker context inspect 2019-box
```

## Get Started with the Setup Script

To help you get up and running quickly, I’ve included an example `Dockerfile`, `compose.yaml` and other scripts to this [repository](https://github.com/ganyinghung/iis-docker-on-mac). It includes the essential configurations for setting up a Windows VM with Docker, configuring file syncs, and preparing the IIS environment. Feel free to clone or fork the repository to adapt it to your specific needs.

## Troubleshooting Common Issues

Setting up IIS on a Mac through Docker and a Windows VM can come with a few quirks. Here are some common issues you might encounter and tips on how to resolve them:

### Docker Context Not Switching

Issue: After setting up, Docker commands still execute on the Mac instead of the Windows VM.

Solution: Make sure you’ve switched Docker’s context to the Windows VM by using the correct docker context command. It should be the one you picked in the Vagrant script but you can always verify the current context with:
```bash
$ docker context ls
```
Double-check that your VM is up and Docker is running within it.

If you got a connection error when trying to `docker compose up`, you may try to provision the VM again:
```bash
$ cd window-docker-machine
$ vagrant provision
```

### File Changes Not Reflecting in IIS

Issue: Modifications to files on your Mac aren’t showing up in IIS.

Solution: It could be that the PowerShell script `register_task.ps1` hadn't been executed properly. Try to fix any issues, or you can do a manual copy:
```bash
$ docker exec website-docker-iis-1 powershell -NoProfile -Command "robocopy C:\\WEBSITE C:\\INETPUB\\WWWROOT /S /XF .DS_Store /XD .git"
```

### VirtualBox Networking Issues

Issue: The VM isn’t accessible, or Docker on the Mac cannot communicate with the VM.

Solution: Check VirtualBox’s network settings to ensure the VM is on a network that allows host-machine communication, like “Bridged Adapter” mode. Also, verify that firewall settings aren’t blocking Docker’s traffic between the Mac and the VM. 

## Conclusion

Setting up IIS on a Mac isn’t straightforward due to the lack of native support, but with VirtualBox, Vagrant, and Docker, we managed to create a functional workaround. By using an automated VM setup, a Docker context switch, and file syncing with ROBOCOPY, this setup allows us to maintain a consistent and flexible development environment, enabling easy updates to our legacy application on macOS.

## Useful Links

IIS Docker
- <http://www.codesin.net/post/Containerise-Legacy-Windows-Apps/>
- <https://github.com/StefanScherer/windows-docker-machine>
- <https://github.com/microsoft/iis-docker>

Windows PowerShell and Features
- <https://learn.microsoft.com/en-us/previous-versions/iis/6.0-sdk/ms525185(v=vs.90)>
- <https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/register-scheduledtask?view=windowsserver2019-ps>
- <https://learn.microsoft.com/en-us/powershell/module/webadministration/add-webconfiguration?view=windowsserver2019-ps>
- <https://www.iis.net/downloads/microsoft/url-rewrite>
- <https://ss64.com/nt/robocopy.html>
