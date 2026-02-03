<script lang="ts">
  interface Entry {
    name: string;
    type: "dir" | "file";
    path: string;
  }

  interface Props {
    mode: "directory" | "file";
    initialPath?: string;
    onSelect: (path: string, content?: string) => void;
    onClose: () => void;
  }

  let { mode, initialPath = "~", onSelect, onClose }: Props = $props();

  let currentPath = $state(initialPath);
  let entries = $state<Entry[]>([]);
  let parentPath = $state<string | null>(null);
  let loading = $state(false);
  let error = $state<string | null>(null);
  let resolvedPath = $state(initialPath);

  async function browse(path: string) {
    loading = true;
    error = null;
    try {
      const browseMode = mode === "directory" ? "all" : "all";
      const res = await fetch(
        `/v1/files/browse?path=${encodeURIComponent(path)}&mode=${browseMode}`,
      );
      if (!res.ok) {
        error = `Failed to browse: ${res.status}`;
        return;
      }
      const data = (await res.json()) as {
        entries: Entry[];
        current: string;
        parent: string | null;
        error?: string;
      };
      if (data.error) {
        error = data.error;
        return;
      }
      entries = data.entries;
      currentPath = path;
      resolvedPath = data.current;
      parentPath = data.parent;
    } catch {
      error = "Failed to connect to server";
    } finally {
      loading = false;
    }
  }

  async function selectFile(entry: Entry) {
    if (entry.type === "dir") {
      await browse(entry.path);
      return;
    }
    // File mode — read the file content and return it
    loading = true;
    try {
      const res = await fetch(
        `/v1/files/read-file?path=${encodeURIComponent(entry.path)}`,
      );
      if (!res.ok) {
        const detail = await res.text();
        error = `Failed to read file: ${detail}`;
        return;
      }
      const data = (await res.json()) as {
        path: string;
        name: string;
        content: string;
      };
      onSelect(data.path, data.content);
    } catch {
      error = "Failed to read file";
    } finally {
      loading = false;
    }
  }

  function selectCurrentDirectory() {
    onSelect(resolvedPath);
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === "Escape") {
      onClose();
    }
  }

  // Build breadcrumb segments from resolved path
  const breadcrumbs = $derived(() => {
    const parts = resolvedPath.split("/").filter(Boolean);
    const segments: Array<{ name: string; path: string }> = [];
    let accumulated = "";
    for (const part of parts) {
      accumulated += "/" + part;
      segments.push({ name: part, path: accumulated });
    }
    return segments;
  });

  // Initial browse
  $effect(() => {
    browse(initialPath);
  });
</script>

<svelte:window onkeydown={handleKeydown} />

<!-- Backdrop -->
<button
  type="button"
  class="fixed inset-0 bg-black/60 z-[10000] cursor-default"
  onclick={onClose}
  aria-label="Close browser"
></button>

<!-- Modal -->
<div
  class="fixed inset-x-4 top-[10%] bottom-[10%] sm:inset-x-auto sm:left-1/2 sm:-translate-x-1/2 sm:w-[560px] sm:max-h-[70vh] z-[10001] flex flex-col command-panel rounded overflow-hidden"
>
  <!-- Header -->
  <div
    class="flex items-center justify-between px-4 py-3 border-b border-exo-medium-gray/30"
  >
    <span class="text-xs font-mono tracking-wider uppercase text-exo-yellow">
      {mode === "directory" ? "SELECT PROJECT FOLDER" : "PROJECT FILES"}
    </span>
    <button
      type="button"
      onclick={onClose}
      class="text-exo-light-gray/50 hover:text-exo-yellow transition-colors cursor-pointer"
    >
      <svg
        class="w-4 h-4"
        fill="none"
        viewBox="0 0 24 24"
        stroke="currentColor"
        stroke-width="2"
      >
        <path
          stroke-linecap="round"
          stroke-linejoin="round"
          d="M6 18L18 6M6 6l12 12"
        />
      </svg>
    </button>
  </div>

  <!-- Breadcrumb path -->
  <div
    class="flex items-center gap-1 px-4 py-2 border-b border-exo-medium-gray/20 overflow-x-auto scrollbar-none"
  >
    <button
      type="button"
      onclick={() => browse("/")}
      class="text-xs font-mono text-exo-light-gray/50 hover:text-exo-yellow transition-colors cursor-pointer flex-shrink-0"
    >
      /
    </button>
    {#each breadcrumbs() as segment, i}
      <span class="text-exo-medium-gray/50 text-xs">/</span>
      <button
        type="button"
        onclick={() => browse(segment.path)}
        class="text-xs font-mono transition-colors cursor-pointer flex-shrink-0 {i ===
        breadcrumbs().length - 1
          ? 'text-exo-yellow'
          : 'text-exo-light-gray/60 hover:text-exo-yellow'}"
      >
        {segment.name}
      </button>
    {/each}
  </div>

  <!-- Entries list -->
  <div class="flex-1 overflow-y-auto min-h-0">
    {#if loading && entries.length === 0}
      <div class="flex items-center justify-center py-8">
        <span
          class="w-4 h-4 border-2 border-exo-yellow border-t-transparent rounded-full animate-spin"
        ></span>
      </div>
    {:else if error}
      <div class="px-4 py-3 text-xs font-mono text-red-400">{error}</div>
    {:else}
      <!-- Parent directory -->
      {#if parentPath}
        <button
          type="button"
          onclick={() => browse(parentPath ?? "/")}
          class="w-full flex items-center gap-3 px-4 py-2 text-left hover:bg-exo-medium-gray/20 transition-colors cursor-pointer group"
        >
          <svg
            class="w-4 h-4 text-exo-light-gray/40 group-hover:text-exo-yellow transition-colors flex-shrink-0"
            fill="none"
            viewBox="0 0 24 24"
            stroke="currentColor"
            stroke-width="2"
          >
            <path
              stroke-linecap="round"
              stroke-linejoin="round"
              d="M15 19l-7-7 7-7"
            />
          </svg>
          <span
            class="text-xs font-mono text-exo-light-gray/50 group-hover:text-exo-yellow transition-colors"
            >..</span
          >
        </button>
      {/if}

      {#each entries as entry}
        <button
          type="button"
          onclick={() =>
            entry.type === "dir" && mode === "directory"
              ? browse(entry.path)
              : selectFile(entry)}
          class="w-full flex items-center gap-3 px-4 py-2 text-left hover:bg-exo-medium-gray/20 transition-colors cursor-pointer group"
        >
          {#if entry.type === "dir"}
            <svg
              class="w-4 h-4 text-exo-yellow/60 group-hover:text-exo-yellow transition-colors flex-shrink-0"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="2"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M3 7v10a2 2 0 002 2h14a2 2 0 002-2V9a2 2 0 00-2-2h-6l-2-2H5a2 2 0 00-2 2z"
              />
            </svg>
          {:else}
            <svg
              class="w-4 h-4 text-exo-light-gray/40 group-hover:text-exo-light-gray transition-colors flex-shrink-0"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
              stroke-width="2"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"
              />
            </svg>
          {/if}
          <span
            class="text-xs font-mono truncate {entry.type === 'dir'
              ? 'text-exo-yellow/80 group-hover:text-exo-yellow'
              : 'text-exo-light-gray group-hover:text-white'} transition-colors"
          >
            {entry.name}
          </span>
        </button>
      {/each}

      {#if entries.length === 0 && !loading}
        <div
          class="px-4 py-6 text-xs font-mono text-exo-light-gray/40 text-center"
        >
          Empty directory
        </div>
      {/if}
    {/if}
  </div>

  <!-- Footer -->
  {#if mode === "directory"}
    <div
      class="flex items-center justify-between px-4 py-3 border-t border-exo-medium-gray/30"
    >
      <span class="text-[10px] font-mono text-exo-light-gray/40 truncate mr-3">
        {resolvedPath}
      </span>
      <button
        type="button"
        onclick={selectCurrentDirectory}
        class="px-4 py-1.5 rounded text-xs font-mono tracking-wider uppercase bg-exo-yellow text-exo-black hover:bg-exo-yellow-darker transition-all cursor-pointer flex-shrink-0"
      >
        SELECT
      </button>
    </div>
  {/if}
</div>
