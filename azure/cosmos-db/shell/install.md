---
title: Install Azure Cosmos DB Shell
description: Step-by-step guide to install Azure Cosmos DB Shell using VS Code Marketplace, NuGet package, or self-contained binaries.
author: sajeetharan
ms.author: sasinnat
ms.reviewer: mjbrown
ms.service: azure-cosmos-db
ms.topic: how-to
ms.date: 05/04/2024
---

# Install Azure Cosmos DB Shell

Azure Cosmos DB Shell is available through three distribution methods. Choose the one that best fits your workflow.

## Option 1: VS Code extension (recommended)

The VS Code extension provides seamless integration with the code editor.

### Method 1a: Install from VS Code Marketplace

1. **Open VS Code**
   - Launch Visual Studio Code on your machine

2. **Open Extensions Marketplace**
   - Click the Extensions icon (Ctrl+Shift+X / Cmd+Shift+X)
   - Or go to View > Extensions

3. **Search for Cosmos DB**
   - Type "Cosmos DB" in the search box
   - Look for "Azure Cosmos DB Shell" by Microsoft

4. **Install the Extension**
   - Click the Install button
   - Wait for installation to complete

5. **Verify Installation**
   - The extension should appear in your installed extensions list
   - You can now access Cosmos DB Shell from the command palette

### Method 1b: Install from VSIX file

1. **Download Extension**
   - Download `vscode-cosmosdb-0.33.3.vsix` (or latest version) from the releases

2. **Install Extension**
   - Open VS Code
   - Go to Extensions (Ctrl+Shift+X / Cmd+Shift+X)
   - Click the "..." menu and select "Install from VSIX"
   - Select the downloaded file

3. **Complete Installation**
   - Wait for the installation to finish
   - Reload VS Code if prompted

### VS Code extension requirements

- **VS Code Version**: 1.85 or later
- **Supported Platforms**: Windows, macOS, Linux
- **Disk Space**: ~50 MB

For detailed VS Code integration features, see [Visual Studio Code Extension Setup](visual-studio-code.md).

## Option 2: NuGet package (.NET global tool)

Install as a .NET global tool for command-line access.

### Prerequisites

- **.NET SDK**: Version 10.0 or later
- **Platform Support**: Windows, macOS (Intel and Apple Silicon), Linux (x64 and ARM)

### Installation steps

1. **Open Terminal or Command Prompt**
   - Windows: Open Command Prompt or PowerShell
   - macOS/Linux: Open Terminal

2. **Install Global Tool**
   ```bash
   dotnet tool install --global CosmosDBShell --prerelease
   ```

3. **Verify Installation**
   ```bash
   cosmosdbshell --version
   ```

### Update to latest version

```bash
dotnet tool update --global CosmosDBShell --prerelease
```

### Uninstall

```bash
dotnet tool uninstall --global CosmosDBShell
```

### Package details

- **Package Name**: CosmosDBShell
- **Version**: 1.0.213-preview
- **NuGet URL**: [CosmosDBShell on NuGet.org](https://www.nuget.org/packages/CosmosDBShell/1.0.213-preview)
- **License**: MIT

## Option 3: Self-contained binary (no .NET required)

Download and extract a self-contained binary for your platform.

### Prerequisites

- No runtime requirements (includes .NET runtime)
- Sufficient disk space (~200 MB)

### Installation steps

1. **Download**
   - Choose your platform:
     - **Windows (x64)**: `cosmosdbshell-win-x64.zip`
     - **Windows (ARM)**: `cosmosdbshell-win-arm64.zip`
     - **macOS (Intel)**: `cosmosdbshell-macos-x64.tar.gz`
     - **macOS (Apple Silicon)**: `cosmosdbshell-macos-arm64.tar.gz`
     - **Linux (x64)**: `cosmosdbshell-linux-x64.tar.gz`
     - **Linux (ARM64)**: `cosmosdbshell-linux-arm64.tar.gz`

2. **Extract Archive**
   
   **Windows:**
   ```powershell
   Expand-Archive -Path cosmosdbshell-win-x64.zip -DestinationPath "C:\Program Files\CosmosDBShell"
   ```
   
   **macOS/Linux:**
   ```bash
   mkdir -p ~/cosmosdbshell
   tar -xzf cosmosdbshell-macos-x64.tar.gz -C ~/cosmosdbshell
   ```

3. **Add to PATH (Optional)**
   
   **Windows:**
   - Add `C:\Program Files\CosmosDBShell` to your system PATH
   - Or use the full path: `C:\Program Files\CosmosDBShell\cosmosdbshell.exe`
   
   **macOS/Linux:**
   ```bash
   export PATH="$HOME/cosmosdbshell:$PATH"
   # Add to ~/.bashrc or ~/.zshrc for persistence
   ```

4. **Run**
   ```bash
   cosmosdbshell
   ```

## Verification

### Test your installation

1. **For NuGet Installation:**
   ```bash
   cosmosdbshell --version
   ```

2. **For Binary Installation:**
   ```bash
   ./cosmosdbshell --version
   ```

3. **For VS Code Extension:**
   - Open the command palette (Ctrl+Shift+P / Cmd+Shift+P)
   - Type "Cosmos DB Shell"
   - You should see Cosmos DB Shell commands

## Troubleshooting

### "Command not found" (NuGet installation)

- Ensure .NET SDK 10.0+ is installed: `dotnet --version`
- Restart your terminal after installation
- Check that `$HOME/.dotnet/tools` is in your PATH

### "File not found" (binary installation)

- Verify the archive extracted successfully
- Check file permissions: `chmod +x cosmosdbshell` (macOS/Linux)
- Use the full path to the executable

### VS Code extension not appearing

- Ensure VS Code version is 1.85 or later
- Restart VS Code
- Check the Extensions marketplace for "Azure Cosmos DB Shell"

## Next steps

- [Quick Start Guide](get-started.md) - Get started with basic commands
- [Command Reference](command-reference.md) - Learn all available commands
- [Security Best Practices](security.md) - Secure your setup

## Support

If you encounter issues:
- [View Troubleshooting Guide](troubleshooting.md)
- [Check Security Considerations](security.md)

## See also

- [Azure Cosmos DB Shell Overview](overview.md)
- [Azure Cosmos DB Documentation](overview.md)
