<template>
    <aside class="sidebar">
        <div class="sidebar-content">
            <h3>MoeKoe Music</h3>
            <ul>
                <li>
                    <router-link to="/" exact-active-class="active">{{ $t('shou-ye') }}</router-link>
                </li>
                <li>
                    <router-link to="/discover" exact-active-class="active">{{ $t('fa-xian') }}</router-link>
                </li>
                <li>
                    <router-link to="/library" exact-active-class="active">{{ $t('yin-le-ku') }}</router-link>
                </li>
            </ul>
            我的
            <ul class="song-list">
                <li>
                    <a v-if="!isLoading" @click="goToCloudDrive">
                        <img :src="`./assets/images/cloud-disk.png`" class="album-image" />
                        <div class="album-info">
                            <h3>我的云盘</h3>
                            <p>(*/ω＼*)</p>
                        </div>
                    </a>
                </li>
                <li v-for="(item, index) in (selectedCategory === 0 ? userPlaylists : selectedCategory === 1 ? collectedPlaylists : collectedAlbums)"
                    :key="index">
                    <router-link :to="{
                        path: '/PlaylistDetail',
                        query: { global_collection_id: item.list_create_gid || item.global_collection_id, listid: item.listid }
                    }">
                        <img :src="item.pic ? $getCover(item.pic, 480) : './assets/images/live.png'"
                            class="album-image" />
                        <div class="album-info">
                            <h3>{{ item.name }}</h3>
                            <p>{{ item.count }} <span>{{ $t('shou-ge') }}</span></p>
                        </div>
                    </router-link>
                </li>
                <li>
                    <a v-if="!isLoading" @click="createPlaylist">
                        <i class="fas fa-plus"></i>
                        <div class="album-info">
                            <h3>{{ $t('chuang-jian-ge-dan') }}</h3>
                        </div>
                    </a>
                </li>
            </ul>
        </div>
    </aside>
</template>

<script setup>
import { ref, onMounted } from 'vue';
import { get } from '../utils/request';
import { MoeAuthStore } from '../stores/store';
import { useRouter } from 'vue-router';
import { useI18n } from 'vue-i18n';
const { t } = useI18n();
const router = useRouter();
const MoeAuth = MoeAuthStore();
const user = ref({});
const userPlaylists = ref([]); // 创建的歌单
const collectedPlaylists = ref([]); // 收藏的歌单
const collectedAlbums = ref([]); // 收藏的专辑
const collectedFriends = ref([]); // 好友
const followedArtists = ref([]); // 关注的歌手
const listenHistory = ref([]); // 听歌历史
const userVip = ref({});
const categories = ref([t('wo-chuang-jian-de-ge-dan'), t('wo-shou-cang-de-ge-dan'), t('wo-shou-cang-de-zhuan-ji'), t('wo-guan-zhu-de-ge-shou'), t('wo-guan-zhu-de-hao-you')]);
const selectedCategory = ref(0);
const isLoading = ref(true);

const selectCategory = (index) => {
    selectedCategory.value = index;
    router.replace({ path: '/library', query: { category: index } });
};

const playSong = (hash, name, img, author) => {
    props.playerControl.addSongToQueue(hash, name, img, author);
};

const props = defineProps({
    playerControl: Object
});

onMounted(() => {
    if (MoeAuth.isAuthenticated) {
        user.value = MoeAuth.UserInfo;
        // 获取用户vip信息
        getVipInfo();
    }
});
const getUserDetails = () => {
    // 获取用户听歌历史
    getlisten().finally(() => {
        isLoading.value = false;
    })
    // 获取用户创建和收藏的歌单
    getplaylist()
    // 获取用户关注的歌手
    getfollow()
    selectedCategory.value = parseInt(router.currentRoute.value.query.category || 0);
}
const getVipInfo = async () => {
    try {
        const VipInfoResponse = await get('/user/vip/detail');
        if (VipInfoResponse.status === 1) {
            userVip.value = VipInfoResponse.data.busi_vip
            getUserDetails();
        }
    } catch (error) {
        window.$modal.alert(t('deng-lu-shi-xiao-qing-zhong-xin-deng-lu'));
        router.push('/login');
    }
}

const getlisten = async () => {
    const historyResponse = await get('/user/listen', { type: 1 });
    if (historyResponse.status === 1) {
        const allLists = historyResponse.data.lists;
        const shuffled = allLists.sort(() => 0.5 - Math.random());
        listenHistory.value = shuffled.slice(0, 16);
    }
}
const getfollow = async () => {
    const followResponse = await get('/user/follow');
    if (followResponse.status === 1) {
        if (followResponse.data.total == 0) return;
        const artists = followResponse.data.lists.map(artist => ({
            ...artist,
            pic: artist.pic.replace('/100/', '/480/')
        }));
        collectedFriends.value = artists.filter(artist => !artist.singerid);
        followedArtists.value = artists.filter(artist => artist.source == 7);
    }
}
const getplaylist = async () => {
    try {
        const playlistResponse = await get('/user/playlist', {
            pagesize: 100,
            t: localStorage.getItem('t')
        });
        if (playlistResponse.status === 1) {
            userPlaylists.value = playlistResponse.data.info.filter(playlist => {
                if (playlist.name == '我喜欢') {
                    localStorage.setItem('like', playlist.listid);
                }
                return playlist.list_create_userid === user.value.userid || playlist.name === '我喜欢';
            }).sort((a, b) => a.name === '我喜欢' ? -1 : 1);
            collectedPlaylists.value = playlistResponse.data.info.filter(playlist => playlist.list_create_userid !== user.value.userid && !playlist.authors);
            collectedAlbums.value = playlistResponse.data.info.filter(playlist => playlist.list_create_userid !== user.value.userid && playlist.authors);

            const collectedIds = [];
            playlistResponse.data.info.forEach(playlist => {
                if (playlist.list_create_userid !== user.value.userid) {
                    collectedIds.push({ list_create_listid: playlist.list_create_listid, listid: playlist.listid });
                }
            });
            localStorage.setItem('collectedPlaylists', JSON.stringify(collectedIds));
        }
    } catch (error) {
        window.$modal.alert(t('xin-zeng-zhang-hao-qing-xian-zai-guan-fang-ke-hu-duan-zhong-deng-lu-yi-ci'));
    }
}
const createPlaylist = async () => {
    const result = await window.$modal.prompt(t('qing-shu-ru-xin-de-ge-dan-ming-cheng'), '');
    if (result) {
        try {
            const playlistResponse = await get('/playlist/add', { name: result, list_create_userid: user.value.userid });
            if (playlistResponse.status === 1) {
                localStorage.setItem('t', Date.now());
                getplaylist()
            }
        } catch (error) {
            window.$modal.alert(t('chuang-jian-shi-bai'));
        }
    }
}

const goToCloudDrive = () => {
    router.push('/CloudDrive');
}

const goToArtistDetail = (artist) => {
    if (!artist.singerid) return;
    router.push({
        path: '/PlaylistDetail',
        query: {
            singerid: artist.singerid,
            unfollow: true
        }
    });
};
const signIn = async () => {
    try {
        const res = await get('/youth/vip');
        if (res.status === 1) {
            window.$modal.alert(`签到成功，获得${res.data.award_vip_hour}小时VIP时长`);
        }
    } catch (error) {
        window.$modal.alert('签到失败，请勿频繁签到');
    }
}
const getVip = async () => {
    try {
        const vipResponse = await get('/youth/day/vip');
        if (vipResponse.status === 1) {
            window.$modal.alert(`签到成功，获得1天畅听VIP`);
        }
    } catch (error) {
        window.$modal.alert('获取VIP失败, 一天仅限一次');
    }
}
</script>

<style scoped>
.sidebar {
    position: fixed;
    top: 0;
    bottom: 82px;
    left: 0;
    width: 260px;
    box-shadow: 4px 2px 10px rgba(0, 0, 0, 0.1);
}

.sidebar-content {
    height: 100%;
    display: flex;
    flex-direction: column;
    padding: 20px;
}

.sidebar-content ul {
    margin-top: 4px;
    margin-bottom: 16px;
    list-style-type: none;
    padding: 0;
    border-bottom: 1px solid #ccc;
}

.sidebar-content ul li {
    margin-bottom: 6px;
}

.sidebar-content ul li a {
    text-decoration: none;
    display: flex;
    align-items: center;
    gap: 4px;
    padding: 10px;
    border-radius: 8px;
    transition: 0.2s ease;
    cursor: pointer;
}

.sidebar-content ul li a i {
    display: flex;
    justify-content: center;
    align-items: center;
    width: 46px;
    height: 46px;
    border: solid 1px;
    border-radius: 8px;
}

.sidebar-content ul li a:hover,
.sidebar-content ul li .active {
    color: #fff;
    background-color: var(--primary-color);
}


.sign-in {
    cursor: pointer;
    color: var(--primary-color);
    margin-left: 10px;
    border-radius: 5px;
    padding: 2px 8px;
    border: 1px solid var(--primary-color);
    font-size: 12px;
}

.library-page {
    padding: 20px;
}

.user-level {
    width: 50px;
    margin-left: 10px;
    cursor: pointer;
}


.section-title {
    font-size: 28px;
    font-weight: bold;
    margin-bottom: 30px;
    color: var(--primary-color);
}

.profile-section {
    display: flex;
    align-items: center;
}

.profile-pic {
    border-radius: 50%;
    width: 100px;
    height: 100px;
    margin-right: 15px;
}

.favorite-section {
    display: flex;
    justify-content: space-between;
}

.favorite-playlist {
    background-color: var(--background-color);
    padding: 20px;
    border-radius: 12px;
    flex: 1;
    margin-right: 20px;
    border: 1px solid var(--secondary-color);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    margin-bottom: 20px;
}

.playlist-info p {
    margin: 10px 0;
}

.play-button {
    display: inline-flex;
    align-items: center;
    gap: 5px;
    background-color: var(--secondary-color);
    color: white;
    border: none;
    border-radius: 25px;
    padding: 10px 15px;
    cursor: pointer;
}

.play-button i {
    font-size: 16px;
}

.song-list {
    overflow-y: auto;
}

.category-tabs {
    display: flex;
    gap: 20px;
    margin-bottom: 20px;
}

.category-tabs button {
    padding: 10px 15px;
    border: none;
    background-color: #f5f5f5;
    border-radius: 20px;
    cursor: pointer;
}

.category-tabs button.active {
    background-color: var(--primary-color);
    color: white;
}

.music-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    gap: 20px;
}

.album-image {
    width: 48px;
    height: 48px;
    border-radius: 8px;
}

.album-info h3 {
    margin: 0;
    font-size: 16px;
}

.album-info p {
    margin: 0;
    color: #666;
    font-size: 14px;
}

.song-item {
    display: flex;
    align-items: center;
    margin-bottom: 10px;
}

.album-cover {
    width: 50px;
    height: 50px;
    margin-right: 10px;
    border-radius: 5px;
}

.song-info {
    display: flex;
    flex-direction: column;
    justify-content: center;
    max-width: 190px;
}

.album-name,
.singer-name {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.album-name {
    font-weight: bold;
    margin-bottom: -5px;
    font-size: 14px;
    color: #333;
}

.singer-name {
    font-size: 12px;
    color: #666;
}

.skeleton-loader {
    display: flex;
    flex-wrap: wrap;
    justify-content: space-between;
    margin-top: 10px;
}

.skeleton-item {
    display: flex;
    align-items: center;
    margin-bottom: 10px;
    width: 250px;
    border-radius: 10px;
    padding-left: 10px;
    background-color: #f0f0f0;
    height: 68px;
}

.skeleton-cover {
    width: 50px;
    height: 50px;
    margin-right: 10px;
    border-radius: 10px;
    background-color: #e0e0e0;
}

.skeleton-info {
    display: flex;
    flex-direction: column;
    justify-content: center;
    max-width: 190px;
}

.skeleton-line {
    height: 10px;
    background-color: #e0e0e0;
    margin-bottom: 5px;
    border-radius: 5px;
    width: 150px;
}
</style>