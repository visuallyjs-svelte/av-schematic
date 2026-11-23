<script lang="ts">
  import { onMount } from 'svelte';
  import { Editor } from './editor';
  import type { LibraryDevice } from './devices';
  import type { DeviceNodeData, DevicePort, PortDirection } from './definitions';

  let root: HTMLDivElement;
  let revision = $state(0);
  const refresh = () => { revision++; };
  const controller = new Editor(refresh);
  let editor = $derived.by(() => {
    // A new wrapper invalidates template reads while keeping the model outside Svelte proxies.
    void revision;
    return new Proxy(controller, {
      get(target, key) {
        const value = Reflect.get(target, key);
        return typeof value === 'function' ? value.bind(target) : value;
      },
    });
  });

  function mount(root: ParentNode, editor: Editor) {
    const part = (name: string) => root.querySelector<HTMLElement>(`[data-avs-part="${name}"]`)!;
    editor.mount(part('surface'), part('palette'), part('controls'), part('miniview'));
    return () => editor.destroy();
  }
  onMount(() => mount(root, controller));
</script>

{#snippet libraryItem(item: LibraryDevice, segments?: {text: string; match: boolean}[])}
  <button class={["vjs-avs-library-item", editor.dark && "dark"].filter(Boolean).join(' ')} type="button" data-vjs-type="device" data-library-id={item.libraryId} onclick={() => { editor.editTemplate(item.libraryId); refresh(); }}>
    <span class="vjs-avs-library-item-text">
      <strong>
        {#if segments}
          {#each segments as segment, index (index)}
            {#if segment.match}
              <mark>
                {segment.text}
              </mark>
            {:else}
              <span>
                {segment.text}
              </span>
            {/if}
          {/each}
        {:else}
           {item.template.manufacturer} {item.template.model} 
        {/if}
      </strong>
      <small>
        {item.template.category}
      </small>
    </span>
    <svg class="icon drag-grip" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true">
      <circle cx="6" cy="4.5" r="1" />
      <circle cx="10" cy="4.5" r="1" />
      <circle cx="6" cy="8" r="1" />
      <circle cx="10" cy="8" r="1" />
      <circle cx="6" cy="11.5" r="1" />
      <circle cx="10" cy="11.5" r="1" />
    </svg>
  </button>
{/snippet}

{#snippet deviceForm(item: DeviceNodeData, template: boolean)}
  <div class="field-group">
    {#if !template}
      <label>
        Device ID 
        <input value={item.deviceId ?? ''} oninput={(event: Event) => { editor.editDevice(item, {deviceId: (event.currentTarget as HTMLInputElement).value}); refresh(); }} />
      </label>
    {/if}
    <label>
      Manufacturer 
      <input value={item.manufacturer ?? ''} oninput={(event: Event) => { editor.editDevice(item, {manufacturer: (event.currentTarget as HTMLInputElement).value}); refresh(); }} />
    </label>
    <label>
      Model 
      <input value={item.model ?? ''} oninput={(event: Event) => { editor.editDevice(item, {model: (event.currentTarget as HTMLInputElement).value}); refresh(); }} />
    </label>
    <label>
      Category 
      <input list="device-categories" value={item.category ?? ''} oninput={(event: Event) => { editor.editDevice(item, {category: (event.currentTarget as HTMLInputElement).value}); refresh(); }} />
    </label>
    {#if !template}
      <label>
        Location 
        <input value={item.location ?? ''} oninput={(event: Event) => { editor.editDevice(item, {location: (event.currentTarget as HTMLInputElement).value}); refresh(); }} />
      </label>
    {/if}
  </div>
  <h3 class="section-heading">
    Ports
  </h3>
  {#if template}
    {@render portList(item, item.ports)}
    <button class="btn-full" type="button" onclick={() => { editor.addPort(item); refresh(); }}>
      <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" aria-hidden="true">
        <path d="M8 3.5v9M3.5 8h9" />
      </svg>
       Add port 
    </button>
  {:else}
    <div class="port-tabs" role="tablist" tabindex="-1" aria-label="Port direction" onkeydown={(event: KeyboardEvent) => { editor.onPortTabKey(event); refresh(); }}>
      {#each editor.portTabs as direction (direction)}
        <button type="button" role="tab" id={'port-tab-' + direction} aria-selected={editor.activePortTab === direction} tabindex={editor.activePortTab === direction ? 0 : -1} onclick={() => { editor.activePortTab = direction; refresh(); }}>
          <span>
            {direction === 'input' ? 'Input ports' : 'Output ports'}
          </span>
          <span class="tab-count">
            {editor.portsFor(item, direction).length}
          </span>
        </button>
      {/each}
    </div>
    {#each editor.portTabs as direction (direction)}
      <div role="tabpanel" id={'port-panel-' + direction} aria-labelledby={'port-tab-' + direction} hidden={editor.activePortTab !== direction}>
        {#if editor.activePortTab === direction}
          {@render portList(item, editor.portsFor(item, direction), direction)}
        {/if}
      </div>
    {/each}
  {/if}
{/snippet}

{#snippet portList(item: DeviceNodeData, ports: DevicePort[], direction?: PortDirection)}
  <div class="port-cards">
    {#each ports as port, i (port.id)}
      {@const last = i === ports.length - 1}
      <div class="port-card">
        <div class="port-row-top">
          <input aria-label="Port label" placeholder="Port label" value={port.label ?? ''} oninput={(event: Event) => { editor.editPort(item, port, {label: (event.currentTarget as HTMLInputElement).value}); refresh(); }} />
          <select aria-label="Port direction" value={port.direction ?? ''} onchange={(event: Event) => { editor.editPort(item, port, {direction: (event.currentTarget as HTMLSelectElement).value as PortDirection}); refresh(); }}>
            <option value="input">
              Input
            </option>
            <option value="output">
              Output
            </option>
          </select>
        </div>
        <div class="port-row-bottom">
          <input list="connector-types" aria-label="Connector type" placeholder="Connector" value={port.connectorType ?? ''} oninput={(event: Event) => { editor.editPort(item, port, {connectorType: (event.currentTarget as HTMLInputElement).value}); refresh(); }} />
          <div class="port-actions">
            <button class="btn-icon" type="button" aria-label="Move port up" title="Move up" disabled={i === 0} onclick={() => { editor.movePort(item, port, -1, direction); refresh(); }}>
              <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                <path d="m4 10 4-4 4 4" />
              </svg>
            </button>
            <button class="btn-icon" type="button" aria-label="Move port down" title="Move down" disabled={last} onclick={() => { editor.movePort(item, port, 1, direction); refresh(); }}>
              <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                <path d="m4 6 4 4 4-4" />
              </svg>
            </button>
            <button class="btn-icon btn-danger-ghost" type="button" aria-label="Remove port" title="Remove port" onclick={() => { editor.removePort(item, port); refresh(); }}>
              <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                <path d="M2.5 4.5h11M6 4.5V3a1 1 0 0 1 1-1h2a1 1 0 0 1 1 1v1.5M4.5 4.5l.7 8.5a1 1 0 0 0 1 .9h3.6a1 1 0 0 0 1-.9l.7-8.5" />
              </svg>
            </button>
          </div>
        </div>
      </div>
    {:else}
      <p class="empty-state">
        {direction ? 'No ' + direction + ' ports' : 'No ports'}
      </p>
    {/each}
  </div>
{/snippet}

<div bind:this={root}>
  <main class={["vjs-avs-container", editor.dark && "dark"].filter(Boolean).join(' ')}>
    <header class="vjs-avs-toolbar">
      <h1>
        AV Schematic
      </h1>
      <button type="button" aria-expanded={editor.libraryOpen} onclick={() => { editor.libraryOpen = !editor.libraryOpen; refresh(); }}>
        <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <rect x="2" y="2.5" width="12" height="11" rx="1.5" />
          <path d="M6 2.5v11" />
        </svg>
         Library 
      </button>
      <button type="button" aria-expanded={editor.propertiesOpen} onclick={() => { editor.propertiesOpen = !editor.propertiesOpen; refresh(); }}>
        <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <rect x="2" y="2.5" width="12" height="11" rx="1.5" />
          <path d="M10 2.5v11" />
        </svg>
         Properties 
      </button>
      <button type="button" onclick={() => { editor.toggleTheme(); refresh(); }}>
        {#if editor.dark}
          <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <circle cx="8" cy="8" r="3" />
            <path d="M8 1.5v1.5M8 13v1.5M1.5 8H3M13 8h1.5M3.4 3.4l1.1 1.1M11.5 11.5l1.1 1.1M12.6 3.4l-1.1 1.1M4.5 11.5l-1.1 1.1" />
          </svg>
           Light theme 
        {:else}
          <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
            <path d="M13.5 9.8A5.5 5.5 0 0 1 6.2 2.5a5.5 5.5 0 1 0 7.3 7.3Z" />
          </svg>
           Dark theme 
        {/if}
      </button>
      <button type="button" onclick={() => { editor.downloadDxf(); refresh(); }}>
        <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
          <path d="M9.5 2H4a1.5 1.5 0 0 0-1.5 1.5v9A1.5 1.5 0 0 0 4 14h8a1.5 1.5 0 0 0 1.5-1.5V6L9.5 2Z" />
          <path d="M9.5 2v4h4" />
          <path d="M8 8.5v3.5m0 0-1.5-1.5M8 12l1.5-1.5" />
        </svg>
         Export DXF 
      </button>
    </header>
    <div class={["vjs-avs-workspace", !editor.libraryOpen && "vjs-avs-library-closed", !editor.propertiesOpen && "vjs-avs-properties-closed"].filter(Boolean).join(' ')}>
      <aside class="vjs-avs-pane vjs-avs-library-pane" hidden={!editor.libraryOpen}>
        <div class="vjs-avs-pane-header">
          <h2>
            Device library
          </h2>
          <div class="search-field">
            <svg class="icon search-icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
              <circle cx="7" cy="7" r="4.5" />
              <path d="m10.5 10.5 3 3" />
            </svg>
            <input type="search" placeholder="Search devices" aria-label="Search devices" value={editor.query ?? ''} oninput={(event: Event) => { editor.query = (event.currentTarget as HTMLInputElement).value; refresh(); }} />
          </div>
          <button class="btn-primary btn-full" type="button" onclick={() => { editor.editTemplate(); refresh(); }}>
            <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" aria-hidden="true">
              <path d="M8 3.5v9M3.5 8h9" />
            </svg>
             Add template 
          </button>
        </div>
        <div class="vjs-avs-pane-body" data-avs-part="palette">
          {#if editor.draft}
            <h3 class="section-heading">
              {editor.draftId ? 'Edit template' : 'New template'}
            </h3>
            {@render deviceForm(editor.draft!, true)}
          {:else if editor.searchQuery}
            <div class="vjs-avs-library-items">
              {#each editor.filteredLibrary as result (result.item.libraryId)}
                {@render libraryItem(result.item, result.segments)}
              {:else}
                <p class="empty-state">
                  No matching devices
                </p>
              {/each}
            </div>
          {:else}
            {#each editor.libraryGroups as group (group.key)}
              <details class="library-group" open={editor.isCategoryOpen(group.key)} ontoggle={(event: Event) => editor.setCategoryOpen(group.key, (event.currentTarget as HTMLDetailsElement).open)}>
                <summary>
                  <span class="library-label">
                    {group.label}
                  </span>
                  <span class="library-count">
                    {group.items.length}
                  </span>
                </summary>
                <div class="vjs-avs-library-items">
                  {#each group.items as item (item.libraryId)}
                    {@render libraryItem(item)}
                  {/each}
                </div>
              </details>
            {:else}
              <p class="empty-state">
                No devices in library
              </p>
            {/each}
          {/if}
        </div>
        {#if editor.draft}
          <div class="vjs-avs-pane-footer">
            <button class="btn-primary" type="button" onclick={() => { editor.saveTemplate(); refresh(); }}>
              <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                <path d="M3.5 8.5 6.5 11.5 12.5 4.5" />
              </svg>
               Save 
            </button>
            <button type="button" onclick={() => { editor.draft = null; refresh(); }}>
              <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" aria-hidden="true">
                <path d="M4 4l8 8M12 4l-8 8" />
              </svg>
               Discard 
            </button>
            {#if editor.draftId}
              <button class="btn-danger" type="button" aria-label="Delete template" title="Delete template" onclick={() => { editor.deleteTemplate(); refresh(); }}>
                <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                  <path d="M2.5 4.5h11M6 4.5V3a1 1 0 0 1 1-1h2a1 1 0 0 1 1 1v1.5M4.5 4.5l.7 8.5a1 1 0 0 0 1 .9h3.6a1 1 0 0 0 1-.9l.7-8.5" />
                </svg>
              </button>
            {/if}
          </div>
        {/if}
      </aside>
      <section class="vjs-avs-canvas">
        <div class="avs-surface" data-avs-part="surface"></div>
        <div data-avs-part="controls"></div>
        <div class="avs-miniview">
          <div data-avs-part="miniview"></div>
        </div>
      </section>
      <aside class="vjs-avs-pane vjs-avs-properties-pane" hidden={!editor.propertiesOpen}>
        <div class="vjs-avs-pane-header">
          <h2>
            Properties
          </h2>
        </div>
        <div class="avs-inspector">
          {#if editor.selectedDevice || editor.selectedCable}
            <div class="vjs-avs-pane-body">
              {#if editor.selectedDevice}
                {@render deviceForm(editor.selectedDevice!, false)}
              {:else if editor.selectedCable}
                <h3 class="section-heading">
                  Cable
                </h3>
                <div class="field-group">
                  <label>
                    Cable ID 
                    <input value={editor.selectedCable!.cableId ?? ''} oninput={(event: Event) => { editor.editCable({cableId: (event.currentTarget as HTMLInputElement).value}); refresh(); }} />
                  </label>
                  <label>
                    Signal type 
                    <select value={editor.selectedCable!.cableType ?? ''} onchange={(event: Event) => { editor.editCable({cableType: (event.currentTarget as HTMLSelectElement).value || undefined}); refresh(); }}>
                      <option value="">
                        Unspecified
                      </option>
                      {#each editor.signalTypes as type (type)}
                        <option value={type}>
                          {type}
                        </option>
                      {/each}
                    </select>
                  </label>
                </div>
              {/if}
            </div>
            {#if editor.selectedDevice}
              <div class="vjs-avs-pane-footer">
                <button class="btn-primary" type="button" onclick={() => { editor.addPort(editor.selectedDevice!, editor.activePortTab); refresh(); }}>
                  <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6" stroke-linecap="round" aria-hidden="true">
                    <path d="M8 3.5v9M3.5 8h9" />
                  </svg>
                   Add {editor.activePortTab} port 
                </button>
                <button class="btn-danger" type="button" onclick={() => { editor.deleteDevice(); refresh(); }}>
                  <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                    <path d="M2.5 4.5h11M6 4.5V3a1 1 0 0 1 1-1h2a1 1 0 0 1 1 1v1.5M4.5 4.5l.7 8.5a1 1 0 0 0 1 .9h3.6a1 1 0 0 0 1-.9l.7-8.5" />
                  </svg>
                   Delete device 
                </button>
              </div>
            {:else if editor.selectedCable}
              <div class="vjs-avs-pane-footer">
                <button type="button" onclick={() => { editor.resetRouting(); refresh(); }}>
                  <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                    <path d="M3 7a5 5 0 1 1 1.5 3.6M3 3.5V7h3.5" />
                  </svg>
                   Reset routing 
                </button>
                <button class="btn-danger" type="button" onclick={() => { editor.deleteCable(); refresh(); }}>
                  <svg class="icon" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round" aria-hidden="true">
                    <path d="M2.5 4.5h11M6 4.5V3a1 1 0 0 1 1-1h2a1 1 0 0 1 1 1v1.5M4.5 4.5l.7 8.5a1 1 0 0 0 1 .9h3.6a1 1 0 0 0 1-.9l.7-8.5" />
                  </svg>
                   Delete cable 
                </button>
              </div>
            {/if}
          {:else}
            <div class="vjs-avs-pane-body">
              <p class="empty-state">
                Select a device or cable on the canvas to inspect and edit its properties.
              </p>
            </div>
          {/if}
        </div>
      </aside>
    </div>
  </main>
  <datalist id="device-categories">
    {#each editor.deviceTypes as type (type.name)}
      <option value={type.name}></option>
    {/each}
  </datalist>
  <datalist id="connector-types">
    {#each editor.connectors as type (type)}
      <option value={type}></option>
    {/each}
  </datalist>
</div>
