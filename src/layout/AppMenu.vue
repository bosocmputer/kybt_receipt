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
    const menu = [
        {
            label: 'Home',
            items: [{ label: 'Dashboard', icon: 'pi pi-fw pi-home', to: '/' }]
        }
    ];

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

    // เพิ่มเมนูอื่นๆ ตามสิทธิ์
    if (permissions.value) {
        const items = [];

        if (permissions.value.receive_screen === '1') {
            items.push({
                label: 'รับเอกสาร',
                icon: 'pi pi-fw pi-inbox',
                to: '/pages/receive'
            });
            items.push({
                label: 'ใบรับสินค้า',
                icon: 'pi pi-fw pi-file-check',
                to: '/pages/receivedoc'
            });
        }

        if (permissions.value.history_screen === '1') {
            items.push({
                label: 'ประวัติ',
                icon: 'pi pi-fw pi-history',
                to: '/pages/history'
            });
        }

        if (items.length > 0) {
            menu.push({
                label: 'เมนูหลัก',
                items
            });
        }
    }

    // เมนู UI Components (เก็บไว้สำหรับ development)
    menu.push({
        label: 'UI Components',
        items: [
            { label: 'Form Layout', icon: 'pi pi-fw pi-id-card', to: '/uikit/formlayout' },
            { label: 'Input', icon: 'pi pi-fw pi-check-square', to: '/uikit/input' },
            { label: 'Button', icon: 'pi pi-fw pi-mobile', to: '/uikit/button', class: 'rotated-icon' },
            { label: 'Table', icon: 'pi pi-fw pi-table', to: '/uikit/table' },
            { label: 'List', icon: 'pi pi-fw pi-list', to: '/uikit/list' },
            { label: 'Tree', icon: 'pi pi-fw pi-share-alt', to: '/uikit/tree' },
            { label: 'Panel', icon: 'pi pi-fw pi-tablet', to: '/uikit/panel' },
            { label: 'Overlay', icon: 'pi pi-fw pi-clone', to: '/uikit/overlay' },
            { label: 'Media', icon: 'pi pi-fw pi-image', to: '/uikit/media' },
            { label: 'Menu', icon: 'pi pi-fw pi-bars', to: '/uikit/menu' },
            { label: 'Message', icon: 'pi pi-fw pi-comment', to: '/uikit/message' },
            { label: 'File', icon: 'pi pi-fw pi-file', to: '/uikit/file' },
            { label: 'Chart', icon: 'pi pi-fw pi-chart-bar', to: '/uikit/charts' },
            { label: 'Timeline', icon: 'pi pi-fw pi-calendar', to: '/uikit/timeline' },
            { label: 'Misc', icon: 'pi pi-fw pi-circle', to: '/uikit/misc' }
        ]
    });

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
