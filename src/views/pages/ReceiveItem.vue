<script setup>
import ReceiveDocService from '@/service/ReceiveDocService';
import { useToast } from 'primevue/usetoast';
import { computed, onMounted, ref } from 'vue';
import { useRoute, useRouter } from 'vue-router';

const route = useRoute();
const router = useRouter();
const toast = useToast();

// Document info
const docno = ref(route.params.docno || '');
const loading = ref(false);

// SO Details & Received Items
const soDetails = ref([]);
const receivedDetails = ref([]);

// Search & Scanning
const selectedItem = ref(null);
const searchLoading = ref(false);
const searchResults = ref([]);

// Barcode Scanning
const barcodeInput = ref('');

// Input Mode Selection
const inputMode = ref('barcode'); // 'barcode' or 'search'
const inputModes = ref([
    { name: 'Scan Barcode', value: 'barcode', icon: 'pi pi-qrcode' },
    { name: 'ค้นหาสินค้า', value: 'search', icon: 'pi pi-search' }
]);

// Scanned Items List
const scannedItems = ref([]);

// Summary Dialog
const showSummaryDialog = ref(false);

// SO Details toggle
const showSODetails = ref(false);

onMounted(async () => {
    if (!docno.value) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'ไม่พบเลขที่เอกสาร',
            life: 3000
        });
        router.push({ name: 'receivedoc' });
        return;
    }
    await loadReceiveDocDetail();
});

// โหลดรายละเอียดใบรับสินค้า
async function loadReceiveDocDetail() {
    loading.value = true;
    try {
        const result = await ReceiveDocService.getReceiveDocDetail(docno.value);

        if (result.success) {
            soDetails.value = result.details_so || [];
            receivedDetails.value = result.details_receive || [];

            // ถ้ามีสินค้าที่รับแล้ว ให้โหลดมาแสดงใน scannedItems
            if (receivedDetails.value.length > 0) {
                scannedItems.value = receivedDetails.value.map((item) => ({
                    item_code: item.item_code,
                    item_name: item.item_name || '',
                    unit_code: item.unit_code,
                    barcode: item.barcode || item.item_code,
                    item_year: item.item_year || '',
                    qty: parseInt(item.qty) || 1,
                    max_qty: getMaxQty(item.item_code)
                }));
            }
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to load document detail',
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while loading document detail',
            life: 3000
        });
    } finally {
        loading.value = false;
    }
}

// หาจำนวนสูงสุดที่รับได้สำหรับสินค้าแต่ละตัว
function getMaxQty(itemCode) {
    const soItem = soDetails.value.find((item) => item.item_code === itemCode);
    return soItem ? parseInt(soItem.qty) || 0 : 0;
}

// หาจำนวนที่รับไปแล้วจาก scannedItems
function getScannedQty(itemCode) {
    return scannedItems.value.filter((item) => item.item_code === itemCode).reduce((sum, item) => sum + item.qty, 0);
}

// คำนวณจำนวนคงเหลือที่ยังรับได้
function getRemainingQty(itemCode) {
    return getMaxQty(itemCode) - getScannedQty(itemCode);
}

// แยก Barcode Input รูปแบบ 10*04-101-0000539#2024
function parseBarcodeInput(input) {
    const trimmedInput = input.trim();

    // กำหนดค่า default
    let qty = 1;
    let barcode = '';
    let item_year = '';

    // แยก quantity ถ้ามี * (เช่น 10*)
    let remaining = trimmedInput;
    if (remaining.includes('*')) {
        const parts = remaining.split('*');
        const qtyPart = parseInt(parts[0]);
        if (!isNaN(qtyPart) && qtyPart > 0) {
            qty = qtyPart;
        }
        remaining = parts.slice(1).join('*'); // เอาส่วนหลัง * มาต่อกัน
    }

    // แยก year ถ้ามี # (เช่น #2024)
    if (remaining.includes('#')) {
        const parts = remaining.split('#');
        barcode = parts[0];
        item_year = parts[1] || '';
    } else {
        barcode = remaining;
    }

    return { qty, barcode, item_year };
}

// ประมวลผล Barcode Input
async function processBarcodeInput() {
    if (!barcodeInput.value || !barcodeInput.value.trim()) {
        return;
    }

    const { qty, barcode, item_year } = parseBarcodeInput(barcodeInput.value);

    if (!barcode) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'รูปแบบ Barcode ไม่ถูกต้อง',
            life: 3000
        });
        barcodeInput.value = '';
        return;
    }

    // ค้นหาสินค้าจาก barcode
    searchLoading.value = true;
    try {
        const result = await ReceiveDocService.getItemSearch(barcode);

        if (result.success && result.data && result.data.length > 0) {
            const item = result.data[0];

            // ตรวจสอบว่าสินค้าอยู่ใน SO หรือไม่
            const soItem = soDetails.value.find((so) => so.item_code === item.item_code);

            if (!soItem) {
                toast.add({
                    severity: 'error',
                    summary: 'Error',
                    detail: `สินค้า ${item.item_code} ไม่อยู่ใน SO นี้`,
                    life: 3000
                });
                barcodeInput.value = '';
                searchLoading.value = false;
                return;
            }

            // ตรวจสอบจำนวนที่จะเพิ่ม
            const maxQty = getMaxQty(item.item_code);
            const currentQty = getScannedQty(item.item_code);
            const remainingQty = maxQty - currentQty;

            if (remainingQty <= 0) {
                toast.add({
                    severity: 'error',
                    summary: 'Error',
                    detail: `สินค้า ${item.item_code} รับครบแล้ว (${maxQty} ${item.unit_code})`,
                    life: 3000
                });
                barcodeInput.value = '';
                searchLoading.value = false;
                return;
            }

            // ตรวจสอบจำนวนที่สแกนไม่ให้เกินที่เหลือ
            const actualQty = Math.min(qty, remainingQty);

            if (actualQty < qty) {
                toast.add({
                    severity: 'warn',
                    summary: 'Warning',
                    detail: `สามารถรับได้เพียง ${actualQty} ชิ้น (เหลือ ${remainingQty} ชิ้น)`,
                    life: 3000
                });
            }

            // เพิ่มสินค้าตามจำนวนที่สแกน (หรือจำนวนที่เหลือ)
            let addedCount = 0;
            for (let i = 0; i < actualQty; i++) {
                const success = addItemToScanned(
                    {
                        ...item,
                        barcode: barcode,
                        item_year: item_year,
                        qty: 1
                    },
                    false
                );
                if (success) {
                    addedCount++;
                } else {
                    break; // หยุดถ้าไม่สามารถเพิ่มได้
                }
            }

            if (addedCount > 0) {
                toast.add({
                    severity: 'success',
                    summary: 'Success',
                    detail: `เพิ่มสินค้า ${item.item_code} จำนวน ${addedCount} ชิ้น`,
                    life: 2000
                });
            }
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: `ไม่พบสินค้า Barcode: ${barcode}`,
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'เกิดข้อผิดพลาดในการค้นหา',
            life: 3000
        });
    } finally {
        searchLoading.value = false;
        barcodeInput.value = '';
    }
}

// ค้นหาสินค้า (AutoComplete)
async function handleSearch(event) {
    const query = event.query;

    if (!query || !query.trim()) {
        searchResults.value = [];
        return;
    }

    searchLoading.value = true;
    try {
        // แยก barcode และ year ถ้ามี #
        let searchTerm = query.trim();
        let year = '';

        if (searchTerm.includes('#')) {
            const parts = searchTerm.split('#');
            searchTerm = parts[0];
            year = parts[1] || '';
        }

        const result = await ReceiveDocService.getItemSearch(searchTerm);

        if (result.success && result.data && result.data.length > 0) {
            searchResults.value = result.data.map((item) => ({
                ...item,
                item_year: year
            }));
        } else {
            searchResults.value = [];
        }
    } catch (error) {
        searchResults.value = [];
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'เกิดข้อผิดพลาดในการค้นหา',
            life: 3000
        });
    } finally {
        searchLoading.value = false;
    }
}

// เลือกสินค้าจาก AutoComplete
function onItemSelect(event) {
    if (event.value) {
        addItemToScanned(event.value);
        // Clear search ด้วย timeout เล็กน้อยเพื่อให้ AutoComplete ทำงานเสร็จก่อน
        setTimeout(() => {
            selectedItem.value = null;
            searchResults.value = [];
        }, 100);
    }
}

// เพิ่มสินค้าที่เลือกเข้า scannedItems
function addItemToScanned(item, showToast = true) {
    // ตรวจสอบว่าสินค้านี้อยู่ใน SO หรือไม่
    const soItem = soDetails.value.find((so) => so.item_code === item.item_code);

    if (!soItem) {
        if (showToast) {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: `สินค้า ${item.item_code} ไม่อยู่ใน SO นี้`,
                life: 3000
            });
        }
        return false;
    }

    const maxQty = getMaxQty(item.item_code);
    const currentQty = getScannedQty(item.item_code);

    // ตรวจสอบไม่ให้รับเกิน
    if (currentQty >= maxQty) {
        if (showToast) {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: `สินค้า ${item.item_code} รับครบแล้ว (${maxQty} ${item.unit_code})`,
                life: 3000
            });
        }
        return false;
    }

    // เพิ่มสินค้า
    scannedItems.value.push({
        item_code: item.item_code,
        item_name: item.item_name || '',
        unit_code: item.unit_code,
        barcode: item.barcode || item.item_code,
        item_year: item.item_year || '',
        qty: item.qty || 1,
        max_qty: maxQty
    });

    if (showToast) {
        toast.add({
            severity: 'success',
            summary: 'Success',
            detail: `เพิ่มสินค้า ${item.item_code}`,
            life: 2000
        });
    }

    return true;
}

// ลบสินค้าจาก scannedItems
function removeScannedItem(index) {
    scannedItems.value.splice(index, 1);
}

// อัพเดทจำนวน
function updateQty(index, newQty) {
    const item = scannedItems.value[index];
    const otherQty = scannedItems.value.filter((i, idx) => idx !== index && i.item_code === item.item_code).reduce((sum, i) => sum + i.qty, 0);

    const maxQty = getMaxQty(item.item_code);

    if (newQty + otherQty > maxQty) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: `ไม่สามารถรับเกิน ${maxQty} ${item.unit_code}`,
            life: 3000
        });
        scannedItems.value[index].qty = maxQty - otherQty;
    } else {
        scannedItems.value[index].qty = newQty;
    }
}

// แสดง Summary Dialog
function showSummary() {
    if (scannedItems.value.length === 0) {
        toast.add({
            severity: 'warn',
            summary: 'Warning',
            detail: 'กรุณาเพิ่มสินค้าก่อนบันทึก',
            life: 3000
        });
        return;
    }

    showSummaryDialog.value = true;
}

// สรุปข้อมูลสินค้าที่จะบันทึก (group by item_code)
const summaryItems = computed(() => {
    const grouped = {};

    scannedItems.value.forEach((item) => {
        if (!grouped[item.item_code]) {
            grouped[item.item_code] = {
                item_code: item.item_code,
                item_name: item.item_name,
                unit_code: item.unit_code,
                qty: 0,
                max_qty: item.max_qty
            };
        }
        grouped[item.item_code].qty += item.qty;
    });

    return Object.values(grouped);
});

// คำนวณยอดรวมของ SO ทั้งหมด
const totalSOQty = computed(() => {
    return soDetails.value.reduce((sum, item) => sum + (parseInt(item.qty) || 0), 0);
});

// คำนวณยอดที่สแกนแล้วทั้งหมด
const totalScannedQty = computed(() => {
    return scannedItems.value.reduce((sum, item) => sum + item.qty, 0);
});

// คำนวณ Progress Percentage
const progressPercentage = computed(() => {
    if (totalSOQty.value === 0) return 0;
    return (totalScannedQty.value / totalSOQty.value) * 100;
});

// บันทึกการรับสินค้า (จาก Summary Dialog)
async function saveReceiveDoc() {
    loading.value = true;

    // ปิด dialog หลังจาก set loading เพื่อให้แสดง loading overlay
    setTimeout(() => {
        showSummaryDialog.value = false;
    }, 100);

    try {
        // เตรียมข้อมูลสำหรับ API
        const details = scannedItems.value.map((item) => ({
            barcode: item.barcode,
            item_year: item.item_year || '',
            item_code: item.item_code,
            unit_code: item.unit_code,
            qty: item.qty.toString()
        }));

        const result = await ReceiveDocService.updateReceiveDoc(docno.value, details);

        if (result.success) {
            toast.add({
                severity: 'success',
                summary: 'Success',
                detail: 'บันทึกการรับสินค้าสำเร็จ',
                life: 3000
            });

            // กลับไปหน้ารายการใบรับ
            setTimeout(() => {
                router.push({ name: 'receivedoc' });
            }, 1500);
        } else {
            loading.value = false;
            showSummaryDialog.value = true;
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to save receive document',
                life: 3000
            });
        }
    } catch (error) {
        loading.value = false;
        showSummaryDialog.value = true;
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while saving',
            life: 3000
        });
    }
} // ยกเลิกและกลับไปหน้ารายการ
function cancelAndGoBack() {
    router.push({ name: 'receivedoc' });
}
</script>

<template>
    <ConfirmDialog />
    <div class="min-h-screen bg-surface-50 dark:bg-surface-900">
        <!-- Fixed Header - Enlarged -->
        <div class="sticky top-0 z-10 bg-surface-0 dark:bg-surface-800 border-b border-surface-200 dark:border-surface-700 shadow-md">
            <!-- Top Bar with Document Info and Actions -->
            <div class="flex items-center justify-between p-4 border-b border-surface-200 dark:border-surface-700">
                <div class="flex items-center gap-3 flex-1 min-w-0">
                    <Button icon="pi pi-arrow-left" text rounded @click="cancelAndGoBack" />
                    <div class="flex-1 min-w-0">
                        <div class="font-bold text-lg text-primary truncate">{{ docno }}</div>
                    </div>
                </div>
                <div class="flex items-center gap-2">
                    <Button icon="pi pi-list" text rounded @click="showSODetails = true" v-badge.danger="soDetails.length" v-tooltip.bottom="'ดูรายการ SO'" />
                    <Button label="บันทึก" icon="pi pi-check" severity="success" @click="showSummary" :disabled="scannedItems.length === 0" />
                </div>
            </div>

            <!-- Progress Section -->
            <div class="px-4 py-3 bg-surface-50 dark:bg-surface-900">
                <div class="flex items-center justify-between mb-2">
                    <span class="font-semibold">ความคืบหน้าการรับสินค้า</span>
                    <span class="text-sm text-muted-color">{{ totalScannedQty }} / {{ totalSOQty }} ชิ้น</span>
                </div>
                <ProgressBar :value="progressPercentage" :showValue="false" class="h-3 mb-2" />
                <div class="text-sm text-center font-semibold" :class="progressPercentage === 100 ? 'text-success' : 'text-primary'">{{ progressPercentage.toFixed(1) }}% เสร็จสิ้น</div>
            </div>

            <!-- Input Mode Selection -->
            <div class="px-4 py-3">
                <SelectButton v-model="inputMode" :options="inputModes" optionLabel="name" optionValue="value" fluid>
                    <template #option="slotProps">
                        <i :class="slotProps.option.icon" class="mr-2"></i>
                        <span>{{ slotProps.option.name }}</span>
                    </template>
                </SelectButton>
            </div>

            <!-- Barcode Scanner Input -->
            <div v-if="inputMode === 'barcode'" class="px-4 pb-4">
                <IconField>
                    <InputIcon class="pi pi-qrcode" />
                    <InputText v-model="barcodeInput" @keyup.enter="processBarcodeInput" placeholder="Scan Barcode..." fluid class="text-lg font-mono" :disabled="searchLoading" autofocus />
                </IconField>
                <div class="text-xs text-muted-color mt-2 flex items-center gap-2">
                    <i class="pi pi-info-circle"></i>
                    <span>รูปแบบ: [จำนวน*]barcode[#ปี] เช่น 04-101-0000539#2024 หรือ 10*04-101-0000539#2024 </span>
                </div>
            </div>

            <!-- AutoComplete Search -->
            <div v-if="inputMode === 'search'" class="px-4 pb-4">
                <IconField>
                    <InputIcon class="pi pi-search" />
                    <AutoComplete
                        v-model="selectedItem"
                        :suggestions="searchResults"
                        @complete="handleSearch"
                        @item-select="onItemSelect"
                        placeholder="ค้นหาสินค้า..."
                        optionLabel="item_code"
                        fluid
                        class="text-base"
                        :loading="searchLoading"
                        completeOnFocus
                        :minLength="1"
                        autofocus
                    >
                        <template #option="slotProps">
                            <div class="flex items-start gap-2 py-2">
                                <div class="flex-1 min-w-0">
                                    <div class="font-semibold text-sm text-primary mb-1">{{ slotProps.option.item_code }}</div>
                                    <div class="text-xs text-muted-color mb-1 line-clamp-1">{{ slotProps.option.item_name || '-' }}</div>
                                    <div class="flex items-center gap-2">
                                        <Tag :value="slotProps.option.unit_code" severity="secondary" class="text-xs" />
                                        <span v-if="slotProps.option.barcode" class="font-mono font-bold text-xs text-primary bg-primary-50 dark:bg-primary-900/20 px-2 py-1 rounded">{{ slotProps.option.barcode }}</span>
                                    </div>
                                </div>
                            </div>
                        </template>
                    </AutoComplete>
                </IconField>
            </div>
        </div>

        <!-- Main Content -->
        <div class="p-2">
            <!-- Scanned Items List -->
            <div v-if="scannedItems.length === 0" class="text-center py-12 text-muted-color">
                <i class="pi pi-qrcode text-5xl mb-3 opacity-30"></i>
                <p class="text-sm">สแกน Barcode เพื่อเริ่มรับสินค้า</p>
            </div>

            <div v-else class="space-y-2">
                <div v-for="(item, index) in scannedItems" :key="index" class="bg-surface-0 dark:bg-surface-800 rounded-lg border border-surface-200 dark:border-surface-700 overflow-hidden">
                    <!-- Item Header -->
                    <div class="flex items-start p-3 pb-2">
                        <div class="flex-1 min-w-0">
                            <div class="font-semibold text-primary text-sm truncate mb-1">{{ item.item_code }}</div>
                            <div class="text-xs text-muted-color truncate mb-1">{{ item.item_name || '-' }}</div>
                            <div class="flex items-center gap-2">
                                <i class="pi pi-qrcode text-xs text-muted-color"></i>
                                <span class="font-mono font-bold text-xs text-primary bg-primary-50 dark:bg-primary-900/20 px-2 py-1 rounded">{{ item.barcode }}</span>
                                <span v-if="item.item_year" class="text-xs font-semibold text-orange-600 bg-orange-50 dark:bg-orange-900/20 px-2 py-1 rounded">#{{ item.item_year }}</span>
                            </div>
                        </div>
                        <Button icon="pi pi-trash" severity="danger" text rounded size="small" @click="removeScannedItem(index)" class="ml-2 -mt-1" />
                    </div>

                    <!-- Quantity Controls -->
                    <div class="flex items-center justify-between px-3 pb-3 pt-1">
                        <div class="flex items-center gap-1">
                            <Tag :value="item.unit_code" severity="secondary" class="text-xs" />
                        </div>
                        <div class="flex items-center gap-2">
                            <Button icon="pi pi-minus" text rounded size="small" severity="secondary" @click="item.qty > 1 && updateQty(index, item.qty - 1)" :disabled="item.qty <= 1" />
                            <div class="bg-primary-50 dark:bg-primary-900/20 rounded px-3 py-1 min-w-[3.5rem] text-center">
                                <div class="text-xl font-bold text-primary leading-none">{{ item.qty }}</div>
                                <div class="text-[10px] text-muted-color">/ {{ item.max_qty }}</div>
                            </div>
                            <Button icon="pi pi-plus" text rounded size="small" severity="success" @click="updateQty(index, item.qty + 1)" :disabled="item.qty >= item.max_qty" />
                        </div>
                    </div>

                    <!-- Progress Bar -->
                    <div class="h-1 bg-surface-100 dark:bg-surface-700">
                        <div class="h-full bg-success-500 transition-all" :style="{ width: `${(item.qty / item.max_qty) * 100}%` }"></div>
                    </div>
                </div>
            </div>

            <!-- Bottom Spacer -->
            <div class="h-20"></div>
        </div>

        <!-- SO Details Dialog - Full Screen -->
        <Dialog v-model:visible="showSODetails" header="รายการใน SO" :style="{ width: '100vw', height: '100vh', maxWidth: '100%' }" :modal="true" :dismissableMask="true" :draggable="false" class="so-details-dialog">
            <template #header>
                <div class="flex items-center justify-between w-full pr-8">
                    <div class="flex items-center gap-3">
                        <i class="pi pi-list text-xl"></i>
                        <div>
                            <div class="font-bold text-lg">รายการใน SO</div>
                            <div class="text-xs text-muted-color">{{ docno }}</div>
                        </div>
                    </div>
                    <Tag :value="`${soDetails.length} รายการ`" severity="info" />
                </div>
            </template>

            <!-- Stats Summary -->
            <div class="grid grid-cols-3 gap-3 mb-4">
                <div class="bg-primary-100 dark:bg-primary-900/30 rounded-lg p-3 text-center">
                    <div class="text-xs text-primary-600 dark:text-primary-400 mb-1">รายการทั้งหมด</div>
                    <div class="text-2xl font-bold text-primary">{{ soDetails.length }}</div>
                </div>
                <div class="bg-success-100 dark:bg-success-900/30 rounded-lg p-3 text-center">
                    <div class="text-xs text-success-600 dark:text-success-400 mb-1">ยอดรวม SO</div>
                    <div class="text-2xl font-bold text-success">{{ totalSOQty }}</div>
                </div>
                <div class="bg-orange-100 dark:bg-orange-900/30 rounded-lg p-3 text-center">
                    <div class="text-xs text-orange-600 dark:text-orange-400 mb-1">รับแล้ว</div>
                    <div class="text-2xl font-bold text-orange-600 dark:text-orange-400">{{ totalScannedQty }}</div>
                </div>
            </div>

            <!-- SO Items List -->
            <div class="space-y-3">
                <div v-for="(so, idx) in soDetails" :key="idx" class="bg-surface-50 dark:bg-surface-800 rounded-lg border border-surface-200 dark:border-surface-700 p-4">
                    <!-- Item Header -->
                    <div class="flex items-start justify-between mb-3">
                        <div class="flex-1 min-w-0">
                            <div class="font-bold text-lg text-primary mb-1">{{ so.item_code }}</div>
                            <div class="text-sm text-muted-color truncate">{{ so.item_name || '-' }}</div>
                        </div>
                        <Tag :value="so.unit_code" severity="secondary" class="ml-2" />
                    </div>

                    <!-- Quantity Stats -->
                    <div class="grid grid-cols-3 gap-3">
                        <div class="bg-blue-50 dark:bg-blue-900/20 rounded-lg p-3 text-center">
                            <div class="text-xs text-blue-600 dark:text-blue-400 mb-1">จำนวน SO</div>
                            <div class="text-xl font-bold text-blue-600 dark:text-blue-400">{{ so.qty }}</div>
                        </div>
                        <div :class="['rounded-lg p-3 text-center', getScannedQty(so.item_code) >= parseInt(so.qty) ? 'bg-green-50 dark:bg-green-900/20' : 'bg-orange-50 dark:bg-orange-900/20']">
                            <div class="text-xs mb-1" :class="getScannedQty(so.item_code) >= parseInt(so.qty) ? 'text-green-600 dark:text-green-400' : 'text-orange-600 dark:text-orange-400'">รับแล้ว</div>
                            <div class="text-xl font-bold" :class="getScannedQty(so.item_code) >= parseInt(so.qty) ? 'text-green-600 dark:text-green-400' : 'text-orange-600 dark:text-orange-400'">
                                {{ getScannedQty(so.item_code) }}
                            </div>
                        </div>
                        <div class="bg-surface-100 dark:bg-surface-700 rounded-lg p-3 text-center">
                            <div class="text-xs text-muted-color mb-1">คงเหลือ</div>
                            <div class="text-xl font-bold">{{ getRemainingQty(so.item_code) }}</div>
                        </div>
                    </div>

                    <!-- Progress Bar -->
                    <div class="mt-3">
                        <div class="flex items-center justify-between mb-1">
                            <span class="text-xs text-muted-color">ความคืบหน้า</span>
                            <span class="text-xs font-semibold">{{ ((getScannedQty(so.item_code) / parseInt(so.qty)) * 100).toFixed(0) }}%</span>
                        </div>
                        <ProgressBar :value="(getScannedQty(so.item_code) / parseInt(so.qty)) * 100" :showValue="false" class="h-2" />
                    </div>

                    <!-- Status Badge -->
                    <div class="mt-3 flex justify-end">
                        <Tag v-if="getScannedQty(so.item_code) >= parseInt(so.qty)" value="รับครบแล้ว" severity="success" icon="pi pi-check-circle" />
                        <Tag v-else-if="getScannedQty(so.item_code) > 0" value="รับบางส่วน" severity="warn" icon="pi pi-clock" />
                        <Tag v-else value="ยังไม่รับ" severity="danger" icon="pi pi-times-circle" />
                    </div>
                </div>
            </div>

            <template #footer>
                <Button label="ปิด" icon="pi pi-times" text @click="showSODetails = false" />
            </template>
        </Dialog>

        <!-- Summary Dialog -->
        <Dialog v-model:visible="showSummaryDialog" header="สรุปการรับสินค้า" :style="{ width: '95vw', maxWidth: '500px' }" :modal="true" :closable="!loading">
            <div class="mb-3 text-xs">
                <span class="text-muted-color">เอกสาร:</span>
                <span class="font-semibold text-primary ml-1">{{ docno }}</span>
            </div>

            <div class="space-y-2">
                <div v-for="(item, idx) in summaryItems" :key="idx" class="bg-surface-50 dark:bg-surface-800 rounded p-2 border border-surface-200 dark:border-surface-700">
                    <div class="flex items-center justify-between mb-1">
                        <div class="font-semibold text-sm text-primary">{{ item.item_code }}</div>
                        <Tag :value="item.unit_code" severity="secondary" class="text-xs" />
                    </div>
                    <div class="text-xs text-muted-color mb-2 truncate">{{ item.item_name }}</div>
                    <div class="flex items-center justify-between">
                        <span class="text-xs text-muted-color">จำนวน:</span>
                        <Tag :value="`${item.qty} / ${item.max_qty}`" :severity="item.qty === item.max_qty ? 'success' : 'warning'" class="text-xs" />
                    </div>
                </div>
            </div>

            <template #footer>
                <Button label="ยกเลิก" icon="pi pi-times" text @click="showSummaryDialog = false" size="small" :disabled="loading" />
                <Button label="ยืนยันบันทึก" icon="pi pi-check" severity="success" @click="saveReceiveDoc" size="small" :loading="loading" />
            </template>
        </Dialog>

        <!-- Full Screen Loading Overlay -->
        <div v-if="loading && !showSummaryDialog" class="fixed inset-0 bg-surface-900/80 backdrop-blur-sm z-50 flex items-center justify-center">
            <div class="bg-surface-0 dark:bg-surface-200 rounded-lg p-6 shadow-2xl text-center">
                <i class="pi pi-spin pi-spinner text-4xl text-primary mb-3"></i>
                <div class="font-semibold text-lg mb-2">กำลังบันทึกข้อมูล...</div>
                <div class="text-sm text-muted-color">กรุณารอสักครู่</div>
            </div>
        </div>
    </div>
</template>

<style scoped>
.line-clamp-1 {
    display: -webkit-box;
    -webkit-line-clamp: 1;
    -webkit-box-orient: vertical;
    overflow: hidden;
}

.space-y-2 > * + * {
    margin-top: 0.5rem;
}

.space-y-3 > * + * {
    margin-top: 0.75rem;
}

/* Full screen dialog styles */
:deep(.so-details-dialog .p-dialog) {
    margin: 0 !important;
    max-height: 100vh !important;
}

:deep(.so-details-dialog .p-dialog-content) {
    flex: 1;
    overflow-y: auto;
}
</style>
