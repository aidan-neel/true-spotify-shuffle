<script>
    import { goto } from '$app/navigation';
    import Icon from '@iconify/svelte';
    import '../app.css';

    /**
     * Generates a random string containing numbers and letters
     * @param {number} length The length of the string
     */
    const generateRandomString = (length) => {
        let text = '';
        const possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';

        for (let i = 0; i < length; i++) {
            text += possible.charAt(Math.floor(Math.random() * possible.length));
        }

        return text;
    }

    const scope = 'playlist-read-private playlist-read-collaborative user-read-playback-state user-modify-playback-state user-read-currently-playing user-library-read user-read-playback-position'
    const state = generateRandomString(16);

    goto(`https://accounts.spotify.com/authorize?response_type=code&client_id=${import.meta.env.CLIENT_ID}&scope=${scope}&redirect_uri=${import.meta.env.REDIRECT_URI}&state=${state}`);
</script>

<main class="h-screen w-screen flex items-center bg-zinc-100 justify-center pb-64 flex-col">
    <header class="text-center flex items-center justify-center flex-col">
        <h1 class="font-extrabold text-zinc-900 text-4xl">
            True Shuffle for Spotify
        </h1>
        <p class="w-2/3 text-zinc-600 mt-1">
            Shuffle your Spotify playlists without repeats, and without bias. A true random experience.
        </p>
    </header>
    <div class="mt-3">
        <button class="bg-green-500 flex flex-row items-center justify-center gap-2 px-8 p-2.5 hover:bg-green-600 rounded-full text-white shadow-md font-medium">
            Login with Spotify <Icon icon="simple-icons:spotify" class="text-white h-5 w-5" />
        </button>
    </div>
</main>