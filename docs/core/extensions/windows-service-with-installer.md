---
title: Create a Windows Service installer
description: Learn how to create a Windows Service installer project.
author: IEvangelist
ms.author: dapine
ms.date: 12/13/2023
ms.topic: tutorial
---

# Create a Windows Service installer

When you create a .NET Windows Service (not to be mistaken with a .NET Framework Windows Service), you may want to create an installer for your service. Without an installer, users would have to know how to install and configure your service. An installer bundles your app's executables and exposes a customizable installation user experience. This tutorial is a continuation of the [Create a Windows Service](windows-service.md) tutorial. It shows how to create an installer for your .NET Windows Service.

In this tutorial, you'll learn how to:

> [!div class="checklist"]
>
> - Install the Visual Studio Installer Projects extension.
> - Create a setup project.
> - Update an existing .NET Worker project to support installation.
> - Automate the installation and uninstallation with the Windows Service Control Manager.

## Prerequisites

- You're expected to have completed the [Create a Windows Service](windows-service.md) tutorial, or be prepared to clone the sample repo.
- The [.NET 8.0 SDK or later](https://dotnet.microsoft.com/download/dotnet)
- A Windows OS
- A .NET integrated development environment (IDE)
  - Feel free to use [Visual Studio](https://visualstudio.microsoft.com)
- An existing .NET Windows Service

## Install tooling dependencies

# [Wix Toolset](#tab/wix)

Start by installing the Wix Toolset. The Wix Toolset is a set of tools that build Windows installation packages from XML source code.

```dotnetcli
dotnet tool install --global wix
```

Next, install the [HeatWave for VS2022 extension](https://marketplace.visualstudio.com/items?itemName=FireGiant.FireGiantHeatWaveDev17). After installing, restart Visual Studio and you'll see new project templates available.

# [Microsoft installer extension](#tab/ext)

Install the [Microsoft Visual Studio Installer Projects extension](https://marketplace.visualstudio.com/items?itemName=VisualStudioClient.MicrosoftVisualStudio2022InstallerProjects). After installing, restart Visual Studio and you'll see new project templates available.

---

## Get existing project

This tutorial is based on the app created as part of the [Create a Windows Service using BackgroundService](windows-service.md) tutorial. You can either clone the sample repo or use the app you built in the previous tutorial.

[!INCLUDE [workers-samples-browser](includes/workers-samples-browser.md)]

# [Wix Toolset](#tab/wix)

Open the solution in Visual Studio, and select <kbd>F5</kbd> to ensure that the app builds and runs as expected. Press <kbd>Ctrl</kbd>+<kbd>C</kbd> to stop the app.

# [Microsoft installer extension](#tab/ext)

Open the solution in Visual Studio, and select <kbd>F5</kbd> to ensure that the app builds and runs as expected. Press <kbd>Ctrl</kbd>+<kbd>C</kbd> to stop the app.

### Handle installation switches

The Windows Service app needs to handle installation switches. The setup project will call into the Windows Service app with `/Install` and `/Uninstall` switches during installation and uninstallation respectively. When these switches are present, the app will behave differently, in that it will only perform installation or uninstallation using the Windows Service Control Manager executable (_sc.exe_).

For the app to call a separate process, install the [CliWrap](https://www.nuget.org/packages/CliWrap) NuGet package as a convenience. To install the `CliWrap` package, use the following command.

```dotnetcli
dotnet add App.WindowsService.csproj package CliWrap
```

For more information, see [dotnet package add](../tools/dotnet-package-add.md).

With `CliWrap` installed, open the _Program.cs_ file of the `App.WindowsService` project. After the `using` directives, but before the `IHost` is created, add the following code:

```csharp
using CliWrap;

const string ServiceName = ".NET Joke Service";

if (args is { Length: 1 })
{
    try
    {
        string executablePath =
            Path.Combine(AppContext.BaseDirectory, "App.WindowsService.exe");

        if (args[0] is "/Install")
        {
            await Cli.Wrap("sc")
                .WithArguments(new[] { "create", ServiceName, $"binPath={executablePath}", "start=auto" })
                .ExecuteAsync();
        }
        else if (args[0] is "/Uninstall")
        {
            await Cli.Wrap("sc")
                .WithArguments(new[] { "stop", ServiceName })
                .ExecuteAsync();

            await Cli.Wrap("sc")
                .WithArguments(new[] { "delete", ServiceName })
                .ExecuteAsync();
        }
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex);
    }

    return;
}
```

The preceding code:

- Declares the service name as a `const string` value.
- Checks the `args` for a single value.
- Gets the executable path from the <xref:System.AppContext.BaseDirectory?displayProperty=nameWithType>.
- When the `"/Install"` switch is present, `sc create ".NET Joke Service" binPath="path/to/App.WindowsService.exe" start=auto` is called.
- When the `"/Uninstall"` switch is present, `sc stop ".NET Joke Service"` and `sc delete ".NET Joke Service"` are called.

When no installation switches are present, the app behaves as it did before, but it now includes installation functionality.

> [!NOTE]
> Defining custom actions, and manually handling installation switches is considered an anti-pattern. While this documentation demonstrates how to do it, you might want to consider using the third-party Wix Toolset instead.

---

## Add new setup project

# [Wix Toolset](#tab/wix)

To add a new Wix setup project, right-click on the solution in the **Solution Explorer** and select **Add > New Project**:

:::image type="content" source="media/workers/new-wix-setup-project.png" alt-text="Add new project dialog: New MSI Package (Wix v4) Project.":::

Select **MSI Package (Wix v4)** from the available templates, then select **Next**. Provide the desired **Name** and **Location**, then select **Create**.

# [Microsoft installer extension](#tab/ext)

To add a new setup project, right-click on the solution in the **Solution Explorer** and select **Add > New Project**:

:::image type="content" source="media/workers/new-setup-project.png" alt-text="Add new project dialog: New Setup Project.":::

Select **Setup Project** from the available templates, then select **Next**. Provide the desired **Name** and **Location**, then select **Create**.

---

### Configure installer project

# [Wix Toolset](#tab/wix)

To configure the setup project, you first must add a reference to the `App.WindowsService` project. Right-click the setup project in the **Solution Explorer**, and then select **Add > Project Reference**.

The template includes example component and localization files. Delete these, files leaving only the _Package.wxs_ file. Your project should now include a `ProjectReference` element, similar to the following:

```xml
<Project Sdk="WixToolset.Sdk/4.0.0">
  <ItemGroup>
    <ProjectReference Include="..\App.WindowsService.csproj" />
  </ItemGroup>
</Project>
```

After the project reference has been added, configure the _Package.wxs_ file. Open the file in the editor, and then replace the contents with the following:

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- Define the variables in "$(var.*) expressions" -->
<?define Name = ".NET Joke Service" ?>
<?define Manufacturer = "Microsoft" ?>
<?define Version = "1.0.0.0" ?>
<?define UpgradeCode = "9ED3FF33-8718-444E-B44B-69A2344B7E98" ?>

<Wix xmlns="http://wixtoolset.org/schemas/v4/wxs">
    <Package Name="$(Name)"
             Manufacturer="$(Manufacturer)"
             Version="$(Version)"
             UpgradeCode="$(var.UpgradeCode)"
             Compressed="true">

        <!-- Allow upgrades and prevent downgrades -->
        <MajorUpgrade DowngradeErrorMessage="A later version of [ProductName] is already installed. Setup will now exit." />

        <!-- Define the directory structure -->
        <Directory Id="TARGETDIR" Name="SourceDir">
            <Directory Id="ProgramFiles64Folder">

                <!-- Create a folder inside program files -->
                <Directory Id="ROOTDIRECTORY" Name="$(var.Manufacturer)">

                    <!-- Create a folder within the parent folder given the name -->
                    <Directory Id="INSTALLFOLDER" Name="$(Name)" />
                </Directory>
            </Directory>
        </Directory>

        <!-- The files inside this DirectoryRef are linked to
             the App.WindowsService directory via INSTALLFOLDER -->
        <DirectoryRef Id="INSTALLFOLDER">

            <!-- Create a single component which is the App.WindowsService.exe file -->
            <Component Id="ServiceExecutable" Bitness="always64">

                <!-- Copies the App.WindowsService.exe file using the
                     project reference preprocessor variables -->
                <File Id="App.WindowsService.exe"
                      Source="$(var.App.WindowsService.TargetDir)publish\App.WindowsService.exe"
                      KeyPath="true" />

                <!-- Remove all files from the INSTALLFOLDER on uninstall -->
                <RemoveFile Id="ALLFILES" Name="*.*" On="both" />

                <!-- Tell WiX to install the Service -->
                <ServiceInstall Id="ServiceInstaller"
                                Type="ownProcess"
                                Name="App.WindowsService"
                                DisplayName="$(Name)"
                                Description="A joke service that periodically logs nerdy humor."
                                Start="auto"
                                ErrorControl="normal" />

                <!-- Tell WiX to start the Service -->
                <ServiceControl Id="StartService"
                                Start="install"
                                Stop="both"
                                Remove="uninstall"
                                Name="App.WindowsService"
                                Wait="true" />
            </Component>
        </DirectoryRef>

        <!-- Tell WiX to install the files -->
        <Feature Id="Service" Title="App.WindowsService Setup" Level="1">
            <ComponentRef Id="ServiceExecutable" />
        </Feature>

    </Package>
</Wix>
```

When you build the project, the output is an MSI file that can be used to install and uninstall the service.

# [Microsoft installer extension](#tab/ext)

To configure the installer project, select the project in the **Solution Explorer**. Select <kbd>F4</kbd> to open the project properties pane. You can configure the app's "add" and "remove" icons, author, manufacturer, product name, title, target platform, and so on.

:::image type="content" source="media/workers/f4-installer-properties.png" alt-text="Installer project properties pane.":::

The installer project needs to define two custom actions for installation behavior. Right-click the project in the **Solution Explorer**, and then select **View > Custom Actions**.

:::image type="content" source="media/workers/custom-actions.png" alt-text="Installer project Custom Actions context menu.":::

From the **Custom Actions** window, select **Install > Add Custom Action**.

:::image type="content" source="media/workers/select-item.png" alt-text="Custom Actions properties dialog: select item.":::

Double-click the **Application Folder** (or select **Add Output...** with **Application Folder** selected), and select **Publish Items from App.WindowsService (Active)**.

:::image type="content" source="media/workers/select-item-publish.png" alt-text="Custom Actions properties dialog: select item app folder.":::

Select **Ok** to confirm the selection. Right-click the added **Publish Items from App.WindowsService (Active)** node under **Install**, then select **Properties**.

:::image type="content" source="media/workers/install-properties.png" alt-text="Publish items properties.":::

Add `/Install` to the **Arguments**. Follow these same steps for **Uninstall**, adding `/Uninstall` to the **Arguments**.

Once you've done both, you should see the following:

:::image type="content" source="media/workers/custom-actions-added.png" alt-text="Custom Actions with Install and Uninstall actions defined.":::

With these updates, the setup project has been configured to delegate its *Install* and *Uninstall* actions to call into the Windows Service app with appropriate arguments.

---

## Test installation

# [Wix Toolset](#tab/wix)

To test the installer, publish the _App.WindowsService_ project. Right-click the project in the **Solution Explorer**, and then select **Publish**. Once published with the profile you created in the previous tutorial, the executable will be in the publish directory. Next, you **Build** the setup project and run the installer.

You need to run the installation as an administrator. To do this, right-click the MSI file, and then select **Run as administrator**.

Once the service is installed, you can open **Services** to see the service running. To uninstall the service, use the **Windows Add or Remove Programs** feature to call the installer.

# [Microsoft installer extension](#tab/ext)

To test the installer, expand **Solution Configurations** in Visual Studio, and select **Release** (assuming **Debug** was selected). Build the solution and then right-click on the setup project and select **Build**. By default, setup projects are not part of the build.

Select **View > Output**, and ensure that the **Show output from** dropdown has **Build** selected. The Microsoft Installer (MSI) file path is displayed. Copy the path and open the installer. Run the installer.

You need to run the installation as an administrator. To do this, right-click the MSI file, and then select **Run as administrator**.

:::row:::
    :::column:::
        **1. Installer welcome dialog**
        :::image type="content" source="media/workers/installer-welcome.png" lightbox="media/workers/installer-welcome.png" alt-text="Installer welcome dialog.":::
    :::column-end:::
    :::column:::
        **2. Installer select folder dialog**
        :::image type="content" source="media/workers/installer-select-folder.png" lightbox="media/workers/installer-select-folder.png" alt-text="Installer select folder dialog.":::
    :::column-end:::
:::row-end:::

:::row:::
    :::column:::
        **3. Installer confirm dialog**
        :::image type="content" source="media/workers/installer-confirm.png" lightbox="media/workers/installer-confirm.png" alt-text="Installer confirm dialog.":::
    :::column-end:::
    :::column:::
        **4. Installer complete dialog**
        :::image type="content" source="media/workers/installer-complete.png" lightbox="media/workers/installer-complete.png" alt-text="Installer complete dialog.":::
    :::column-end:::
:::row-end:::

Once the service is installed, you can open **Services** to start the service. To uninstall the service, use the **Windows Add or Remove Programs** feature to call the installer.

---

## See also

- [Worker Services in .NET](workers.md)
- [Create a Queue Service](queue-service.md)
- [Use scoped services within a `BackgroundService`](scoped-service.md)
- [Implement the `IHostedService` interface](timer-service.md)
