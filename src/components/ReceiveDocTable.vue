<script setup>
import { ref } from 'vue';

defineProps({
    data: {
        type: Array,
        required: true
    },
    loading: {
        type: Boolean,
        default: false
    },
    totalRecords: {
        type: Number,
        default: 0
    },
    currentPage: {
        type: Number,
        default: 1
    },
    pageSize: {
        type: Number,
        default: 20
    },
    mode: {
        type: String,
        default: 'default', // 'default', 'close', 'history'
        validator: (value) => ['default', 'close', 'history'].includes(value)
    }
});

const emit = defineEmits(['page-change', 'receive-item', 'send-approve', 'delete', 'close-job', 'view-detail']);

const dt = ref();
const expandedRows = ref([]);

function formatDate(dateStr) {
    if (!dateStr) return '';
    return new Date(dateStr).toLocaleDateString('th-TH', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit'
    });
}

function formatDateTime(dateStr, timeStr) {
    if (!dateStr) return '-';
    let result = formatDate(dateStr);
    if (timeStr) {
        result += ` ${timeStr}`;
    }
    return result;
}

function onPageChange(event) {
    emit('page-change', {
        page: event.page + 1,
        rows: event.rows
    });
}

function canApprove(docData) {
    const soQty = parseInt(docData.so_qty) || 0;
    const receiveQty = parseInt(docData.receive_qty) || 0;
    return soQty === receiveQty && docData.can_approve === '1';
}
</script>

<template>
    <DataTable
        ref="dt"
        :value="data"
        :loading="loading"
        v-model:expandedRows="expandedRows"
        dataKey="doc_no"
        :lazy="true"
        :paginator="true"
        :rows="pageSize"
        :totalRecords="totalRecords"
        :rowsPerPageOptions="[10, 20, 50, 100]"
        :rowHover="true"
        stripedRows
        paginatorTemplate="FirstPageLink PrevPageLink PageLinks NextPageLink LastPageLink CurrentPageReport RowsPerPageDropdown"
        currentPageReportTemplate="แสดง {first} ถึง {last} จาก {totalRecords} รายการ"
        @page="onPageChange"
    >
        <template #empty>
            <div class="flex flex-col items-center justify-center py-6">
                <i class="pi pi-inbox text-4xl text-muted-color mb-4"></i>
                <span class="text-muted-color">ไม่พบข้อมูล</span>
            </div>
        </template>
        <template #loading>
            <div class="flex items-center justify-center py-6">
                <i class="pi pi-spin pi-spinner text-2xl mr-2"></i>
                <span>กำลังโหลดข้อมูล...</span>
            </div>
        </template>

        <!-- Column Expander -->
        <Column expander style="width: 3rem" />

        <!-- 1. วันที่ -->
        <Column field="doc_date" header="วันที่" :sortable="true" style="min-width: 10rem">
            <template #body="{ data }">
                <div class="flex items-center gap-2">
                    <i class="pi pi-calendar text-muted-color"></i>
                    <span>{{ formatDate(data.doc_date) }}</span>
                </div>
            </template>
        </Column>

        <!-- 2. เลขที่ใบรับ -->
        <Column field="doc_no" header="เลขที่ใบรับ" :sortable="true" style="min-width: 14rem">
            <template #body="{ data }">
                <span class="font-semibold text-primary">{{ data.doc_no }}</span>
            </template>
        </Column>

        <!-- 3. เลขที่ SO -->
        <Column field="doc_ref" header="เลขที่ SO" :sortable="true" style="min-width: 12rem">
            <template #body="{ data }">
                <Tag v-if="data.doc_ref" :value="data.doc_ref" severity="info" />
                <span v-else class="text-muted-color">-</span>
            </template>
        </Column>

        <!-- 4. จำนวนที่ต้องรับ (SO Qty) -->
        <Column field="so_qty" header="จำนวนที่ต้องรับ" :sortable="true" style="min-width: 10rem">
            <template #body="{ data }">
                <div class="flex items-center gap-2">
                    <i class="pi pi-box text-blue-500"></i>
                    <Tag :value="data.so_qty?.toString() || '0'" severity="info" class="font-bold" />
                </div>
            </template>
        </Column>

        <!-- 5. จำนวนรับ (Receive Qty) - เน้นให้เด่น -->
        <Column field="receive_qty" header="จำนวนรับ" :sortable="true" style="min-width: 10rem">
            <template #body="{ data }">
                <div class="flex items-center gap-2">
                    <i class="pi pi-check-circle text-green-500"></i>
                    <Tag :value="data.receive_qty?.toString() || '0'" :severity="canApprove(data) ? 'success' : 'warn'" class="font-bold text-lg" />
                </div>
            </template>
        </Column>

        <!-- 6. เมนูจัดการ -->
        <!-- Default Mode: ใบรับสินค้า -->
        <Column v-if="mode === 'default'" header="จัดการ" :exportable="false" style="min-width: 20rem">
            <template #body="slotProps">
                <div class="flex flex-wrap gap-2">
                    <Button icon="pi pi-box" label="รับสินค้า" size="small" severity="success" @click.stop="emit('receive-item', slotProps.data)" v-tooltip.top="'เริ่มรับสินค้า'" />
                    <Button
                        icon="pi pi-send"
                        label="ส่งอนุมัติ"
                        size="small"
                        severity="info"
                        :disabled="!canApprove(slotProps.data)"
                        @click.stop="emit('send-approve', slotProps.data)"
                        v-tooltip.top="canApprove(slotProps.data) ? 'ส่งอนุมัติ' : 'รอรับสินค้าให้ครบก่อน'"
                    />
                    <Button icon="pi pi-trash" label="ลบ" size="small" severity="danger" @click.stop="emit('delete', slotProps.data)" v-tooltip.top="'ลบใบรับ'" />
                </div>
            </template>
        </Column>

        <!-- Close Mode: ปิดงานใบรับ -->
        <Column v-if="mode === 'close'" header="จัดการ" :exportable="false" style="min-width: 16rem">
            <template #body="slotProps">
                <div class="flex flex-wrap gap-2">
                    <Button icon="pi pi-eye" label="ดูรายละเอียด" size="small" severity="info" @click.stop="emit('view-detail', slotProps.data)" v-tooltip.top="'ดูรายละเอียด'" />
                    <Button icon="pi pi-check-circle" label="ปิดงาน" size="small" severity="success" @click.stop="emit('close-job', slotProps.data)" v-tooltip.top="'ปิดงาน'" />
                </div>
            </template>
        </Column>

        <!-- History Mode: ประวัติใบรับ -->
        <Column v-if="mode === 'history'" header="สถานะ" style="min-width: 8rem">
            <template #body>
                <Tag value="ปิดงานแล้ว" severity="secondary" />
            </template>
        </Column>

        <!-- Row Expansion Template -->
        <template #expansion="slotProps">
            <div class="p-4">
                <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                    <!-- ข้อมูลลูกค้า -->
                    <div class="border border-surface-200 dark:border-surface-700 rounded-lg p-4">
                        <h6 class="font-semibold mb-3 text-primary"><i class="pi pi-user mr-2"></i>ข้อมูลลูกค้า</h6>
                        <div class="space-y-2">
                            <div class="flex justify-between">
                                <span class="text-muted-color">รหัสลูกค้า:</span>
                                <span class="font-semibold">{{ slotProps.data.cust_code || '-' }}</span>
                            </div>
                            <div class="flex justify-between">
                                <span class="text-muted-color">ชื่อลูกค้า:</span>
                                <span class="font-semibold">{{ slotProps.data.cust_name || '-' }}</span>
                            </div>
                        </div>
                    </div>

                    <!-- ข้อมูลพนักงาน -->
                    <div class="border border-surface-200 dark:border-surface-700 rounded-lg p-4">
                        <h6 class="font-semibold mb-3 text-primary"><i class="pi pi-users mr-2"></i>ข้อมูลพนักงาน</h6>
                        <div class="space-y-2">
                            <div class="flex justify-between">
                                <span class="text-muted-color">รหัสพนักงานขาย:</span>
                                <span class="font-semibold">{{ slotProps.data.sale_code || '-' }}</span>
                            </div>
                            <div class="flex justify-between">
                                <span class="text-muted-color">ชื่อพนักงานขาย:</span>
                                <span class="font-semibold">{{ slotProps.data.sale_name || '-' }}</span>
                            </div>
                        </div>
                    </div>

                    <!-- ข้อมูลสาขา -->
                    <div class="border border-surface-200 dark:border-surface-700 rounded-lg p-4">
                        <h6 class="font-semibold mb-3 text-primary"><i class="pi pi-building mr-2"></i>ข้อมูลสาขา</h6>
                        <div class="space-y-2">
                            <div class="flex justify-between">
                                <span class="text-muted-color">รหัสสาขา:</span>
                                <Tag v-if="slotProps.data.branch_code" :value="slotProps.data.branch_code" severity="secondary" />
                                <span v-else>-</span>
                            </div>
                        </div>
                    </div>

                    <!-- หมายเหตุ -->
                    <div class="border border-surface-200 dark:border-surface-700 rounded-lg p-4">
                        <h6 class="font-semibold mb-3 text-primary"><i class="pi pi-comment mr-2"></i>หมายเหตุ</h6>
                        <div class="space-y-2">
                            <p class="text-sm">{{ slotProps.data.remark || '-' }}</p>
                        </div>
                    </div>

                    <!-- ข้อมูลการอนุมัติ (ถ้ามี) -->
                    <div v-if="mode === 'history' || slotProps.data.user_approve" class="border border-surface-200 dark:border-surface-700 rounded-lg p-4">
                        <h6 class="font-semibold mb-3 text-success"><i class="pi pi-check-circle mr-2"></i>ข้อมูลการอนุมัติ</h6>
                        <div class="space-y-2">
                            <div class="flex justify-between">
                                <span class="text-muted-color">ผู้อนุมัติ:</span>
                                <span class="font-semibold">{{ slotProps.data.user_approve_name || '-' }}</span>
                            </div>
                            <div class="flex justify-between">
                                <span class="text-muted-color">วันที่อนุมัติ:</span>
                                <span>{{ formatDateTime(slotProps.data.approve_date, slotProps.data.approve_time) }}</span>
                            </div>
                        </div>
                    </div>

                    <!-- ข้อมูลการปิดงาน (ถ้ามี) -->
                    <div v-if="mode === 'history' || slotProps.data.user_close" class="border border-surface-200 dark:border-surface-700 rounded-lg p-4">
                        <h6 class="font-semibold mb-3 text-secondary"><i class="pi pi-lock mr-2"></i>ข้อมูลการปิดงาน</h6>
                        <div class="space-y-2">
                            <div class="flex justify-between">
                                <span class="text-muted-color">ผู้ปิดงาน:</span>
                                <span class="font-semibold">{{ slotProps.data.user_close_name || '-' }}</span>
                            </div>
                            <div class="flex justify-between">
                                <span class="text-muted-color">วันที่ปิดงาน:</span>
                                <span>{{ formatDateTime(slotProps.data.close_date, slotProps.data.close_time) }}</span>
                            </div>
                        </div>
                    </div>

                    <!-- สรุปจำนวน -->
                    <div class="border border-surface-200 dark:border-surface-700 rounded-lg p-4 md:col-span-2">
                        <h6 class="font-semibold mb-3 text-primary"><i class="pi pi-chart-bar mr-2"></i>สรุปจำนวน</h6>
                        <div class="flex gap-6">
                            <div class="flex-1 text-center p-3 bg-blue-50 dark:bg-blue-900/20 rounded">
                                <div class="text-muted-color text-sm mb-1">จำนวนที่ต้องรับ</div>
                                <div class="text-2xl font-bold text-blue-600">{{ slotProps.data.so_qty || '0' }}</div>
                            </div>
                            <div class="flex-1 text-center p-3 bg-green-50 dark:bg-green-900/20 rounded">
                                <div class="text-muted-color text-sm mb-1">จำนวนที่รับแล้ว</div>
                                <div class="text-2xl font-bold text-green-600">{{ slotProps.data.receive_qty || '0' }}</div>
                            </div>
                            <div class="flex-1 text-center p-3 bg-orange-50 dark:bg-orange-900/20 rounded">
                                <div class="text-muted-color text-sm mb-1">ยังต้องรับ</div>
                                <div class="text-2xl font-bold text-orange-600">
                                    {{ (parseInt(slotProps.data.so_qty) || 0) - (parseInt(slotProps.data.receive_qty) || 0) }}
                                </div>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </template>
    </DataTable>
</template>

<style scoped lang="scss">
// Removed cursor-pointer-rows class since we're using expansion instead of row-click
</style>
