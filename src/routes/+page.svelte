<script>
    import { goto } from '$app/navigation';
    import Icon from '@iconify/svelte';
    import { onMount } from 'svelte';
    import { fade } from 'svelte/transition';
    import '../app.css';
    import '../app.pcss';

    const CLIENT_ID = "5f68cc8f5281480d920786c9f5c5b680"; // Replace with your Spotify Client ID
    const REDIRECT_URI = "https://shuffle.aidan-neel.com/authorize"; // Replace with your redirect URI
    const scope = 'playlist-read-private playlist-read-collaborative user-read-playback-state user-modify-playback-state user-read-currently-playing user-library-read user-read-playback-position';

    let isOn = false;
    let loading = true;
    let access_token;
    let playbackData;
    let userData;
    let shuffling = false;
    let queue = [];
    let currentQueueIndex = 0;

    const generateRandomString = (length) => {
        let text = '';
        const possible = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
        for (let i = 0; i < length; i++) {
            text += possible.charAt(Math.floor(Math.random() * possible.length));
        }
        return text;
    }

    const state = generateRandomString(16);
    const url = `https://accounts.spotify.com/authorize?response_type=token&client_id=${CLIENT_ID}&scope=${encodeURIComponent(scope)}&redirect_uri=${encodeURIComponent(REDIRECT_URI)}&state=${state}`;

    const handleUnauthorized = () => {
        goto(url);
    };

    const fetchData = async (endpoint, method = 'GET', body = null) => {
        const response = await fetch(endpoint, {
            method: method,
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${access_token}`
            },
            body: body
        });

        if (response.status === 401) {
            handleUnauthorized();
            return null;
        }

        if (response.status !== 200 && response.status !== 204) {
            throw new Error(`API request failed with status ${response.status}`);
        }

        return response.status === 204 ? null : await response.json();
    };

    const updatePlaybackData = async () => {
        console.log('Updating playback data')
        const data = await fetchData('https://api.spotify.com/v1/me/player/currently-playing');
        if (data) {
            playbackData = {
                status: 200,
                data: data
            };
        }
    };

    const getMeData = async () => {
        return await fetchData('https://api.spotify.com/v1/me');
    };

    onMount(async () => {
        access_token = localStorage.getItem("access_token");
        loading = false;

        if (access_token) {
            try {
                userData = await getMeData();                
                await updatePlaybackData();
                setInterval(updatePlaybackData, 1000);
            } catch (error) {
                console.error('Error:', error);
            }
        } else {
            handleUnauthorized();
        }
    });

    function formatTime(milliseconds) {
        const totalSeconds = Math.floor(milliseconds / 1000);
        const minutes = Math.floor(totalSeconds / 60);
        const seconds = totalSeconds % 60;
        return `${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
    }

    async function startShuffle() {
        console.log('Starting shuffle');
        shuffling = true;
        let songs = [];
        let alreadyPlayed = [];
        queue = [];
        currentQueueIndex = 0;

        const context = playbackData?.data.context;
        if (context && context.type === 'playlist' && shuffling) {
            const playlistId = context.uri.split(':')[2];
            const playlistData = await fetchData(`https://api.spotify.com/v1/playlists/${playlistId}/tracks`);
            const playlistLength = playlistData.items.length;

            // Function to select a random, unplayed song
            function selectRandomSong() {
                let randomIndex;
                do {
                    randomIndex = Math.floor(Math.random() * playlistLength);
                } while (alreadyPlayed.includes(randomIndex)); // Ensure the song hasn't been played
                return randomIndex;
            }

            let songIndex = selectRandomSong();
            alreadyPlayed.push(songIndex); // Update the already played list

            console.log("Selected song index:", songIndex);
            console.log("Already played:", alreadyPlayed);

            // Make sure it's not local
            if (!playlistData.items[songIndex].track.is_local) {
                const songUri = playlistData.items[songIndex].track.uri;
                console.log("Adding to queue:", songUri);

                queue = playlistData.items.map(item => item.track.uri);
                currentQueueIndex = queue.indexOf(songUri);

                // Push to queue and skip to this song
                await playTrack(queue[currentQueueIndex]);
            }
        }
    }

    async function playTrack(uri) {
        await fetchData(`https://api.spotify.com/v1/me/player/play`, 'PUT', JSON.stringify({ uris: [uri] }));
    }

    const controlPlayback = async (action) => {
        if (action === 'next') {
            currentQueueIndex = (currentQueueIndex + 1) % queue.length;
            await playTrack(queue[currentQueueIndex]);
        } else if (action === 'previous') {
            currentQueueIndex = (currentQueueIndex - 1 + queue.length) % queue.length;
            await playTrack(queue[currentQueueIndex]);
        } else {
            // Handle pause and play normally
            let endpoint = action === 'pause' ? `https://api.spotify.com/v1/me/player/pause` : `https://api.spotify.com/v1/me/player/${action}`;
            await fetchData(endpoint, action === 'next' || action === 'previous' ? 'POST' : 'PUT');
            if (action !== 'pause') await updatePlaybackData();
        }
    };
    
    const loginFunction = async() => {
        goto(url);
    }
</script>

<svelte:head>
    <title>
        True Shuffle for Spotify
    </title>
    <meta name="description" content="Shuffle your Spotify playlists properly" />
    <meta name="keywords" content="Spotify, Shuffle, True Shuffle, Shuffle Spotify, Spotify Shuffle, Spotify True Shuffle, True Shuffle for Spotify, Spotify True Shuffl
e, Spotify Shuffle Fix, Spotify Shuffle Broken, Spotify Shuffle Broken Fix, Spotify Shuffle Broken Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle
    Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed,
    Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotif
    >" />

    <meta property="og:title" content="True Shuffle for Spotify" />
    <meta property="og:description" content="Shuffle your Spotify playlists properly" />
</svelte:head>

<style>
    .fadeUp {
        animation: fadeUp 0.4s ease-in-out;
    }

    @keyframes fadeUp {
        0% {
            transform: translateY(20px);
            opacity: 0;
        }
        100% {
            transform: translateY(0px);
            opacity: 1;
        }
    }
</style>

{#if !playbackData && loading}
    <main transition:fade={{ delay: 350, duration: 400 }} class="bg-zinc-950 w-screen h-screen flex items-center justify-center absolute z-20">
    </main>
{/if}

<main class="bg-zinc-950 w-screen h-screen flex items-center justify-center">
    {#if access_token && playbackData && playbackData?.data.is_playing}
        <img on:click={() => {
            goto(`https://open.spotify.com/track/${playbackData?.data.item.id}`);
        }} src={playbackData?.data.item.album.images[0].url} class="w-screen h-screen blur-lg opacity-20 object-cover absolute z-0 shadow-md" />
        <nav class="top-8 left-8 absolute rounded-2xl flex flex-row items-center justify-center gap-2">
            <img src={userData?.images[0].url} alt="Profile picture" class="w-10 h-10 rounded-full shadow-md" />
            <p class='text-white font-semibold'>
                {userData?.display_name}
            </p>
        </nav>

        <nav class="absolute top-8 right-8 z-20">
            <h1 class="text-white font-medium">
                True Shuffle for Spotify
            </h1> 
        </nav>

        <main class="w-full h-full z-10 flex items-center justify-end pt-24 px-24 flex-col">
            <div class="flex items-center justify-start w-full flex-row">
                <img src={playbackData?.data.item.album.images[0].url} alt="Album cover" class="w-[32rem] h-[32rem] rounded-2xl shadow-md" />
                <header class='flex flex-col text-white ml-12 h-[32rem] pb-12 justify-end items-start'>
                    <h1 class="font-bold text-6xl w-full">
                        {playbackData?.data.item.name}
                    </h1>
                    <p class="text-xl text-white/80 mt-4">
                        {playbackData?.data.item.artists[0].name} &middot; {playbackData?.data.item.album.name}
                    </p>
                </header>
            </div>
            <div class="mb-24 w-full mt-5 flex items-center justify-center flex-col">
                <div class="text-white flex flex-row items-center justify-center w-full">
                    <p>{formatTime(playbackData?.data.progress_ms)}</p>
                    <div class="w-full"> 
                        <div class="h-1 bg-white/20 rounded-full mx-4">
                            <div class="h-full bg-white/80 rounded-full" style="width: {playbackData?.data.progress_ms / playbackData?.data.item.duration_ms * 100}%"></div>
                        </div>
                    </div>
                    <p>{formatTime(playbackData?.data.item.duration_ms)}</p>
                </div>
                <div class='text-white w-full flex gap-4 items-center justify-center mt-5'>
                    <button on:click={() => {controlPlayback('next')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
                        <Icon icon="ic:sharp-skip-previous" class=" h-full w-full" />
                    </button>
                    {#if playbackData?.data.is_playing}
                        <button on:click={() => {controlPlayback('pause')}} class="text-white hover:text-white/50 h-20 w-20 duration-100 rounded-lg">
                            <Icon icon="gridicons:pause" class="h-full w-full" />
                        </button>
                    {:else}
                        <button on:click={() => {controlPlayback('play')}} class="text-white hover:text-white/50 h-20 w-20 duration-100 rounded-lg">
                            <Icon icon="gridicons:play" class="h-full w-full" />
                        </button>
                    {/if}
                    <button on:click={() => {controlPlayback('previous')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
                        <Icon icon="ic:sharp-skip-next" class="h-full w-full" />
                    </button>
                </div>
                <button on:click={startShuffle} class="p-2 z-10 px-4 mt-8 {shuffling ? 'border-green-500/70 text-green-500 shadow-green-500/10' : ''} text-white/80 rounded-xl shadow-md hover:bg-zinc-900 hover:bg-opacity-70 duration-150 shadow-white/[0.025] bg-zinc-950 border border-white/20">
                    {shuffling ? 'Stop' : 'Start'} shuffling
                </button>
            </div>
        </main>
    {:else if playbackData?.data.is_playing === false}
    <main class="w-full h-full z-10 flex items-center justify-end pt-24 px-24 flex-col">
        <div class="flex items-center justify-start w-full flex-row">
            <header class='flex flex-col text-white h-[32rem] pb-12 justify-end items-start'>
                <h1 class="font-bold text-6xl w-full">
                    Currently not playing anything
                </h1>
                <p class="text-xl text-white/80 mt-4">
                    Start playing something on Spotify
                </p>
            </header>
        </div>
        <div class="mb-24 w-full mt-5 flex items-center justify-center flex-col">
            <div class="text-white flex flex-row items-center justify-center w-full">
                <p>00:00</p>
                <div class="w-full"> 
                    <div class="h-1 bg-white/20 rounded-full mx-4">
                        <div class="h-full bg-white/80 rounded-full" style="width: {1 / 1 * 100}%"></div>
                    </div>
                </div>
                <p>00:00</p>
            </div>
            <div class='text-white w-full flex gap-4 items-center justify-center mt-5'>
                <button on:click={() => {controlPlayback('next')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
                    <Icon icon="ic:sharp-skip-previous" class=" h-full w-full" />
                </button>
                {#if playbackData?.data.is_playing}
                    <button on:click={() => {controlPlayback('pause')}} class="text-white hover:text-white/50 h-20 w-20 duration-100 rounded-lg">
                        <Icon icon="gridicons:pause" class="h-full w-full" />
                    </button>
                {:else}
                    <button on:click={() => {controlPlayback('play')}} class="text-white hover:text-white/50 h-20 w-20 duration-100 rounded-lg">
                        <Icon icon="gridicons:play" class="h-full w-full" />
                    </button>
                {/if}
                <button on:click={() => {controlPlayback('previous')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
                    <Icon icon="ic:sharp-skip-next" class="h-full w-full" />
                </button>
            </div>
        </div>
    </main>
    {:else}
        <button on:click={loginFunction} class="p-2 z-10 px-4 text-white/80 rounded-xl shadow-md hover:bg-zinc-900 hover:bg-opacity-70 duration-150 shadow-white/[0.025] bg-zinc-950 border border-white/20">
            Sign in with Spotify
        </button>
    {/if}
</main>
