<script>
 	import { onMount, tick } from 'svelte';
 	import { get } from 'svelte/store';
 	import Nav from 'labs/packages/global-navbar/src/Nav.svelte';
 	import SideBar from '$lib/SideBar.svelte';
 	// 第1步：添加ProgressAddon导入
 	import { ProgressAddon } from '@xterm/addon-progress';
 	import '$lib/global.css';
 	import '@xterm/xterm/css/xterm.css'
 	import '@fortawesome/fontawesome-free/css/all.min.css'
 	import { networkInterface, startLogin } from '$lib/network.js'
 	import { cpuActivity, diskActivity, cpuPercentage, diskLatency } from '$lib/activities.js'
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
 	// 第2步：添加进度条相关变量（添加响应式声明）
 	// ===== 进度条相关变量 =====
 	let progressAddon = null;
 	let totalImageSize = 0;
 	let downloadedSize = 0;
 	let downloadSpeed = 0;
 	let lastBytes = 0;
 	let lastTime = Date.now();
 	let isDownloading = false;
 	// 为模板中使用的变量添加响应式声明
 	$: downloadPercent = totalImageSize > 0 ? Math.round((downloadedSize / totalImageSize) * 100) : 0;
 	$: formattedDownloaded = formatBytes(downloadedSize);
 	$: formattedTotal = formatBytes(totalImageSize);
 	$: formattedSpeed = formatBytes(downloadSpeed) + '/s';
 	function writeData(buf, vt)
 	{
 		if(vt != 1)
 			return;
 		term.write(new Uint8Array(buf));
 	}
 	// 第3步：添加格式化字节和进度发送辅助函数（添加非空判断）
 	// 格式化字节数
 	function formatBytes(bytes) {
 	    if (bytes === 0 || isNaN(bytes)) return '0 B';
 	    const k = 1024;
 	    const sizes = ['B', 'KB', 'MB', 'GB'];
 	    const i = Math.floor(Math.log(bytes) / Math.log(k));
 	    return parseFloat((bytes / Math.pow(k, i)).toFixed(1)) + ' ' + sizes[i];
 	}
 	// 发送进度到终端（添加非空判断）
 	function sendProgressToTerminal(percent, downloaded, total) {
 	    if (!progressAddon || !term) return;
 	    
 	    // 进度条转义序列 (ESC ] 9 ; 4 ; 1 ; <percent> BEL)
 	    const sequence = `\x1b]9;4;1;${percent}\x07`;
 	    term.write(sequence);
 	    
 	    // 同时在终端标题显示详细信息
 	    const titleSeq = `\x1b]0;下载: ${percent}% (${formatBytes(downloaded)} / ${formatBytes(total)}) 速度: ${formatBytes(downloadSpeed)}/s\x07`;
 	    term.write(titleSeq);
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
 			// NOTE: The first event could be before the limit,
 			//       we need at least one event to correctly mark
 			//       active time when there is long time under load
 			var eTime = e.t;
 			if(eTime < limitTime)
 				eTime = limitTime;
 			if(e.state == "ready")
 			{
 				// Inactive state, add the time from lastActiveTime
 				totalActiveTime += (eTime - lastActiveTime);
 				lastWasActive = false;
 			}
 			else
 			{
 				// Active state
 				lastActiveTime = eTime;
 				lastWasActive = true;
 			}
 		}
 		// Add the last interval if needed
 		if(lastWasActive)
 		{
 			totalActiveTime += (curTime - lastActiveTime);
 		}
 		cpuPercentage.set(Math.ceil((totalActiveTime / 10000) * 100));
 	}
 	function hddCallback(state)
 	{
 		diskActivity.set(state != "ready");
 	}
 	function latencyCallback(latency)
 	{
 		diskLatencies.push(latency);
 		if(diskLatencies.length > 30)
 			diskLatencies.shift();
 		// Average the latency over at most 30 blocks
 		var total = 0;
 		for(var i=0;i<diskLatencies.length;i++)
 			total += diskLatencies[i];
 		var avg = total / diskLatencies.length;
 		diskLatency.set(Math.ceil(avg));
 	}
 	function cpuCallback(state)
 	{
 		cpuActivity.set(state != "ready");
 		var curTime = Date.now();
 		var limitTime = curTime - 10000;
 		expireEvents(cpuActivityEvents, curTime, limitTime);
 		cpuActivityEvents.push({t: curTime, state: state});
 		computeCpuActivity(curTime, limitTime);
 		// Start an interval timer to cleanup old samples when no further activity is received
 		if(activityEventsInterval != 0)
 			clearInterval(activityEventsInterval);
 		activityEventsInterval = setInterval(cleanupEvents, 2000);
 	}
 	function computeXTermFontSize()
 	{
 		return parseInt(getComputedStyle(document.body).fontSize);
 	}
 	// 第4步：添加fetch拦截函数（修正拦截规则、添加异常捕获、备份原生fetch）
 	// 备份原生fetch，用于后续还原
 	let originalFetch = window.fetch;
 	// 拦截 fetch 下载进度
 	function setupProgressInterceptor() {
 	    if (window.fetch !== originalFetch) {
 	        console.warn('Fetch interceptor already set up, skipping.');
 	        return;
 	    }
 	    
 	    window.fetch = function(url, options) {
 	        // 修正：拦截匹配configObj.diskImageUrl的请求，而不仅仅是.ext2后缀
 	        if (typeof url === 'string' && configObj && configObj.diskImageUrl && url.includes(configObj.diskImageUrl)) {
 	            console.log('检测到镜像下载:', url);
 	            isDownloading = true;
 	            totalImageSize = 0;
 	            downloadedSize = 0;
 	            lastBytes = 0;
 	            lastTime = Date.now();
 	            downloadSpeed = 0;
 	            
 	            return originalFetch(url, options).then(async response => {
 	                // 获取文件总大小
 	                const contentLength = response.headers.get('content-length');
 	                if (!contentLength) {
 	                    console.warn('无法获取文件大小');
 	                    isDownloading = false;
 	                    return response;
 	                }
 	                
 	                totalImageSize = parseInt(contentLength, 10);
 	                
 	                // 获取响应体的 reader
 	                const reader = response.body.getReader();
 	                
 	                // 创建新的可读流来跟踪进度
 	                const stream = new ReadableStream({
 	                    start(controller) {
                         function push() {
                             reader.read().then(({done, value}) => {
                                 if (done) {
                                     controller.close();
                                     sendProgressToTerminal(100, totalImageSize, totalImageSize);
                                     isDownloading = false;
                                     console.log('下载完成');
                                     return;
                                 }
                                 
                                 // 更新已下载字节数
                                 downloadedSize += value.length;
                                 
                                 // 计算速度（添加0值兜底）
                                 const now = Date.now();
                                 const timeDiff = now - lastTime;
                                 if (timeDiff > 500 && timeDiff > 0) {
                                     downloadSpeed = (downloadedSize - lastBytes) / (timeDiff / 1000);
                                     lastBytes = downloadedSize;
                                     lastTime = now;
                                 }
                                 
                                 // 计算百分比并发送
                                 const percent = Math.round((downloadedSize / totalImageSize) * 100);
                                 sendProgressToTerminal(percent, downloadedSize, totalImageSize);
                                 
                                 controller.enqueue(value);
                                 push();
                             }).catch(error => {
                                 // 捕获读取流时的错误
                                 console.error('下载流读取错误:', error);
                                 controller.error(error);
                                 isDownloading = false;
                                 sendProgressToTerminal(0, 0, 0);
                             });
                         }
                         push();
                     }
                 });
                 
                 // 返回新的响应
                 return new Response(stream, {
                     headers: response.headers,
                     status: response.status,
                     statusText: response.statusText
                 });
             }).catch(error => {
                 // 捕获fetch请求本身的错误
                 console.error('镜像下载请求失败:', error);
                 isDownloading = false;
                 sendProgressToTerminal(0, 0, 0);
                 throw error; // 继续抛出错误，让原代码处理
             });
         }
         
         // 非镜像请求，直接返回
         return originalFetch(url, options);
     };
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
 		// Compute the size to be used for AI screenshots
 		var screenshotMult = 1.0;
 		var maxWidth = 1024;
 		var maxHeight = 768;
 		if(internalWidth > maxWidth)
 			screenshotMult = maxWidth / internalWidth;
 		if(internalHeight > maxHeight)
 			screenshotMult = Math.min(screenshotMult, maxHeight / internalHeight);
 		var screenshotWidth = Math.floor(internalWidth * screenshotMult);
 		var screenshotHeight = Math.floor(internalHeight * screenshotMult);
 		// Track the state of the mouse as requested by the AI, to avoid losing the position due to user movement
 		displayConfig.set({width: screenshotWidth, height: screenshotHeight, mouseMult: internalMult * screenshotMult});
 	}
 	var curInnerWidth = 0;
 	var curInnerHeight = 0;
 	function handleResize()
 	{
 		// Avoid spurious resize events caused by the soft keyboard
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
 		// 第5步：加载进度条插件（添加非空判断）
 		// 加载进度条插件
 		try {
 		    progressAddon = new ProgressAddon();
 		    term.loadAddon(progressAddon);
 		} catch (error) {
 		    console.error('Failed to load ProgressAddon:', error);
 		    progressAddon = null; // 加载失败时置为null
 		}
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
 		// Avoid undesired default DnD handling
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
 			printIntroMessage(introMessage); // 保留原文件的写法，不做修改
 		try
 		{
 			await initCheerpX();
 		}
 		catch(e)
 		{
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
 		// Raise the display to the foreground
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
 		switch(configObj.diskImageType)
 		{
 			case "cloud":
 				try
 				{
 					blockDevice = await CheerpX.CloudDevice.create(configObj.diskImageUrl);
 				}
 				catch(e)
 				{
 					// Report the failure and try again with plain HTTP
 					var wssProtocol = "wss:";
 					if(configObj.diskImageUrl.startsWith(wssProtocol))
 					{
 						// WebSocket protocol failed, try agin using plain HTTP
 						tryPlausible("WS Disk failure");
 						blockDevice = await CheerpX.CloudDevice.create("https:" + configObj.diskImageUrl.substr(wssProtocol.length));
 					}
 					else
 					{
 						// No other recovery option
 						throw e;
 					}
 				}
 				break;
 			case "bytes":
 				blockDevice = await CheerpX.HttpBytesDevice.create(configObj.diskImageUrl);
 				break;
 			case "github":
 				blockDevice = await CheerpX.GitHubDevice.create(configObj.diskImageUrl);
 				break;
 			default:
 				throw new Error("Unrecognized device type");
 		}
 		blockCache = await CheerpX.IDBDevice.create(cacheId);
 		var overlayDevice = await CheerpX.OverlayDevice.create(blockDevice, blockCache);
 		var webDevice = await CheerpX.WebDevice.create("");
 		var documentsDevice = await CheerpX.WebDevice.create("documents");
 		var dataDevice = await CheerpX.DataDevice.create();
 		var mountPoints = [
 			// The root filesystem, as an Ext2 image
 			{type:"ext2", dev:overlayDevice, path:"/"},
 			// Access to files on the Web server, relative to the current page
 			{type:"dir", dev:webDevice, path:"/web"},
 			// Access to read-only data coming from JavaScript
 			{type:"dir", dev:dataDevice, path:"/data"},
 			// Automatically created device files
 			{type:"devs", path:"/dev"},
 			// Pseudo-terminals
 			{type:"devpts", path:"/dev/pts"},
 			// The Linux 'proc' filesystem which provides information about running processes
 			{type:"proc", path:"/proc"},
 			// The Linux 'sysfs' filesystem which is used to enumerate emulated devices
 			{type:"sys", path:"/sys"},
 			// Convenient access to sample documents in the user directory
 			{type:"dir", dev:documentsDevice, path:"/home/user/documents"}
 		];
 		try
 		{
 			cx = await CheerpX.Linux.create({mounts: mountPoints, networkInterface: networkInterface});
 		}
 		catch(e)
 		{
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
 		// Run the command in a loop, in case the user exits
 		while (true)
 		{
 			await cx.run(configObj.cmd, configObj.args, configObj.opts);
 		}
 	}
 	// 第6步：修改onMount，先启动拦截器再初始化终端
 	onMount(() => {
 	    setupProgressInterceptor();  // 先设置拦截器
 	    initTerminal();              // 再初始化终端
 	});
 	// 添加组件销毁时的清理逻辑，还原原生fetch
 	onDestroy(() => {
 	    if (window.fetch !== originalFetch) {
 	        window.fetch = originalFetch;
 	        console.log('Fetch interceptor restored.');
 	    }
 	});
 	async function handleConnect()
 	{
 		const w = window.open("login.html", "_blank");
 		cx.networkLogin();
 		w.location.href = await startLogin();
 	}
 	async function handleReset()
 	{
 		// Be robust before initialization
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
 		// Make sure the pinning state of reflected in the layout
 		await tick();
 		// Adjust the layout based on the new sidebar state
 		triggerResize();
 	}
 </script>
 <main class="relative w-full h-full">
 	<Nav />
 	<div class="absolute top-10 bottom-0 left-0 right-0">
 		<SideBar on:connect={handleConnect} on:reset={handleReset} handleTool={!configObj.needsDisplay || curVT == 7 ? handleTool : null} on:sidebarPinChange={handleSidebarPinChange}>
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
 	<!-- 第7步：添加可视化进度条浮窗（使用响应式变量） -->
 	<!-- 下载进度浮动窗 -->
 	{#if isDownloading && totalImageSize > 0}
 	    <div class="fixed top-12 right-4 bg-gray-800 text-white p-4 rounded-lg shadow-xl z-50 w-80 border border-blue-500">
 	        <div class="text-sm font-semibold mb-2 flex items-center">
 	            <svg class="animate-spin -ml-1 mr-2 h-4 w-4 text-blue-400" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
 	                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
 	                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
 	            </svg>
 	            正在加载系统镜像
 	        </div>
 	        
 	        <!-- 进度条 -->
 	        <div class="w-full bg-gray-600 rounded-full h-2.5 mb-2">
 	            <div class="bg-blue-600 h-2.5 rounded-full transition-all duration-300" 
 	                 style="width: {downloadPercent}%"></div>
 	        </div>
 	        
 	        <!-- 进度数字 -->
 	        <div class="flex justify-between text-xs mb-1">
 	            <span>{downloadPercent}%</span>
 	            <span>{formattedDownloaded} / {formattedTotal}</span>
 	        </div>
 	        
 	        <!-- 下载速度 -->
 	        <div class="text-xs text-gray-300">
 	            速度: {formattedSpeed}
 	        </div>
 	    </div>
 	{/if}
 </main>