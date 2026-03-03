<script>
	import BlogPost from './BlogPost.svelte';
	import { page } from '$app/stores';
</script>
<h1 class="text-lg font-bold">博客文章</h1>
<div class="overflow-y-scroll scrollbar flex flex-col gap-2">
{#each $page.data.posts as post}
	<BlogPost
		title={post.title}
		image={post.image}
		url={post.url}
	/>
{/each}
</div>
