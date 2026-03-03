<script>
	import { networkData, startLogin, updateButtonData } from '$lib/network.js'
	import { createEventDispatcher } from 'svelte';
	import PanelButton from './PanelButton.svelte';
	var dispatch = createEventDispatcher();
	var connectionState = networkData.connectionState;
	var exitNode = networkData.exitNode;

	function handleConnect() {
		connectionState.set("DOWNLOADING");
		dispatch('connect');
	}

	let buttonData = null;
	$: buttonData = updateButtonData($connectionState, handleConnect);
</script>
<h1 class="text-lg font-bold">网络连接</h1>
<PanelButton buttonImage="assets/tailscale.svg" clickUrl={buttonData.clickUrl} clickHandler={buttonData.clickHandler} rightClickHandler={buttonData.rightClickHandler} buttonTooltip={buttonData.buttonTooltip} buttonText={buttonData.buttonText}>
	{#if $connectionState == "CONNECTED"}
		<i class='fas fa-circle fa-xs ml-auto {$exitNode ? 'text-green-500' : 'text-amber-500'}' title={$exitNode ? '就绪' : '未配置出口节点'}></i>
	{/if}
</PanelButton>
<p>WebVM 可以通过 Tailscale 连接到互联网</p>
<p>由于浏览器目前尚不支持 TCP/UDP 套接字，因此需要使用 Tailscale 来实现网络连接。</p>
