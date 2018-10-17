# Webpack bin/webpack.js
看完这篇文章你应该可以回答以下问题：
1. 知道`/bin/webpack.js`文件都做了些什么

当我们从命令行中输入`webpack`时, 系统就会从webpack的`package.json`文件中寻找cli的入口文件。因为webpack的package.json中定义了`"bin": "./bin/webpack.js"`这一项，所以命令行输入后这个文件就会被执行。我们来看看这个文件的代码都做了些什么。

```javascript
// shebang 告诉系统用node执行脚本，一般cli脚本都要带这个
#!/usr/bin/env node

// 指定process的exitCode的默认值是0，也就是正常退出，关于更多process exitCode
// https://nodejs.org/api/process.html#process_exit_codes
process.exitCode = 0;

// runCommand是一个帮助方法，功能就是在child process里面运行一些shell命令，然后通过promise将脚本运行的结果返回
/**
 * @param {string} command process to run
 * @param {string[]} args commandline arguments
 * @returns {Promise<void>} promise
 */
const runCommand = (command, args) => {
	const cp = require("child_process");
	return new Promise((resolve, reject) => {
		const executedCommand = cp.spawn(command, args, {
			stdio: "inherit",
			shell: true
		});

		executedCommand.on("error", error => {
			reject(error);
		});

		executedCommand.on("exit", code => {
      // 根据child process的exitCode来判断子进程是否是正常退出
			if (code === 0) {
				resolve();
			} else {
				reject();
			}
		});
	});
};

// 这是个帮助函数，用来判断某个包有没有被安装
/**
 * @param {string} packageName name of the package
 * @returns {boolean} is the package installed?
 */
const isInstalled = packageName => {
	try {
    // require.resolve()方法会使用require()内部的包解析机制帮你寻找这个包名实际对应的
    // 文件，如果可以找到该包就返回文件的绝对路径，如果不可以就会抛出一个错误，这会被视为该包
    // 没有被安装
		require.resolve(packageName);

		return true;
	} catch (err) {
		return false;
	}
};


// 以下是webpack目前的两个CLI工具，一个是webpack-cli, 一个是webpack-command
// 其中webpack-cli是被推荐使用的cli工具，是under development的，webpack-command不推荐被使用，因为已经被achived了
/**
 * @typedef {Object} CliOption // 定义CliOption的schema
 * @property {string} name display name
 * @property {string} package npm package name
 * @property {string} binName name of the executable file
 * @property {string} alias shortcut for choice
 * @property {boolean} installed currently installed?
 * @property {boolean} recommended is recommended
 * @property {string} url homepage
 * @property {string} description description
 */

/** @type {CliOption[]} */
const CLIs = [
	{
		name: "webpack-cli",
		package: "webpack-cli",
		binName: "webpack-cli",
		alias: "cli",
		installed: isInstalled("webpack-cli"),
		recommended: true,
		url: "https://github.com/webpack/webpack-cli",
		description: "The original webpack full-featured CLI."
	},
	{
		name: "webpack-command",
		package: "webpack-command",
		binName: "webpack-command",
		alias: "command",
		installed: isInstalled("webpack-command"),
		recommended: false,
		url: "https://github.com/webpack-contrib/webpack-command",
		description: "A lightweight, opinionated webpack CLI."
	}
];

// 获取现在系统已经被安装的webpack的cli工具
const installedClis = CLIs.filter(cli => cli.installed);

/**
如果没有一个工具被安装，就会帮用户安装recommended的那一个，这里是webpack-cli
**/
if (installedClis.length === 0) {
	const path = require("path");
	const fs = require("fs");
	const readLine = require("readline");

	let notify =
		"One CLI for webpack must be installed. These are recommended choices, delivered as separate packages:";

	for (const item of CLIs) {
		if (item.recommended) {
			notify += `\n - ${item.name} (${item.url})\n   ${item.description}`;
		}
	}

	console.error(notify);

  // 判断用户当前使用的是哪个package manager, yarn还是npm
  // 判断package manager 是yarn的条件是，当前process的current working directory有没有yarn.lock这个文件，凡是用yarn安装过工具包的包的根目录都会有一个yarn.lock文件，用npm安装包的会有一个npm.lock文件
	const isYarn = fs.existsSync(path.resolve(process.cwd(), "yarn.lock"));

	const packageManager = isYarn ? "yarn" : "npm";
	const installOptions = [isYarn ? "add" : "install", "-D"];

	console.error(
		`We will use "${packageManager}" to install the CLI via "${packageManager} ${installOptions.join(
			" "
		)}".`
	);

	let question = `Do you want to install 'webpack-cli' (yes/no): `;

	const questionInterface = readLine.createInterface({
		input: process.stdin,
		output: process.stderr
	});
	questionInterface.question(question, answer => {
		questionInterface.close();

		const normalizedAnswer = answer.toLowerCase().startsWith("y");

		if (!normalizedAnswer) {
			console.error(
				"You need to install 'webpack-cli' to use webpack via CLI.\n" +
					"You can also install the CLI manually."
			);
			process.exitCode = 1;

			return;
		}

		const packageName = "webpack-cli";

		console.log(
			`Installing '${packageName}' (running '${packageManager} ${installOptions.join(
				" "
			)} ${packageName}')...`
		);

    // 使用系统命令行安装webpack-cli
		runCommand(packageManager, installOptions.concat(packageName))
			.then(() => {
        // 安装成功后, 加载webpack-cli，这会执行webpack-cli bin目录下的cli.js
				require(packageName); //eslint-disable-line
			})
			.catch(error => {
        // webpack-cli安装失败，设置process的exitCode是1，然后退出程序
				console.error(error);
				process.exitCode = 1;
			});
	});
} else if (installedClis.length === 1) {
  /**
    用户如果已经安装了任何一个cli工具，则加载该cli工具
  **/
	const path = require("path");
  // 找出该包的package.json文件
	const pkgPath = require.resolve(`${installedClis[0].package}/package.json`);
	// eslint-disable-next-line node/no-missing-require
  // 获取该cli包的package.json的内容
	const pkg = require(pkgPath);
	// eslint-disable-next-line node/no-missing-require
  // 因为cli包的入口文件可以从package.json拿到，就是'bin'字段的值
  // 直接require这个文件的效果等同于在命令行直接输入该命令
	require(path.resolve(
		path.dirname(pkgPath),
		pkg.bin[installedClis[0].binName]
	));
} else {
  /**
  如果用户安装了多于一个cli包时，webpack不会自己判断需要使用哪个cli包，而是告知用户并且退出进程
  **/
	console.warn(
		`You have installed ${installedClis
			.map(item => item.name)
			.join(
				" and "
			)} together. To work with the "webpack" command you need only one CLI package, please remove one of them or use them directly via their binary.`
	);
	process.exitCode = 1;
}
```

## 概述
上面详细解析了webpack bin/webpack.js文件的代码，其实这个文件的逻辑很简单，可以简单叙述为如下：

> 判断用户有没有安装webpack相关的cli工具，现在有webpack-cli和webpack-command这两个工具，其中webpack-command已经archived了，所以推荐使用webpack-cli。如果用户只安装了webpack-cli就执行webpack-cli的bin入口文件，这相当于直接在命令行输入webpack-cli。如果用户安装了超过一个webpack cli工具，webpack不做任何选择，退出进程。如果用户没有安装任何cli工具，webpack安装webpack-cli并且执行。


因为bin/webpack.js执行了webpack-cli/bin/cli.js，我们接着看一下这个文件都做了什么。
