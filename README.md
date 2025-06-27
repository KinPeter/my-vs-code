# My VS Code

Extension for Visual Studio Code that contains my preferred themes, keybindings, and snippets.

## Credits for contents

The themes are based on the [Dracula IntelliJ Theme](https://github.com/kevinvn1709/vscode-dracula-color-theme) by Minh Tri Nguyen.

The keybindings are based on the [IntelliJ IDEA Keybindings](https://github.com/kasecato/vscode-intellij-idea-keybindings) by Keisuke Kato.

Thanks a lot to both authors for their work!

## Building and installing the extension

1. Install [Node.js](https://nodejs.org/en/download/) version >=22.5.1.

2. Install `vsce` which is the Visual Studio Code Extension Manager:

   ```bash
   npm ci
   ```

   Or in case you want to install it globally:

   ```bash
   npm install -g vsce
   ```

3. Build (package) the extension:

   ```bash
   npm run build
   ```

   or using the global installation of `vsce`:

   ```bash
   vsce package
   ```

   This will generate a new `.vsix` file in the project root directory, typically named `pk-stuff-<version>.vsix`.

4. Install the extension in Visual Studio Code in one of the following ways:

   - Open the Command Palette (Ctrl+Shift+P) and select "Extensions: Install from VSIX..." and choose the generated `.vsix` file.
   - Also in the Command Palette, you can use "Developer: Install Extension from location..." and select the folder containing the `.vsix` file.
   - Open the Extensions view (Ctrl+Shift+X), click on the ellipsis (...) in the top right corner, and select "Install from VSIX..." to choose the generated `.vsix` file.
   - Alternatively, you can drag and drop the `.vsix` file into the Extensions view in Visual Studio Code.
