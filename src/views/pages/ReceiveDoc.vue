<script setup>
import ReceiveDocService from '@/service/ReceiveDocService';
import { FilterMatchMode } from '@primevue/core/api';
import ConfirmDialog from 'primevue/confirmdialog';
import { useConfirm } from 'primevue/useconfirm';
import { useToast } from 'primevue/usetoast';
import { onMounted, ref } from 'vue';

const toast = useToast();
const confirmDialog = useConfirm();
const dt = ref();
const receiveDocs = ref([]);
const loading = ref(false);
const searchQuery = ref('');
const fromDate = ref('2025-01-01');
const toDate = ref('2025-12-31');
const currentPage = ref(1);
const pageSize = ref(20);
const totalRecords = ref(0);
const totalPages = ref(0);

// Dialog states
const createDialog = ref(false);
const currentStep = ref('1');
const selectedSO = ref(null);
const remark = ref('');
const soList = ref([]);
const soLoading = ref(false);
const soSearchQuery = ref('');
const soFromDate = ref('2020-01-01');
const soToDate = ref('2025-01-31');
const soCurrentPage = ref(1);
const soPageSize = ref(20);
const soTotalRecords = ref(0);
const soTotalPages = ref(0);

const filters = ref({
    global: { value: null, matchMode: FilterMatchMode.CONTAINS }
});

onMounted(async () => {
    await loadReceiveDocs();
});

async function loadReceiveDocs() {
    loading.value = true;
    try {
        const result = await ReceiveDocService.getReceiveDocList(searchQuery.value, fromDate.value, toDate.value, currentPage.value, pageSize.value);

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
    currentPage.value = event.page + 1;
    pageSize.value = event.rows;
    loadReceiveDocs();
}

// Open create dialog -> show SO list
function openCreateDialog() {
    currentStep.value = '1';
    selectedSO.value = null;
    remark.value = '';
    soSearchQuery.value = '';
    soCurrentPage.value = 1;
    createDialog.value = true;
    loadSOList();
}

async function loadSOList() {
    soLoading.value = true;
    try {
        const result = await ReceiveDocService.getSODocList(soSearchQuery.value, soFromDate.value, soToDate.value, soCurrentPage.value, soPageSize.value);

        if (result.success) {
            soList.value = result.data;
            soTotalRecords.value = result.total;
            soTotalPages.value = result.totalPages;
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to load SO documents',
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while loading SO documents',
            life: 3000
        });
    } finally {
        soLoading.value = false;
    }
}

async function handleSOSearch() {
    soCurrentPage.value = 1;
    await loadSOList();
}

function onSOPageChange(event) {
    soCurrentPage.value = event.page + 1;
    soPageSize.value = event.rows;
    loadSOList();
}

function selectSO(event) {
    selectedSO.value = { ...event.data };
    currentStep.value = '2';
}

async function createReceiveDoc() {
    if (!selectedSO.value) {
        toast.add({
            severity: 'warn',
            summary: 'Warning',
            detail: 'กรุณาเลือก SO ก่อน',
            life: 3000
        });
        return;
    }

    loading.value = true;
    try {
        const result = await ReceiveDocService.createReceiveDoc({
            doc_ref: selectedSO.value.doc_no,
            cust_code: selectedSO.value.cust_code,
            sale_code: selectedSO.value.sale_code || '',
            branch_code: selectedSO.value.branch_code,
            remark: remark.value
        });

        if (result.success) {
            toast.add({
                severity: 'success',
                summary: 'Success',
                detail: 'สร้างใบรับสินค้าสำเร็จ',
                life: 3000
            });

            createDialog.value = false;
            selectedSO.value = null;
            remark.value = '';
            await loadReceiveDocs();
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to create receive document',
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while creating receive document',
            life: 3000
        });
    } finally {
        loading.value = false;
    }
}

function hideCreateDialog() {
    createDialog.value = false;
    currentStep.value = '1';
    selectedSO.value = null;
    remark.value = '';
}

function backToSOList() {
    currentStep.value = '1';
    selectedSO.value = null;
}

function formatDate(dateStr) {
    if (!dateStr) return '';
    return new Date(dateStr).toLocaleDateString('th-TH', {
        year: 'numeric',
        month: '2-digit',
        day: '2-digit'
    });
}

async function confirmSendApprove(docData) {
    confirmDialog.require({
        message: `ต้องการส่งอนุมัติใบรับสินค้า ${docData.doc_no} หรือไม่?`,
        header: 'ยืนยันการส่งอนุมัติ',
        icon: 'pi pi-send',
        acceptLabel: 'ยืนยัน',
        rejectLabel: 'ยกเลิก',
        acceptClass: 'p-button-info',
        rejectClass: 'p-button-secondary p-button-outlined',
        accept: async () => {
            await handleSendApprove(docData);
        }
    });
}

async function handleSendApprove(docData) {
    loading.value = true;
    try {
        const result = await ReceiveDocService.sendApprove(docData.doc_no);

        if (result.success) {
            toast.add({
                severity: 'success',
                summary: 'Success',
                detail: 'ส่งอนุมัติสำเร็จ',
                life: 3000
            });
            await loadReceiveDocs();
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to send approve',
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while sending approve',
            life: 3000
        });
    } finally {
        loading.value = false;
    }
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

async function confirmDelete(docData) {
    confirmDialog.require({
        message: `ต้องการลบใบรับสินค้า ${docData.doc_no} หรือไม่?`,
        header: 'ยืนยันการลบ',
        icon: 'pi pi-exclamation-triangle',
        acceptLabel: 'ลบ',
        rejectLabel: 'ยกเลิก',
        acceptClass: 'p-button-danger',
        rejectClass: 'p-button-secondary p-button-outlined',
        accept: async () => {
            await handleDelete(docData);
        }
    });
}

async function handleDelete(docData) {
    loading.value = true;
    try {
        const result = await ReceiveDocService.deleteReceiveDoc(docData.doc_no);

        if (result.success) {
            toast.add({
                severity: 'success',
                summary: 'Success',
                detail: 'ลบใบรับสินค้าสำเร็จ',
                life: 3000
            });
            await loadReceiveDocs();
        } else {
            toast.add({
                severity: 'error',
                summary: 'Error',
                detail: result.message || 'Failed to delete receive document',
                life: 3000
            });
        }
    } catch (error) {
        toast.add({
            severity: 'error',
            summary: 'Error',
            detail: 'An error occurred while deleting receive document',
            life: 3000
        });
    } finally {
        loading.value = false;
    }
}

function canApprove(docData) {
    const soQty = parseInt(docData.so_qty) || 0;
    const receiveQty = parseInt(docData.receive_qty) || 0;
    return soQty <= receiveQty && docData.can_approve === '1';
}
</script>

<template>
    <ConfirmDialog />
    <div class="flex flex-col gap-6">
        <div class="card">
            <div class="font-semibold text-xl mb-2">ใบรับสินค้า</div>
            <p class="text-muted-color m-0 mb-6">จัดการใบรับสินค้าทั้งหมด</p>

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
                    <Button label="สร้างใบรับ" icon="pi pi-plus" @click="openCreateDialog" fluid severity="success" />
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
                    <div class="flex gap-2">
                        <Button icon="pi pi-refresh" severity="secondary" text rounded v-tooltip.top="'รีเฟรช'" @click="loadReceiveDocs" :loading="loading" />
                        <Button label="สร้างใบรับ" icon="pi pi-plus" @click="openCreateDialog" />
                    </div>
                </template>
            </Toolbar>

            <DataTable
                ref="dt"
                v-model:filters="filters"
                :value="receiveDocs"
                :loading="loading"
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

                <Column field="doc_no" header="เลขที่เอกสาร" :sortable="true" style="min-width: 14rem">
                    <template #body="{ data }">
                        <span class="font-semibold text-primary">{{ data.doc_no }}</span>
                    </template>
                </Column>

                <Column field="doc_ref" header="อ้างอิง SO" :sortable="true" style="min-width: 12rem">
                    <template #body="{ data }">
                        <Tag v-if="data.doc_ref" :value="data.doc_ref" severity="info" />
                        <span v-else class="text-muted-color">-</span>
                    </template>
                </Column>

                <Column field="doc_date" header="วันที่" :sortable="true" style="min-width: 10rem">
                    <template #body="{ data }">
                        <div class="flex items-center gap-2">
                            <i class="pi pi-calendar text-muted-color"></i>
                            <span>{{ formatDate(data.doc_date) }}</span>
                        </div>
                    </template>
                </Column>

                <Column field="cust_name" header="ลูกค้า" :sortable="true" style="min-width: 12rem">
                    <template #body="{ data }">
                        <div class="flex items-center gap-2">
                            <i class="pi pi-user text-muted-color"></i>
                            <span>{{ data.cust_name || '-' }}</span>
                        </div>
                    </template>
                </Column>

                <Column field="sale_name" header="พนักงานขาย" :sortable="true" style="min-width: 12rem">
                    <template #body="{ data }">
                        {{ data.sale_name || '-' }}
                    </template>
                </Column>

                <Column field="branch_code" header="สาขา" :sortable="true" style="min-width: 8rem">
                    <template #body="{ data }">
                        <Tag v-if="data.branch_code" :value="data.branch_code" severity="secondary" />
                        <span v-else class="text-muted-color">-</span>
                    </template>
                </Column>

                <Column field="remark" header="หมายเหตุ" style="min-width: 12rem">
                    <template #body="{ data }">
                        <span v-if="data.remark" v-tooltip.top="data.remark" class="max-w-[200px] truncate block">{{ data.remark }}</span>
                        <span v-else class="text-muted-color">-</span>
                    </template>
                </Column>

                <Column field="receive_qty" header="จำนวนรับ" :sortable="true" style="min-width: 8rem">
                    <template #body="{ data }">
                        <Tag :value="data.receive_qty?.toString() || '0'" severity="success" />
                    </template>
                </Column>

                <Column header="จัดการ" :exportable="false" style="min-width: 16rem">
                    <template #body="slotProps">
                        <div class="flex flex-wrap gap-2">
                            <Button
                                icon="pi pi-send"
                                label="ส่งอนุมัติ"
                                size="small"
                                severity="info"
                                :disabled="!canApprove(slotProps.data)"
                                @click="confirmSendApprove(slotProps.data)"
                                v-tooltip.top="canApprove(slotProps.data) ? 'ส่งอนุมัติ' : 'รอรับสินค้าให้ครบก่อน'"
                            />
                            <Button icon="pi pi-check-circle" label="ปิดงาน" size="small" severity="success" @click="confirmCloseJob(slotProps.data)" v-tooltip.top="'ปิดงาน'" />
                            <Button icon="pi pi-trash" label="ลบ" size="small" severity="danger" @click="confirmDelete(slotProps.data)" v-tooltip.top="'ลบใบรับ'" />
                        </div>
                    </template>
                </Column>
            </DataTable>
        </div>

        <!-- Dialog สร้างใบรับสินค้า -->
        <Dialog v-model:visible="createDialog" :style="{ width: '80vw', height: '80vh' }" header="สร้างใบรับสินค้า" :modal="true" :draggable="false" :breakpoints="{ '960px': '90vw', '640px': '95vw' }">
            <!-- Stepper -->
            <div class="mb-4">
                <Stepper :value="currentStep" linear>
                    <StepList>
                        <Step value="1">เลือก SO</Step>
                        <Step value="2">กรอกรายละเอียด</Step>
                    </StepList>
                </Stepper>
            </div>

            <div v-if="currentStep === '1'">
                <!-- Step 1: เลือก SO -->
                <div class="mb-4">
                    <div class="flex flex-wrap items-center gap-2 mb-4">
                        <InputText v-model="soSearchQuery" placeholder="ค้นหา SO..." class="w-full md:w-[15rem]" @keyup.enter="handleSOSearch" />
                        <DatePicker v-model="soFromDate" dateFormat="yy-mm-dd" placeholder="จากวันที่" class="w-full md:w-[12rem]" />
                        <DatePicker v-model="soToDate" dateFormat="yy-mm-dd" placeholder="ถึงวันที่" class="w-full md:w-[12rem]" />
                        <Button label="ค้นหา" icon="pi pi-search" @click="handleSOSearch" :loading="soLoading" />
                    </div>
                </div>

                <DataTable
                    :value="soList"
                    :loading="soLoading"
                    dataKey="doc_no"
                    :lazy="true"
                    :paginator="true"
                    :rows="soPageSize"
                    :totalRecords="soTotalRecords"
                    :rowsPerPageOptions="[10, 20, 50]"
                    paginatorTemplate="FirstPageLink PrevPageLink PageLinks NextPageLink LastPageLink CurrentPageReport RowsPerPageDropdown"
                    currentPageReportTemplate="แสดง {first} ถึง {last} จาก {totalRecords} รายการ"
                    @page="onSOPageChange"
                    selectionMode="single"
                    @row-select="selectSO"
                    class="cursor-pointer"
                >
                    <template #empty> ไม่พบข้อมูล SO </template>
                    <template #loading> กำลังโหลดข้อมูล... </template>

                    <Column field="doc_no" header="เลขที่ SO" :sortable="true" style="min-width: 12rem">
                        <template #body="{ data }">
                            <span class="font-semibold text-primary">{{ data.doc_no }}</span>
                        </template>
                    </Column>

                    <Column field="doc_date" header="วันที่" :sortable="true" style="min-width: 10rem">
                        <template #body="{ data }">
                            {{ formatDate(data.doc_date) }}
                        </template>
                    </Column>

                    <Column field="cust_name" header="ลูกค้า" :sortable="true" style="min-width: 12rem"></Column>

                    <Column field="sale_name" header="พนักงานขาย" style="min-width: 12rem"></Column>

                    <Column field="branch_code" header="สาขา" :sortable="true" style="min-width: 8rem"></Column>

                    <Column field="remark" header="หมายเหตุ" style="min-width: 15rem"></Column>
                </DataTable>
            </div>

            <div v-else-if="currentStep === '2' && selectedSO">
                <!-- Step 2: แสดง SO ที่เลือกและกรอก remark -->
                <div class="flex flex-col gap-6">
                    <div class="card bg-primary-50 dark:bg-primary-400/10">
                        <h6 class="mb-3">SO ที่เลือก</h6>
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                            <div>
                                <label class="text-sm text-muted-color">เลขที่ SO</label>
                                <p class="font-semibold text-lg">{{ selectedSO.doc_no }}</p>
                            </div>
                            <div>
                                <label class="text-sm text-muted-color">วันที่</label>
                                <p class="font-semibold">{{ formatDate(selectedSO.doc_date) }}</p>
                            </div>
                            <div>
                                <label class="text-sm text-muted-color">ลูกค้า</label>
                                <p class="font-semibold">{{ selectedSO.cust_name }}</p>
                            </div>
                            <div>
                                <label class="text-sm text-muted-color">พนักงานขาย</label>
                                <p class="font-semibold">{{ selectedSO.sale_name || '-' }}</p>
                            </div>
                            <div>
                                <label class="text-sm text-muted-color">สาขา</label>
                                <p class="font-semibold">{{ selectedSO.branch_code }}</p>
                            </div>
                            <div>
                                <label class="text-sm text-muted-color">หมายเหตุ SO</label>
                                <p>{{ selectedSO.remark || '-' }}</p>
                            </div>
                        </div>
                    </div>

                    <div class="flex flex-col gap-2">
                        <label for="remark" class="font-bold">หมายเหตุใบรับสินค้า</label>
                        <Textarea id="remark" v-model="remark" rows="4" placeholder="กรอกหมายเหตุ..." />
                    </div>
                </div>
            </div>

            <template #footer>
                <div v-if="currentStep === '1'">
                    <Button label="ยกเลิก" icon="pi pi-times" text @click="hideCreateDialog" />
                </div>
                <div v-else-if="currentStep === '2'">
                    <Button label="ย้อนกลับ" icon="pi pi-arrow-left" text @click="backToSOList" />
                    <Button label="สร้างใบรับ" icon="pi pi-check" @click="createReceiveDoc" :loading="loading" />
                </div>
            </template>
        </Dialog>
    </div>
</template>

<style scoped lang="scss">
.cursor-pointer :deep(tbody tr) {
    cursor: pointer;
}
</style>
