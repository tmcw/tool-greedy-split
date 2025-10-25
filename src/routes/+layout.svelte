<script lang="ts">
  import favicon from "$lib/assets/favicon.svg";
  import "$lib/assets/style.css?text";

  let { children } = $props();
</script>

<svelte:head>
  <link rel="icon" href={favicon} />
  <title>Greedy split</title>
</svelte:head>

{@render children?.()}
