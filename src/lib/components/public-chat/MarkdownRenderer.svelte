<script>
  import { marked } from 'marked';
  import { onMount } from 'svelte';

  export let markdownText = '';
  let htmlContent = '';

  // Initialize marked with plugins
  onMount(() => {
    marked.setOptions({
      gfm: true,
      breaks: true,
      smartLists: true,
      smartypants: true
    });
  });

  // Convert markdown to HTML when markdownText changes
  $: {
    htmlContent = marked.parse(markdownText || '');
  }
</script>

<div class="ai-response">
  <div class="ai-prefix"></div>
  {@html htmlContent}
</div>

<style>
  /* General container for the AI response */
  .ai-response {
    line-height: 1.5;
    font-family: Inter, sans-serif; /* Normal font instead of monospace */
    /* padding: 16px; */
    font-size: 16px;
    /* color: #0000AA; Dark blue text color similar to the image */
    /* white-space: pre-wrap; */
    /* background-color: #f5f5f5; Light gray background */
    border-radius: 8px;
  }

  /* AI message prefix styling */
  .ai-prefix {
    font-weight: bold;
    /* margin-bottom: 8px; */
  }

  /* Override default styles to match the formatting in the image */
  .ai-response :global(h1),
  .ai-response :global(h2),
  .ai-response :global(h3),
  .ai-response :global(h4) {
    margin: 16px 0 8px 0;
    font-size: inherit;
    font-weight: bold;
    /* color: #0000AA; */
    font-family: Inter, sans-serif;
  }

  /* Reset margins for paragraphs to match the format */
  .ai-response :global(p) {
    /* margin: 8px 0; */
    padding: 0;
    font-family: Inter, sans-serif;
  }

  /* Style lists */
  .ai-response :global(ul),
  .ai-response :global(ol) {
    padding-left: 20px;
    /* margin: 8px 0; */
    list-style-type: disc;
    font-family: Inter, sans-serif;
    margin-bottom: 20px;
  }

  /* Style list items */
  .ai-response :global(li) {
    margin: 4px 0;
    list-style-type:decimal;
    
    font-family: Inter, sans-serif;
  }
  .ai-response :global(li li) {
  margin-left: 1rem; /* indent the nested item */
  list-style-type: '• '; /* different bullet symbol */
  color: #666; /* for example, lighter text */
  font-size: 0.95rem;
  }

  .ai-response :global(li li li) {
  margin-left: 1rem; /* indent the nested item */
  list-style-type: '- '; /* different bullet symbol */
  color: #666; /* for example, lighter text */
  font-size: 0.95rem;
  }

  /* Override link styling to match the blue color in the image */
  .ai-response :global(a) {
    text-decoration: underline;
    font-family: Inter, sans-serif;
  }

  /* Style code blocks */
  .ai-response :global(code),
  .ai-response :global(pre) {
    font-family: Inter, sans-serif;
    background-color: transparent;
    padding: 0;
    margin: 0;
    /* color: #0000AA; */
  }
</style>