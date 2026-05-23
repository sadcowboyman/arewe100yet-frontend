<script lang="ts">
	import './layout.css';
	import favicon from '$lib/assets/favicon.svg';

	let { children } = $props();
</script>

<!-- SVG BRUTALIST FAVICON -->
<link rel="icon" type="image/svg+xml" href="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 32 32'%3E%3Crect width='32' height='32' rx='6' fill='%23f7f7f7'/%3E%3Crect width='32' height='32' rx='6' stroke='%23000000' stroke-width='4' fill='none'/%3E%3Ctext x='16' y='22' font-family='sans-serif' font-weight='900' font-size='16' fill='%23f97316' text-anchor='middle' tracking='-1'%3E100%3C/text%3E%3C/svg%3E">
{@render children()}
