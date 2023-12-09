<script>
    import { goto } from '$app/navigation';
    import Icon from '@iconify/svelte';
    import { onMount } from 'svelte';
    import { fade, fly } from 'svelte/transition';
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
            console.log(playlistLength, playlistData)
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
            }

            // Push queue to spotify queue
            

            if(alreadyPlayed.length === playlistLength) {
                console.log('All songs played, resetting');
                alreadyPlayed = [];
            }
        }
    }

    async function playTrack(uri) {
        await fetchData(`https://api.spotify.com/v1/me/player/play`, 'PUT', JSON.stringify({ uris: [uri] }));
    }

    const controlPlayback = async (action) => {
        if (action === 'next') {
            console.log(queue, queue.length);
            console.log('next')
            if(queue.length > 0) {
                currentQueueIndex = (currentQueueIndex + 1) % queue.length;
                await playTrack(queue[currentQueueIndex]);
            } else {
                await fetchData(`https://api.spotify.com/v1/me/player/next`, 'POST');
                await updatePlaybackData();
            }
        } else if (action === 'previous') {
            console.log(queue, queue.length);
            console.log('previous')
            if(queue.length > 0) {
                currentQueueIndex = (currentQueueIndex - 1) % queue.length;
                await playTrack(queue[currentQueueIndex]);
                console.log(queue);
            } else {
                await fetchData(`https://api.spotify.com/v1/me/player/previous`, 'POST');
                await updatePlaybackData();
            }
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

    let showingmessage = false;
</script>

<svelte:head>
    <title>
        {playbackData?.data.item.name || 'True Shuffle for Spotify'}
    </title>
    <meta name="description" content="Shuffle your Spotify playlists properly" />
    <meta name="keywords" content="Spotify, Shuffle, True Shuffle, Shuffle Spotify, Spotify Shuffle, Spotify True Shuffle, True Shuffle for Spotify, Spotify True Shuffl
e, Spotify Shuffle Fix, Spotify Shuffle Broken, Spotify Shuffle Broken Fix, Spotify Shuffle Broken Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle
    Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed,
    Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotify Shuffle Fixed, Spotif
    >" />

    <meta property="og:url" content="https://shuffle.aidan-neel.com" />
    <meta property="og:type" content="website" />
    <meta property="og:image" content="https://media.discordapp.net/attachments/1139174900590968862/1183187080625528843/Untitled-2.png?ex=65876bd6&is=6574f6d6&hm=7bdf7a453bc3cc5569908c054f2c7fac4ad6ac7624ac351e13c1978028601111&=&format=webp&quality=lossless&width=1537&height=905" />
    <meta property="og:title" content="True Shuffle for Spotify" />
    <meta property="og:description" content="Shuffle your Spotify playlists properly with no bias or repeats. Currently waiting for Spotify's approval to be published." />
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
    <main transition:fade={{ delay: 350, duration: 400 }} class="bg-zinc-950 w-screen h-screen flex items-center justify-center absolute z-50">
    </main>
{/if}

{#if showingmessage}
<main transition:fade={{ delay: 350, duration: 400 }} class="bg-zinc-950 bg-opacity-50 backdrop-blur-sm w-screen h-screen flex items-center justify-center absolute z-50">
    <div transition:fly={{ y: 20, duration: 300 }} class="w-[32rem] p-4 bg-white bg-opacity-5 rounded-lg border border-white/10 backdrop-blur-md">
        <h1 class="text-white font-semibold text-2xl flex items-start justify-between w-full">
            Warning!<button on:click={() => {showingmessage = false}}>
                <Icon icon="ic:baseline-close" class="w-6 h-6 text-white/80"  />
            </button>
        </h1>
        <p class="text-white/80 text-lg">
            This app is currently waitlisted until Spotify approves it. This means you cannot use the app unless added to the waitlist manually. Sorry.
        </p>
    </div>
</main>
{/if}

<main class="bg-zinc-950 overflow-hidden w-screen h-screen flex items-center justify-center">
    <aside class="absolute md:hidden bottom-6 w-screen px-4 text-center text-white/40">
        <p>
            Not intended for mobile devices. Functionality isn't guaranteed.
        </p>
    </aside>
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

        <main class="w-full h-full overflow-hidden z-10 flex items-center justify-end md:pt-24 px-8 lg:px-32 flex-col relative">
            <div class="md:items-end md:flex hidden h-full justify-start text-center md:text-left md:justify-start w-full flex-col relative md:flex-row">
                <img src={playbackData?.data.item.album.images[0].url} alt="Album cover" class="md:w-[32rem] w-[20rem] sm:w-[24rem] h-[20rem] sm:h-[24rem] md:h-[32rem] rounded-2xl shadow-md" />
                <header class='flex flex-col w-full text-white mt-2 md:mt-0 md:pl-12 h-[32rem] md:pb-12 md:justify-end md:items-start'>
                    <h1 class="font-bold text-4xl truncate h-12 xl:h-16 xl:text-5xl w-[80%]">
                        {playbackData?.data.item.name}
                    </h1>
                    <p class="text-xl text-white/80">
                        {playbackData?.data.item.artists[0].name} &middot; {playbackData?.data.item.album.name}
                    </p>
                </header>
            </div>
            <div class="md:hidden flex flex-col items-center justify-center text-center absolute">
                <img src={playbackData?.data.item.album.images[0].url} alt="Album cover" class="md:hidden w-[20rem] sm:w-[24rem] h-[20rem] sm:h-[24rem] rounded-2xl shadow-md" />
                <header class='flex flex-col text-white mt-3 md:mt-0 md:ml-12 h-[32rem] md:pb-12 md:justify-end md:items-start'>
                    <h1 class="font-bold truncate pb-3 whitespace-pre-wrap break-words text-5xl w-screen px-12">
                        {playbackData?.data.item.name}
                    </h1>
                    <p class="text-xl text-white/80">
                        {playbackData?.data.item.artists[0].name} &middot; {playbackData?.data.item.album.name}
                    </p>
                </header>
            </div>
            <div class="mb-24 w-full mt-5 flex items-center justify-center flex-col">
                <button on:click={() => {
                    if(shuffling === false) {
                        startShuffle();
                    } else {
                        console.log('Stopping shuffle');
                        shuffling = false;
                    }
                }} class="flex md:hidden {shuffling ? 'text-green-400 bg-green-500 hover:bg-green-600 hover:bg-opacity-20 border-green-400' : 'text-white bg-white hover:bg-white/10 border-white/10'} bg-opacity-5 border justify-center duration-100 flex items-center gap-2 p-2 px-4 mb-4  rounded-lg">
                    <Icon icon="solar:shuffle-linear" class="{shuffling ? 'text-green-400' : 'text-white'}" />{shuffling ? 'Shuffling...' : 'Shuffle'}
                </button>
                <div class="text-white flex flex-row items-center justify-center w-full">
                    <p>{formatTime(playbackData?.data.progress_ms)}</p>
                    <div class="w-full"> 
                        <div class="h-1 bg-white/20 rounded-full mx-4">
                            <div class="h-full bg-white/80 rounded-full" style="width: {playbackData?.data.progress_ms / playbackData?.data.item.duration_ms * 100}%"></div>
                        </div>
                    </div>
                    <p>{formatTime(playbackData?.data.item.duration_ms)}</p>
                </div>
                <div class='text-white w-full flex gap-4 z-20 items-center justify-center mt-5 relative'>
                    <button on:click={() => {
                        if(shuffling === false) {
                            startShuffle();
                        } else {
                            console.log('Stopping shuffle');
                            shuffling = false;
                        }
                    }} class="hidden md:flex {shuffling ? 'text-green-400 bg-green-500 hover:bg-green-600 hover:bg-opacity-20 border-green-400' : 'text-white bg-white hover:bg-white/10 border-white/10'} bg-opacity-5 border justify-center duration-100 flex items-center gap-2 p-2 px-4 absolute left-0 rounded-lg">
                        <Icon icon="solar:shuffle-linear" class="{shuffling ? 'text-green-400' : 'text-white'}" />{shuffling ? 'Shuffling...' : 'Shuffle'}
                    </button>
                    <button on:click={() => {controlPlayback('previous')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
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
                    <button on:click={() => {controlPlayback('next')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
                        <Icon icon="ic:sharp-skip-next" class="h-full w-full" />
                    </button>
                </div>
            </div>
        </main>
    {:else if playbackData?.data.is_playing === false}
        <img src={playbackData?.data.item.album.images[0].url} class="w-screen h-screen blur-lg opacity-20 object-cover absolute z-0 shadow-md" />
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

        <main class="w-full h-full overflow-hidden z-10 flex items-center justify-end md:pt-24 px-8 lg:px-32 flex-col relative">
            <div class="md:items-end md:flex hidden h-full justify-start text-center md:text-left md:justify-start w-full flex-col relative md:flex-row">
                <img on:click={() => {
                    goto(`https://open.spotify.com/track/${playbackData?.data.item.id}`);
                }} src={playbackData?.data.item.album.images[0].url} alt="Album cover" class="md:w-[32rem] hover:cursor-pointer w-[20rem] sm:w-[24rem] h-[20rem] sm:h-[24rem] md:h-[32rem] rounded-2xl shadow-md" />
                <header class='flex flex-col w-full text-white mt-2 md:mt-0 md:pl-12 h-[32rem] md:pb-12 md:justify-end md:items-start'>
                    <h1 class="font-bold text-4xl truncate h-12 xl:h-16 xl:text-5xl w-[80%]">
                        {playbackData?.data.item.name}
                    </h1>
                    <p class="text-xl text-white/80">
                        {playbackData?.data.item.artists[0].name} &middot; {playbackData?.data.item.album.name}
                    </p>
                </header>
            </div>
            <div class="md:hidden flex flex-col items-center justify-center text-center absolute">
                <img src={playbackData?.data.item.album.images[0].url} alt="Album cover" class="md:hidden w-[20rem] sm:w-[24rem] h-[20rem] sm:h-[24rem] rounded-2xl shadow-md" />
                <header class='flex flex-col text-white mt-3 md:mt-0 md:ml-12 h-[32rem] md:pb-12 md:justify-end md:items-start'>
                    <h1 class="font-bold truncate pb-3 whitespace-pre-wrap break-words text-5xl w-screen px-12">
                        {playbackData?.data.item.name}
                    </h1>
                    <p class="text-xl text-white/80">
                        {playbackData?.data.item.artists[0].name} &middot; {playbackData?.data.item.album.name}
                    </p>
                </header>
            </div>
            <div class="mb-24 w-full mt-5 flex items-center justify-center flex-col">
                <button on:click={() => {
                    if(shuffling === false) {
                        startShuffle();
                    } else {
                        console.log('Stopping shuffle');
                        shuffling = false;
                    }
                }} class="flex md:hidden {shuffling ? 'text-green-400 bg-green-500 hover:bg-green-600 hover:bg-opacity-20 border-green-400' : 'text-white bg-white hover:bg-white/10 border-white/10'} bg-opacity-5 border justify-center duration-100 flex items-center gap-2 p-2 px-4 mb-4  rounded-lg">
                    <Icon icon="solar:shuffle-linear" class="{shuffling ? 'text-green-400' : 'text-white'}" />{shuffling ? 'Shuffling...' : 'Shuffle'}
                </button>
                <div class="text-white flex flex-row items-center justify-center w-full">
                    <p>{formatTime(playbackData?.data.progress_ms)}</p>
                    <div class="w-full"> 
                        <div class="h-1 bg-white/20 rounded-full mx-4">
                            <div class="h-full bg-white/80 rounded-full" style="width: {playbackData?.data.progress_ms / playbackData?.data.item.duration_ms * 100}%"></div>
                        </div>
                    </div>
                    <p>{formatTime(playbackData?.data.item.duration_ms)}</p>
                </div>
                <div class='text-white w-full flex gap-4 z-20 items-center justify-center mt-5 relative'>
                    <button on:click={() => {
                        if(shuffling === false) {
                            startShuffle();
                        } else {
                            console.log('Stopping shuffle');
                            shuffling = false;
                        }
                    }} class="hidden md:flex {shuffling ? 'text-green-400 bg-green-500 hover:bg-green-600 hover:bg-opacity-20 border-green-400' : 'text-white bg-white hover:bg-white/10 border-white/10'} bg-opacity-5 border justify-center duration-100 flex items-center gap-2 p-2 px-4 absolute left-0 rounded-lg">
                        <Icon icon="solar:shuffle-linear" class="{shuffling ? 'text-green-400' : 'text-white'}" />{shuffling ? 'Shuffling...' : 'Shuffle'}
                    </button>
                    <button on:click={() => {controlPlayback('previous')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
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
                    <button on:click={() => {controlPlayback('next')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
                        <Icon icon="ic:sharp-skip-next" class="h-full w-full" />
                    </button>
                </div>
            </div>
        </main>
    {:else if access_token && playbackData === undefined || null || ''}
        <img src={'https://www.shutterstock.com/blog/wp-content/uploads/sites/5/2020/02/Usign-Gradients-Featured-Image.jpg'} class="w-screen h-screen blur-lg opacity-[0.05] mix-blend-luminosity object-cover absolute z-0 shadow-md" />
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

        <main class="w-full h-full overflow-hidden z-10 flex items-center justify-end md:pt-24 px-8 lg:px-32 flex-col relative">
            <div class="md:items-end md:flex hidden h-full justify-start text-center md:text-left md:justify-start w-full flex-col relative md:flex-row">
                <div class="md:w-[32rem] flex-shrink-0 bg-white/40 w-[20rem] sm:w-[24rem] h-[20rem] sm:h-[24rem] md:h-[32rem] rounded-2xl shadow-md" />
                <header class='flex flex-col w-full text-white mt-2 md:mt-0 md:pl-12 h-[32rem] md:pb-12 md:justify-end md:items-start'>
                    <h1 class="font-bold text-4xl truncate h-12 xl:h-16 xl:text-5xl w-[80%]">
                        Currently not playing
                    </h1>
                    <p class="text-xl text-white/80">
                        Play a song on Spotify to get started
                    </p>
                </header>
            </div>
            <div class="md:hidden flex flex-col items-center justify-center text-center absolute">
                <img src={playbackData?.data.item.album.images[0].url} alt="Album cover" class="md:hidden w-[20rem] sm:w-[24rem] h-[20rem] sm:h-[24rem] rounded-2xl shadow-md" />
                <header class='flex flex-col text-white mt-3 md:mt-0 md:ml-12 h-[32rem] md:pb-12 md:justify-end md:items-start'>
                    <h1 class="font-bold truncate pb-3 whitespace-pre-wrap break-words text-5xl w-screen px-12">
                        {playbackData?.data.item.name}
                    </h1>
                    <p class="text-xl text-white/80">
                        {playbackData?.data.item.artists[0].name} &middot; {playbackData?.data.item.album.name}
                    </p>
                </header>
            </div>
            <div class="mb-24 w-full mt-5 flex items-center justify-center flex-col">
                <div class="text-white flex flex-row items-center justify-center w-full">
                    <p>00:00</p>
                    <div class="w-full"> 
                        <div class="h-1 bg-white/20 rounded-full mx-4">
                            <div class="h-full bg-white/80 rounded-full" style="width: {playbackData?.data.progress_ms / playbackData?.data.item.duration_ms * 100}%"></div>
                        </div>
                    </div>
                    <p>00:00</p>
                </div>
                <div class='text-white w-full flex gap-4 z-20 items-center justify-center mt-5 relative'>
                    <button on:click={() => {controlPlayback('previous')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
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
                    <button on:click={() => {controlPlayback('next')}} class="text-white hover:text-white/50 h-16 w-16 duration-100 rounded-lg">
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
