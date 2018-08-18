## doc-templite

> doc-templite 主要转换文件

1. 拿到单个文件的内容, 看看有没有`doc-templite注释`,且将 单一注释模版的内容组合 添加到`数组`

> 期间可能会出现 `doc-templite注释不闭合` 的错误

2. 对`数组`中每个注释模版操作

2.1 逐个注释模版, 找 toml 注释(单行/多行)

- 其中 找toml的注释 是分 找单行 和 找多行 注释

2.2 切掉 前后注释符号, 交给 toml 解析成js对象

> 期间出现 `toml注释不闭合` 与 `toml解析` 的错误

2.3 从js对象中拿到 模版id (docTempliteId 默认:`yobrave), 并运行模版引擎

> 期间 会出现`id模版找不到` 错误

2.4 组合 生成的文档模版, 覆盖上一文件内容(原文/上一模版的覆盖)拿到全新的文件内容, 汇总一下数据, 继续下个模版

> 注意: 只有文件中所有模版都成功后, 才能说这个文件成功转换

3. `数组`循环结束, 组合数据 返回

供给`cli.js`

``` js
	let result = {
		path: opts.path, // 文件路径
		toml: blocksTomls, // 每个模版中的toml对象 :[{},{}]
		transformed: transformed, // 是否成功转换
		data: data // 新文件内容
	}
```

``` js
'use strict';

const toml = require('toml');
const os = require('os')
const dlv = require('dlv');
const templiteParse = require('templite')
const { oneOra,loggerText } = require('two-log')
const {c,g} = require('./src/util')

const updateTemplite = require('./src/updateTemplite')
const errorInfo = require('./src/errorInfo')
const {toS} = require('./src/util')

const START = '<!-- doc-templite START generated -->'
const END   = '<!-- doc-templite END generated -->'

function matchesStart(line) {
	return (/<!-- doc-templite START/).test(line);
  }

function matchesEnd(line) {
	return (/<!-- doc-templite END/).test(line);
}

function remarkStart(line) {
	return (/<!--/).test(line);
  }

function remarkEnd(line) {
	return (/-->/).test(line);
}

function isRemark(line){
	return line.startsWith('<!--') && line.endsWith('-->')
}

function removeTag(line){
	if(isRemark(line)){
		line = line.slice(4,-3)
	}
	return line
}

module.exports = function docTemplite(content, opts){
	if (typeof content !== 'string') {
		throw new TypeError(`Expected a string, got ${c(typeof content)}`);
	}
	let blocksTomls = [];
	let transformed = false;
	let data = content;
	let currentBlocks = []

	// must had doc-templite tag
	let lines = content.split(os.EOL)

	loggerText(c(`${opts.path} searching doc-templite <-tags->`))
	loggerText(g('all:'+lines.length))

//  ====》 1 ❤️ start
	let tags = updateTemplite.parse(lines, matchesStart, matchesEnd); 
	tags.forEach(function(tag){
		if(tag.hasStart && tag.hasEnd){
			currentBlocks.push(lines.slice(tag.startIdx + 1, tag.endIdx))
		}else if(tag.hasStart || tag.hasEnd){
			let E = errorInfo('tag',{tag,lines})
			throw new Error(`${c(opts.path)}\n - doc-templite tag not Closed:\n${E}`)
		}
    })
//  ====》 1 ❤️ end

//  ====》 2 ❤️ start
	// currentBlock : [[],[]] || []
	if (currentBlocks.length) {
		loggerText(c(`${opts.path} had <-tags->`))

		// Support md more tags with templite
		for(let i = 0; i < currentBlocks.length; i ++){
			let indexBlock = currentBlocks[i]
			loggerText(`<-tag-${i}->: ${toS(tags[i])}`)

			loggerText(g(`block-${i}:${toS(indexBlock)}`))
//  ====》 2.1 ❤️ start

			let singleRemark = indexBlock.filter(function(line){
				line = line.trim()
				return isRemark(line)
			})
			let removeSingle = indexBlock.filter(function(line){
				line = line.trim()
				return !isRemark(line)
			})

			let mulitRemarkPtn = updateTemplite.parse(removeSingle, remarkStart, remarkEnd)
			let currentRemarks = []

			mulitRemarkPtn.forEach(function(tag){
				if(tag.hasStart && tag.hasEnd){
					currentRemarks.push(removeSingle.slice(tag.startIdx,tag.endIdx+1))
				}else if(tag.hasStart || tag.hasEnd){
					let E = errorInfo('remark',{tag,lines:removeSingle})
					throw new Error(`${c(opts.path)} - Toml not Closed:\n'${E}`)
				}
			})

			let mulitRemark = []
			if(currentRemarks.length){

				for(let remarkIdx = 0; remarkIdx < currentRemarks.length; remarkIdx++){
					let indexRemark = currentRemarks[remarkIdx]
					mulitRemark.push(indexRemark.join(os.EOL))
					loggerText(`tomls-${remarkIdx}: ${toS(indexRemark)}`)
				}
			}


			let tomlRemark = singleRemark.concat(mulitRemark)
//  ====》 2.1 ❤️ end

//  ====》 2.2 ❤️ start

			let userTomls = tomlRemark.map(function(line){
				line = line.trim()
				let ready2Toml = removeTag(line)

				loggerText(c('toml:'+ready2Toml))
				let userToml = {};
				try{
					userToml = toml.parse(ready2Toml)
				}catch(e){
					let E = errorInfo('toml',{e,line:ready2Toml})
					throw new Error(`${c(opts.path)} TOML Parse error:\n${E}`);
				}
				return userToml
			})
			let mergeOpts = {}
			userTomls.forEach(function(singleTomlOpt){
				!!Object.keys(singleTomlOpt).length && (mergeOpts = Object.assign(mergeOpts, singleTomlOpt))
			})
			blocksTomls.push(mergeOpts)

			loggerText(c('toml -> object:\n'+toS(mergeOpts)))
//  ====》 2.2 ❤️ end

//  ====》 2.3 ❤️ start
			let ID = mergeOpts.docTempliteId || 'readme';
			let id2Templite = dlv(opts.templite, ID)
			loggerText(c(`templite <${ID}>:\n`+id2Templite))

			if(id2Templite){

				let templiteTransformed = templiteParse(id2Templite, mergeOpts)
				loggerText(c('Transformed:\n'+templiteTransformed))
//  ====》 2.4 ❤️ start
				let update = `${START}${os.EOL}${tomlRemark.join(os.EOL)}${os.EOL}${templiteTransformed}${os.EOL}${ END}`

				data = updateTemplite(data, update, matchesStart, matchesEnd, i)

				if (data) {
					// loggerText(c(toS(data)))
					if(i == currentBlocks.length - 1){
						transformed = true;
					}
                }
//  ====》 2.4 ❤️ end
                
			}else{
				throw new Error(`${c(opts.path)} no match doc-templite \nid:${g(mergeOpts.docTempliteId)} `)
            }
//  ====》 2.3 ❤️ end
            
		}
	}else{
		loggerText(c(`${opts.path} no <-tags->`))
    }
//  ====》 2 ❤️ end

//  ====》 3 ❤️
	let result = {
		path: opts.path,
		toml: blocksTomls,
		transformed: transformed,
		data: data
	}

	return result;
};

```