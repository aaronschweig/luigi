<script>
  import { beforeUpdate, createEventDispatcher, onMount } from 'svelte';
  import { LuigiI18N } from '../core-api';
  import { GenericHelpers } from '../utilities/helpers';
  import { Routing } from '../services/routing';
  import {
    KEYCODE_ARROW_UP,
    KEYCODE_ARROW_DOWN,
    KEYCODE_ENTER,
    KEYCODE_ESC
  } from '../utilities/keycode.js';
  export let isSearchFieldVisible;
  export let searchResult = [];
  export let displaySearchResult;
  export let displayCustomSearchResult;
  export let inputElem;
  export let luigiCustomSearchRenderer__slot;
  export let luigiCustomSearchItemRenderer__slotContainer;
  export let globalSearchConfig;
  const dispatch = createEventDispatcher();
  const searchApiObj = {
    fireItemSelected: item => {
      search.searchProvider.onSearchResultItemSelected(item);
    }
  };
  let search = {};
  let isCustomSearchRenderer;
  let isCustomSearchResultItemRenderer;

  onMount(async () => {
    search = globalSearchConfig;
    let inputElement = inputElem;
    const placeHolder = getSearchPlaceholder(search.searchProvider);
    if (placeHolder) {
      inputElement.placeholder = placeHolder;
    }
    getCustomRenderer();
  });

  beforeUpdate(() => {
    search = globalSearchConfig;
    getCustomRenderer();
  });

  function getCustomRenderer() {
    if(!search.searchProvider)return;
    isCustomSearchRenderer = GenericHelpers.isFunction(
      search.searchProvider.customSearchResultRenderer
    );
    isCustomSearchResultItemRenderer = GenericHelpers.isFunction(
      search.searchProvider.customSearchResultItemRenderer
    );
  }

  function getSearchPlaceholder(searchProvider) {
    if (!searchProvider || !searchProvider.inputPlaceholder) {
      return undefined;
    }
    const currentLocale = LuigiI18N.getCurrentLocale();
    if (GenericHelpers.isFunction(searchProvider.inputPlaceholder)) {
      return searchProvider.inputPlaceholder();
    }
    if (typeof searchProvider.inputPlaceholder === 'string') {
      const translated = LuigiI18N.getTranslation(searchProvider.inputPlaceholder);
      if (!!translated && translated.trim().length > 0) {
        return translated;
      }
      return searchProvider.inputPlaceholder;
    }
    if (typeof searchProvider.inputPlaceholder === 'object') {
      return searchProvider.inputPlaceholder[currentLocale];
    }
  }

  function renderCustomSearchItem(item, slotContainer, index) {
    setTimeout(() => {
      search.searchProvider.customSearchResultItemRenderer(
        item,
        slotContainer.children[index],
        searchApiObj
      );
    });
    return '';
  }

  function closeSearchResult() {
    dispatch('closeSearchResult');
  }

  function onKeyUp({ keyCode }) {
    if (search && search.searchProvider) {
      if (
        GenericHelpers.isFunction(search.searchProvider.onEnter) &&
        keyCode === KEYCODE_ENTER
      ) {
        search.searchProvider.onEnter();
      } else if (
        GenericHelpers.isFunction(search.searchProvider.onEscape) &&
        keyCode === KEYCODE_ESC
      ) {
        search.searchProvider.onEscape();
      } else if (keyCode === KEYCODE_ARROW_DOWN) {
        if (displaySearchResult) {
          document
            .querySelector('.luigi-search-result-item__0')
            .childNodes[0].setAttribute('aria-selected', 'true');
          document.querySelector('.luigi-search-result-item__0').focus();
        }
      } else if (GenericHelpers.isFunction(search.searchProvider.onInput)) {
        search.searchProvider.onInput();
      }
    } else {
      console.warn('GlobalSearch is not available.');
    }
  }

  function calcSearchResultItemSelected(direction) {
    let renderedSearchResultItems = luigiCustomSearchItemRenderer__slotContainer.children;
    if (renderedSearchResultItems) {
      for (let index = 0; index < renderedSearchResultItems.length; index++) {
        let { childNodes, nextSibling, previousSibling } = renderedSearchResultItems[
          index
        ];
        let nodeSibling;
        if (childNodes[0].getAttribute('aria-selected') === 'true') {
          if (direction === KEYCODE_ARROW_DOWN) {
            nodeSibling =
              nextSibling !== null ? nextSibling : renderedSearchResultItems[0];
          }
          if (direction === KEYCODE_ARROW_UP) {
            nodeSibling =
              previousSibling !== null
                ? previousSibling
                : renderedSearchResultItems[renderedSearchResultItems.length - 1];
          }
          childNodes[0].setAttribute('aria-selected', 'false');
          nodeSibling.childNodes[0].setAttribute('aria-selected', 'true');
          nodeSibling.focus();
          break;
        }
      }
    }
  }

  function clearAriaSelected() {
    let renderedSearchResultItems = luigiCustomSearchItemRenderer__slotContainer.children;
    if (renderedSearchResultItems) {
      for (let index = 0; index < renderedSearchResultItems.length; index++) {
        let element = renderedSearchResultItems[index];
        if (element.childNodes[0].getAttribute('aria-selected') === 'true') {
          element.childNodes[0].setAttribute('aria-selected', 'false');
        }
      }
    }
  }

  function onSearchResultItemSelected(searchResultItem) {
    if (
      search &&
      GenericHelpers.isFunction(search.searchProvider.onSearchResultItemSelected)
    ) {
      search.searchProvider.onSearchResultItemSelected(searchResultItem);
    } else if (
      GenericHelpers.isFunction(search.searchProvider.onEscape) &&
      event.keyCode === KEYCODE_ESC
    ) {
      search.searchProvider.onEscape();
    }
  }

  function handleKeydown(result, { keyCode }) {
    if (keyCode === KEYCODE_ENTER) {
      search.searchProvider.onSearchResultItemSelected(result);
    }
    if (keyCode === KEYCODE_ARROW_UP || keyCode === KEYCODE_ARROW_DOWN) {
      calcSearchResultItemSelected(keyCode);
    } else if (
      GenericHelpers.isFunction(search.searchProvider.onEscape) &&
      keyCode === KEYCODE_ESC
    ) {
      clearAriaSelected();
      setTimeout(() => {
        inputElem.focus();
      });
      search.searchProvider.onEscape();
    }
  }

  export function onActionClick(searchResultItem) {
    let node = searchResultItem.pathObject;
    if (node.externalLink) {
      Routing.navigateToLink(node);
    } else {
      dispatch('handleSearchNavigation', { node });
    }
  }

  function setFocusOnGlobalSearchFieldDesktop() {
    if (inputElem) {
      inputElem.focus();
    }
  }

  export function toggleSearch() {
    if (!isSearchFieldVisible)
      setTimeout(() => {
        setFocusOnGlobalSearchFieldDesktop();
      });
    else {
      displaySearchResult = false;
    }
    dispatch('toggleSearch', {
      isSearchFieldVisible,
      inputElem,
      luigiCustomSearchRenderer__slot
    });

    if (search && search.searchProvider && GenericHelpers.isFunction(search.searchProvider.toggleSearch)) {
      const fieldVisible =
        isSearchFieldVisible === undefined ? true : !isSearchFieldVisible;
      search.searchProvider.toggleSearch(inputElem, fieldVisible);
    }
  }
</script>

<svelte:window on:click={closeSearchResult} on:blur={closeSearchResult} />
<div
  class="fd-shellbar__action {isSearchFieldVisible
    ? 'luigi-search-shell__mobile'
    : ''}"
>
  <div class="fd-popover">
    <div
      class="fd-popover__control luigi-search"
      on:click|stopPropagation={() => {}}
      aria-hidden={!isSearchFieldVisible}
      aria-haspopup="true"
    >
      <div class="fd-input-group fd-shellbar__input-group">
        {#if search && search.disableInputHandlers}
          <input
            type="text"
            class="fd-input fd-input-group__input fd-shellbar__input-group-input luigi-search__input"
            data-testid="luigi-search-input__no-handlers"
            autofocus
          />
        {:else}
          <input
            type="text"
            on:keyup={event => onKeyUp(event)}
            class="fd-input fd-input-group__input fd-shellbar__input-group-input luigi-search__input"
            data-testid="luigi-search-input"
            autofocus
            bind:this={inputElem}
          />
        {/if}
      </div>
      {#if !isCustomSearchRenderer}
        <div
          class="fd-popover__body fd-popover__body--right luigi-search-popover__body"
          aria-hidden={!displaySearchResult}
        >
          <nav class="fd-menu">
            {#if searchResult}
              <ul
                class="fd-menu__list fd-menu__list--top"
                bind:this={luigiCustomSearchItemRenderer__slotContainer}
              >
                {#each searchResult as result, index}
                  <li
                    class="fd-menu__item luigi-search-result-item__{index}"
                    on:click={event =>
                      onSearchResultItemSelected(result, event)}
                    on:keyup={event => handleKeydown(result, event)}
                    tabindex="0"
                  >
                    {#if !isCustomSearchResultItemRenderer}
                      <a
                        class="fd-menu__link"
                        on:click|preventDefault={() => {}}
                      >
                        <div class="fd-product-switch__text">
                          <div class="fd-product-switch__title">
                            {result.label}
                          </div>
                          <div class="fd-product-switch__subtitle">
                            {result.description}
                          </div>
                        </div>
                      </a>
                    {:else}
                      {@html renderCustomSearchItem(
                        result,
                        luigiCustomSearchItemRenderer__slotContainer,
                        index
                      )}
                    {/if}
                  </li>
                {/each}
              </ul>
            {/if}
          </nav>
        </div>
      {:else}
        <div bind:this={luigiCustomSearchRenderer__slot} />
      {/if}
    </div>
  </div>
</div>
<div class="fd-shellbar__action fd-shellbar__action--desktop">
  <div on:click|stopPropagation={() => {}}>
    <button
      class="fd-button fd-button--transparent fd-shellbar__button"
      aria-haspopup="true"
      aria-expanded={!isSearchFieldVisible}
      on:click={toggleSearch}
      data-testid="luigi-search-btn-desktop"
    >
      <i class="sap-icon sap-icon--search" />
    </button>
  </div>
</div>

<style type="text/scss">
  //remove default browser outline on focus for search results
  .luigi-search-popover__body {
    li[class*='luigi-search-result']:focus {
      outline: none;
    }
  }
  .luigi-search {
    &[aria-hidden='true'] {
      visibility: hidden;
      width: 0;
    }

    &__input {
      width: 184px;
    }
    &-popover__body {
      top: calc(2.25rem + 5px);

      &:before,
      &:after {
        display: none;
      }
      .fd-menu {
        max-height: 70vh;
        overflow: auto;
        .fd-menu__item {
          margin-bottom: 10px;
          &:first-child {
            margin-top: 10px;
          }
          &:last-child .fd-menu__link {
            border-radius: 0;
          }
          .fd-product-switch__title {
            font-weight: 600;
          }
        }
      }
    }
  }

  //remove arrow from the search popover
  @media screen and (max-width: 1024px) {
    .luigi-search-shell__mobile {
      position: relative;
      height: calc(2.25rem + 2px);
      padding: 0;

      .fd-popover {
        vertical-align: top;
      }

      .luigi-search {
        position: absolute;
        top: 0;
        right: 0;
        background: var(--sapShellColor);
        z-index: 2;
      }
      .fd-menu {
        max-width: 12rem;
      }
    }
  }
</style>
