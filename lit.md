```js
#!/usr/bin/env node

'use strict'

```
 # lit
 This is the actual `lit` executable which is being declared as `bin` in the
 `package.json` file. Whenever a user runs `lit`, this is the file that is
 being run.

 ## module dependencies
 We're using [`commander`](https://www.npmjs.com/package/commander) for
 parsing command line arguments and generating the `--help` output. We tried
 out [`vorpal`](https://www.npmjs.com/package/vorpal) as well, but it
 significantly increases startup time, which we obviously want to avoid.
```js
const program  = require('commander')
```
 The [`package.json`](./package.json) is our source of truth.
```js
const version  = require('./package.json').version
const fs       = require('fs')
const path     = require('path')
const split    = require('split')
const through2 = require('through2')

```
 ## program
 This is where we declare our `lit` command. Currently there are no
 sub-commands, but this might change in the future as `lit` grows.
```js
program
  .version(version)
  .option('-o, --output', 'Output directory')
  .option('-p, --prefix', 'Comment prefix')
  .option('-h', '--help', 'Print usage info')
  .parse(process.argv)

const cwd = process.cwd()

const files = program.args.map(file => path.join(cwd, file))

const isDoc = /^\s*\/\//
const isBlank = /^\s*$/

const Transform = through2.ctor(function (chunk, enc, cb) {
	const {extname} = this.options
	const prevIsCodeBlock = this.isCodeBlock

```
 ignore empty lines
```js
	if (isBlank.test(chunk)) {
		this.push(chunk)
		this.push('\n')
		cb()
		return
	}

	this.isCodeBlock = !isDoc.test(chunk)

	if (this.isCodeBlock && !prevIsCodeBlock) {
		this.push(`\`\`\`${extname}\n`)
	}

	if (!this.isCodeBlock && prevIsCodeBlock) {
		this.push('```\n')
	}

	if (this.isCodeBlock) {
		this.push(chunk)
	} else {
		this.push(String(chunk).replace(isDoc, ''))
	}

	this.push('\n')
	cb()
}, function (cb) {
	if (this.isCodeBlock) {
		this.push('```\n')
	}
	cb()
})

const doc = file =>
	fs.createReadStream(file)
		.pipe(split())
		.pipe(new Transform({
			extname: path.extname(file).substr(1) || 'js'
		}))
		.pipe(fs.createWriteStream(`${file}.md`))

function handleEnd () {
	const toPath = path.relative(cwd, this.path)
	console.log('wrote', this.bytesWritten, 'bytes to', toPath)
}

files.map(doc).forEach(stream => stream.on('close', handleEnd))

```