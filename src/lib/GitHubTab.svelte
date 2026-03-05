<script>
	import PanelButton from './PanelButton.svelte';
	import GitHubStarCount from 'labs/packages/astro-theme/components/nav/GitHubStarCount.svelte'
</script>

<h1 class="text-lg font-bold">GitHub</h1>
<PanelButton buttonImage="assets/github-mark-white.svg" clickUrl="https://github.com/leaningtech/webvm" buttonText="GitHub 仓库">
	<i class='fas fa-star fa-xs ml-auto'></i>
	<span class="ml-1"><GitHubStarCount repo="leaningtech/webvm"/></span>
</PanelButton>
<p>喜欢 WebVM？<a class="underline" href="https://github.com/leaningtech/webvm" target="_blank">给我们点个星！</a></p>
<p>WebVM 是开源软件，你可以 fork 构建自己的版本，并开始基于 CheerpX 开发项目</p>
<p>发现 bug？请提交一个 <a class="underline" href="https://github.com/leaningtech/webvm/issues" target="_blank">GitHub 问题</a></p>
