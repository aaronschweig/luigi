<script>
  import { beforeUpdate, createEventDispatcher, onMount, getContext } from 'svelte';
  import { Navigation } from './services/navigation';
  import { NavigationHelpers, RoutingHelpers } from '../utilities/helpers';
  import { LuigiConfig } from '../core-api';

  export let children;
  export let pathData;
  export let pathParams;
  let previousPathData;
  export let hideNavComponent;
  export let virtualGroupPrefix = NavigationHelpers.virtualGroupPrefix;
  let selectedNode;
  export let selectedNodeForTabNav;
  export let dropDownStates = {};
  export let isMoreBtnExpanded = false;
  export let resizeTabNavToggle;
  let previousResizeTabNavToggle;
  let getTranslation = getContext('getTranslation');

  //TODO refactor
  const __this = {
    get: () => ({
      children,
      pathData,
      hideNavComponent,
      virtualGroupPrefix,
      selectedNode,
      selectedNodeForTabNav,
      dropDownStates,
      isMoreBtnExpanded
    }),
    set: obj => {
      if (obj) {
        Object.getOwnPropertyNames(obj).forEach(prop => {
          if (prop === 'pathData') {
            pathData = obj.pathData;
          } else if (prop === 'context') {
            context = obj.context;
          } else if (prop === 'children') {
            children = obj.children;
          } else if (prop === 'selectedNode') {
            selectedNode = obj.selectedNode;
          } else if (prop === 'selectedNodeForTabNav') {
            selectedNodeForTabNav = obj.selectedNodeForTabNav;
          }
        });
      }
    }
  };

  const dispatch = createEventDispatcher();

  const setTabNavData = async () => {
    const componentData = __this.get();
    const tabNavData = await Navigation.getTabNavData(
      { ...componentData },
      componentData
    );
    if (!tabNavData) {
      return;
    }
    __this.set({ ...tabNavData });
    previousPathData = pathData;
    window['LEFTNAVDATA'] = tabNavData.groupedChildren;
    setTimeout(calcTabsContainer);
  };

  const calcTabsContainer = () => {
    clearTapNav();
    let tabsContainer = document.getElementsByClassName('luigi-tabsContainer')[0];
    let morebtn = document.getElementsByClassName('luigi-tabsMoreButton')[0];
    let moreLink = document.getElementsByClassName('luigi__more')[0];
    let tabsContainerOffsetWidth;
    let totalTabsSize = 0;
    let hasMoreBtnElements = false;
    let componentData = __this.get();
    let style;
    let margin;
    moreLink && moreLink.setAttribute('aria-selected', 'false');
    if (tabsContainer) {
      tabsContainerOffsetWidth = tabsContainer.offsetWidth;
      let tabs = [...tabsContainer.children];
      tabs.forEach(element => {
        style = element.currentStyle || window.getComputedStyle(element);
        margin = parseFloat(style.marginLeft) + parseFloat(style.marginRight);
        totalTabsSize += element.offsetWidth + margin;
        let uid = element.getAttribute('uid');
        if (totalTabsSize >= tabsContainerOffsetWidth) {
          element.classList.add('hide_element');
          if (element.getAttribute('isSelected') === 'true') {
            moreLink.setAttribute('aria-selected', 'true');
          }
          document
            .querySelector('li[uid="' + uid + '"]')
            .classList.remove('hide_element');
          hasMoreBtnElements = true;
        } else {
          document.querySelector('li[uid="' + uid + '"]').classList.add('hide_element');
        }
      });
      !hasMoreBtnElements
        ? morebtn.classList.add('hide_element')
        : morebtn.classList.remove('hide_element');
    }
  };

  const clearTapNav = () => {
    let tabsContainer = document.getElementsByClassName('luigi-tabsContainer')[0];
    if (tabsContainer !== undefined) {
      const tabs = [...tabsContainer.children];
      tabs.forEach(element => {
        element.classList.remove('hide_element');
      });
    }
  };

  onMount(() => {
    hideNavComponent = LuigiConfig.getConfigBooleanValue('settings.hideNavigation');
  });

  // [svelte-upgrade warning]
  // beforeUpdate and afterUpdate handlers behave
  // differently to their v2 counterparts
  beforeUpdate(() => {
    if (!previousPathData || previousPathData != pathData) {
      setTabNavData();
    }
    if (
      previousResizeTabNavToggle === undefined ||
      previousResizeTabNavToggle !== resizeTabNavToggle
    ) {
      previousResizeTabNavToggle = resizeTabNavToggle;
      setTabNavData();
    }
  });

  function isOpenUIiconName(name) {
    return NavigationHelpers.isOpenUIiconName(name);
  }

  function isExpanded(nodes, expandedList) {
    return expandedList && expandedList.indexOf(nodes.metaInfo.categoryUid) >= 0;
  }

  function isSelectedCat(key, selectedNodeForTabNav) {
    return (
      selectedNodeForTabNav &&
      selectedNodeForTabNav.category &&
      (selectedNodeForTabNav.category === key ||
        selectedNodeForTabNav.category.label === key)
    );
  }

  function getRouteLink(node) {
    return RoutingHelpers.getNodeHref(node, pathParams);
  }

  // [svelte-upgrade suggestion]
  // review these functions and remove unnecessary 'export' keywords
  export function handleClick(node) {
    closeAllDropdowns();
    dispatch('handleClick', { node });
  }

  export function toggleDropdownState(name) {
    let ddnStates = {};
    if (!dropDownStates[name]) {
      ddnStates[name] = true;
    }
    dropDownStates = ddnStates;
  }

  export function closeAllDropdowns() {
    dropDownStates = {};
    isMoreBtnExpanded = false;
  }

  export function onResize() {
    clearTapNav();
    calcTabsContainer();
  }
  export function toggleMoreBtn() {
    isMoreBtnExpanded = !isMoreBtnExpanded;
  }
</script>

<svelte:window
  on:click={closeAllDropdowns}
  on:blur={closeAllDropdowns}
  on:resize={onResize}
/>
{#if children && pathData.length > 1}
  <nav
    class="fd-tabs fd-tabs--l"
    role="tablist"
    id="tabsContainer"
    on:toggleDropdownState={event => toggleDropdownState(event.name)}
  >
    <div class="tabsContainerWrapper">
      <div class="tabsContainer luigi-tabsContainer">
        {#each Object.entries(children) as [key, nodes], index}
          {#if key === 'undefined' || key.indexOf(virtualGroupPrefix) === 0}
            {#each nodes as node, index2}
              {#if !node.hideFromNav}
                {#if node.label}
                  <span
                    class="fd-tabs__item {node === selectedNodeForTabNav
                      ? 'is-selected'
                      : ''}"
                    uid="{index}-{index2}"
                    isSelected={node === selectedNodeForTabNav}
                  >
                    <a
                      class="fd-tabs__link"
                      href={getRouteLink(node)}
                      role="tab"
                      aria-selected={node === selectedNodeForTabNav}
                      on:click|preventDefault={() => handleClick(node)}
                    >
                      <span class="fd-tabs__tag"
                        >{$getTranslation(node.label)}</span
                      >
                    </a>
                  </span>
                {/if}
              {/if}
            {/each}
          {:else}
            <span
              class="fd-tabs__item"
              uid="{index}-0"
              on:click={event => event.stopPropagation()}
              isSelected={isSelectedCat(key, selectedNodeForTabNav)}
            >
              <div class="fd-popover">
                <div class="fd-popover__control">
                  <a
                    class="fd-tabs__link has-child"
                    aria-expanded="false"
                    role="tab"
                    on:click|preventDefault={() => toggleDropdownState(key)}
                    aria-selected={isSelectedCat(key, selectedNodeForTabNav)}
                  >
                    <span class="label fd-tabs__tag"
                      >{$getTranslation(key)}</span
                    >
                    <span class="sap-icon--dropdown luigi-icon--dropdown" />
                  </a>
                </div>
                <div
                  class="fd-popover__body fd-popover__body--no-arrow"
                  aria-hidden={!dropDownStates[key]}
                >
                  <nav class="fd-menu">
                    <ul class="fd-menu__list fd-menu__list--no-shadow">
                      {#each nodes as node}
                        {#if !node.hideFromNav}
                          {#if node.label}
                            <li class="fd-menu__item">
                              <a
                                href={getRouteLink(node)}
                                class="fd-menu__link"
                                on:click|preventDefault={() =>
                                  handleClick(node)}
                                aria-selected={node === selectedNodeForTabNav}
                              >
                                <span class="fd-menu__title"
                                  >{$getTranslation(node.label)}</span
                                >
                              </a>
                            </li>
                          {/if}
                        {/if}
                      {/each}
                    </ul>
                  </nav>
                </div>
              </div>
            </span>
          {/if}
        {/each}
      </div>
    </div>

    <div class="luigi-tabsMoreButton">
      <span class="fd-tabs__item" on:click={event => event.stopPropagation()}>
        <div class="fd-popover fd-popover--right">
          <a
            class="fd-tabs__link fd-popover__control has-child luigi__more"
            aria-expanded="false"
            role="tab"
            on:click|preventDefault={toggleMoreBtn}
          >
            <span class="label fd-tabs__tag">More</span>
            <span class="sap-icon--dropdown luigi-icon--dropdown" />
          </a>
          <div
            class="fd-popover__body fd-popover__body--right fd-popover__body--no-arrow"
            aria-hidden={!isMoreBtnExpanded}
          >
            <ul
              class="fd-nested-list fd-nested-list--compact fd-nested-list--text-only"
            >
              {#each Object.entries(children) as [key, nodes], index}
                {#if key === 'undefined' || key.indexOf(virtualGroupPrefix) === 0}
                  {#each nodes as node, index2}
                    <li class="fd-nested-list__item" uid="{index}-{index2}">
                      <a
                        href={getRouteLink(node)}
                        class="fd-nested-list__link"
                        on:click|preventDefault={() => handleClick(node)}
                        aria-selected={node === selectedNodeForTabNav}
                      >
                        <span class="fd-nested-list__title"
                          >{$getTranslation(node.label)}</span
                        >
                      </a>
                    </li>
                  {/each}
                {:else}
                  <li class="fd-nested-list__item" uid="{index}-0">
                    <div class="fd-nested-list__content has-child" tabindex="0">
                      <a
                        href="javascript:void(null)"
                        tabindex="-1"
                        class="fd-nested-list__link"
                        id="tabnav_list_level1_{index}"
                        aria-haspopup="true"
                        aria-expanded={dropDownStates[key + index]}
                        aria-selected={isSelectedCat(
                          key,
                          selectedNodeForTabNav
                        )}
                        on:click|preventDefault={() =>
                          toggleDropdownState(key + index)}
                      >
                        <span class="fd-nested-list__title"
                          >{$getTranslation(key)}</span
                        >
                      </a>
                      <button
                        class="fd-button fd-nested-list__button"
                        href="#"
                        tabindex="-1"
                        aria-label="Expand submenu"
                        aria-haspopup="true"
                        aria-expanded={dropDownStates[key + index]}
                        on:click|preventDefault={() =>
                          toggleDropdownState(key + index)}
                      >
                        <i
                          class={dropDownStates[key + index]
                            ? 'sap-icon--navigation-down-arrow'
                            : 'sap-icon--navigation-right-arrow'}
                          role="presentation"
                        />
                      </button>
                    </div>
                    <ul
                      class="fd-nested-list level-2"
                      aria-hidden={!dropDownStates[key + index]}
                    >
                      {#each nodes as node}
                        <li
                          class="fd-nested-list__item"
                          aria-labelledby="tabnav_list_level1_{index}"
                        >
                          <a
                            class="fd-nested-list__link"
                            href={getRouteLink(node)}
                            on:click|preventDefault={() => handleClick(node)}
                            aria-selected={node === selectedNodeForTabNav}
                          >
                            <span class="fd-nested-list__title"
                              >{$getTranslation(node.label)}</span
                            >
                          </a>
                        </li>
                      {/each}
                    </ul>
                  </li>
                {/if}
              {/each}
            </ul>
          </div>
        </div>
      </span>
    </div>
  </nav>
{/if}

<style type="text/scss">
  @import 'src/styles/_mixins.scss';
  @import 'src/styles/_variables.scss';

  .tabsContainer {
    width: 100%;
  }

  .tabsContainerWrapper {
    flex-grow: 1;
    flex-basis: auto;
    flex-shrink: 1;
  }

  .luigi-tabsMoreButton {
    .fd-popover__body {
      > .fd-nested-list {
        max-height: calc(100vh - 110px);
        overflow-y: auto !important;
        padding-top: 4px;
        padding-bottom: 4px;

        .fd-nested-list__item:last-child {
          .fd-nested-list__content {
            border-bottom: none;
          }
        }
      }
    }
    .fd-tabs__link {
      padding-right: 0;

      &.has-child {
        .luigi-icon--dropdown {
          right: 0;
        }
      }
    }
  }

  :global(.luigi-tabsMoreButton.hide_element) {
    display: none;
  }
  :global(.luigi-tabsMoreButton .fd-nested-list__item.hide_element) {
    display: none;
  }

  .fd-tabs {
    flex-wrap: nowrap;
    align-items: stretch;
    @include box-shadow(1px 1px 2px 0 rgba(0, 0, 0, 0.05));
    border: none;
    position: absolute;
    right: 0;
    left: var(--luigi__left-sidenav--width);

    @media (max-width: 599px) {
      left: 0;
    }

    &__item {
      white-space: nowrap;
      display: inline-block;
    }

    &__link {
      &.has-child {
        .label {
          padding-right: 17px;
        }
        .luigi-icon--dropdown {
          position: absolute;
          top: 0.4em;
          right: 14px;
        }
      }
    }
  }

  :global(.fd-tabs__item.hide_element) {
    display: none;
  }
</style>
