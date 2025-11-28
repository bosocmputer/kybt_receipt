<script setup>
import { ref, computed, onMounted } from 'vue';
import AuthService from '@/service/AuthService';
import AppMenuItem from './AppMenuItem.vue';

const isSuperAdmin = ref(false);
const permissions = ref(null);

onMounted(() => {
    isSuperAdmin.value = AuthService.isSuperAdmin();
    permissions.value = AuthService.getPermissions();
});

const model = computed(() => {
    const menu = [];

    // เพิ่มเมนู "กำหนดสิทธิ์" เฉพาะ SUPERADMIN
    if (isSuperAdmin.value) {
        menu.push({
            label: 'Admin',
            items: [
                {
                    label: 'กำหนดสิทธิ์',
                    icon: 'pi pi-fw pi-shield',
                    to: '/pages/permissions'
                }
            ]
        });
    }

    // เมนูหลักสำหรับ Production
    if (permissions.value && permissions.value.receive_screen === '1') {
        menu.push({
            label: 'เมนูหลัก',
            items: [
                {
                    label: 'ใบรับสินค้า',
                    icon: 'pi pi-fw pi-file-check',
                    to: '/pages/receivedoc'
                },
                {
                    label: 'ปิดงานใบรับ',
                    icon: 'pi pi-fw pi-check-circle',
                    to: '/pages/closejobreceive'
                },
                {
                    label: 'ประวัติใบรับ',
                    icon: 'pi pi-fw pi-history',
                    to: '/pages/receivehistory'
                }
            ]
        });
    }

    return menu;
});
</script>

<template>
    <ul class="layout-menu">
        <template v-for="(item, i) in model" :key="item">
            <app-menu-item v-if="!item.separator" :item="item" :index="i"></app-menu-item>
            <li v-if="item.separator" class="menu-separator"></li>
        </template>
    </ul>
</template>

<style lang="scss" scoped></style>
