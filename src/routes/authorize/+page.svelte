<script lang="ts">
    import { goto } from "$app/navigation";
    import { page } from "$app/stores";
    import { onMount } from "svelte";

    onMount(() => {
        const data = $page.url.hash.slice(14);
        const access_token = data.split("&")[0];
        const token_type = data.split("&")[1];
        const expires_in = data.split("&")[2];
        const state = data.split("&")[3];

        localStorage.setItem("access_token", access_token);
        localStorage.setItem("token_type", token_type);
        localStorage.setItem("expires_in", expires_in);
        localStorage.setItem("state", state);

        goto("/");
    })
</script>

<main class="bg-zinc-950 w-screen h-screen flex items-center justify-center">

</main>