## cli.js

> doc-templite 命令行文件

1. `twoLog(cli.flags['D'])` 根据命令行的是否Debug, 切换 [ora] / [winston]

[ora]: https://github.com/sindresorhus/ora
[winston]: https://github.com/winstonjs/winston

2. 确认用户的输入和模版文件`.doc-templite.js`, 否则 帮助/错误

3. 根据输入文件/目录找md文件, 汇总组合成数组

4. 逐个{读取文件内容 运行 `doc-templite.js` 转换, 拿到结果},拿到结果数组, 期间发生错误, 打印错误退出

5. 根据`--OR`命令选项与结果中成功转换的文件, 是否写入覆盖文件


``` js
#!/usr/bin/env node
'use strict';
const meow = require('meow');
const fs = require('fs')
const { twoLog } = require('two-log')
const {g,c,y,b,m,r} = require('./src/util')

const file = require('./src/file')
const docTemplite = require('./doc-templite');
let files;

const cli = meow(`
  Usage
  	$ doc-templite [folder/file name] [Optioins]

	Example
		$ doc-templite readme.md

	${b(`⭐ [Options]`)}
		${g(`-D debug`)} <default:false>

	${m(`⭐ [High Options]`)}
		${g(`--OR`)} only Read, no reWrite files <default:false>
`);

// log
const log = twoLog(cli.flags['D'])      //  ====》 1 ❤️

// ====》 2 ❤️ START
// need path name     
if(!cli.input[0]){
	return console.log(y("--> v"+cli.pkg.version),cli.help)
}
// .doc-templite.js
let templite = {};
try {
	const path = require('path')
	let templitePath = path.join(process.cwd(),'.doc-templite.js')
	templite = require(templitePath)
} catch (error) {
	return console.error(y(`.doc-templite.js IS NOT EXITE\n${error}`))
}
// onlyread
const onlyRead = !!cli.flags['or']
//  ====》 2 ❤️ END

function cleanPath(path) {
	var homeExpanded = (path.indexOf('~') === 0) ? process.env.HOME + path.substr(1) : path;
	//  所有 空格 去除
	return homeExpanded.replace(/\s/g, '\\ ');
}
function transformAndSave(files, opts){
	log.text('2. ready to transform')
//  ====》 4 ❤️
	let transformeds = files.map(function(x){
		let content = fs.readFileSync(x.path, 'utf8');
		opts.path = x.path
		let result = docTemplite(content, opts)

		// result.path = x.path
		return result
	})

	let changeds = transformeds.filter(function (x) { return x.transformed; })
    let unchangeds = transformeds.filter(function (x) { return !x.transformed; })

	unchangeds.forEach(function (x) {
		log.text(`"${x.path}" no transform`);
	  });
//  ====》 5 ❤️
	changeds.forEach(function (x) {
		if (onlyRead) {
		  log.one(`===> ${c(x.path)} need to update`)
		} else {
		  log.one(`${c(x.path)} updated`);
		  fs.writeFileSync(x.path, x.data, 'utf8');
		}
	});
}

log.start(`starting doc-templite`)
for (let i = 0; i < cli.input.length; i++){
	let target = cleanPath(cli.input[i]);
	let stat = fs.statSync(target);

    log.text(`1. files getting...`)
//  ====》 3 ❤️
	if (stat.isDirectory()){
		files = file.findMarkdownFiles(target)
	}else{
		files = [{ path: target}]
	}
	let opts = cli.flags
	opts.templite = templite

	try{
		transformAndSave(files, opts)
	}catch(e){
		console.log(r('\n'+e.stack))
		process.exitCode = 1
	}
	// log.text(`search markdown file ... ${JSON.stringify(files,null,2)}`)
}

log.stop(`doc-templite done [${onlyRead?"onlyRead":"Write"} mode]`,{ora:'succeed'})
```