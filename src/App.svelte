<script lang="ts">
  import wardrobe from './data/wardrobe.json';

  type Item = {
    id: string;
    name: string;
    category: string;
    image: string;
    colors: string[];
    seasons: string[];
    occasions: string[];
    formality: number;
    warmth: number;
    pattern: string;
    material: string;
    notes?: string;
    favorite: boolean;
    rarelyUsed?: boolean;
  };
  type View = 'wardrobe' | 'outfits' | 'builder';
  type Outfit = { id: string; name: string; items: string[]; occasions: string[]; seasons: string[]; rating: number; notes?: string; variants?: string[][]; favorite?: boolean };
  type Compatibility = { items: string[]; score: number; contexts?: string[]; note?: string };
  const categories = { all: 'Viss', tops: 'Virsdaļas', bottoms: 'Apakšdaļas', outerwear: 'Virsslāņi', shoes: 'Apavi', socks: 'Zeķes', gloves: 'Cimdi', scarves: 'Šalles', hats: 'Cepures', bags: 'Somas' };
  const bodyTiers = [['hats', 'scarves'], ['outerwear', 'tops', 'gloves'], ['bottoms'], ['socks', 'shoes']];
  const carriedCategories = ['bags'];
  const defaultItems = wardrobe.items as Item[];

  function loadItems() {
    if (typeof localStorage === 'undefined') return defaultItems;
    try {
      const saved = localStorage.getItem('fitmap-items');
      return saved ? (JSON.parse(saved) as Item[]) : defaultItems;
    } catch {
      return defaultItems;
    }
  }
  function loadCompatibility(): Compatibility[] {
    if (typeof localStorage === 'undefined') return wardrobe.compatibility as Compatibility[];
    try {
      const saved = localStorage.getItem('fitmap-compatibility');
      return saved ? (JSON.parse(saved) as Compatibility[]) : wardrobe.compatibility as Compatibility[];
    } catch {
      return wardrobe.compatibility as Compatibility[];
    }
  }

  let view: View = 'wardrobe';
  let category = 'all';
  let search = '';
  let favoritesOnly = false;
  let seasonFilter = 'all';
  let occasionFilter = 'all';
  let warmthFilter = 'all';
  let showFilters = false;
  let wardrobeMode: 'grid' | 'matrix' | 'suggest' = 'grid';
  let suggestItemIds: string[] = [];
  let selected: Item | null = null;
  let builderItems: string[] = [];
  let showEditor = false;
  let editorMode: 'list' | 'form' = 'list';
  let compatibilityCategory = 'all';
  // The editor uses the truthiness of this value as its open/closed state.
  // Keep the runtime empty state while allowing the template's guarded editor
  // section to access the selected item without repeated nullable assertions.
  let editingItem: Item = null as unknown as Item;
  let draft: Item = emptyItem();
  let toast = '';
  let items = loadItems();
  let compatibility = loadCompatibility();
  let favorites = new Set(items.filter((item) => item.favorite).map((item) => item.id));
  let outfits: Outfit[] = loadOutfits();
  let showOutfitEditor = false;
  let editingOutfit: Outfit | null = null;
  let outfitDraft: Outfit = emptyOutfit();
  let outfitBuilderItems: string[] = [];
  let history: string[] = [];

  function emptyItem(): Item {
    return { id: '', name: '', category: 'tops', image: '', colors: [], seasons: ['Visu gadu'], occasions: ['Ikdiena'], formality: 3, warmth: 2, pattern: 'Vienkrāsains', material: '', notes: '', favorite: false, rarelyUsed: false };
  }
  function emptyOutfit(): Outfit { return { id: '', name: '', items: [], occasions: ['Ikdiena'], seasons: ['Visu gadu'], rating: 3, notes: '', variants: [], favorite: false }; }
  function loadOutfits() {
    if (typeof localStorage === 'undefined') return wardrobe.outfits as Outfit[];
    try { return JSON.parse(localStorage.getItem('fitmap-outfits') ?? JSON.stringify(wardrobe.outfits)) as Outfit[]; } catch { return wardrobe.outfits as Outfit[]; }
  }

  $: if (typeof localStorage !== 'undefined') localStorage.setItem('fitmap-items', JSON.stringify(items));
  $: if (typeof localStorage !== 'undefined') localStorage.setItem('fitmap-compatibility', JSON.stringify(compatibility));
  $: if (typeof localStorage !== 'undefined') localStorage.setItem('fitmap-outfits', JSON.stringify(outfits));

  $: visibleItems = rarelyUsedLast(items.filter((item) => {
    const matchesCategory = category === 'all' || item.category === category;
    const matchesSearch = item.name.toLocaleLowerCase().includes(search.toLocaleLowerCase()) || item.colors.join(' ').toLocaleLowerCase().includes(search.toLocaleLowerCase());
    const matchesSeason = seasonFilter === 'all' || item.seasons.includes(seasonFilter);
    const matchesOccasion = occasionFilter === 'all' || item.occasions.includes(occasionFilter);
    const matchesWarmth = warmthFilter === 'all' || (warmthFilter === 'warm' ? item.warmth >= 4 : item.warmth <= 2);
    return matchesCategory && matchesSearch && matchesSeason && matchesOccasion && matchesWarmth && (!favoritesOnly || favorites.has(item.id));
  }));
  $: selectedMatches = selectedMatchesFor(selected?.id);
  $: builderMatches = items.filter((item) => !builderItems.includes(item.id) && !builderItems.some((id) => items.find((candidate) => candidate.id === id)?.category === item.category)).map((item) => ({ item, rarelyUsed: item.rarelyUsed ?? false, ...matchDetails(item.id, builderItems) })).sort(compareMatches);
  $: builderScoreLabel = builderItems.length < 2 ? 'Izvēlies vēl vienu apģērba gabalu' : selectedCompatibilityLabel();
  $: suggestedItems = items.filter((item) => !suggestItemIds.includes(item.id)).map((item) => ({ item, rarelyUsed: item.rarelyUsed ?? false, ...matchDetails(item.id, suggestItemIds) })).sort(compareMatches);
  $: orderedItems = rarelyUsedLast(items);

  function rarelyUsedLast(list: Item[]) {
    const slots = new Map<string, number[]>();
    list.forEach((item, index) => slots.set(item.category, [...(slots.get(item.category) ?? []), index]));
    const ordered = [...list];
    for (const positions of slots.values()) {
      const group = positions.map((index) => list[index]).sort((first, second) => Number(first.rarelyUsed ?? false) - Number(second.rarelyUsed ?? false));
      positions.forEach((position, offset) => (ordered[position] = group[offset]));
    }
    return ordered;
  }
  function outfitPieces(itemIds: string[]) {
    return itemIds.map((id) => items.find((item) => item.id === id)).filter((item): item is Item => Boolean(item));
  }
  function bodyRows(itemIds: string[]) {
    const worn = outfitPieces(itemIds).filter((item) => !carriedCategories.includes(item.category));
    const rows = bodyTiers.map((tier) => rarelyUsedLast(worn.filter((item) => tier.includes(item.category)).sort((first, second) => tier.indexOf(first.category) - tier.indexOf(second.category))));
    const placed = new Set(rows.flat().map((item) => item.id));
    return [...rows, rarelyUsedLast(worn.filter((item) => !placed.has(item.id)))].filter((row) => row.length > 0);
  }
  function carriedPieces(itemIds: string[]) {
    return rarelyUsedLast(outfitPieces(itemIds).filter((item) => carriedCategories.includes(item.category)));
  }
  function bodyOrder(itemIds: string[]) {
    return [...bodyRows(itemIds).flat(), ...carriedPieces(itemIds)];
  }
  function builderPairScore(first: string, second: string) { return compatibility.find((pair) => pair.items.includes(first) && pair.items.includes(second))?.score ?? 0; }
  function matchDetails(id: string, selectedIds: string[]) {
    const scores = selectedIds.map((selectedId) => getCompatibility(selectedId, id)?.score).filter((score): score is number => score !== undefined);
    const hasBadMatch = scores.includes(0);
    const complete = selectedIds.length > 0 && scores.length === selectedIds.length;
    const score = scores.length ? scores.reduce((sum, value) => sum + value, 0) / scores.length : 0;
    const status = hasBadMatch ? 'bad' : complete ? 'confirmed' : scores.length ? 'partial' : 'unknown';
    return { score, knownCount: scores.length, status };
  }
  function compareMatches(first: { score: number; knownCount: number; status: string; rarelyUsed: boolean }, second: { score: number; knownCount: number; status: string; rarelyUsed: boolean }) {
    const rank = { confirmed: 3, partial: 2, unknown: 1, bad: 0 };
    return rank[second.status as keyof typeof rank] - rank[first.status as keyof typeof rank] || second.score - first.score || Number(first.rarelyUsed) - Number(second.rarelyUsed) || second.knownCount - first.knownCount;
  }
  function selectedCompatibilityLabel() {
    const scores: number[] = [];
    for (let first = 0; first < builderItems.length; first += 1) {
      for (let second = first + 1; second < builderItems.length; second += 1) {
        const score = getCompatibility(builderItems[first], builderItems[second])?.score;
        if (score !== undefined) scores.push(score);
      }
    }
    if (scores.includes(0)) return '0% saderība';
    if (!scores.length) return 'Nav novērtēts';
    const average = scores.reduce((sum, score) => sum + score, 0) / scores.length;
    const expectedPairs = builderItems.length * (builderItems.length - 1) / 2;
    const prefix = scores.length < expectedPairs ? 'Daļēji novērtēts · ' : '';
    return `${prefix}${Math.round(average / 3 * 100)}% saderība`;
  }
  function selectedMatchesFor(id: string | undefined) {
    if (!id) return [];
    return compatibility.filter((pair) => pair.items.includes(id) && pair.score > 0).map((pair) => ({ item: items.find((item) => item.id === pair.items.find((pairId) => pairId !== id)), score: pair.score, note: pair.note }));
  }
  function selectItem(id: string) {
    const item = items.find((candidate) => candidate.id === id);
    if (item) { selected = item; history = [id, ...history.filter((entry) => entry !== id)].slice(0, 8); }
  }
  function toggleFavorite(item: Item) {
    favorites.has(item.id) ? favorites.delete(item.id) : favorites.add(item.id);
    favorites = new Set(favorites);
    items = items.map((candidate) => candidate.id === item.id ? { ...candidate, favorite: favorites.has(candidate.id) } : candidate);
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
  function openEditor(item?: Item) {
    showEditor = true;
    if (item) editItem(item);
    else editorMode = 'list';
  }
  function editItem(item: Item) {
    editingItem = item;
    draft = { ...item, colors: [...item.colors], seasons: [...item.seasons], occasions: [...item.occasions] };
    editorMode = 'form';
  }
  function getCompatibility(first: string, second: string) {
    return compatibility.find((pair) => pair.items.includes(first) && pair.items.includes(second));
  }
  function compatibilityCount(item: Item) {
    return compatibility.filter((pair) => {
      const relatedId = pair.items.find((id) => id !== item.id);
      const relatedItem = items.find((candidate) => candidate.id === relatedId);
      return pair.items.includes(item.id) && relatedItem?.category !== item.category;
    }).length;
  }
  function setCompatibility(first: string, second: string, score: number) {
    const existing = getCompatibility(first, second);
    if (existing) {
      existing.score = score;
      compatibility = [...compatibility];
    }
    else compatibility = [...compatibility, { items: [first, second], score }];
    items = [...items];
  }
  function setCompatibilityNote(first: string, second: string, note: string) {
    const existing = getCompatibility(first, second);
    if (existing) {
      existing.note = note;
      compatibility = [...compatibility];
    }
    else compatibility = [...compatibility, { items: [first, second], score: 2, note }];
    items = [...items];
  }
  function clearCompatibility(first: string, second: string) {
    compatibility = compatibility.filter((pair) => !(pair.items.includes(first) && pair.items.includes(second)));
    items = [...items];
  }
  function newItem() {
    editingItem = null as unknown as Item;
    draft = emptyItem();
    editorMode = 'form';
  }
  function saveItem() {
    if (!draft.name.trim()) return;
    const saved = { ...draft, id: draft.id || `item-${Date.now()}`, name: draft.name.trim(), material: draft.material.trim() || 'Nav norādīts' };
    items = editingItem ? items.map((item) => item.id === saved.id ? saved : item) : [...items, saved];
    favorites = new Set(items.filter((item) => item.favorite).map((item) => item.id));
    editorMode = 'list';
    toast = 'Saglabāts';
    setTimeout(() => (toast = ''), 1800);
  }
  function removeItem(item: Item) {
    if (!confirm(`Dzēst “${item.name}”?`)) return;
    items = items.filter((candidate) => candidate.id !== item.id);
    favorites.delete(item.id);
    favorites = new Set(favorites);
  }
  function handlePhoto(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => (draft = { ...draft, image: String(reader.result) });
    reader.readAsDataURL(file);
  }
  function importData(event: Event) {
    const input = event.currentTarget as HTMLInputElement;
    const file = input.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const parsed = JSON.parse(String(reader.result));
        if (!Array.isArray(parsed.items)) throw new Error('invalid');
        items = parsed.items as Item[];
        if (Array.isArray(parsed.outfits)) outfits = parsed.outfits as Outfit[];
        if (Array.isArray(parsed.compatibility)) compatibility = parsed.compatibility as Compatibility[];
        favorites = new Set(items.filter((item) => item.favorite).map((item) => item.id));
        editorMode = 'list';
        toast = 'Garderobe importēta';
        setTimeout(() => (toast = ''), 1800);
      } catch {
        toast = 'Šo failu nevar importēt';
        setTimeout(() => (toast = ''), 1800);
      }
    };
    reader.readAsText(file);
  }
  function clearFilters() { seasonFilter = 'all'; occasionFilter = 'all'; warmthFilter = 'all'; search = ''; favoritesOnly = false; }
  function openSuggest(item?: Item) { suggestItemIds = item ? [item.id] : []; wardrobeMode = 'suggest'; }
  function toggleSuggestItem(item: Item) { suggestItemIds = suggestItemIds.includes(item.id) ? suggestItemIds.filter((id) => id !== item.id) : [...suggestItemIds, item.id].slice(-2); }
  function openOutfitEditor(outfit?: Outfit) {
    editingOutfit = outfit ?? null;
    outfitDraft = outfit ? { ...outfit, items: [...outfit.items], variants: outfit.variants?.map((variant) => [...variant]) } : emptyOutfit();
    outfitBuilderItems = outfit?.items ?? [];
    showOutfitEditor = true;
  }
  function saveOutfit() {
    if (!outfitDraft.name.trim() || outfitBuilderItems.length < 2) return;
    const saved = { ...outfitDraft, id: outfitDraft.id || `outfit-${Date.now()}`, name: outfitDraft.name.trim(), items: outfitBuilderItems };
    outfits = editingOutfit ? outfits.map((outfit) => outfit.id === saved.id ? saved : outfit) : [...outfits, saved];
    showOutfitEditor = false;
    toast = 'Tērps saglabāts'; setTimeout(() => (toast = ''), 1800);
  }
  function removeOutfit(outfit: Outfit) { if (confirm(`Dzēst “${outfit.name}”?`)) outfits = outfits.filter((candidate) => candidate.id !== outfit.id); }
  function addVariant() { if (outfitBuilderItems.length >= 2) outfitDraft = { ...outfitDraft, variants: [...(outfitDraft.variants ?? []), [...outfitBuilderItems]] }; }
  function exportData() {
    const blob = new Blob([JSON.stringify({ ...wardrobe, items: items.map((item) => ({ ...item, favorite: favorites.has(item.id) })), compatibility, outfits }, null, 2)], { type: 'application/json' });
    const url = URL.createObjectURL(blob); const link = document.createElement('a'); link.href = url; link.download = 'mana-garderobe.json'; link.click(); URL.revokeObjectURL(url);
    toast = 'Garderobe lejupielādēta'; setTimeout(() => (toast = ''), 1800);
  }
</script>

<svelte:head><title>FitMap — mana garderobe</title><meta name="description" content="Vizuāla garderobes un tērpu plānošanas lietotne." /></svelte:head>

<div class="app-shell">
  <main class="main-content">
    <div class="top-navigation">
      <div class="brand"><span class="brand-mark">fm</span><span>fitmap</span></div>
    <nav aria-label="Galvenā navigācija">
      <button class:active={view === 'wardrobe'} on:click={() => (view = 'wardrobe')}><span>01</span> Garderobe</button>
      <button class:active={view === 'outfits'} on:click={() => (view = 'outfits')}><span>02</span> Mani tērpi</button>
      <button class:active={view === 'builder'} on:click={() => (view = 'builder')}><span>03</span> Izveidot tērpu</button>
    </nav>
    </div>
    <header class="topbar"><div><span class="eyebrow">{view === 'wardrobe' ? 'Tava kolekcija' : view === 'outfits' ? 'Gatavi salikumi' : 'Saderības studija'}</span><h1>{view === 'wardrobe' ? 'Garderobe' : view === 'outfits' ? 'Mani tērpi' : 'Izveido tērpu'}</h1></div><button class="icon-button" aria-label="Meklēt">⌕</button></header>

    {#if view === 'wardrobe'}
      <section class="intro-row"><p class="lead">Apģērbi, kas tev jau ir.<br /><em>Idejas, ko ar tiem iesākt.</em></p><div class="stat"><strong>{items.length}</strong><span>gabali garderobē</span></div><div class="stat"><strong>{wardrobe.outfits.length}</strong><span>saglabāti tērpi</span></div><button class="primary-button compact-button edit-wardrobe-button" on:click={() => openEditor()}>Rediģēt garderobi <span>↗</span></button></section>
      <div class="controls"><div class="search-wrap"><span>⌕</span><input bind:value={search} placeholder="Meklēt pēc nosaukuma vai krāsas" /></div><button class:active={favoritesOnly} class="filter-button" on:click={() => (favoritesOnly = !favoritesOnly)}>♡ Izlase</button></div>
      <div class="category-tabs">{#each Object.entries(categories) as [key, label]}<button class:active={category === key} on:click={() => (category = key)}>{label}</button>{/each}</div>
      <div class="view-tools"><div class="view-tabs"><button class:active={wardrobeMode === 'grid'} on:click={() => (wardrobeMode = 'grid')}>Režģis</button><button class:active={wardrobeMode === 'suggest'} on:click={() => openSuggest()}>Ko vilkt?</button><button class:active={wardrobeMode === 'matrix'} on:click={() => (wardrobeMode = 'matrix')}>Matrica</button></div><button class="filter-button" class:active={showFilters} on:click={() => (showFilters = !showFilters)}>Filtri</button></div>
      {#if showFilters}<div class="filter-panel"><label>Sezona<select bind:value={seasonFilter}><option value="all">Visas sezonas</option>{#each ['Pavasaris', 'Vasara', 'Rudens', 'Ziema', 'Visu gadu'] as season}<option value={season}>{season}</option>{/each}</select></label><label>Gadījums<select bind:value={occasionFilter}><option value="all">Visi gadījumi</option>{#each ['Ikdiena', 'Darbs', 'Vakariņas', 'Ceļojums', 'Svinīgs'] as occasion}<option value={occasion}>{occasion}</option>{/each}</select></label><label>Siltums<select bind:value={warmthFilter}><option value="all">Jebkurš</option><option value="warm">Silts</option><option value="light">Viegls</option></select></label><button class="text-button" on:click={clearFilters}>Notīrīt filtrus</button></div>{/if}
      {#if wardrobeMode === 'suggest'}<section class="suggest-panel"><div class="suggest-copy"><span class="eyebrow">Ātra izvēle</span><h2>Ko vilkt kopā?</h2><p>Izvēlies vienu vai divus gabalus, un parādīšu garderobē saglabātās saderīgās izvēles.</p></div><div class="suggest-picks">{#each orderedItems as item}<button class:chosen={suggestItemIds.includes(item.id)} on:click={() => toggleSuggestItem(item)}><img src={item.image} alt="" /><span>{item.name}</span></button>{/each}</div><div class="suggest-results">{#if suggestItemIds.length === 0}<p class="builder-empty">Izvēlies apģērbu augstāk, lai redzētu ieteikumus.</p>{:else}{#each suggestedItems as match}<button class:unknown={match.status === 'unknown'} class:bad={match.status === 'bad'} on:click={() => selectItem(match.item.id)}><img src={match.item.image} alt="" /><span>{match.item.name}<small>{match.status === 'confirmed' ? 'Lieliski sader' : match.status === 'partial' ? 'Daļēji novērtēts' : match.status === 'bad' ? 'Neiesaku' : 'Nav novērtēts'}</small></span></button>{/each}{/if}</div></section>{:else if wardrobeMode === 'matrix'}<section class="matrix-wrap"><h2>Visu apģērbu saderība</h2><p class="matrix-help">Zvaigznes rāda ievadītu saderību. `?` nozīmē, ka šis salikums vēl nav novērtēts.</p><div class="matrix" style={`--columns: ${orderedItems.length}`}><div></div>{#each orderedItems as column}<div class="matrix-label">{column.name}</div>{/each}{#each orderedItems as row}<div class="matrix-label">{row.name}</div>{#each orderedItems as column}<button class:great={builderPairScore(row.id, column.id) === 3} class:okay={builderPairScore(row.id, column.id) === 2} class:unknown={!getCompatibility(row.id, column.id) && row.id !== column.id} disabled={row.id === column.id} aria-label={`${row.name} ar ${column.name}`} on:click={() => { if (row.id !== column.id) { suggestItemIds = [row.id, column.id]; wardrobeMode = 'suggest'; } }}>{row.id === column.id ? '' : getCompatibility(row.id, column.id) ? '★'.repeat(builderPairScore(row.id, column.id)) || '×' : '?'}</button>{/each}{/each}</div></section>{:else}<section class="item-grid" aria-label="Garderobes apģērbi">{#each visibleItems as item}<article class="item-card" on:click={() => selectItem(item.id)} on:keydown={(event) => event.key === 'Enter' && selectItem(item.id)} role="button" tabindex="0"><div class="item-image"><img src={item.image} alt={item.name} /><button class="favorite" class:chosen={favorites.has(item.id)} aria-label="Pievienot izlasei" on:click|stopPropagation={() => toggleFavorite(item)}>{favorites.has(item.id) ? '♥' : '♡'}</button><button class="item-edit" on:click|stopPropagation={() => openEditor(item)}>Rediģēt</button><span class="category-label">{categories[item.category as keyof typeof categories]}{#if item.rarelyUsed}&nbsp;· reti lietots{/if}</span></div><div class="item-info"><h2>{item.name}</h2><p>{item.colors.join(' · ')} <span>·</span> {item.material}</p></div></article>{/each}</section>{/if}
      {#if visibleItems.length === 0}<div class="empty"><strong>Šeit nekā nav.</strong><span>Izmēģini citu meklējumu vai noņem filtru.</span></div>{/if}
    {:else if view === 'outfits'}
      <section class="intro-row"><p class="lead">Tērpi, kas jau ir<br /><em>pierādījuši sevi.</em></p><div class="stat"><strong>{outfits.length}</strong><span>gatavi tērpi</span></div><button class="primary-button compact-button" on:click={() => openOutfitEditor()}>+ Jauns tērps</button></section>
      <section class="outfit-grid">{#each outfits as outfit}{@const carried = carriedPieces(outfit.items)}<article class="outfit-card"><div class="outfit-body"><div class="body-silhouette">{#each bodyRows(outfit.items) as row}<div class="body-tier">{#each row as piece}<img src={piece.image} alt={piece.name} title={piece.name} />{/each}</div>{/each}</div>{#if carried.length}<div class="body-carried">{#each carried as piece}<img src={piece.image} alt={piece.name} title={piece.name} />{/each}</div>{/if}</div><div class="outfit-copy"><div><h2>{outfit.name}</h2><span class="rating">{'★'.repeat(outfit.rating)}{'☆'.repeat(3 - outfit.rating)}</span></div><p>{outfit.notes}</p><div class="outfit-tags">{#each outfit.occasions as occasion}<span>{occasion}</span>{/each}<span>{outfit.seasons[0]}</span></div>{#if outfit.variants?.length}<div class="variant-list"><strong>Varianti ({outfit.variants.length})</strong>{#each outfit.variants as variant, index}<div class="variant-row"><div>{#each bodyOrder(variant) as piece}<img src={piece.image} alt={piece.name} title={piece.name} />{/each}</div><span>{index + 1}. variants</span></div>{/each}</div>{/if}<div class="outfit-card-actions"><button on:click={() => openOutfitEditor(outfit)}>Rediģēt</button><button on:click={() => removeOutfit(outfit)}>Dzēst</button></div></div></article>{/each}</section>
    {:else}
      <section class="builder-head"><div><p class="lead">Sāc ar vienu lietu.<br /><em>Es piemeklēšu pārējo.</em></p><div class="score"><span class="score-dot"></span><strong>{builderScoreLabel}</strong></div></div><button class="secondary-button" on:click={() => (builderItems = [])}>Notīrīt visu</button></section>
      <section class="builder-selected"><h2>Tavs salikums <span>{builderItems.length} izvēlēti</span></h2><div class="selected-row">{#if builderItems.length === 0}<div class="builder-empty">Izvēlies apģērbu zemāk, lai sāktu.</div>{:else}{#each bodyOrder(builderItems) as piece}<div class="selected-piece"><img src={piece.image} alt="" /><button on:click={() => (builderItems = builderItems.filter((id) => id !== piece.id))}>×</button><span>{piece.name}</span></div>{/each}{/if}</div></section>
      <section class="builder-choices"><h2>{builderItems.length ? 'Kas vēl piestāv' : 'Izvēlies pirmo gabalu'}</h2><div class="choice-grid">{#each builderMatches as match}<button class="choice-card" class:weak={match.status === 'bad'} class:unknown={match.status === 'unknown'} on:click={() => addToBuilder(match.item)}><img src={match.item.image} alt="" /><span>{match.item.name}</span>{#if builderItems.length > 0}<small>{match.status === 'confirmed' ? `${Math.round(match.score / 3 * 100)}% saderība` : match.status === 'partial' ? `${Math.round(match.score / 3 * 100)}% · Daļēji novērtēts` : match.status === 'bad' ? 'Neiesaku' : 'Nav novērtēts'}</small>{/if}</button>{/each}</div></section>
    {/if}
  </main>

  {#if selected}<div class="overlay" role="presentation" on:click={() => (selected = null)}><aside class="detail-panel" on:click|stopPropagation><button class="close" aria-label="Aizvērt" on:click={() => (selected = null)}>×</button><img class="detail-image" src={selected.image} alt={selected.name} /><div class="detail-body"><div class="detail-title"><div><span class="eyebrow">{categories[selected.category as keyof typeof categories]}</span><h2>{selected.name}</h2></div><button class="detail-heart" class:chosen={favorites.has(selected.id)} on:click={toggleSelectedFavorite}>{favorites.has(selected.id) ? '♥' : '♡'}</button></div><p class="detail-notes">{selected.notes ?? 'Pievieno piezīmes rediģēšanas režīmā.'}</p><div class="tag-list">{#each [...selected.colors, selected.material, ...selected.occasions] as tag}<span>{tag}</span>{/each}</div><button class="primary-button" on:click={() => selected && addToBuilder(selected)}>Pievienot tērpa veidotājam <span>↗</span></button><h3>Labi sader ar</h3><div class="match-list">{#each selectedMatches.slice(0, 3) as match}{#if match.item}<button on:click={() => selectItem(match.item?.id ?? '')}><img src={match.item?.image} alt="" /><span>{match.item?.name}<small>{match.note ?? 'Viegli kombinēt'}</small></span><b>{'★'.repeat(match.score)}</b></button>{/if}{/each}</div></div></aside></div>{/if}
  {#if showOutfitEditor}<div class="overlay" role="presentation" on:click={() => (showOutfitEditor = false)}><div class="editor-modal outfit-editor" on:click|stopPropagation><button class="close" aria-label="Aizvērt" on:click={() => (showOutfitEditor = false)}>×</button><div class="editor-heading"><div><span class="eyebrow">Tērpu kolekcija</span><h2>{editingOutfit ? 'Rediģēt tērpu' : 'Jauns tērps'}</h2></div></div><div class="editor-form"><label>Nosaukums<input bind:value={outfitDraft.name} placeholder="Piemēram, piektdienas vakars" /></label><label>Gadījums<select bind:value={outfitDraft.occasions[0]}>{#each ['Ikdiena', 'Darbs', 'Vakariņas', 'Ceļojums', 'Svinīgs'] as occasion}<option value={occasion}>{occasion}</option>{/each}</select></label><label>Sezona<select bind:value={outfitDraft.seasons[0]}>{#each ['Visu gadu', 'Pavasaris', 'Vasara', 'Rudens', 'Ziema'] as season}<option value={season}>{season}</option>{/each}</select></label><label>Novērtējums<select bind:value={outfitDraft.rating}><option value={3}>Lielisks</option><option value={2}>Labs</option><option value={1}>Ārkārtas variants</option></select></label><label class="wide-field">Piezīmes<textarea bind:value={outfitDraft.notes} rows="3" placeholder="Kas padara šo tērpu īpašu?"></textarea></label></div><div class="editor-section-heading"><div><h3 class="editor-section-title">Izvēlies gabalus</h3><p class="editor-hint">Pieskaries attēlam, lai pievienotu vai noņemtu apģērbu.</p></div><strong class="selection-count">{outfitBuilderItems.length} izvēlēti</strong></div><div class="outfit-picker">{#each orderedItems as item}<button class:chosen={outfitBuilderItems.includes(item.id)} aria-pressed={outfitBuilderItems.includes(item.id)} on:click={() => outfitBuilderItems = outfitBuilderItems.includes(item.id) ? outfitBuilderItems.filter((id) => id !== item.id) : [...outfitBuilderItems, item.id]}><span class="picker-image"><img src={item.image} alt="" />{#if outfitBuilderItems.includes(item.id)}<span class="selected-badge">✓ Izvēlēts</span>{/if}</span><span>{item.name}</span></button>{/each}</div><div class="variant-bar"><div><strong>Varianti</strong><p>Variants ir cita šī tērpa versija, piemēram, ar citu jaku vai apaviem.</p></div><button class="secondary-button" disabled={outfitBuilderItems.length < 2} on:click={addVariant}>Saglabāt pašreizējo kā variantu</button><span>{outfitDraft.variants?.length ?? 0} saglabāti</span></div><div class="form-actions"><button class="secondary-button" on:click={() => (showOutfitEditor = false)}>Atcelt</button><button class="primary-button compact-button" on:click={saveOutfit}>Saglabāt tērpu</button></div></div></div>{/if}
  {#if showEditor}<div class="overlay" role="presentation" on:click={() => (showEditor = false)}><div class="editor-modal wardrobe-editor" on:click|stopPropagation><button class="close" aria-label="Aizvērt" on:click={() => (showEditor = false)}>×</button>{#if editorMode === 'list'}<div class="editor-heading"><div><span class="eyebrow">Tava kolekcija</span><h2>Rediģēt garderobi</h2></div><button class="primary-button compact-button" on:click={newItem}>+ Pievienot apģērbu</button></div><p>Pievieno jaunus apģērbus, maini informāciju vai nomaini foto.</p><div class="editor-actions"><label class="secondary-button file-button">Importēt garderobi<input type="file" accept="application/json" on:change={importData} /></label><button class="secondary-button" on:click={exportData}>Eksportēt garderobi ↓</button></div><div class="editor-list">{#each orderedItems as item}<div class="editor-row">{#if item.image}<img src={item.image} alt="" />{:else}<div class="editor-placeholder">?</div>{/if}<div><strong>{item.name}</strong><span>{categories[item.category as keyof typeof categories]} · {item.material}{#if item.rarelyUsed}&nbsp;· reti lietots{/if}</span></div><button aria-label="Rediģēt" on:click={() => editItem(item)}>Rediģēt</button><button class="delete-button" aria-label="Dzēst" on:click={() => removeItem(item)}>×</button></div>{/each}</div>{:else}<div class="editor-heading"><div><span class="eyebrow">{editingItem ? 'Mainīt apģērbu' : 'Jauns apģērbs'}</span><h2>{editingItem ? 'Rediģēt apģērbu' : 'Pievienot apģērbu'}</h2></div></div><div class="editor-form"><label>Nosaukums<input bind:value={draft.name} placeholder="Piemēram, balta lina blūze" /></label><label>Kategorija<select bind:value={draft.category}>{#each Object.entries(categories).filter(([key]) => key !== 'all') as [key, label]}<option value={key}>{label}</option>{/each}</select></label><label>Materiāls<input bind:value={draft.material} placeholder="Piemēram, lins" /></label><label>Krāsa<input value={draft.colors.join(', ')} on:input={(event) => (draft = { ...draft, colors: (event.currentTarget as HTMLInputElement).value.split(',').map((value) => value.trim()).filter(Boolean) })} placeholder="Bēšs, balts" /></label><label>Piezīmes<textarea bind:value={draft.notes} rows="3" placeholder="Kas jāatceras par šo apģērbu?"></textarea></label><label class="checkbox-field wide-field"><input type="checkbox" bind:checked={draft.rarelyUsed} /><span><strong>Reti lietots</strong>Šos gabalus rādīšu sarakstu beigās, starp līdzīgiem apģērbiem.</span></label><label class="photo-upload">{#if draft.image}<img src={draft.image} alt="Izvēlētais foto" />{:else}<span>Izvēlies foto no telefona vai datora</span>{/if}<strong>{draft.image ? 'Nomainīt foto' : 'Pievienot foto'}</strong><input type="file" accept="image/*" on:change={handlePhoto} /></label></div>{#if editingItem}<section class="compatibility-editor"><div class="compatibility-heading"><div><h3>Saderība</h3><p>Šeit ieraksti, ko ChatGPT iesaka kopā ar šo apģērbu.</p></div><div class="compatibility-controls"><label>Rādīt<select bind:value={compatibilityCategory}><option value="all">Visi pārējie</option>{#each Object.entries(categories).filter(([key]) => key !== 'all' && key !== editingItem.category) as [key, label]}<option value={key}>{label}</option>{/each}</select></label><span>{compatibilityCount(editingItem)} ieraksti</span></div></div>{#each orderedItems.filter((item) => item.id !== editingItem?.id && item.category !== editingItem?.category && (compatibilityCategory === 'all' || item.category === compatibilityCategory)) as relatedItem}{@const relation = getCompatibility(editingItem.id, relatedItem.id)}<div class="compatibility-row"><div class="compatibility-item"><img src={relatedItem.image} alt="" /><strong>{relatedItem.name}</strong></div><div class="compatibility-options"><button class:active={relation?.score === 3} on:click={() => setCompatibility(editingItem.id, relatedItem.id, 3)}>Labi</button><button class:active={relation?.score === 2} on:click={() => setCompatibility(editingItem.id, relatedItem.id, 2)}>Var mēģināt</button><button class:active={relation?.score === 0} on:click={() => setCompatibility(editingItem.id, relatedItem.id, 0)}>Neiesaku</button><button class="clear-compatibility" aria-label="Noņemt saderības ierakstu" on:click={() => clearCompatibility(editingItem.id, relatedItem.id)}>×</button></div><input class="compatibility-note" value={relation?.note ?? ''} on:input={(event) => setCompatibilityNote(editingItem.id, relatedItem.id, (event.currentTarget as HTMLInputElement).value)} placeholder="Piezīme, piemēram, “labs krāsu līdzsvars”" /></div>{/each}</section>{/if}<div class="form-actions"><button class="secondary-button" on:click={() => (editorMode = 'list')}>Atcelt</button><button class="primary-button compact-button" on:click={saveItem}>Saglabāt</button></div>{/if}</div></div>{/if}
  {#if toast}<div class="toast">{toast}</div>{/if}
</div>
