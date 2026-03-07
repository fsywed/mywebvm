<script>
	import { onMount, tick } from 'svelte';
	import { get } from 'svelte/store';
	import Nav from 'labs/packages/global-navbar/src/Nav.svelte';
	import SideBar from '$lib/SideBar.svelte';
	import '$lib/global.css';
	import '@xterm/xterm/css/xterm.css'
	import '@fortawesome/fontawesome-free/css/all.min.css'
	import { networkInterface, startLogin } from '$lib/network.js'
	// 引入新增的downloadProgress进度store
	import { cpuActivity, diskActivity, cpuPercentage, diskLatency, downloadProgress } from '$lib/activities.js'
	import { introMessage, errorMessage, unexpectedErrorMessage } from '$lib/messages.js'
	import { displayConfig, handleToolImpl } from '$lib/anthropic.js'
	import { tryPlausible } from '$lib/plausible.js'
	export let configObj = null;
	export let processCallback = null;
	export let cacheId = null;
	export let cpuActivityEvents = [];
	export let diskLatencies = [];
	export let activityEventsInterval = 0;
	var term = null;
	var cx = null;
	var fitAddon = null;
	var cxReadFunc = null;
	var blockCache = null;
	var processCount = 0;
	var curVT = 0;
	var sideBarPinned = false;
    // ===== 下载块数统计 =====
let downloadedBlocks = 0;      // 已下载的块数
let isDownloading = false;     // 是否正在下载
let lastBlockTime = 0;         // 上一个块的时间戳
	function writeData(buf, vt)
	{
		if(vt != 1)
			return;
		term.write(new Uint8Array(buf));
	}
	function readData(str)
	{
		if(cxReadFunc == null)
			return;
		for(var i=0;i<str.length;i++)
			cxReadFunc(str.charCodeAt(i));
	}
	function printMessage(msg)
	{
		for(var i=0;i<msg.length;i++)
			term.write(msg[i] + "\n");
	}
	function expireEvents(list, curTime, limitTime)
	{
		while(list.length > 1)
		{
			if(list[1].t < limitTime)
			{
				list.shift();
			}
			else
			{
				break;
			}
		}
	}
	function cleanupEvents()
	{
		var curTime = Date.now();
		var limitTime = curTime - 10000;
		expireEvents(cpuActivityEvents, curTime, limitTime);
		computeCpuActivity(curTime, limitTime);
		if(cpuActivityEvents.length == 0)
		{
			clearInterval(activityEventsInterval);
			activityEventsInterval = 0;
		}
	}
	function computeCpuActivity(curTime, limitTime)
	{
		var totalActiveTime = 0;
		var lastActiveTime = limitTime;
		var lastWasActive = false;
		for(var i=0;i<cpuActivityEvents.length;i++)
		{
			var e = cpuActivityEvents[i];
			var eTime = e.t;
			if(eTime < limitTime)
				eTime = limitTime;
			if(e.state == "ready")
			{
				totalActiveTime += (eTime - lastActiveTime);
				lastWasActive = false;
			}
			else
			{
				lastActiveTime = eTime;
				lastWasActive = true;
			}
		}
		if(lastWasActive)
		{
			totalActiveTime += (curTime - lastActiveTime);
		}
		cpuPercentage.set(Math.ceil((totalActiveTime / 10000) * 100));
	}
	function hddCallback(state)
{
    diskActivity.set(state != "ready");
    
    // 当变成空闲状态时，认为下载完成
    if (state == "ready" && isDownloading) {
        isDownloading = false;
        term.write(`\n✅ 下载完成！总块数: ${downloadedBlocks}\n`);
        downloadProgress.set(-1);
    }
}
function latencyCallback(latency)
{
    // 原有逻辑（保留）
    diskLatencies.push(latency);
    if(diskLatencies.length > 30)
        diskLatencies.shift();
    var total = 0;
    for(var i=0;i<diskLatencies.length;i++)
        total += diskLatencies[i];
    var avg = total / diskLatencies.length;
    diskLatency.set(Math.ceil(avg));
    
    // ===== 简化版：只要触发就算一个块 =====
    if (!isDownloading) {
        isDownloading = true;
        downloadedBlocks = 0;
    }
    
    downloadedBlocks++;
    downloadProgress.set(downloadedBlocks);
    term.write(`\r📦 块 ${downloadedBlocks} (延迟: ${latency}ms)`);
}
	function cpuCallback(state)
	{
		cpuActivity.set(state != "ready");
		var curTime = Date.now();
		var limitTime = curTime - 10000;
		expireEvents(cpuActivityEvents, curTime, limitTime);
		cpuActivityEvents.push({t: curTime, state: state});
		computeCpuActivity(curTime, limitTime);
		if(activityEventsInterval != 0)
			clearInterval(activityEventsInterval);
		activityEventsInterval = setInterval(cleanupEvents, 2000);
	}
	function computeXTermFontSize()
	{
		return parseInt(getComputedStyle(document.body).fontSize);
	}
	function setScreenSize(display)
	{
		var internalMult = 1.0;
		var displayWidth = display.offsetWidth;
		var displayHeight = display.offsetHeight;
		var minWidth = 1024;
		var minHeight = 768;
		if(displayWidth < minWidth)
			internalMult = minWidth / displayWidth;
		if(displayHeight < minHeight)
			internalMult = Math.max(internalMult, minHeight / displayHeight);
		var internalWidth = Math.floor(displayWidth * internalMult);
		var internalHeight = Math.floor(displayHeight * internalMult);
		cx.setKmsCanvas(display, internalWidth, internalHeight);
		var screenshotMult = 1.0;
		var maxWidth = 1024;
		var maxHeight = 768;
		if(internalWidth > maxWidth)
			screenshotMult = maxWidth / internalWidth;
		if(internalHeight > maxHeight)
			screenshotMult = Math.min(screenshotMult, maxHeight / internalHeight);
		var screenshotWidth = Math.floor(internalWidth * screenshotMult);
		var screenshotHeight = Math.floor(internalHeight * screenshotMult);
		displayConfig.set({width: screenshotWidth, height: screenshotHeight, mouseMult: internalMult * screenshotMult});
	}
	var curInnerWidth = 0;
	var curInnerHeight = 0;
	function handleResize()
	{
		if(curInnerWidth == window.innerWidth && curInnerHeight == window.innerHeight)
			return;
		curInnerWidth = window.innerWidth;
		curInnerHeight = window.innerHeight;
		triggerResize();
	}
	function triggerResize()
	{
		term.options.fontSize = computeXTermFontSize();
		fitAddon.fit();
		const display = document.getElementById("display");
		if(display)
			setScreenSize(display);
	}
	async function initTerminal()
	{
		const { Terminal } = await import('@xterm/xterm');
		const { FitAddon } = await import('@xterm/addon-fit');
		const { WebLinksAddon } = await import('@xterm/addon-web-links');
		term = new Terminal({cursorBlink:true, convertEol:true, fontFamily:"monospace", fontWeight: 400, fontWeightBold: 700, fontSize: computeXTermFontSize()});
		fitAddon = new FitAddon();
		term.loadAddon(fitAddon);
		var linkAddon = new WebLinksAddon();
		term.loadAddon(linkAddon);
		const consoleDiv = document.getElementById("console");
		term.open(consoleDiv);
		term.scrollToTop();
		fitAddon.fit();
		window.addEventListener("resize", handleResize);
		term.focus();
		term.onData(readData);
		function preventDefaults (e) {
			e.preventDefault()
			e.stopPropagation()
		}
		consoleDiv.addEventListener("dragover", preventDefaults, false);
		consoleDiv.addEventListener("dragenter", preventDefaults, false);
		consoleDiv.addEventListener("dragleave", preventDefaults, false);
		consoleDiv.addEventListener("drop", preventDefaults, false);
		curInnerWidth = window.innerWidth;
		curInnerHeight = window.innerHeight;
		if(configObj.printIntro)
			printMessage(introMessage);
		try
		{
			await initCheerpX();
		}
		catch(e)
		{
			downloadProgress.set(-1); // 初始化失败重置进度
			printMessage(unexpectedErrorMessage);
			printMessage([e.toString()]);
			return;
		}
	}
	function handleActivateConsole(vt)
	{
		if(curVT == vt)
			return;
		curVT = vt;
		if(vt != 7)
			return;
		const display = document.getElementById("display");
		display.parentElement.style.zIndex = 5;
		tryPlausible("Display activated");
	}
	function handleProcessCreated()
	{
		processCount++;
		if(processCallback)
			processCallback(processCount);
	}
	async function initCheerpX()
	{
		const CheerpX = await import('@leaningtech/cheerpx');
		var blockDevice = null;
		// 进度回调通用方法（复用，避免重复代码）
		const progressCallback = (loaded, total) => {
			if (total <= 0) return;
			const progress = Math.ceil((loaded / total) * 100);
			downloadProgress.set(progress);
			// 终端实时打印进度（覆盖当前行，不刷屏）
			term.write(`\r[WebVM] 磁盘镜像加载中：${progress}% (${loaded}/${total} 字节)`);
		};
		try
		{
			switch(configObj.diskImageType)
			{
				case "cloud":
					try
					{
						downloadProgress.set(0);
						blockDevice = await CheerpX.CloudDevice.create(configObj.diskImageUrl, {
							progress: progressCallback
						});
						downloadProgress.set(-1);
						term.write("\r[WebVM] 云磁盘镜像加载完成！\n");
					}
					catch(e)
					{
						var wssProtocol = "wss:";
						if(configObj.diskImageUrl.startsWith(wssProtocol))
						{
							tryPlausible("WS Disk failure");
							downloadProgress.set(0);
							blockDevice = await CheerpX.CloudDevice.create("https:" + configObj.diskImageUrl.substr(wssProtocol.length), {
								progress: progressCallback
							});
							downloadProgress.set(-1);
							term.write("\r[WebVM] 云磁盘镜像(HTTP)加载完成！\n");
						}
						else
						{
							downloadProgress.set(-1); // 失败重置
							throw e;
						}
					}
					break;
				case "bytes":
					downloadProgress.set(0);
					blockDevice = await CheerpX.HttpBytesDevice.create(configObj.diskImageUrl, {
						progress: progressCallback
					});
					downloadProgress.set(-1);
					term.write("\r[WebVM] 字节流磁盘镜像加载完成！\n");
					break;
				case "github":
					downloadProgress.set(0);
					blockDevice = await CheerpX.GitHubDevice.create(configObj.diskImageUrl, {
						progress: progressCallback
					});
					downloadProgress.set(-1);
					term.write("\r[WebVM] GitHub磁盘镜像加载完成！\n");
					break;
				default:
					downloadProgress.set(-1); // 未知类型重置
					throw new Error("Unrecognized device type");
			}
		}
		catch(e)
		{
			downloadProgress.set(-1); // 设备创建失败重置进度
			printMessage(["[WebVM] 磁盘镜像加载失败：" + e.toString()]);
			throw e;
		}

		blockCache = await CheerpX.IDBDevice.create(cacheId);
		var overlayDevice = await CheerpX.OverlayDevice.create(blockDevice, blockCache);
		var webDevice = await CheerpX.WebDevice.create("");
		var documentsDevice = await CheerpX.WebDevice.create("documents");
		var dataDevice = await CheerpX.DataDevice.create();
		var mountPoints = [
			{type:"ext2", dev:overlayDevice, path:"/"},
			{type:"dir", dev:webDevice, path:"/web"},
			{type:"dir", dev:dataDevice, path:"/data"},
			{type:"devs", path:"/dev"},
			{type:"devpts", path:"/dev/pts"},
			{type:"proc", path:"/proc"},
			{type:"sys", path:"/sys"},
			{type:"dir", dev:documentsDevice, path:"/home/user/documents"}
		];
		try
		{
			cx = await CheerpX.Linux.create({mounts: mountPoints, networkInterface: networkInterface});
		}
		catch(e)
		{
			downloadProgress.set(-1); // Linux创建失败重置进度
			printMessage(errorMessage);
			printMessage([e.toString()]);
			return;
		}
		cx.registerCallback("cpuActivity", cpuCallback);
		cx.registerCallback("diskActivity", hddCallback);
		cx.registerCallback("diskLatency", latencyCallback);
		cx.registerCallback("processCreated", handleProcessCreated);
		term.scrollToBottom();
		cxReadFunc = cx.setCustomConsole(writeData, term.cols, term.rows);
		const display = document.getElementById("display");
		if(display)
		{
			setScreenSize(display);
			cx.setActivateConsole(handleActivateConsole);
		}
		while (true)
		{
			await cx.run(configObj.cmd, configObj.args, configObj.opts);
		}
	}
	onMount(initTerminal);
	async function handleConnect()
	{
		const w = window.open("login.html", "_blank");
		cx.networkLogin();
		w.location.href = await startLogin();
	}
	async function handleReset()
	{
		if(blockCache == null)
			return;
		await blockCache.reset();
		location.reload();
	}
	async function handleTool(tool)
	{
		return await handleToolImpl(tool, term);
	}
	async function handleSidebarPinChange(event)
	{
		sideBarPinned = event.detail;
		await tick();
		triggerResize();
	}
</script>

<main class="relative w-full h-full">
	<Nav />
	<!-- 下载块数显示 -->
{#if $downloadProgress >= 0}
	<div class="fixed top-12 right-4 z-50 bg-black/80 text-white px-4 py-2 rounded text-sm font-mono border border-blue-500">
		📦 已下载块数: {$downloadProgress}
	</div>
{/if}
	<!-- 原有布局 -->
	<div class="absolute top-10 bottom-0 left-0 right-0">
		<SideBar 
			on:connect={handleConnect} 
			on:reset={handleReset} 
			handleTool={!configObj.needsDisplay || curVT == 7 ? handleTool : null} 
			on:sidebarPinChange={handleSidebarPinChange}
		>
			<slot></slot>
		</SideBar>
		{#if configObj.needsDisplay}
			<div class="absolute top-0 bottom-0 {sideBarPinned ? 'left-[23.5rem]' : 'left-14'} right-0">
				<canvas class="w-full h-full cursor-none" id="display"></canvas>
			</div>
		{/if}
		<div class="absolute top-0 bottom-0 {sideBarPinned ? 'left-[23.5rem]' : 'left-14'} right-0 p-1 scrollbar" id="console">
		</div>
	</div>
</main>
