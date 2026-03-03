<script>
	import PanelButton from './PanelButton.svelte';
	import { createEventDispatcher } from 'svelte';
	import { diskLatency } from './activities.js'
	var dispatch = createEventDispatcher();
	let state = "START";
	function handleReset()
	{
		if(state == "START")
			state = "CONFIRM";
		else if (state == "CONFIRM") {
			state = "RESETTING";
			dispatch('reset');
		}
	}
	function getButtonText(state)
	{
		if(state == "START")
			return "重置磁盘";
		else if (state == "RESETTING")
			return "重置中...";
		else
			return "重置磁盘，确认吗？";
	}
	function getBgColor(state)
	{
		if(state == "START")
		{
			// 使用默认样式
			return undefined;
		}
		else
		{
			return "bg-red-900";
		}
	}
	function getHoverColor(state)
	{
		if(state == "START")
		{
			// 使用默认样式
			return undefined;
		}
		else
		{
			return "hover:bg-red-700";
		}
	}
</script>
<h1 class="text-lg font-bold">磁盘</h1>
<PanelButton buttonIcon="fa-solid fa-trash-can" clickHandler={handleReset} buttonText={getButtonText(state)} bgColor={getBgColor(state)} hoverColor={getHoverColor(state)}>
</PanelButton>
{#if state == "CONFIRM"}
	<p><span class="font-bold">警告：</span>WebVM 将会重新加载</p>
{:else if state == "RESETTING"}
	<p><span class="font-bold">重置进行中：</span>请稍候...</p>
{:else}
	<p><span class="font-bold">后端延迟：</span>{$diskLatency}ms</p>
{/if}
<p>WebVM 运行在一个完整的 Linux 发行版之上</p>
<p>支持最大 2GB 的文件系统，数据将完全根据需求按需下载</p>
<p>WebVM 云端后端采用 WebSockets 技术，并通过全球 CDN 进行分发，以最大限度降低下载延迟</p>
