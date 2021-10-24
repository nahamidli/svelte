<script lang="ts">
    import { onMount, tick } from 'svelte';
    import MainLink from './MainLink.svelte';
    import { modules } from '@src/stores/navigation';
    import VesselSelector from './VesselSelector.svelte';

    type Mode = 'company' | 'vessel';

    let collapsed = false;
    let mode: Mode = '' as Mode;
    let vesselSelector: VesselSelector;
    let vesselSelectorTarget: HTMLElement;

    $: navigationModules = $modules.filter((m) => !m.settings);
    $: settingsModule = $modules.find((m) => m.settings);

    onMount(async () => {
        const mods = modules.init();
        mode = (window.smcRouter.currentRoute.query.mode as Mode) ?? setMode('company');

        window.smcRouter.addRender('checkRouteFrame', async (route, m) => {
            if (route.query?.mode && route.query.mode !== mode) {
                mode = route.query.mode as any;
            }

            await tick();
            modules.updateNavEntries();
        });

        await tick();
        // When root path is specified navigates to the first module to not show an empty page
        if (
            mods &&
            mods.length > 0 &&
            !mods.map((n) => n.name).some((name) => window.smcRouter.currentRoute.fullPath.includes(name))
        )
            window.smcRouter.navigate(mods[0].path, true);
    });

    $: {
        updateNav(mode);
    }

    // this is called 2 times each mode change...
    async function updateNav(mode: Mode) {
        const im = navigationModules.find((m) => m.name === 'itemmanagement');
        const imChildren = im?.children;

        if (!imChildren) return;

        const pro = imChildren?.find((c) => c.subpath === '/procurableitems');
        const equipreg = imChildren?.find((c) => c.subpath === '/equipmentregister');
        const tech = imChildren?.find((c) => c.subpath === '/technicalitems');
        pro!.hidden = mode === 'vessel';
        equipreg!.hidden = mode === 'vessel';
        tech!.hidden = mode === 'company';

        navigationModules = navigationModules;

        if (im?.active) await modules.navigateTo(im);
    }

    async function setMode(newMode: Mode) {
        console.log('::Navigation - Setting query Param:', 'mode=' + newMode);

        await window.smcRouter.setQueryParam('mode', newMode);
        console.log('::Navigation - Param set:', 'mode=' + newMode);

        mode = newMode;
        return mode;
    }

    function toggle() {
        collapsed = !collapsed;
    }
</script>

<div id="navigation" class="z-50 flex flex-col h-full text-white bg-blue-800" style="width:{collapsed ? 48 : 200}px">
    <div data-test="tabs" class="flex flex-row text-blue-600 bg-blue-900 select-none">
        <div
            class="flex-1 py-4 text-center cursor-pointer"
            on:click={() => setMode('company')}
            class:active={mode === 'company'}
        >
            C{#if !collapsed}ompany{/if}
        </div>
        <div
            class="flex-1 py-4 text-center cursor-pointer"
            on:click={() => setMode('vessel')}
            class:active={mode === 'vessel'}
        >
            V{#if !collapsed}essel{/if}
        </div>
    </div>

    <div data-test="filter" class="h-12 border-b border-blue-700">
        {#if mode === 'vessel'}
            {#if collapsed}
                <div
                    class="flex items-center justify-center w-full h-full cursor-pointer hover:bg-blue-700"
                    bind:this={vesselSelectorTarget}
                    on:click={vesselSelector.show}
                    data-test="vesselSelector"
                >
                    <i class="fal fa-ship fa-fw" />
                    <VesselSelector collapsed bind:this={vesselSelector} externalTarget={vesselSelectorTarget} />
                </div>
            {:else}
                <div class="px-4 py-2">
                    <VesselSelector />
                </div>
            {/if}
        {/if}
    </div>

    <div data-test="modules" class="z-10 text-base">
        {#each navigationModules as module}
            <MainLink {module} {collapsed} />
        {/each}
    </div>

    <div
        class="flex {collapsed ? 'flex-col' : 'flex-row'} mt-auto border-t border-blue-700 cursor-pointer select-none"
        class:settings-active={settingsModule?.active}
        class:collapsed
    >
        {#if settingsModule}
            <div
                class="flex items-center {collapsed
                    ? 'justify-center'
                    : 'pl-3 settings'} h-12 text-base hover:bg-blue-700 menu-entry"
                on:click={() => modules.navigateToSettings()}
                data-test="settings"
            >
                <i class="fal fa-cog fa-fw" />
                {#if !collapsed}
                    <span class="ml-2">{settingsModule?.label}</span>
                {/if}
            </div>
        {/if}

        <span
            class="flex items-center justify-center w-full h-12 group hover:bg-blue-700 collapser"
            on:click={toggle}
            data-test="collapser"
        >
            <i
                class="text-2xl text-blue-400 group-hover:text-blue-500 fal {collapsed
                    ? 'fa-angle-double-right'
                    : 'fa-angle-double-left'}"
            />
        </span>
    </div>
</div>

<style>
    .active {
        @apply bg-blue-800;
        @apply text-white;
    }

    .settings-active {
        @apply bg-white;
        @apply text-blue-700;
        @apply border-digital-blue-200;
    }

    .settings {
        min-width: 152px;
    }
    .settings-active:not(.collapsed) .menu-entry:hover,
    .settings-active:not(.collapsed) .collapser:hover {
        @apply bg-neutral-100;
    }

    .settings-active:not(.collapsed) .collapser {
        @apply text-blue-400;
    }

    .settings-active.collapsed .collapser {
        @apply bg-blue-800;
        @apply text-blue-400;
    }

    .settings-active.collapsed .menu-entry:hover {
        @apply bg-neutral-100;
    }

    .settings-active.collapsed .collapser:hover {
        @apply bg-blue-700;
    }
</style>
