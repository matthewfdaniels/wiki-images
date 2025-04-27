<script>
  import { onMount } from 'svelte';
  import { Deck } from '@deck.gl/core';
  import { OrthographicView } from '@deck.gl/core';
  import { IconLayer } from '@deck.gl/layers';
  import * as d3 from 'd3';

  let deckgl;
  let tooltipDiv;
  let searchTerm = 'rainbow';
  let isLoading = false;

  onMount(async () => {
    // Initialize deck.gl
    deckgl = new Deck({
      views: new OrthographicView(),
      initialViewState: { target: [0, 0, 0], zoom: 3 },
      controller: true,
      getTooltip: ({object}) => object && `${object.title}\n${object.snippet}`
    });

    await updateData();
  });

  async function updateData() {
    isLoading = true;
    try {
      // Clear existing layers
      deckgl.setProps({ layers: [] });
      
      const data = await getData();
      const circles = data.map((d, i) => {
        return {
          title: d.title,
          snippet: d.snippet,
          url: d.imageUrl,
          r: Math.log(d.size + 1)
        };
      });

      d3.packSiblings(circles);
      renderLayers(circles);
    } catch (error) {
      console.error('Error fetching data:', error);
    } finally {
      isLoading = false;
    }
  }

  function renderLayers(data) {
    const iconLayer = new IconLayer({
      id: 'images',
      data,
      getIcon: d => ({
        url: d.url,
        width: 128,
        height: 128
      }),
      getSize: d => d.r * 1.4,
      getPosition: d => [d.x, d.y],
      sizeUnits: 'common',
      pickable: true
    });

    deckgl.setProps({ layers: [iconLayer] });
  }

  async function getData() {
    // First get the search results
    const searchResponse = await fetch(
      'https://commons.wikimedia.org/w/api.php?' + 
      new URLSearchParams({
        action: 'query',
        format: 'json',
        list: 'search',
        srsearch: searchTerm,
        srnamespace: '6',
        srlimit: '50',
        origin: '*'
      })
    );
    const searchData = await searchResponse.json();
    const searchResults = searchData.query.search;

    // Then get image URLs for each result
    const resultsWithUrls = await Promise.all(
      searchResults.map(async (result) => {
        const imageResponse = await fetch(
          'https://commons.wikimedia.org/w/api.php?' + 
          new URLSearchParams({
            action: 'query',
            format: 'json',
            prop: 'imageinfo',
            iiprop: 'url',
            titles: result.title,
            origin: '*'
          })
        );
        const imageData = await imageResponse.json();
        const pages = Object.values(imageData.query.pages);
        const imageUrl = pages[0]?.imageinfo?.[0]?.url;
        
        return {
          ...result,
          imageUrl
        };
      })
    );

    return resultsWithUrls;
  }

  function handleSearch(event) {
    if (event.key === 'Enter') {
      updateData();
    }
  }
</script>

<div class="container">
  <div class="search-box">
    <input 
      type="text" 
      bind:value={searchTerm} 
      on:keydown={handleSearch}
      placeholder="Search for images..."
    />
    <button on:click={updateData}>Search</button>
  </div>
  <div bind:this={tooltipDiv} id="tooltip"></div>
  {#if isLoading}
    <div class="loading">Loading...</div>
  {/if}
</div>

<style>
  .container {
    width: 100vw;
    height: 100vh;
    margin: 0;
    background: #f7f7f7;
    overflow: hidden;
    position: relative;
  }

  .search-box {
    position: absolute;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    z-index: 1000;
    display: flex;
    gap: 10px;
  }

  input {
    padding: 8px 12px;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-size: 16px;
    width: 300px;
  }

  button {
    padding: 8px 16px;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }

  button:hover {
    background: #0056b3;
  }

  .loading {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    background: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 20px 40px;
    border-radius: 8px;
    font-size: 18px;
  }

  :global(.deck-tooltip) {
    font-family: Helvetica, Arial, sans-serif;
    padding: 6px !important;
    margin: 8px;
    max-width: 300px;
    font-size: 10px;
  }

  :global(canvas) {
    top: 0;
  }
</style> 