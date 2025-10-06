<template>
    <DrawerPro v-model="dialogVisible" :header="$t('database.databaseConnInfo')" @close="handleClose" size="small">
        <el-form @submit.prevent v-loading="loading" ref="formRef" :model="form" label-position="top">
            <el-form-item :label="$t('database.containerConn')" v-if="form.from === 'local'">
                <el-card class="mini-border-card">
                    <el-descriptions :column="1">
                        <el-descriptions-item :label="$t('database.connAddress')">
                            <el-tooltip
                                v-if="loadMongoInfo(true).length > 48"
                                :content="loadMongoInfo(true)"
                                placement="top"
                            >
                                {{ loadMongoInfo(true).substring(0, 48) }}...
                            </el-tooltip>
                            <span else>
                                {{ loadMongoInfo(true) }}
                            </span>
                            <CopyButton :content="loadMongoInfo(true)" />
                        </el-descriptions-item>
                        <el-descriptions-item :label="$t('commons.table.port')">
                            27017
                            <CopyButton content="27017" />
                        </el-descriptions-item>
                    </el-descriptions>
                </el-card>
                <span class="input-help">
                    {{ $t('database.containerConnHelper') }}
                </span>
            </el-form-item>
            <el-form-item :label="$t('database.remoteConn')">
                <el-card class="mini-border-card">
                    <el-descriptions :column="1">
                        <el-descriptions-item :label="$t('database.connAddress')">
                            <el-tooltip
                                v-if="loadMongoInfo(false).length > 48"
                                :content="loadMongoInfo(false)"
                                placement="top"
                            >
                                {{ loadMongoInfo(false).substring(0, 48) }}...
                            </el-tooltip>
                            <span else>
                                {{ loadMongoInfo(false) }}
                            </span>
                            <CopyButton :content="loadMongoInfo(false)" />
                        </el-descriptions-item>
                        <el-descriptions-item :label="$t('commons.table.port')">
                            {{ form.port }}
                            <CopyButton :content="form.port + ''" />
                        </el-descriptions-item>
                    </el-descriptions>
                </el-card>
                <span class="input-help">
                    {{ $t('database.remoteConnHelper2') }}
                </span>
            </el-form-item>

            <el-divider border-style="dashed" />
            
            <div v-if="form.from === 'local'">
                <el-form-item :label="$t('commons.login.username')">
                    <el-tag>root</el-tag>
                    <CopyButton content="root" />
                </el-form-item>
                <el-form-item :label="$t('commons.login.password')">
                    <el-tag>{{ form.password }}</el-tag>
                    <CopyButton :content="form.password" />
                </el-form-item>
            </div>

            <div v-if="form.from !== 'local'">
                <el-form-item :label="$t('commons.login.username')">
                    <el-tag>{{ form.username }}</el-tag>
                    <CopyButton :content="form.username" />
                </el-form-item>
                <el-form-item :label="$t('commons.login.password')">
                    <el-tag>{{ form.password }}</el-tag>
                    <CopyButton :content="form.password" />
                </el-form-item>
            </div>
        </el-form>

        <template #footer>
            <span class="dialog-footer">
                <el-button :disabled="loading" @click="dialogVisible = false">
                    {{ $t('commons.button.cancel') }}
                </el-button>
            </span>
        </template>
    </DrawerPro>
</template>

<script lang="ts" setup>
import { reactive, ref } from 'vue';
import i18n from '@/lang';
import { ElForm } from 'element-plus';
import { getDatabase } from '@/api/modules/database';
import { getAppConnInfo } from '@/api/modules/app';
import { getAgentSettingInfo } from '@/api/modules/setting';
import { GlobalStore } from '@/store';
const globalStore = GlobalStore();

const loading = ref(false);

const dialogVisible = ref(false);
const form = reactive({
    status: '',
    type: '',
    systemIP: '',
    password: '',
    username: '',
    serviceName: '',
    containerName: '',
    port: 0,

    from: '',
    database: '',
    remoteIP: '',
});

type FormInstance = InstanceType<typeof ElForm>;
const formRef = ref<FormInstance>();

interface DialogProps {
    from: string;
    type: string;
    database: string;
}
const acceptParams = (params: DialogProps): void => {
    form.password = '';
    form.username = '';
    form.from = params.from;
    form.type = params.type;
    form.database = params.database;
    loadPassword();
    dialogVisible.value = true;
};
const handleClose = () => {
    dialogVisible.value = false;
};

const loadPassword = async () => {
    if (form.from === 'local') {
        const res = await getAppConnInfo(form.type, form.database);
        form.status = res.data.status;
        form.password = res.data.password || '';
        form.port = res.data.port || 27017;
        form.serviceName = res.data.serviceName || '';
        form.containerName = res.data.containerName || '';
        loadSystemIP();
        return;
    }
    const res = await getDatabase(form.database);
    form.password = res.data.password || '';
    form.username = res.data.username || '';
    form.port = res.data.port || 27017;
    form.remoteIP = res.data.address;
};

const loadSystemIP = async () => {
    const res = await getAgentSettingInfo();
    form.systemIP = res.data.systemIP || globalStore.currentNodeAddr || i18n.global.t('database.localIP');
};

function loadMongoInfo(isContainer: boolean) {
    if (isContainer) {
        return form.from === 'local' ? form.containerName : form.systemIP;
    } else {
        return form.from === 'local' ? form.systemIP : form.remoteIP;
    }
}

defineExpose({
    acceptParams,
});
</script>

<style lang="scss" scoped>
.copy_button {
    border-radius: 0px;
    border-left-width: 0px;
}
:deep(.el-input__wrapper) {
    border-top-right-radius: 0px;
    border-bottom-right-radius: 0px;
}
</style>
