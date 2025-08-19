<script lang="ts">
import Icon from "@iconify/svelte";
import { onDestroy, onMount } from "svelte";
import { slide } from "svelte/transition";
import { musicPlayerConfig } from "../../config";
import Key from "../../i18n/i18nKey";
import { i18n } from "../../i18n/translation";

// 新增：播放器模式，可选 "local" 或 "meting"
let mode = musicPlayerConfig.mode ?? "meting"; // local or meting
let meting_api = musicPlayerConfig.meting_api ?? "域名/api?server=:server&type=:type&id=:id&auth=:auth&r=:r";
let meting_id = musicPlayerConfig.id ?? "7039487815";
let meting_server = musicPlayerConfig.server ?? "netease";
let meting_type = musicPlayerConfig.type ?? "playlist";

let isPlaying = false;
let isExpanded = false;
let isHidden = false;
let showPlaylist = false;
let currentTime = 0;
let duration = 0;
let volume = 0.7;
let isMuted = false;
let isLoading = false;
let isShuffled = false;
let isRepeating = 0; // 0: 不循环, 1: 单曲循环, 2: 列表循环
let errorMessage = "";
let showError = false;

// 当前歌曲信息
let currentSong = {
    title: "示例歌曲",
    artist: "示例艺术家",
    cover: "/favicon/favicon-light-192.png",
    url: "",
    duration: 0,
};

let playlist = [];
let currentIndex = 0;
let audio: HTMLAudioElement;
let progressBar: HTMLElement;
let volumeBar: HTMLElement;

// 本地音乐列表
const localPlaylist = [
    {
        id: 1,
        title: "ひとり上手",
        artist: "Kaya",
        cover: "assets/music/cover/hitori.jpg",
        url: "assets/music/url/hitori.mp3",
        duration: 240,
    },
    {
        id: 2,
        title: "眩耀夜行",
        artist: "スリーズブーケ",
        cover: "assets/music/cover/xryx.jpg",
        url: "assets/music/url/xryx.mp3",
        duration: 180,
    },
    {
        id: 3,
        title: "春雷の頃",
        artist: "22/7",
        cover: "assets/music/cover/cl.jpg",
        url: "assets/music/url/cl.mp3",
        duration: 200,
    },
];

// meting 歌单获取
async function fetchMetingPlaylist() {
    if (!meting_api || !meting_id) return;
    isLoading = true;
    const apiUrl = meting_api
        .replace(":server", meting_server)
        .replace(":type", meting_type)
        .replace(":id", meting_id)
        .replace(":auth", "")
        .replace(":r", Date.now().toString());
    try {
        const res = await fetch(apiUrl);
        if (!res.ok) throw new Error("meting api error");
        const list = await res.json();
        playlist = list.map((song) => {
            let title = song.name ?? song.title ?? "未知歌曲";
            let artist = song.artist ?? song.author ?? "未知艺术家";
            let dur = song.duration ?? 0;
            if (dur > 10000) dur = Math.floor(dur / 1000); // 毫秒转秒
            if (!Number.isFinite(dur) || dur <= 0) dur = 0;
            return {
                id: song.id,
                title,
                artist,
                cover: song.pic ?? "",
                url: song.url ?? "",
                duration: dur,
            };
        });
        if (playlist.length > 0) {
            loadSong(playlist[0]);
        }
        isLoading = false;
    } catch (e) {
        showErrorMessage("Meting 歌单获取失败");
        isLoading = false;
    }
}

// 切换播放/暂停
function togglePlay() {
    if (!audio || !currentSong.url) return;
    if (isPlaying) {
        audio.pause();
    } else {
        audio.play();
    }
}

// 切换展开/收缩状态
function toggleExpanded() {
    isExpanded = !isExpanded;
    if (isExpanded) {
        showPlaylist = false;
        isHidden = false;
    }
}

// 切换隐藏状态
function toggleHidden() {
    isHidden = !isHidden;
    if (isHidden) {
        isExpanded = false;
        showPlaylist = false;
    }
}

// 切换播放列表显示
function togglePlaylist() {
    showPlaylist = !showPlaylist;
}

// 切换随机播放
function toggleShuffle() {
    isShuffled = !isShuffled;
}

// 切换循环模式
function toggleRepeat() {
    isRepeating = (isRepeating + 1) % 3;
}

// 上一首
function previousSong() {
    if (playlist.length <= 1) return;
    const newIndex = currentIndex > 0 ? currentIndex - 1 : playlist.length - 1;
    playSong(newIndex);
}

// 下一首
function nextSong() {
    if (playlist.length <= 1) return;
    let newIndex: number;
    if (isShuffled) {
        do {
            newIndex = Math.floor(Math.random() * playlist.length);
        } while (newIndex === currentIndex && playlist.length > 1);
    } else {
        newIndex = currentIndex < playlist.length - 1 ? currentIndex + 1 : 0;
    }
    playSong(newIndex);
}

// 播放指定歌曲
function playSong(index: number) {
    if (index < 0 || index >= playlist.length) return;
    const wasPlaying = isPlaying;
    currentIndex = index;
    if (audio) audio.pause();
    loadSong(playlist[currentIndex]);
    if (wasPlaying || !isPlaying) {
        setTimeout(() => {
            if (!audio) return;
            if (audio.readyState >= 2) {
                audio.play().catch(() => {});
            } else {
                audio.addEventListener("canplay", () => {
                    audio.play().catch(() => {});
                }, { once: true });
            }
        }, 100);
    }
}

// 获取正确的资源路径
function getAssetPath(path: string): string {
    if (path.startsWith("http://") || path.startsWith("https://")) return path;
    if (path.startsWith("/")) return path;
    return `/${path}`;
}

// 加载歌曲
function loadSong(song: typeof currentSong) {
    if (!song || !audio) return;
    currentSong = { ...song };
    if (song.url) {
        isLoading = true;
        audio.pause();
        audio.currentTime = 0;
        currentTime = 0;
        duration = song.duration ?? 0;
        audio.removeEventListener("loadeddata", handleLoadSuccess);
        audio.removeEventListener("error", handleLoadError);
        audio.removeEventListener("loadstart", handleLoadStart);
        audio.addEventListener("loadeddata", handleLoadSuccess, { once: true });
        audio.addEventListener("error", handleLoadError, { once: true });
        audio.addEventListener("loadstart", handleLoadStart, { once: true });
        audio.src = getAssetPath(song.url);
        audio.load();
    } else {
        isLoading = false;
    }
}

function handleLoadSuccess() {
    isLoading = false;
    if (audio?.duration && audio.duration > 1) {
        duration = Math.floor(audio.duration);
        if (playlist[currentIndex]) playlist[currentIndex].duration = duration;
        currentSong.duration = duration;
    }
}

function handleLoadError(event: Event) {
    isLoading = false;
    showErrorMessage(`无法播放 "${currentSong.title}"，正在尝试下一首...`);
    if (playlist.length > 1) setTimeout(() => nextSong(), 1000);
    else showErrorMessage("播放列表中没有可用的歌曲");
}

function handleLoadStart() {}

function showErrorMessage(message: string) {
    errorMessage = message;
    showError = true;
    setTimeout(() => { showError = false; }, 3000);
}
function hideError() { showError = false; }

function setProgress(event: MouseEvent) {
    if (!audio || !progressBar) return;
    const rect = progressBar.getBoundingClientRect();
    const percent = (event.clientX - rect.left) / rect.width;
    const newTime = percent * duration;
    audio.currentTime = newTime;
    currentTime = newTime;
}

function setVolume(event: MouseEvent) {
    if (!audio || !volumeBar) return;
    const rect = volumeBar.getBoundingClientRect();
    const percent = Math.max(0, Math.min(1, (event.clientX - rect.left) / rect.width));
    volume = percent;
    audio.volume = volume;
    isMuted = volume === 0;
}

function toggleMute() {
    if (!audio) return;
    isMuted = !isMuted;
    audio.muted = isMuted;
}

function formatTime(seconds: number): string {
    if (!Number.isFinite(seconds) || seconds < 0) return "0:00";
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins}:${secs.toString().padStart(2, "0")}`;
}

function handleAudioEvents() {
    if (!audio) return;
    audio.addEventListener("play", () => { isPlaying = true; });
    audio.addEventListener("pause", () => { isPlaying = false; });
    audio.addEventListener("timeupdate", () => { currentTime = audio.currentTime; });
    audio.addEventListener("ended", () => {
        if (isRepeating === 1) {
            audio.currentTime = 0;
            audio.play().catch(() => {});
        } else if (isRepeating === 2 || currentIndex < playlist.length - 1 || isShuffled) {
            nextSong();
        } else {
            isPlaying = false;
        }
    });
    audio.addEventListener("error", (event) => { isLoading = false; });
    audio.addEventListener("stalled", () => {});
    audio.addEventListener("waiting", () => {});
}

onMount(() => {
    audio = new Audio();
    audio.volume = volume;
    handleAudioEvents();
    if (mode === "meting") {
        fetchMetingPlaylist();
    } else {
        playlist = localPlaylist;
        if (playlist.length > 0) loadSong(playlist[0]);
    }
});

onDestroy(() => {
    if (audio) {
        audio.pause();
        audio.src = "";
    }
});
</script>

{#if musicPlayerConfig.enable}
{#if showError}
<div class="fixed bottom-20 left-4 z-[60] max-w-sm">
    <div class="bg-red-500 text-white px-4 py-3 rounded-lg shadow-lg flex items-center gap-3 animate-slide-up">
        <Icon icon="material-symbols:error" class="text-xl flex-shrink-0" />
        <span class="text-sm flex-1">{errorMessage}</span>
        <button on:click={hideError} class="text-white/80 hover:text-white transition-colors">
            <Icon icon="material-symbols:close" class="text-lg" />
        </button>
    </div>
</div>
{/if}

<div class="music-player fixed bottom-4 left-4 z-50 transition-all duration-300 ease-in-out"
     class:expanded={isExpanded}
     class:hidden-mode={isHidden}>
    <!-- 隐藏状态的小圆球 -->
    <div class="orb-player w-12 h-12 bg-[var(--primary)] rounded-full shadow-lg cursor-pointer transition-all duration-500 ease-in-out flex items-center justify-center hover:scale-110 active:scale-95"
         class:opacity-0={!isHidden}
         class:scale-0={!isHidden}
         class:pointer-events-none={!isHidden}
         on:click={toggleHidden}
         on:keydown={(e) => {
            if (e.key === 'Enter' || e.key === ' ') {
                e.preventDefault();
                toggleHidden();
            }
         }}
         role="button"
         tabindex="0"
         aria-label="显示音乐播放器">
        {#if isLoading}
            <Icon icon="eos-icons:loading" class="text-white text-lg" />
        {:else if isPlaying}
            <div class="flex space-x-0.5">
                <div class="w-0.5 h-3 bg-white rounded-full animate-pulse"></div>
                <div class="w-0.5 h-4 bg-white rounded-full animate-pulse" style="animation-delay: 150ms;"></div>
                <div class="w-0.5 h-2 bg-white rounded-full animate-pulse" style="animation-delay: 300ms;"></div>
            </div>
        {:else}
            <Icon icon="material-symbols:music-note" class="text-white text-lg" />
        {/if}
    </div>
    <!-- 收缩状态的迷你播放器 -->
    <div class="mini-player card-base bg-[var(--float-panel-bg)] shadow-xl rounded-2xl p-3 transition-all duration-500 ease-in-out"
         class:opacity-0={isExpanded || isHidden}
         class:scale-95={isExpanded || isHidden}
         class:pointer-events-none={isExpanded || isHidden}
         on:click={toggleExpanded}
         on:keydown={(e) => {
            if (e.key === 'Enter' || e.key === ' ') {
                e.preventDefault();
                toggleExpanded();
            }
         }}
         role="button"
         tabindex="0"
         aria-label="展开音乐播放器">
        <div class="flex items-center gap-3 cursor-pointer">
            <div class="cover-container relative w-12 h-12 rounded-xl overflow-hidden">
                <img src={getAssetPath(currentSong.cover)} alt="封面"
                     class="w-full h-full object-cover transition-transform duration-300"
                     class:animate-spin={isPlaying && !isLoading}
                     class:animate-pulse={isLoading} />
                <div class="absolute inset-0 bg-black/20 flex items-center justify-center opacity-0 hover:opacity-100 transition-opacity">
                    {#if isLoading}
                        <Icon icon="eos-icons:loading" class="text-white text-xl" />
                    {:else if isPlaying}
                        <Icon icon="material-symbols:pause" class="text-white text-xl" />
                    {:else}
                        <Icon icon="material-symbols:play-arrow" class="text-white text-xl" />
                    {/if}
                </div>
            </div>
            <div class="flex-1 min-w-0">
                <div class="text-sm font-medium text-90 truncate">{currentSong.title}</div>
                <div class="text-xs text-50 truncate">{currentSong.artist}</div>
            </div>
            <div class="flex items-center gap-1">
                <button class="btn-plain w-8 h-8 rounded-lg flex items-center justify-center"
                        on:click|stopPropagation={toggleHidden}
                        title="隐藏播放器">
                    <Icon icon="material-symbols:visibility-off" class="text-lg" />
                </button>
                <button class="btn-plain w-8 h-8 rounded-lg flex items-center justify-center"
                        on:click|stopPropagation={toggleExpanded}>
                    <Icon icon="material-symbols:expand-less" class="text-lg" />
                </button>
            </div>
        </div>
    </div>
    <!-- 展开状态的完整播放器 -->
    <div class="expanded-player card-base bg-[var(--float-panel-bg)] shadow-xl rounded-2xl p-4 transition-all duration-500 ease-in-out"
         class:opacity-0={!isExpanded}
         class:scale-95={!isExpanded}
         class:pointer-events-none={!isExpanded}>
        <div class="flex items-center gap-4 mb-4">
            <div class="cover-container relative w-16 h-16 rounded-xl overflow-hidden flex-shrink-0">
                <img src={getAssetPath(currentSong.cover)} alt="封面"
                     class="w-full h-full object-cover transition-transform duration-300"
                     class:animate-spin={isPlaying && !isLoading}
                     class:animate-pulse={isLoading} />
            </div>
            <div class="flex-1 min-w-0">
                <div class="song-title text-lg font-bold text-90 truncate mb-1">{currentSong.title}</div>
                <div class="song-artist text-sm text-50 truncate">{currentSong.artist}</div>
                <div class="text-xs text-30 mt-1">
                    {formatTime(currentTime)} / {formatTime(duration)}
                </div>
            </div>
            <div class="flex items-center gap-1">
                <button class="btn-plain w-8 h-8 rounded-lg flex items-center justify-center"
                        on:click={toggleHidden}
                        title="隐藏播放器">
                    <Icon icon="material-symbols:visibility-off" class="text-lg" />
                </button>
                <button class="btn-plain w-8 h-8 rounded-lg flex items-center justify-center"
                        on:click={toggleExpanded}>
                    <Icon icon="material-symbols:expand-more" class="text-lg" />
                </button>
            </div>
        </div>
        <div class="progress-section mb-4">
            <div class="progress-bar flex-1 h-2 bg-[var(--btn-regular-bg)] rounded-full cursor-pointer"
                 bind:this={progressBar}
                 on:click={setProgress}
                 on:keydown={(e) => {
                     if (e.key === 'Enter' || e.key === ' ') {
                         e.preventDefault();
                         const rect = progressBar.getBoundingClientRect();
                         const percent = 0.5;
                         const newTime = percent * duration;
                         if (audio) {
                             audio.currentTime = newTime;
                             currentTime = newTime;
                         }
                     }
                 }}
                 role="slider"
                 tabindex="0"
                 aria-label="播放进度"
                 aria-valuemin="0"
                 aria-valuemax="100"
                 aria-valuenow={duration > 0 ? (currentTime / duration * 100) : 0}>
                <div class="h-full bg-[var(--primary)] rounded-full transition-all duration-100"
                     style="width: {duration > 0 ? (currentTime / duration) * 100 : 0}%"></div>
            </div>
        </div>
        <div class="controls flex items-center justify-center gap-2 mb-4">
            <button class="btn-plain w-10 h-10 rounded-lg"
                    class:text-[var(--primary)]={isShuffled}
                    on:click={toggleShuffle}
                    disabled={playlist.length <= 1}>
                <Icon icon="material-symbols:shuffle" class="text-lg" />
            </button>
            <button class="btn-plain w-10 h-10 rounded-lg" on:click={previousSong}
                    disabled={playlist.length <= 1}>
                <Icon icon="material-symbols:skip-previous" class="text-xl" />
            </button>
            <button class="btn-regular w-12 h-12 rounded-full"
                    class:opacity-50={isLoading}
                    disabled={isLoading}
                    on:click={togglePlay}>
                {#if isLoading}
                    <Icon icon="eos-icons:loading" class="text-xl" />
                {:else if isPlaying}
                    <Icon icon="material-symbols:pause" class="text-xl" />
                {:else}
                    <Icon icon="material-symbols:play-arrow" class="text-xl" />
                {/if}
            </button>
            <button class="btn-plain w-10 h-10 rounded-lg" on:click={nextSong}
                    disabled={playlist.length <= 1}>
                <Icon icon="material-symbols:skip-next" class="text-xl" />
            </button>
            <button class="btn-plain w-10 h-10 rounded-lg"
                    class:text-[var(--primary)]={isRepeating > 0}
                    on:click={toggleRepeat}>
                {#if isRepeating === 1}
                    <Icon icon="material-symbols:repeat-one" class="text-lg" />
                {:else if isRepeating === 2}
                    <Icon icon="material-symbols:repeat" class="text-lg" />
                {:else}
                    <Icon icon="material-symbols:repeat" class="text-lg opacity-50" />
                {/if}
            </button>
        </div>
        <div class="bottom-controls flex items-center gap-2">
            <button class="btn-plain w-8 h-8 rounded-lg" on:click={toggleMute}>
                {#if isMuted || volume === 0}
                    <Icon icon="material-symbols:volume-off" class="text-lg" />
                {:else if volume < 0.5}
                    <Icon icon="material-symbols:volume-down" class="text-lg" />
                {:else}
                    <Icon icon="material-symbols:volume-up" class="text-lg" />
                {/if}
            </button>
            <div class="flex-1 h-2 bg-[var(--btn-regular-bg)] rounded-full cursor-pointer"
                 bind:this={volumeBar}
                 on:click={setVolume}
                 on:keydown={(e) => {
                     if (e.key === 'Enter' || e.key === ' ') {
                         e.preventDefault();
                         if (e.key === 'Enter') toggleMute();
                     }
                 }}
                 role="slider"
                 tabindex="0"
                 aria-label="音量控制"
                 aria-valuemin="0"
                 aria-valuemax="100"
                 aria-valuenow={volume * 100}>
                <div class="h-full bg-[var(--primary)] rounded-full transition-all duration-100"
                     style="width: {volume * 100}%"></div>
            </div>
            <button class="btn-plain w-8 h-8 rounded-lg"
                    class:text-[var(--primary)]={showPlaylist}
                    on:click={togglePlaylist}>
                <Icon icon="material-symbols:queue-music" class="text-lg" />
            </button>
        </div>
    </div>
    {#if showPlaylist}
        <div class="playlist-panel float-panel fixed bottom-20 left-4 w-80 max-h-96 overflow-hidden z-50"
             transition:slide={{ duration: 300, axis: 'y' }}>
            <div class="playlist-header flex items-center justify-between p-4 border-b border-[var(--line-divider)]">
                <h3 class="text-lg font-semibold text-90">{i18n(Key.playlist)}</h3>
                <button class="btn-plain w-8 h-8 rounded-lg" on:click={togglePlaylist}>
                    <Icon icon="material-symbols:close" class="text-lg" />
                </button>
            </div>
            <div class="playlist-content overflow-y-auto max-h-80">
                {#each playlist as song, index}
                    <div class="playlist-item flex items-center gap-3 p-3 hover:bg-[var(--btn-plain-bg-hover)] cursor-pointer transition-colors"
                         class:bg-[var(--btn-plain-bg)]={index === currentIndex}
                         class:text-[var(--primary)]={index === currentIndex}
                         on:click={() => playSong(index)}
                         on:keydown={(e) => {
                             if (e.key === 'Enter' || e.key === ' ') {
                                 e.preventDefault();
                                 playSong(index);
                             }
                         }}
                         role="button"
                         tabindex="0"
                         aria-label="播放 {song.title} - {song.artist}">
                        <div class="w-6 h-6 flex items-center justify-center">
                            {#if index === currentIndex && isPlaying}
                                <Icon icon="material-symbols:graphic-eq" class="text-[var(--primary)] animate-pulse" />
                            {:else if index === currentIndex}
                                <Icon icon="material-symbols:pause" class="text-[var(--primary)]" />
                            {:else}
                                <span class="text-sm text-[var(--content-meta)]">{index + 1}</span>
                            {/if}
                        </div>
                        <div class="w-10 h-10 rounded-lg overflow-hidden bg-[var(--btn-regular-bg)] flex-shrink-0">
                            <img src={getAssetPath(song.cover)} alt={song.title} class="w-full h-full object-cover" />
                        </div>
                        <div class="flex-1 min-w-0">
                            <div class="font-medium truncate" class:text-[var(--primary)]={index === currentIndex} class:text-90={index !== currentIndex}>
                                {song.title}
                            </div>
                            <div class="text-sm text-[var(--content-meta)] truncate" class:text-[var(--primary)]={index === currentIndex}>
                                {song.artist}
                            </div>
                        </div>
                        <!-- 歌单时间移除 -->
                    </div>
                {/each}
            </div>
        </div>
    {/if}
</div>

<style>
/* 小圆球播放器样式 */
.orb-player {
	position: relative;
	backdrop-filter: blur(10px);
	-webkit-backdrop-filter: blur(10px);
}

.orb-player::before {
	content: '';
	position: absolute;
	inset: -2px;
	background: linear-gradient(45deg, var(--primary), transparent, var(--primary));
	border-radius: 50%;
	z-index: -1;
	opacity: 0;
	transition: opacity 0.3s ease;
}

.orb-player:hover::before {
	opacity: 0.3;
	animation: rotate 2s linear infinite;
}

/* 播放状态的音波动画 */
.orb-player .animate-pulse {
	animation: musicWave 1.5s ease-in-out infinite;
}

@keyframes rotate {
	from { transform: rotate(0deg); }
	to { transform: rotate(360deg); }
}

@keyframes musicWave {
	0%, 100% { transform: scaleY(0.5); }
	50% { transform: scaleY(1); }
}

/* 隐藏模式下的容器调整 */
.music-player.hidden-mode {
	width: 48px;
	height: 48px;
}

/* 过渡动画优化 */
.orb-player {
	transform-origin: center;
	position: absolute;
	bottom: 0;
	left: 0;
}

.mini-player {
	transform-origin: bottom left;
}

.expanded-player {
	transform-origin: bottom left;
}

/* 确保播放器始终固定在左下角 */
.music-player {
	position: fixed !important;
	bottom: 1rem !important;
	left: 1rem !important;
}

/* 确保动画期间元素不会闪烁 */
.opacity-0 {
	visibility: hidden;
}

.pointer-events-none {
	pointer-events: none;
}

.music-player {
    max-width: 320px;
    user-select: none;
}

.mini-player {
    width: 280px;
    position: absolute;
    bottom: 0;
    left: 0;
}

.expanded-player {
    width: 320px;
}

.cover-container img.animate-spin {
    animation: spin 3s linear infinite;
}

@keyframes spin {
    from {
        transform: rotate(0deg);
    }
    to {
        transform: rotate(360deg);
    }
}

.animate-pulse {
    animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

@keyframes pulse {
    0%, 100% {
        opacity: 1;
    }
    50% {
        opacity: 0.5;
    }
}

/* 进度条和音量条的交互效果 */
.progress-section div:hover,
.bottom-controls > div:hover {
    transform: scaleY(1.2);
    transition: transform 0.2s ease;
}

/* 响应式设计 */
@media (max-width: 768px) {
    .music-player {
        max-width: 280px;
        left: 8px !important;
        bottom: 8px !important;
    }
    
    .music-player.expanded {
        width: calc(100vw - 16px);
        max-width: none;
        left: 8px !important;
        right: 8px !important;
    }
    
    .playlist-panel {
        width: calc(100vw - 16px) !important;
        left: 8px !important;
        right: 8px !important;
        max-width: none;
    }
    
    .controls {
        gap: 8px;
    }
    
    .controls button {
        width: 36px;
        height: 36px;
    }
    
    .controls button:nth-child(3) {
        width: 44px;
        height: 44px;
    }
}

@media (max-width: 480px) {
    .music-player {
        max-width: 260px;
    }
    
    .song-title {
        font-size: 14px;
    }
    
    .song-artist {
        font-size: 12px;
    }
    
    .controls {
        gap: 6px;
        margin-bottom: 12px;
    }
    
    .controls button {
        width: 32px;
        height: 32px;
    }
    
    .controls button:nth-child(3) {
        width: 40px;
        height: 40px;
    }
    
    .playlist-item {
        padding: 8px 12px;
    }
    
    .playlist-item .w-10 {
        width: 32px;
        height: 32px;
    }
}

/* 错误提示动画 */
@keyframes slide-up {
    from {
        transform: translateY(100%);
        opacity: 0;
    }
    to {
        transform: translateY(0);
        opacity: 1;
    }
}

.animate-slide-up {
    animation: slide-up 0.3s ease-out;
}

/* 触摸设备优化 */
@media (hover: none) and (pointer: coarse) {
    .music-player button,
    .playlist-item {
        min-height: 44px;
    }
    
    .progress-section > div,
    .bottom-controls > div:nth-child(2) {
        height: 12px;
    }
}
</style>
{/if}