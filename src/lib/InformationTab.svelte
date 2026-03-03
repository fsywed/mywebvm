<h1 class="text-lg font-bold">系统信息</h1>
<img src="assets/webvm_hero.png" alt="WebVM 标志" class="w-56 h-56 object-contain self-center">
<p>WebVM 是一个通过 WebAssembly 在浏览器中运行的虚拟 Linux 环境</p>
<p>它基于以下核心技术构建：</p>
<ul class="list-disc list-inside">
	<li><a class="underline" target="_blank" href="https://cheerpx.io/">CheerpX</a>：基于 WebAssembly 的 x86 即时编译器</li>
	<li><a class="underline" target="_blank" href="https://xtermjs.org/">Xterm.js</a>：交互式终端组件</li>
	<li>本地/私有 <a class="underline" target="_blank" href="https://cheerpx.io/docs/guides/File-System-support">文件存储</a> 功能</li>
	<li>通过 <a class="underline" target="_blank" href="https://tailscale.com/">Tailscale</a> 实现的 <a class="underline" target="_blank" href="https://cheerpx.io/docs/guides/Networking">网络联网</a> 能力</li>
</ul>
<slot></slot>
