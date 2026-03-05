<script>
	import PanelButton from './PanelButton.svelte';
	import DiscordPresenceCount from 'labs/packages/astro-theme/components/nav/DiscordPresenceCount.svelte'
</script>

<h1 class="text-lg font-bold">Discord</h1>
<PanelButton buttonImage="assets/discord-mark-blue.svg" clickUrl="https://discord.gg/yTNZgySKGa" buttonText="加入我们的 Discord">
	<i class='fas fa-circle fa-xs ml-auto text-green-500'></i>
	<span class="ml-1"><DiscordPresenceCount /></span>
</PanelButton>
<p>你有关于 WebVM 或 CheerpX 的问题吗？</p>
<p>加入我们的社区，我们很乐意为你提供帮助！</p>
