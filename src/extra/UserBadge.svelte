<script lang="ts">
  import { client } from "Client";
  import type { CustomBadge, UserBadge } from "revolt-toolset";
  import { tippy } from "svelte-tippy";
  import { Theme } from "Theme";

  export let data: UserBadge | CustomBadge;
  export let size: number;
</script>

<img
  use:tippy={{
    content: `<div class="flex flex-col gap-0.5"><div class="text-sm">${data.name}</div><div class="text-xs" style="color:${$Theme["tertiary-foreground"]};">${data.description}</div></div>`,
    allowHTML: true,
  }}
  width={size}
  height={size}
  src={"ids" in data ? client.proxyFile(data.image) || data.image : data.image}
  alt={data.name}
  class="shrink-0"
/>
