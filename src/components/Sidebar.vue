<script setup lang="ts">

</script>

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
        </div>
    </aside>
</template>

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
    list-style-type: none;
    padding: 0;
    border-bottom: 1px solid #ccc;
}

.sidebar-content ul li {
    margin-bottom: 6px;
}

.sidebar-content ul li a {
    text-decoration: none;
    display: block;
    padding: 10px;
    border-radius: 8px;
    transition: 0.2s ease;
}

.sidebar-content ul li a:hover,
.sidebar-content ul li .active {
    color: #fff;
    background-color: var(--primary-color);
}

</style>