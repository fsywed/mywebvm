<script>
	import PanelButton from './PanelButton.svelte';
	import { cpuPercentage } from './activities.js'
</script>

<h1 class="text-lg font-bold">引擎</h1>
<PanelButton buttonImage="assets/cheerpx.svg" clickUrl="https://cheerpx.io/docs" buttonText="探索 CheerpX">
</PanelButton>
<p><span class="font-bold">虚拟 CPU：</span>{$cpuPercentage}%</p>
<p>CheerpX 是一个基于 WebAssembly 的 x86 虚拟化引擎</p>
<p>它可以在浏览器中安全运行未经修改的 x86 二进制文件和库</p>
<p>对我们的技术感兴趣？今天就使用 <a class="underline" href="https://cheerpx.io/" target="_blank">CheerpX</a> <a class="underline" href="https://cheerpx.io/docs/getting-started" target="_blank">开始构建</a> 你的项目吧！</p>
