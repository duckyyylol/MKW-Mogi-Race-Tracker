<script lang="ts">
    import "duckylib/styles/globals.css";
    import { browser } from "$app/environment";
    import { goto } from "$app/navigation";
    import favicon from "$lib/assets/favicon.png";
    import { Website, Header, setUserData, getTheme } from "duckylib";
    import { onMount } from "svelte";
    import { page } from "$app/state";

    let { children } = $props();
</script>

<svelte:head>
    <link rel="icon" href={favicon} />
    <title>Ducky's Mogi Track-er</title>
</svelte:head>

<Website options={{ theme: getTheme() }}>
    <Header
        emoji="🏎️"
        label="Ducky's Mogi Track-er"
        defaultNav={false}
        nav={[{ label: "Tracker", pathname: "/", symbol: "list_alt_add" }]}
    />

    {@render children()}
</Website>

<style>
    div {
        height: 100%;
    }

    :global(::-webkit-scrollbar) {
        display: none !important;
        width: 0px !important;
        height: 0px !important;
        background: transparent !important;
    }

    :global(html, body, .chat) {
        scrollbar-width: none !important; /* Firefox fallback */
        -ms-overflow-style: none !important; /* Edge fallback */
    }
</style>
