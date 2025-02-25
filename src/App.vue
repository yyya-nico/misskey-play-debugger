<template>
<div id="root">
	<h1>AiScript (v0.12.4) Playground</h1>
	<div id="grid1">
		<div id="editor" class="container">
			<header>Input<div class="actions"><button @click="setCode">FizzBuzz</button></div></header>
			<div>
				<PrismEditor class="code" v-model="script" :highlight="highlighter" :line-numbers="lineNumbers"/>
			</div>
			<footer>
				<span v-if="isSyntaxError" class="syntaxError">Syntax Error!</span>
				<div class="actions"><button @click="lint">Lint</button></div>
				<div class="actions-right"><button @click="run">RUN</button></div>
			</footer>
		</div>
		<div id="logs" class="container">
			<header>Output</header>
			<div>
				<div v-for="log in logs" class="log" :key="log.id" :class="[{ print: log.print }, log.type]"><span class="type">{{ log.type }}</span> {{ log.text }}</div>
			</div>
		</div>
	</div>
	<div id="grid2">
		<div id="ast" class="container">
			<header>AST</header>
			<div>
				<pre>{{ JSON.stringify(ast, null, '\t') }}</pre>
			</div>
		</div>
		<div id="rootUi" class="container">
			<header>UI</header>
			<div>
				<MkAsUi v-if="rootUi" :component="rootUi" :components="componentsUi" size="medium" align="center"/>
			</div>
		</div>
		<div id="mockAPI" class="container">
			<header>API Mock<div class="actions"><button @click="setMockAPI">Example</button></div></header>
			<div>
				<PrismEditor class="code" v-model="mockAPI" :highlight="highlighter" :line-numbers="false"/>
			</div>
			<footer>
				<span v-if="isMockAPISyntaxError" class="syntaxError">Syntax Error!</span>
			</footer>
		</div>
	</div>
</div>
</template>

<script lang="ts" setup>
import { Ref, ref, watch } from 'vue';
import { Interpreter, Parser, utils, serialize } from '@syuilo/aiscript';
import { AsUiComponent, AsUiRoot, patch, registerAsUiLib, render } from './misskey/scripts/aiscript/ui';
import { createAiScriptEnv } from './misskey/scripts/aiscript/api';
import MkAsUi from './misskey/MkAsUi.vue';
import { setupMisskey } from './setup';
import { lintAst } from './linter';

import { PrismEditor } from 'vue-prism-editor';
import 'vue-prism-editor/dist/prismeditor.min.css';
import 'prismjs';
import { highlight, languages } from 'prismjs/components/prism-core';
import 'prismjs/components/prism-clike';
import 'prismjs/components/prism-javascript';
import 'prismjs/themes/prism-okaidia.css';
import { each } from 'lodash';

setupMisskey();

const exampleMockAPI = `{
	"path/to/api": ["response"],
	"path/to/api-per-param": {
		"paramA": "responseA",
		"paramB": ["responseB"]
	},
	"notes/local-timeline": [],
	"emojis": {}
}`;

const script = ref(window.localStorage.getItem('script') || '<: "Hello, AiScript!"');
const mockAPI = ref(window.localStorage.getItem('mockAPI') || exampleMockAPI);

const ast = ref(null);
const logs = ref([]);
const isSyntaxError = ref(false);
const isMockAPISyntaxError = ref(false);

const rootUi = ref<AsUiRoot>();
const componentsUi: Ref<AsUiComponent>[] = ref([]);

const lineNumbers = true;

watch(script, () => {
	window.localStorage.setItem('script', script.value);
	try {
		ast.value = Parser.parse(script.value);
		isSyntaxError.value = false;
	} catch (e) {
		isSyntaxError.value = true;
		console.error(e);
		return;
	}
}, {
	immediate: true
});

watch(mockAPI, () => {
	window.localStorage.setItem('mockAPI', mockAPI.value);
	try {
		JSON.parse(mockAPI.value);
		isMockAPISyntaxError.value = false;
	} catch (e) {
		isMockAPISyntaxError.value = true;
		console.error(e);
		return;
	}
}, {
	immediate: true
});

const setCode = () => {
	script.value = `for (let i, 100) {
  <: if (i % 15 == 0) "FizzBuzz"
    elif (i % 3 == 0) "Fizz"
    elif (i % 5 == 0) "Buzz"
    else i
}`;
};

const setMockAPI = () => {
	mockAPI.value = exampleMockAPI;
};

const run = async () => {
	logs.value = [];

	const interpreter = new Interpreter({
		...createAiScriptEnv({
			storageKey: 'flash:playground',
			mockAPI: mockAPI,
		}),
		...registerAsUiLib(componentsUi.value, (_root) => {
			rootUi.value = _root.value;
		}),
	}, {
		in: (q) => {
			return new Promise(ok => {
				const res = window.prompt(q);
				ok(res);
			});
		},
		out: (value) => {
			logs.value.push({
				id: Math.random(),
				type: value.type,
				text: value.type === 'str' || value.type === 'num' ? value.value : utils.valToString(value),
				print: true
			});
		},
		log: (type, params) => {
			switch (type) {
				case 'end': logs.value.push({
					id: Math.random(),
					text: utils.valToString(params.val, true),
					print: false
				}); break;
				default: break;
			}
		}
	});

	try {
		await interpreter.exec(ast.value);
	} catch (e) {
		console.error(e);
		window.alert('Error: ' + e);
	}
}

const pushTextLog = text => {
	logs.value.push({
		id: Math.random(),
		type: 'str',
		text: text,
		print: true
	});
}

const lint = () => {
	pushTextLog('=== start lint ===');
	const result = lintAst(script.value, ast.value);
	each(result, pushTextLog);
	pushTextLog('=== end lint ===');
}

const highlighter = code => {
	return highlight(code, languages.js, 'javascript');
};
</script>

<style>
:root {
	--borderThickness: 1px;
}

* {
	font-family: Fira code, Fira Mono, Consolas, Menlo, Courier, monospace;
}

html {
	background: #171717;
	color: #fff;
	tab-size: 2;
}

body {
	margin: 0;
	padding: 0;
}

pre {
	margin: 0;
}

#root {
    display: grid;
    height: 100vh;
    grid-template-columns: 3fr 2fr;
    padding: 16px;
    gap: 16px;
    box-sizing: border-box;
}
#root > h1 {
	font-size: 1.5em;
    margin: 0;
    grid-column: 1 / -1;
}

#grid1, #grid2 {
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    gap: 16px;
    min-height: 0;
}

#grid1 > * {
	min-height: 0;
}
#grid2 {
}
#grid2 > * {
	min-height: 0;
}

#editor {
    flex-basis: 80%;
}
#editor > .code {
	box-sizing: border-box;
	padding: 16px;
}
#editor .syntaxError {
	color: #f00;
}

#logs {
    flex-basis: 20%;
}
#logs .log .type {
	opacity: 0.5;
	color: #fff;
}
#logs .log:not(.print) {
	opacity: 0.7;
}
#logs .log.num {
	color: #0ff;
}
#logs .log.str {
	color: #ff0;
}

#ast, #rootUi, #mockAPI {
    flex-basis: 100%;
}

#ast {

}

#mockAPI {
}
#mockAPI > .code {
	box-sizing: border-box;
	padding: 16px;
}
#mockAPI .syntaxError {
	color: #f00;
}

.container {
	display: flex;
	flex-direction: column;
	border: solid var(--borderThickness) #555;
	border-radius: 8px;
	background: #202020;
}
.container > header {
	display: flex;
	padding: 8px 16px;
	border-bottom: dashed var(--borderThickness) #555;
	font-weight: bold;
}
.container > header > .actions {
	margin-left: auto;
}
.container > div {
	flex: 1;
	overflow: auto;
	padding: 16px;
}
.container > footer {
	display: flex;
	padding: 8px 16px;
	border-top: dashed var(--borderThickness) #555;
}
.container > footer > .actions {
	margin-left: auto;
}
.container > footer > .actions-right {
	margin-left: 16px;
}
</style>
