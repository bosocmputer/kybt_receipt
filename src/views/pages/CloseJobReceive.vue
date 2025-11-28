<script setup>
import ReceiveDocService from '@/service/ReceiveDocService';
import ReceiveDocTable from '@/components/ReceiveDocTable.vue';
import ConfirmDialog from 'primevue/confirmdialog';
import { useConfirm } from 'primevue/useconfirm';
import { useToast } from 'primevue/usetoast';
import { onMounted, ref } from 'vue';

const toast = useToast();
const confirmDialog = useConfirm();
const receiveDocs = ref([]);
const loading = ref(false);
const searchQuery = ref('');
const fromDate = ref('2025-01-01');
const toDate = ref('2025-12-31');
const currentPage = ref(1);
const pageSize = ref(20);
const totalRecords = ref(0);
const totalPages = ref(0);

// Detail Dialog
const detailDialog = ref(false);
const detailLoading = ref(false);
const selectedDoc = ref(null);
const soDetails = ref([]);
const receiveDetails = ref([]);

onMounted(async () => {
    await loadReceiveDocs();
});

async function loadReceiveDocs() {
    loading.value = true;
    try {
        const result = await ReceiveDocService.getReceiveDocListByStatus(searchQuery.value, fromDate.value, toDate.value, 1, currentPage.value, pageSize.value);

        if (result.success) {
            receiveDocs.value = result.data;
            totalRecords.value = result.total;
            totalPages.value = result.totalPages;
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to load receive documents',
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while loading receive documents',
            life: 3000
        });
    } finally {
        loading.value = false;
    }
}

async function handleSearch() {
    currentPage.value = 1;
    await loadReceiveDocs();
}

function onPageChange(event) {
    currentPage.value = event.page;
    pageSize.value = event.rows;
    loadReceiveDocs();
}

function formatDate(dateStr) {
    if (!dateStr) return '';
    return new Date(dateStr).toLocaleDateString('th-TH', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit'
    });
}

async function confirmCloseJob(docData) {
    confirmDialog.require({
        message: `ต้องการปิดงานใบรับสินค้า ${docData.doc_no} หรือไม่?`,
        header: 'ยืนยันการปิดงาน',
        icon: 'pi pi-check-circle',
        acceptLabel: 'ยืนยัน',
        rejectLabel: 'ยกเลิก',
        acceptClass: 'p-button-success',
        rejectClass: 'p-button-secondary p-button-outlined',
        accept: async () => {
            await handleCloseJob(docData);
        }
    });
}

async function handleCloseJob(docData) {
    loading.value = true;
    try {
        const result = await ReceiveDocService.sendCloseJob(docData.doc_no);

        if (result.success) {
            toast.add({
                severity: 'success',
                summary: 'Success',
                detail: 'ปิดงานสำเร็จ',
                life: 3000
            });
            await loadReceiveDocs();
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to close job',
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while closing job',
            life: 3000
        });
    } finally {
        loading.value = false;
    }
}

async function viewDetail(docData) {
    selectedDoc.value = docData;
    detailDialog.value = true;
    detailLoading.value = true;

    try {
        const result = await ReceiveDocService.getReceiveDocDetail(docData.doc_no);

        if (result.success) {
            soDetails.value = result.details_so || [];
            receiveDetails.value = result.details_receive || [];
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to load detail',
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while loading detail',
            life: 3000
        });
    } finally {
        detailLoading.value = false;
    }
}

function closeDetailDialog() {
    detailDialog.value = false;
    selectedDoc.value = null;
    soDetails.value = [];
    receiveDetails.value = [];
}
</script>

<template>
    <ConfirmDialog />
    <div class="flex flex-col gap-6">
        <div class="card">
            <div class="font-semibold text-xl mb-2">ปิดงานใบรับ</div>
            <p class="text-muted-color m-0 mb-6">จัดการปิดงานใบรับสินค้าที่ได้รับอนุมัติแล้ว</p>

            <!-- Mobile Filters -->
            <div class="lg:hidden mb-4">
                <div class="flex flex-col gap-3">
                    <IconField>
                        <InputIcon class="pi pi-search" />
                        <InputText v-model="searchQuery" placeholder="ค้นหาเลขที่เอกสาร, ลูกค้า..." fluid @keyup.enter="handleSearch" />
                    </IconField>
                    <div class="grid grid-cols-2 gap-2">
                        <DatePicker v-model="fromDate" dateFormat="yy-mm-dd" placeholder="จากวันที่" :showIcon="true" fluid />
                        <DatePicker v-model="toDate" dateFormat="yy-mm-dd" placeholder="ถึงวันที่" :showIcon="true" fluid />
                    </div>
                    <div class="grid grid-cols-2 gap-2">
                        <Button label="ค้นหา" icon="pi pi-search" @click="handleSearch" :loading="loading" fluid />
                        <Button label="รีเฟรช" icon="pi pi-refresh" severity="secondary" @click="loadReceiveDocs" :loading="loading" fluid />
                    </div>
                </div>
            </div>

            <!-- Desktop Toolbar -->
            <Toolbar class="mb-6 hidden lg:flex">
                <template #start>
                    <div class="flex flex-wrap items-center gap-2">
                        <IconField>
                            <InputIcon class="pi pi-search" />
                            <InputText v-model="searchQuery" placeholder="ค้นหาเลขที่เอกสาร, ลูกค้า..." style="width: 18rem" @keyup.enter="handleSearch" />
                        </IconField>
                        <DatePicker v-model="fromDate" dateFormat="yy-mm-dd" placeholder="จากวันที่" :showIcon="true" style="width: 11rem" />
                        <DatePicker v-model="toDate" dateFormat="yy-mm-dd" placeholder="ถึงวันที่" :showIcon="true" style="width: 11rem" />
                        <Button label="ค้นหา" icon="pi pi-search" @click="handleSearch" :loading="loading" />
                    </div>
                </template>

                <template #end>
                    <Button icon="pi pi-refresh" severity="secondary" text rounded v-tooltip.top="'รีเฟรช'" @click="loadReceiveDocs" :loading="loading" />
                </template>
            </Toolbar>

            <ReceiveDocTable
                :data="receiveDocs"
                :loading="loading"
                :totalRecords="totalRecords"
                :currentPage="currentPage"
                :pageSize="pageSize"
                mode="close"
                :enableRowClick="true"
                @page-change="onPageChange"
                @row-click="viewDetail"
                @view-detail="viewDetail"
                @close-job="confirmCloseJob"
            />
        </div>

        <!-- Dialog รายละเอียดการรับ -->
        <Dialog v-model:visible="detailDialog" :style="{ width: '90vw', height: '85vh' }" header="รายละเอียดการรับสินค้า" :modal="true" :draggable="false" :breakpoints="{ '960px': '95vw', '640px': '98vw' }">
            <div v-if="detailLoading" class="flex items-center justify-center py-12">
                <i class="pi pi-spin pi-spinner text-4xl mr-3"></i>
                <span class="text-xl">กำลังโหลดข้อมูล...</span>
            </div>

            <div v-else class="flex flex-col gap-4">
                <!-- Header Info -->
                <div class="card bg-primary-50 dark:bg-primary-400/10 p-4">
                    <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
                        <div>
                            <label class="text-sm text-muted-color">เลขที่เอกสาร</label>
                            <p class="font-semibold text-lg">{{ selectedDoc?.doc_no }}</p>
                        </div>
                        <div>
                            <label class="text-sm text-muted-color">วันที่</label>
                            <p class="font-semibold">{{ formatDate(selectedDoc?.doc_date) }}</p>
                        </div>
                        <div>
                            <label class="text-sm text-muted-color">อ้างอิง SO</label>
                            <p class="font-semibold">{{ selectedDoc?.doc_ref || '-' }}</p>
                        </div>
                        <div>
                            <label class="text-sm text-muted-color">ลูกค้า</label>
                            <p class="font-semibold">{{ selectedDoc?.cust_name }}</p>
                        </div>
                        <div>
                            <label class="text-sm text-muted-color">พนักงานขาย</label>
                            <p class="font-semibold">{{ selectedDoc?.sale_name || '-' }}</p>
                        </div>
                        <div>
                            <label class="text-sm text-muted-color">สาขา</label>
                            <p class="font-semibold">{{ selectedDoc?.branch_code }}</p>
                        </div>
                    </div>
                </div>

                <!-- SO Details -->
                <div class="card p-4">
                    <h6 class="mb-3 font-semibold">รายการ SO</h6>
                    <DataTable :value="soDetails" :rows="10" stripedRows scrollable scrollHeight="250px">
                        <template #empty> ไม่พบรายการ SO </template>
                        <Column field="item_code" header="รหัสสินค้า" style="min-width: 10rem"></Column>
                        <Column field="item_name" header="ชื่อสินค้า" style="min-width: 15rem"></Column>
                        <Column field="unit_code" header="หน่วย" style="min-width: 8rem"></Column>
                        <Column field="qty" header="จำนวน" style="min-width: 8rem">
                            <template #body="{ data }">
                                <Tag :value="data.qty?.toString() || '0'" severity="info" />
                            </template>
                        </Column>
                    </DataTable>
                </div>

                <!-- Receive Details -->
                <div class="card p-4">
                    <h6 class="mb-3 font-semibold">รายการที่รับแล้ว</h6>
                    <DataTable :value="receiveDetails" :rows="10" stripedRows scrollable scrollHeight="250px">
                        <template #empty> ไม่พบรายการที่รับ </template>
                        <Column field="item_code" header="รหัสสินค้า" style="min-width: 10rem"></Column>
                        <Column field="item_name" header="ชื่อสินค้า" style="min-width: 15rem"></Column>
                        <Column field="unit_code" header="หน่วย" style="min-width: 8rem"></Column>
                        <Column field="qty" header="จำนวน" style="min-width: 8rem">
                            <template #body="{ data }">
                                <Tag :value="data.qty?.toString() || '0'" severity="success" />
                            </template>
                        </Column>
                    </DataTable>
                </div>
            </div>

            <template #footer>
                <Button label="ปิด" icon="pi pi-times" @click="closeDetailDialog" />
            </template>
        </Dialog>
    </div>
</template>

<style scoped lang="scss"></style>
