# Extending Visual Studio Code

## Intro

[Visual Studio Code](https://code.visualstudio.com) is a lightweight, yet powerful text editor that runs on Windows, Mac, and Linux. 

VS Code supports custom colorization, snippets, commands, and more through a flexible extension API. 

In this talk you'll learn some basics about writing extensions for VS Code, how to publish those extensions into the [Visual Studio Marketplace](https://marketplace.visualstudio.com/VSCode), and get some resources and guidance on considerations for extension development. 

## Getting Started 

### Prerequisites

1. [Visual Studio Code](https://code.visualstudio.com)
2. [Node.js](https://nodejs.org/en/https://nodejs.org/) - VS Code depends on Node.js
3. [Yeoman](http://yeoman.io/) - The template engine used by the scaffolder
4. [Yeoman VS Code Generator](https://www.npmjs.com/package/generator-code) - scaffolds extension project structure
5. [Visual Studio Code Extension Manager](https://www.npmjs.com/package/vsce) (VSCE) - for publishing to the marketplace

## Demo Setup

This talk is about extension development, so the target audience for the talk is extension developers. 

The demo story will demonstrate how to build an extension to make it easy for other extension authors. In fact, the talk will be done using an extension built in order to streamline the demonstrations in the talk. 

> **Action Item**: Create a new extension using the Yeoman scaffolder named **target-extension** before continuing with the demos. Open it up whenever debugging the extension as it is built throughout the demos.

---

## Snippets

### Demo 1 - Create a logging snippet

When you're developing extensions you need to see what's happening in your development instance of VS Code. Let's make a snippet that makes it easy to insert the `console.log` call, since we'll use it a lot during extension debugging. 

1. Create a new empty JS extension named `meta-extension`
2. Open the `meta-extension` folder in VS Code
3. Add the file `snippets/console.json` to the workspace with this JSON

```json
{
    "Print to console": {
        "prefix": "log",
        "body": [
            "console.log('$1');"
        ],
        "description": "Log output to the VS Code console window."
    }
}
```

4. Add this JSON to the `contributes` section of the `package.json` file to load the snippet when the extension is loaded

```json
"contributes": {
    "snippets": [
        {
            "language": "javascript",
            "path": "./snippets/console.json"
        }
    ]
}
```

5. Debug the extension
6. Open the **target-extension** target project in the debugging instance
7. Use the two snippets to extend the **target-extension** project
8. Debug the **target-extension** to show the message appearing in the development instance

--- 

## Commands

### Demo 2 - Commands

The command palette gives customers a convenient way to execute commands contributed by extensions. 

1. Note the `contributes` section and how there is a single command named `sayHello`

```json
"commands": [
        {
            "command": "extension.sayHello",
            "title": "Hello World"
        }
    ],
```

2. Note the handler for the `sayHello` command in `extension.js`

```javascript
var disposable = vscode.commands.registerCommand('extension.sayHello', function () {
    // The code you place here will be executed every time your command is executed

    // Display a message box to the user
    vscode.window.showInformationMessage('Hello World!');
});
```

3. Put a breakpoint on the handler and run the extension
4. Use `Ctrl-Shift-P` or `Cmd-Shift-P` to open the command palette
5. Type **Hello World** to see the command in the palette
6. Select it and note execution stops on the breakpoint

---

### Demo 3 - Add a new command 

Commands are useful for componentizing and isolating parts of the functionality your extension offers so that it can be called via the **Command Palette** or when users perform certain actions, like clicking on a button in the VS Code toolbar. 

1. Note the `activationEvents` section in `package.json`

```json
"activationEvents": [
    "onCommand:extension.sayHello"
]
```

2. With this setting the extension won't activate **until** the `sayHello` command is called, so it needs to be changed so that the extension activates right away. Optionally, we could just add the various commands that could activate the extension. 

```json
"activationEvents": [
    "*"
],
```

3. Add a new command named `meta.slowProcess` to the `package.json` file

```json
{
    "command": "meta.slowProcess",
    "title": "Run Slow Process"
}
```

4. Create a new directory named `commands` in the project and add a file to it named `slowProcess.js`

    > Rather than register all the command handlers manually in the `extension.js` file, the command handler logic can be separated into multiple files. 

    Then add the following code to the file to export a method to wire up the command handler to the VS Code context. 

```javascript
var vscode = require('vscode');

module.exports = exports = function (context) {
    var disposable = vscode.commands.registerCommand('meta.slowProcess', function () {
        vscode.window.showInformationMessage('Starting the slow process...');
    });

    context.subscriptions.push(disposable);
}
```

5. In the `extension.js` file, wire up the command handler after the `sayHello` command has been registered. 

```javascript
// wire up the other commands
slowProcess(context);
```

6. Debug the extension and use `Ctrl-Shift-P` or `Cmd-Shift-P` to run command titled `Run Slow Process`

---

## User Experience

### Demo 4 - Use the output window to give the customer information

Customers who use your extensions need to know what's happening so they typically look at the output window. 

1. Replace the code in the `commands/slowProcess.js` file with this code

```javascript
var vscode = require('vscode');

const outputChannelName = 'META';

module.exports = exports = function (context) {
    var disposable = vscode.commands.registerCommand('meta.slowProcess', function () {
        var outputChannel = vscode.window.createOutputChannel(outputChannelName);
        outputChannel.show(false);
        outputChannel.appendLine('Starting the slow process...');

        setTimeout(() => {
            outputChannel.appendLine('Process complete');
        }, 3000);
    });

    context.subscriptions.push(disposable);
}
```

2. Debug the extension and run the `slowProcess` command a few times. Once with the output window entirely closed, once with a different channel selected. Note how the `outputChannel.show()` method could be used to make for a less-intrusive experience when providing the user feedback. 

--- 

### Demo 5 - Adding buttons to the toolbar

Once commands are working in an extension a good way to trigger them is with a button in the toolbar. 

1. Show the list of icons in the [octicon](https://octicons.github.com/). Point out this icon library ships with VS Code and the icon names are used to specify which button icon should be shown.

2. Create a new file `utils\toolbar.js` in the workspace and add this code to it. This will componentize the toolbar so it can be used throughout the extension's codebase. 

```javascript
var vscode = require('vscode');

module.exports = {
    addButton: function (command, text, tooltip) {
        var customStatusBarItem = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Left, 0);
        customStatusBarItem.color = 'white';
        customStatusBarItem.command = command;
        customStatusBarItem.text = text;
        customStatusBarItem.tooltip = tooltip;
        customStatusBarItem.show();
    }
}
```

3. A command should be able to add itself to the toolbar rather than be centralized, so the `toolbar` will be referenced from the `slowProcess.js`. 

```javascript
var toolbar = require('../utils/toolbar.js');
```

4. The command name needs to be passed to the `addButton` method, so it can be extracted to a variable. This updated code for `slowProcess.js` is below.

```javascript
module.exports = exports = function (context) {
    var commandName = 'meta.slowProcess';
    var disposable = vscode.commands.registerCommand(commandName, function () {
        var outputChannel = vscode.window.createOutputChannel(outputChannelName);
        outputChannel.show(false);
        outputChannel.appendLine('Starting the slow process...');

        setTimeout(() => {
            outputChannel.appendLine('Process complete');
        }, 3000);
    });

    context.subscriptions.push(disposable);
    
    toolbar.addButton(commandName, '$(bug)', 'Submit a bug');
}
```

5. Debug the extension and show how the button is present and how clicking it causes the `slowProcess` command to fire. 

---

### Demo 6 - Giving the user options

Intrinsic validation in the form of drop-down lists is a great way to enable users to interface with your extension. 

1. Change the code in the `slowProcess` so that the user needs to confirm the command's execution. 

```javascript
var vscode = require('vscode');
var toolbar = require('../utils/toolbar.js');

const outputChannelName = 'META';
const startConfirmYes = 'Yes, start the process';
const startConfirmNo = 'Actually, nevermind';

module.exports = exports = function (context) {
    var commandName = 'meta.slowProcess';
    var disposable = vscode.commands.registerCommand(commandName, function () {

        vscode.window
            .showQuickPick([startConfirmYes, startConfirmNo])
            .then((selected) => {
                if (selected == startConfirmYes) {
                    var outputChannel = vscode.window.createOutputChannel(outputChannelName);
                    outputChannel.show(false);
                    outputChannel.appendLine('Starting the slow process...');

                    setTimeout(() => {
                        outputChannel.appendLine('Process complete');
                    }, 3000);
                }
            });
    });

    context.subscriptions.push(disposable);

    toolbar.addButton(commandName, '$(bug)', 'Submit a bug');
}
```

2. The `showQuickPick` method is called, by passing an array of strings as a parameter. 
3. Once the user makes a selection (or hits ESC), the method passed to `then` is called. 

### Collection, confirmation, and error dialogs

Data can be collected from the user, and dialogs to inform the user of events or of error states can be shown. 

1. Replace the code from `slowProcess.js` to match the code below. 

```javascript
var vscode = require('vscode');
var toolbar = require('../utils/toolbar.js');

const outputChannelName = 'META';
const startConfirmYes = 'Yes, start the process';
const startConfirmNo = 'Actually, nevermind';
const password = 'secret';

module.exports = exports = function (context) {
    var commandName = 'meta.slowProcess';
    var disposable = vscode.commands.registerCommand(commandName, function () {

        vscode.window
            .showInputBox({
                password: true,
                prompt: 'Enter the password'
            })
            .then((value) => {
                if (value == password) {
                    vscode.window
                        .showQuickPick([startConfirmYes, startConfirmNo])
                        .then((selected) => {
                            if (selected == startConfirmYes) {
                                var outputChannel = vscode.window.createOutputChannel(outputChannelName);
                                outputChannel.show(false);
                                outputChannel.appendLine('Starting the slow process...');

                                setTimeout(() => {
                                    vscode.window.showInformationMessage('Process complete!')
                                }, 3000);
                            }
                        });
                }
                else {
                    vscode.window.showErrorMessage('Incorrect password');
                }
            });
    });

    context.subscriptions.push(disposable);

    toolbar.addButton(commandName, '$(bug)', 'Submit a bug');
}
```