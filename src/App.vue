<script setup lang="ts">
import { onMounted, ref, watch } from 'vue'
import { WebContainer } from '@webcontainer/api'
import { Terminal } from '@xterm/xterm'
import { files } from './files'

const iframeSrc = ref('./src/loading.html')
const textareaValue = ref(files['index.js'].file.contents)
let webcontainerInstance: WebContainer
const terminalEl = ref<HTMLDivElement >()
onMounted(async () => {
  webcontainerInstance = await WebContainer.boot()
  await webcontainerInstance.mount(files)
  // await readFile()
  const terminal = new Terminal({
    convertEol: true,
  })
  if (terminalEl.value)
    terminal.open(terminalEl.value)
  const exitCode = await installDependencies(terminal)
  if (exitCode !== 0) {
    throw new Error('Installation failed')
  };

  await startDevServer(terminal)
})

// async function readFile() {
//   const packageJSON = await webcontainerInstance.fs.readFile('package.json', 'utf-8')
// }

async function installDependencies(terminal: Terminal) {
  const installProcess = await webcontainerInstance.spawn('npm', ['install'])
  installProcess?.output.pipeTo(new WritableStream({
    write(data) {
      terminal.write(data)
    },
  }))
  return installProcess?.exit
}

async function startDevServer(terminal: Terminal) {
  await webcontainerInstance.spawn('npm', ['run', 'start'])

  webcontainerInstance.on('server-ready', (port, url) => {
    terminal.writeln(`Server ready at ${url} ，port: ${port}`)
    iframeSrc.value = url
  })
}
async function writeIndexJS(content: string) {
  await webcontainerInstance.fs.writeFile('/index.js', content)
};
watch(textareaValue, (value) => {
  writeIndexJS(value)
})
</script>

<template>
  <div wh-screen flex-col p-3>
    <div flex flex-1 gap-3>
      <textarea v-model="textareaValue" flex-1 resize-none border-1 p-2 />
      <iframe :src="iframeSrc" flex-1 border-1 />
    </div>
    <div ref="terminalEl" mt-3 h-428px border-1 />
  </div>
</template>

<style>
.terminal.xterm {
  padding: 10px;
}
</style>
