<script lang="ts">
  import ColorInput from './ColorInput.svelte';
  import { query_style } from '../common/ui-style';
  import type { ConfiguredPages, Preferences } from '../common/types';
  import {
    preferences,
    get_prefs,
    on_prefs_change,
    prefs_keys_with_defaults,
    set_pref,
  } from '../common/shared';
  import { methods } from '../methods/methods';
  import type { Browser } from 'webextension-polyfill';
  export let browser: Browser;

  query_style().catch((error) => console.error(error));

  let current_preferences: Preferences = [];
  let configured_pages: ConfiguredPages = {};
  async function update_prefs() {
    const saved_prefs = await get_prefs();
    current_preferences = preferences.map((pref) => ({
      ...pref,
      value: saved_prefs[pref.name],
    })) as Preferences;

    configured_pages = current_preferences.find(
      (p) => p.name === 'configured_pages',
    )!.value as ConfiguredPages;
  }
  update_prefs().catch((error) => console.error(error));
  on_prefs_change(() => {
    update_prefs().catch((error) => console.error(error));
    query_style().catch((error) => console.error(error));
  });
</script>

<main class="container-fluid">
  <div class="settings-grid">
    <!-- General Options Section -->
    <div class="settings-section">
      <div class="section-header">
        <h2>General Options</h2>
      </div>
      <div class="settings-content">
        {#each current_preferences as pref (pref.name)}
          {#if pref.type !== 'configured_pages' && (pref.type === 'bool' || pref.type === 'menulist')}
            <div class="setting-item">
              <div class="setting-label">
                <label for="labeled_pref_{pref.name}">{pref.title}</label>
              </div>
              <div class="setting-control">
                {#if pref.type === 'bool'}
                  <div class="control-with-action">
                    <input
                      bind:checked={pref.value}
                      on:change={(e) => set_pref(pref.name, e.currentTarget.checked)}
                      id="labeled_pref_{pref.name}"
                      type="checkbox"
                      data-pref-type={pref.type}
                    />
                    <button
                      on:click={() => set_pref(pref.name, prefs_keys_with_defaults[pref.name])}
                      class="btn btn-icon" title="Reset to default">
                      ↺
                    </button>
                  </div>
                {:else if pref.type === 'menulist'}
                  <div class="control-with-action">
                    <select
                      on:change={(e) => set_pref(pref.name, e.currentTarget.selectedIndex)}
                      id="labeled_pref_{pref.name}"
                      data-pref-type={pref.type}
                    >
                      {#each pref.options as option (option.value)}
                        <option selected={pref.value === parseInt(option.value, 10)}>{option.label}</option>
                      {/each}
                    </select>
                    <button
                      on:click={() => set_pref(pref.name, prefs_keys_with_defaults[pref.name])}
                      class="btn btn-icon" title="Reset to default">
                      ↺
                    </button>
                  </div>
                {/if}
              </div>
            </div>
          {/if}
        {/each}
      </div>
    </div>
    
    <!-- Color Settings Section -->
    <div class="settings-section">
      <div class="section-header">
        <h2>Color Settings</h2>
      </div>
      <div class="settings-content">
        {#each current_preferences as pref (pref.name)}
          {#if pref.type === 'color'}
            <div class="setting-item">
              <div class="setting-label">
                <label for="labeled_pref_{pref.name}">{pref.title}</label>
              </div>
              <div class="setting-control">
                <div class="color-control-group">
                  <ColorInput
                    value={pref.value}
                    default={pref.value}
                    on:change={(e) => set_pref(pref.name, e.detail.value)}
                  />
                  <div class="color-buttons">
                    <button
                      on:click={() => set_pref(pref.name, 'transparent')}
                      class="btn transparent-btn" title="Set transparent">
                      Clear
                    </button>
                    <button
                      on:click={() => set_pref(pref.name, prefs_keys_with_defaults[pref.name])}
                      class="btn btn-icon" title="Reset to default">
                      ↺
                    </button>
                  </div>
                </div>
              </div>
            </div>
          {/if}
        {/each}
      </div>
    </div>
    
    <!-- Keyboard Shortcuts Section -->
    {#await browser.runtime.getPlatformInfo() then platformInfo}
      {#if platformInfo.os !== 'android'}
        <div class="settings-section">
          <div class="section-header">
            <h2>Keyboard Shortcuts</h2>
          </div>
          <div class="settings-content">
            <div class="info-note">
              <div class="info-icon">ℹ️</div>
              <div class="info-content">
                <p>Configure shortcuts through Firefox's extension management:</p>
                <ol>
                  <li>Go to about:addons (Menu → Add-ons)</li>
                  <li>Click the cogwheel icon</li>
                  <li>Select "Manage Extension Shortcuts"</li>
                </ol>
                <a href="https://support.mozilla.org/kb/manage-extension-shortcuts-firefox">
                  Mozilla Support - Managing Extension Shortcuts
                </a>
              </div>
            </div>
          </div>
        </div>
      {/if}
    {/await}
    
    <!-- Configured Websites Section -->
    <div class="settings-section websites-section">
      <div class="section-header">
        <h2>Configured Websites</h2>
      </div>
      <div class="settings-content">
        {#each Object.entries(configured_pages) as [page, method_index] (page)}
          <div class="configured-page">
            <div class="site-info">
              <div class="configured-page-url">{page}</div>
              <div class="method-name">{methods[method_index].label}</div>
            </div>
            <div class="site-actions">
              <button
                on:click={() => set_pref(
                  'configured_pages',
                  Object.fromEntries(
                    Object.entries(configured_pages).filter(([k, _]) => k !== page)
                  )
                )}
                class="btn danger-button" title="Remove this configuration">
                <span class="btn-icon">×</span>
              </button>
            </div>
          </div>
        {:else}
          <div class="info-note">
            <div class="info-icon">ℹ️</div>
            <div class="info-content">
              <p>No websites have been configured yet. Add sites by clicking the extension icon in your browser toolbar.</p>
            </div>
          </div>
        {/each}
      </div>
    </div>
  </div>
</main>

<style>
  /* Two-column grid layout */
  .settings-grid {
    display: grid;
    grid-template-columns: 1fr;
    gap: 16px;
  }
  
  @media (min-width: 768px) {
    .settings-grid {
      grid-template-columns: repeat(2, 1fr);
    }
    
    .websites-section {
      grid-column: 1 / -1; /* Span all columns */
    }
  }
  
  /* Section styling */
  .settings-section {
    background-color: var(--ctp-mantle);
    border: 1px solid var(--ctp-surface-0);
    border-radius: var(--border-radius);
    overflow: hidden;
  }
  
  .section-header {
    padding: 12px 16px;
    background-color: var(--ctp-surface-0);
    border-bottom: 1px solid var(--ctp-surface-1);
  }
  
  .settings-content {
    padding: 12px 16px;
  }
  
  /* Section headers */
  h2 {
    font-size: 15px;
    font-weight: 600;
    color: var(--ctp-lavender);
    margin: 0;
    letter-spacing: 0.5px;
  }

  /* Setting items */
  .setting-item {
    margin-bottom: 12px;
    display: flex;
    flex-direction: column;
  }
  
  .setting-label {
    margin-bottom: 4px;
  }
  
  /* Control with reset button */
  .control-with-action {
    display: flex;
    align-items: center;
    gap: 8px;
  }
  
  .control-with-action select {
    flex: 1;
  }
  
  /* Icon button */
  .btn-icon {
    height: 28px;
    width: 28px;
    padding: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 16px;
    border-radius: 4px;
    background-color: var(--ctp-surface-0);
    color: var(--ctp-subtext-0);
  }
  
  .btn-icon:hover {
    background-color: var(--ctp-surface-1);
    color: var(--ctp-text);
  }
  
  /* Color controls */
  .color-control-group {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  
  .color-buttons {
    display: flex;
    gap: 8px;
  }
  
  .transparent-btn {
    flex: 1;
    height: 28px;
    padding: 0 8px;
    font-size: 12px;
  }
  
  /* Info note with icon */
  .info-note {
    display: flex;
    gap: 12px;
    background-color: rgba(137, 220, 235, 0.1);
    border-radius: 6px;
    padding: 12px;
  }
  
  .info-icon {
    font-size: 16px;
    flex-shrink: 0;
  }
  
  .info-content {
    font-size: 13px;
    color: var(--ctp-subtext-0);
  }
  
  .info-content a {
    color: var(--ctp-sky);
    display: inline-block;
    margin-top: 4px;
  }
  
  .info-content ol {
    padding-left: 20px;
    margin: 8px 0;
  }
  
  .info-content li {
    margin-bottom: 4px;
  }
  
  /* Configured websites */
  .configured-page {
    display: flex;
    justify-content: space-between;
    align-items: center;
    background-color: var(--ctp-surface-0);
    border: 1px solid var(--ctp-surface-1);
    border-radius: var(--border-radius);
    padding: 10px;
    margin-bottom: 8px;
    transition: background-color 0.15s;
  }
  
  .configured-page:hover {
    background-color: var(--ctp-surface-1);
  }
  
  .site-info {
    overflow: hidden;
    flex: 1;
  }
  
  .site-actions {
    flex-shrink: 0;
    margin-left: 8px;
  }
  
  .configured-page-url {
    word-break: break-all;
    font-family: 'SF Mono', SFMono-Regular, Consolas, 'Liberation Mono', Menlo, monospace;
    font-size: 12px;
    color: var(--ctp-sky);
    margin-bottom: 2px;
  }
  
  .method-name {
    font-size: 12px;
    color: var(--ctp-green);
  }
  
  .danger-button {
    height: 28px;
    width: 28px;
    padding: 0;
    display: flex;
    align-items: center;
    justify-content: center;
  }
  
  .danger-button .btn-icon {
    font-size: 18px;
    font-weight: bold;
    color: var(--ctp-red);
  }
  
  /* Better checkboxes and inputs */
  input[type="checkbox"] {
    height: 18px;
    width: 18px;
    border-radius: 3px;
  }
  
  select {
    height: 30px;
    padding: 0 28px 0 8px;
  }
  
  /* Label styling */
  label {
    display: inline-block;
    font-weight: 500;
    font-size: 13px;
    color: var(--ctp-subtext-0);
  }
</style>
