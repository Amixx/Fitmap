<script lang="ts">
  import wardrobe from './data/wardrobe.json';

  type Item = (typeof wardrobe.items)[number];
  type View = 'wardrobe' | 'outfits' | 'builder';
  const categories = { all: 'Viss', tops: 'Virsdaļas', bottoms: 'Apakšdaļas', outerwear: 'Virsslāņi', shoes: 'Apavi', bags: 'Somas' };

  let view: View = 'wardrobe';
  let category = 'all';
  let search = '';
  let favoritesOnly = false;
  let selected: Item | null = null;
  let builderItems: string[] = [];
  let showEditor = false;
  let toast = '';
  let favorites = new Set(wardrobe.items.filter((item) => item.favorite).map((item) => item.id));

  $: visibleItems = wardrobe.items.filter((item) => {
    const matchesCategory = category === 'all' || item.category === category;
    const matchesSearch = item.name.toLocaleLowerCase().includes(search.toLocaleLowerCase()) || item.colors.join(' ').toLocaleLowerCase().includes(search.toLocaleLowerCase());
    return matchesCategory && matchesSearch && (!favoritesOnly || favorites.has(item.id));
  });
  $: selectedMatches = selectedMatchesFor(selected?.id);
  $: builderMatches = wardrobe.items.filter((item) => !builderItems.includes(item.id)).map((item) => ({ item, score: builderScore(item.id) })).sort((a, b) => b.score - a.score);
  $: builderScoreLabel = builderItems.length < 2 ? 'Izvēlies vēl vienu apģērba gabalu' : `${Math.round(builderMatches.reduce((sum, match) => sum + match.score, 0) / Math.max(builderMatches.length, 1) * 33)}% saderība`;

  function builderScore(id: string) {
    if (!builderItems.length) return 0;
    const scores = builderItems.map((selectedId) => wardrobe.compatibility.find((pair) => pair.items.includes(selectedId) && pair.items.includes(id))?.score ?? 0);
    return Math.min(...scores);
  }
  function selectedMatchesFor(id: string | undefined) {
    if (!id) return [];
    return wardrobe.compatibility.filter((pair) => pair.items.includes(id)).map((pair) => ({ item: wardrobe.items.find((item) => item.id === pair.items.find((pairId) => pairId !== id)), score: pair.score, note: pair.note }));
  }
  function selectItem(id: string) {
    const item = wardrobe.items.find((candidate) => candidate.id === id);
    if (item) selected = item;
  }
  function toggleFavorite(item: Item) {
    favorites.has(item.id) ? favorites.delete(item.id) : favorites.add(item.id);
    favorites = new Set(favorites);
    toast = favorites.has(item.id) ? 'Pievienots izlasei' : 'Noņemts no izlases';
    setTimeout(() => (toast = ''), 1800);
  }
  function toggleSelectedFavorite() {
    if (selected) toggleFavorite(selected);
  }
  function addToBuilder(item: Item) {
    if (!builderItems.includes(item.id)) builderItems = [...builderItems, item.id];
    view = 'builder';
  }
  function exportData() {
    const blob = new Blob([JSON.stringify({ ...wardrobe, items: wardrobe.items.map((item) => ({ ...item, favorite: favorites.has(item.id) })) }, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob); const link = document.createElement('a'); link.href = url; link.download = 'mana-garderobe.json'; link.click(); URL.revokeObjectURL(url);
    toast = 'Garderobe lejupielādēta'; setTimeout(() => (toast = ''), 1800);
  }
</script>

<svelte:head><title>FitMap — mana garderobe</title><meta name="description" content="Vizuāla garderobes un tērpu plānošanas lietotne." /></svelte:head>

<div class="app-shell">
  <aside class="sidebar">
    <div class="brand"><span class="brand-mark">fm</span><span>fitmap</span></div>
    <p class="sidebar-intro">Tava garderobe,<br />sakārtota dzīvei.</p>
    <nav aria-label="Galvenā navigācija">
      <button class:active={view === 'wardrobe'} on:click={() => (view = 'wardrobe')}><span>01</span> Garderobe</button>
      <button class:active={view === 'outfits'} on:click={() => (view = 'outfits')}><span>02</span> Mani tērpi</button>
      <button class:active={view === 'builder'} on:click={() => (view = 'builder')}><span>03</span> Izveidot tērpu</button>
    </nav>
    <div class="sidebar-bottom"><button class="text-button" on:click={() => (showEditor = true)}>Rediģēt garderobi <span>↗</span></button></div>
  </aside>

  <main class="main-content">
    <header class="topbar"><div><span class="eyebrow">{view === 'wardrobe' ? 'Tava kolekcija' : view === 'outfits' ? 'Gatavi salikumi' : 'Saderības studija'}</span><h1>{view === 'wardrobe' ? 'Garderobe' : view === 'outfits' ? 'Mani tērpi' : 'Izveido tērpu'}</h1></div><button class="icon-button" aria-label="Meklēt">⌕</button></header>

    {#if view === 'wardrobe'}
      <section class="intro-row"><p class="lead">Apģērbi, kas tev jau ir.<br /><em>Idejas, ko ar tiem iesākt.</em></p><div class="stat"><strong>{wardrobe.items.length}</strong><span>gabali garderobē</span></div><div class="stat"><strong>{wardrobe.outfits.length}</strong><span>saglabāti tērpi</span></div></section>
      <div class="controls"><div class="search-wrap"><span>⌕</span><input bind:value={search} placeholder="Meklēt pēc nosaukuma vai krāsas" /></div><button class:active={favoritesOnly} class="filter-button" on:click={() => (favoritesOnly = !favoritesOnly)}>♡ Izlase</button></div>
      <div class="category-tabs">{#each Object.entries(categories) as [key, label]}<button class:active={category === key} on:click={() => (category = key)}>{label}</button>{/each}</div>
      <section class="item-grid" aria-label="Garderobes apģērbi">{#each visibleItems as item}<article class="item-card" on:click={() => (selected = item)} on:keydown={(event) => event.key === 'Enter' && (selected = item)} role="button" tabindex="0"><div class="item-image"><img src={item.image} alt={item.name} /><button class="favorite" class:chosen={favorites.has(item.id)} aria-label="Pievienot izlasei" on:click|stopPropagation={() => toggleFavorite(item)}>{favorites.has(item.id) ? '♥' : '♡'}</button><span class="category-label">{categories[item.category as keyof typeof categories]}</span></div><div class="item-info"><h2>{item.name}</h2><p>{item.colors.join(' · ')} <span>·</span> {item.material}</p></div></article>{/each}</section>
      {#if visibleItems.length === 0}<div class="empty"><strong>Šeit nekā nav.</strong><span>Izmēģini citu meklējumu vai noņem filtru.</span></div>{/if}
    {:else if view === 'outfits'}
      <section class="intro-row"><p class="lead">Tērpi, kas jau ir<br /><em>pierādījuši sevi.</em></p><div class="stat"><strong>{wardrobe.outfits.length}</strong><span>gatavi tērpi</span></div></section>
      <section class="outfit-grid">{#each wardrobe.outfits as outfit}<article class="outfit-card"><div class="outfit-images">{#each outfit.items as itemId}<img src={wardrobe.items.find((item) => item.id === itemId)?.image} alt="" />{/each}</div><div class="outfit-copy"><div><h2>{outfit.name}</h2><span class="rating">{'★'.repeat(outfit.rating)}{'☆'.repeat(3 - outfit.rating)}</span></div><p>{outfit.notes}</p><div class="outfit-tags">{#each outfit.occasions as occasion}<span>{occasion}</span>{/each}<span>{outfit.seasons[0]}</span></div></div></article>{/each}</section>
    {:else}
      <section class="builder-head"><div><p class="lead">Sāc ar vienu lietu.<br /><em>Es piemeklēšu pārējo.</em></p><div class="score"><span class="score-dot"></span><strong>{builderScoreLabel}</strong></div></div><button class="secondary-button" on:click={() => (builderItems = [])}>Notīrīt visu</button></section>
      <section class="builder-selected"><h2>Tavs salikums <span>{builderItems.length} izvēlēti</span></h2><div class="selected-row">{#if builderItems.length === 0}<div class="builder-empty">Izvēlies apģērbu zemāk, lai sāktu.</div>{:else}{#each builderItems as itemId}<div class="selected-piece"><img src={wardrobe.items.find((item) => item.id === itemId)?.image} alt="" /><button on:click={() => (builderItems = builderItems.filter((id) => id !== itemId))}>×</button><span>{wardrobe.items.find((item) => item.id === itemId)?.name}</span></div>{/each}{/if}</div></section>
      <section class="builder-choices"><h2>{builderItems.length ? 'Kas vēl piestāv' : 'Izvēlies pirmo gabalu'}</h2><div class="choice-grid">{#each builderMatches as match}<button class="choice-card" class:weak={builderItems.length > 0 && match.score < 2} on:click={() => addToBuilder(match.item)}><img src={match.item.image} alt="" /><span>{match.item.name}</span>{#if builderItems.length > 0}<small>{match.score === 3 ? 'Lieliski sader' : match.score === 2 ? 'Var pamēģināt' : 'Mazāk ieteicams'}</small>{/if}</button>{/each}</div></section>
    {/if}
  </main>

  {#if selected}<div class="overlay" role="presentation" on:click={() => (selected = null)}><aside class="detail-panel" on:click|stopPropagation><button class="close" aria-label="Aizvērt" on:click={() => (selected = null)}>×</button><img class="detail-image" src={selected.image} alt={selected.name} /><div class="detail-body"><div class="detail-title"><div><span class="eyebrow">{categories[selected.category as keyof typeof categories]}</span><h2>{selected.name}</h2></div><button class="detail-heart" class:chosen={favorites.has(selected.id)} on:click={toggleSelectedFavorite}>{favorites.has(selected.id) ? '♥' : '♡'}</button></div><p class="detail-notes">{selected.notes ?? 'Pievieno piezīmes rediģēšanas režīmā.'}</p><div class="tag-list">{#each [...selected.colors, selected.material, ...selected.occasions] as tag}<span>{tag}</span>{/each}</div><button class="primary-button" on:click={() => selected && addToBuilder(selected)}>Pievienot tērpa veidotājam <span>↗</span></button><h3>Labi sader ar</h3><div class="match-list">{#each selectedMatches.slice(0, 3) as match}{#if match.item}<button on:click={() => selectItem(match.item?.id ?? '')}><img src={match.item?.image} alt="" /><span>{match.item?.name}<small>{match.note ?? 'Viegli kombinēt'}</small></span><b>{'★'.repeat(match.score)}</b></button>{/if}{/each}</div></div></aside></div>{/if}
  {#if showEditor}<div class="overlay" role="presentation" on:click={() => (showEditor = false)}><div class="editor-modal" on:click|stopPropagation><button class="close" on:click={() => (showEditor = false)}>×</button><span class="eyebrow">Tava datu kopija</span><h2>Rediģēt garderobi</h2><p>Lejupielādē garderobes datu kopiju, lai to varētu saglabāt vai papildināt.</p><button class="primary-button" on:click={exportData}>Lejupielādēt JSON <span>↓</span></button></div></div>{/if}
  {#if toast}<div class="toast">{toast}</div>{/if}
</div>
