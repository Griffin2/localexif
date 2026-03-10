<script>
	let { sourceUrl = '#' } = $props();
</script>

<div class="trust-bar">
	<span class="dot"></span>
	<span>Your data never leaves this browser</span>
	<span class="sep">·</span>
	<a href={sourceUrl} target="_blank" rel="noopener">view source</a>
</div>
