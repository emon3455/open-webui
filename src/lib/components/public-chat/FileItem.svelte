<!-- FileItem.svelte -->
<script>
    import { FileText, X, LoaderCircle } from 'lucide-svelte';
    export let file;
    export let onRemove;
    export let isUploading;
  
    const formatFileSize = (size) => {
          if (!size) return '0 B';
          return size < 1024
              ? `${size} B`
              : size < 1048576
                  ? `${(size / 1024).toFixed(1)} KB`
                  : `${(size / 1048576).toFixed(1)} MB`;
      };
    
    $: displayFile = file.isUploaded ? file.file : file.localFile;
    $: displayName = displayFile?.name || displayFile?.filename;
    $: displaySize = displayFile?.size || displayFile?.meta?.size;
    $: formattedName = displayName?.length > 9 
      ? `${displayName?.slice(0, 6)}...${displayName?.slice(-4)}` 
      : displayName;
  </script>
  
  <div class="flex items-center h-14 space-x-1 p-2 text-primaryText bg-gray-50 hover:bg-secondary rounded-lg shadow-sm">
    {#if isUploading}
      <LoaderCircle size={37} class="text-red-600 p-2 rounded-md animate-spin" />
    {:else}
      <FileText size={37} class="text-red-600 p-2 bg-gray-200 rounded-md" />
    {/if}
    <div class="flex flex-col ">
      <div class="flex justify-between items-center gap-5 w-full mb-1">
        <span class="font-medium text-xs w-20">
          {formattedName}
        </span>
        <button
          on:click={() => onRemove(file)}
          class="text-red-500 hover:text-red-700"
          disabled={isUploading}
        >
          <X size={12} />
        </button>
      </div>
      <div class="flex justify-between w-full">
        <p class="text-xs text-gray-600">
          {isUploading ? 'uploading...' : file.isUploaded ? 'ready' : 'pending'}
        </p>
        <p class="text-xs text-gray-600">
          {formatFileSize(displaySize)}
        </p>
      </div>
    </div>
  </div>