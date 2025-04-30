<template>
    <aside>
        <div class="logo">
            MoeKoe Music
        </div>
        <div class="sidebar">
            <div class="sidebar-content">
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
                <hr />
                <p>我的</p>
                <ul class="song-list">
                    <li>
                        <a v-if="!isLoading" @click="goToCloudDrive">
                            <img :src="`./assets/images/cloud-disk.png`" class="album-image"/>
                            <div class="album-info">
                                <h3>我的云盘</h3>
                                <p>(*/ω＼*)</p>
                            </div>
                        </a>
                    </li>
                    <li v-for="(item, index) in (userPlaylists)"
                        :key="index">
                        <router-link :to="{
                        path: '/PlaylistDetail',
                        query: { global_collection_id: item.list_create_gid || item.global_collection_id, listid: item.listid }
                    }">
                            <img :src="item.pic ? $getCover(item.pic, 480) : './assets/images/live.png'"
                                 class="album-image"/>
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
                <hr />
                <p>收藏的歌单</p>
                <ul class="song-list">
                    <li v-for="(item, index) in (collectedPlaylists)"
                        :key="index">
                        <router-link :to="{
                        path: '/PlaylistDetail',
                        query: { global_collection_id: item.list_create_gid || item.global_collection_id, listid: item.listid }
                    }">
                            <img :src="item.pic ? $getCover(item.pic, 480) : './assets/images/live.png'"
                                 class="album-image"/>
                            <div class="album-info">
                                <h3>{{ item.name }}</h3>
                                <p>{{ item.count }} <span>{{ $t('shou-ge') }}</span></p>
                            </div>
                        </router-link>
                    </li>
                </ul>
            </div>
        </div>
    </aside>
</template>

<script setup>
import {ref, onMounted} from 'vue';
import {get} from '../utils/request';
import {MoeAuthStore} from '../stores/store';
import {useRouter} from 'vue-router';
import {useI18n} from 'vue-i18n';

const {t} = useI18n();
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
    router.replace({path: '/library', query: {category: index}});
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
    const historyResponse = await get('/user/listen', {type: 1});
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
                if (playlist.name === '我喜欢') {
                    localStorage.setItem('like', playlist.listid);
                }
                return playlist.list_create_userid === user.value.userid || playlist.name === '我喜欢';
            }).sort((a, b) => a.name === '我喜欢' ? -1 : 1);
            collectedPlaylists.value = playlistResponse.data.info.filter(playlist => playlist.list_create_userid !== user.value.userid && !playlist.authors);
            collectedAlbums.value = playlistResponse.data.info.filter(playlist => playlist.list_create_userid !== user.value.userid && playlist.authors);

            const collectedIds = [];
            playlistResponse.data.info.forEach(playlist => {
                if (playlist.list_create_userid !== user.value.userid) {
                    collectedIds.push({list_create_listid: playlist.list_create_listid, listid: playlist.listid});
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
            const playlistResponse = await get('/playlist/add', {name: result, list_create_userid: user.value.userid});
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
</script>

<style scoped>
.logo {
    position: fixed;
    top: 8px;
    height: 84px;
    width: 260px;
    font-size: 1.5rem;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: 500;
}

.sidebar {
    position: fixed;
    top: 92px;
    bottom: 82px;
    left: 0;
    width: 260px;
    box-shadow: 4px 2px 10px rgba(0, 0, 0, 0.1);
    overflow: auto;
    overscroll-behavior: contain;
}

.sidebar-content {
    height: 100%;
    display: flex;
    flex-direction: column;
    padding: 8px 20px;
}

.sidebar-content ul {
    margin: 0;
    list-style-type: none;
    padding: 0;
}

.sidebar-content hr {
    width: 85%;
    margin: 8px auto;
    border: none;
    border-top: 1px solid #ddd;
}

.sidebar-content ul li {
    margin-bottom: 4px;
}

.sidebar-content ul li a {
    text-decoration: none;
    display: flex;
    align-items: center;
    gap: 8px;
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

.sidebar-content p {
    font-size: 1rem;
    margin: 2px 8px;
}

.sidebar-content .playlist-info p {
    margin: 10px 0;
}

.sidebar-content .play-button i {
    font-size: 16px;
}

.sidebar-content .song-list {
    overflow: auto;
}

.sidebar-content .album-image {
    width: 48px;
    height: 48px;
    border-radius: 8px;
}

.sidebar-content .album-info h3 {
    margin: 0;
    font-size: 1rem;
    font-weight: bold;
}

.sidebar-content .album-info p {
    margin: 0;
    color: #888;
    font-size: 14px;
}

</style>