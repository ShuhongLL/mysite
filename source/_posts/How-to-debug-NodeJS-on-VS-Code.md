---
title: How to debug NodeJS on VS Code
date: 2019-05-08 11:50:53
tags: [VS Code, NodeJS]
photos: ["../images/vscode.JPG"]
---
Here are the steps to start debug mode in VS Code:

1. On the left side bar, click "debug" icon to switch to debug viewlet

2. On the top left, click the gear icon

3. Then `launch.json` will be opened in the editor

4. Replace the content of the file to be:
<!-- more -->
```json
{
	"version": "0.2.0",
	"configurations": [
		{
			"type": "node",
			"request": "launch",
			"name": "Launch app.js",
			"program": "${workspaceRoot}/app.js",
			"stopOnEntry": true,
			"args": [
				"arg1", "arg2", "arg3"
			]
		}
	]
}
```

5. Replace the command line arguments to whatever you need

6. Start the debugger or press `F5`

You are all good to go!

If your program reads from **stdin**, please add a "console" attribute to the launch config:
```json
{
	"version": "0.2.0",
	"configurations": [
		{
			"type": "node",
			"request": "launch",
			"name": "Launch app.js",
			"program": "${workspaceRoot}/app.js",
			"stopOnEntry": true,
			"args": [
				"arg1", "arg2", "arg3"
			],
			"console": "integratedTerminal"
		}
	]
}
```

If you are running the program in the **terminal**, you can change the content alternatively to be:
```json
{
	"version": "0.2.0",
	"configurations": [
		{
			"type": "node",
			"request": "attach",
			"name": "Attach to app.js",
			"port": "5858"
		}
	]
}
```
The port is the **debug port** and it has nothing to do with your program (no matter it is a service or not). Then in the terminal, run:
```shell
node --debug-brk app.js arg1 arg2 arg3...
```
>The`--debug-brk` lets your program wait for the debugger to attach to. So there is no problem that it terminates before the debugger could attach.

Running such command, you may encounter a problem like this:
```
(node:31245) [DEP0062] DeprecationWarning: `node --inspect --debug-brk` is deprecated. Please use `node --inspect-brk` instead.
```
As discussed in [microsoft github offical repository](https://github.com/Microsoft/vscode/issues/32529), currently there is **no way** to prevent this happening. The reason why using `--inspect --debug-brk` is explained [here](https://github.com/microsoft/vscode/issues/27731):
>This combination of args is the only way to enter debug mode across all node versions. At some point I'll switch to inspect-brk if we don't want to support node 6.x anymore, or will do version detection for it and do something for runtimeExecutable scenarios.

>The problem is that we do not really know what version of node a user is using, so we cannot adapt the flags we use to the node version in order to minimize the resulting deprecation warnings.

