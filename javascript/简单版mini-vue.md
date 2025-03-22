### 初始化工程

```shell
yarn init -y 

# init后会生成如下文件
--node_modules
--package.json
--yarn.lock
```

安装babel

```shell
yarn add --dev babel-jest @babel/core @babel/preset-env

#支持ts 
yarn add --dev @babel/preset-typescript


```

添加babel.config.js

```js
module.exports = {
  presets: [
    ["@babel/preset-env", { targets: { node: "current" } }], //支持当前node环境下的配置
    "@babel/preset-typescript",
  ],
};

```

添加rollup

```sh
yarn add rollup --dev

yarn add @rollup/plugin-typescript --dev

yarn add tslib --dev

#创建rollup.config.js
import typescript from "@rollup/plugin-typescript"
export default {
	input: "./src/index.ts",
	output: [
			# 1.cjs->commonjs
			# 2.esm
			{
					format: "cjs",
					file: "lib/guide-mini-vue.cjs.jsÏ"
			},
			{
				format:"es",
				file: "lib/guide-mini-vue.esm.jsÏ"
			}
	],
	plugins:[
		typescript()
	]
}
```

package.json配置rollup

```js
{
	"scripts":{
		"test": "jest",
		"build": "rollup -c rollup.config.js"
	}
}
```

