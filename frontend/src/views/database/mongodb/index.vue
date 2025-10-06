<template>
    <div v-loading="loading">
        <div class="app-status mt-5" v-if="currentDB && currentDB.from === 'remote'">
            <el-card>
                <div class="flex w-full flex-col gap-4 md:flex-row">
                    <div class="flex flex-wrap gap-4 ml-3">
                        <el-tag class="float-left" effect="dark" type="success">MongoDB</el-tag>
                        <el-tag>{{ $t('app.version') }}: {{ currentDB?.version }}</el-tag>
                    </div>
                </div>
            </el-card>
        </div>
        <LayoutContent title="MongoDB">
            <template #app v-if="currentDB?.from === 'local'">
                <AppStatus
                    :app-key="currentDB.type"
                    :app-name="appName"
                    v-model:loading="loading"
                    @before="onBefore"
                    @after="onAfter"
                    @setting="onSetting"
                    ref="appStatusRef"
                ></AppStatus>
            </template>
            <template #leftToolBar v-if="!isOnSetting">
                <el-button v-if="currentDB" type="primary" plain @click="onLoadConn">
                    {{ $t('database.databaseConnInfo') }}
                </el-button>
                <el-button @click="goRemoteDB()" type="primary" plain>
                    {{ $t('database.remoteDB') }}
                </el-button>
                <el-button
                    v-if="currentDB && (currentDB.from !== 'local' || mongodbStatus === 'Running')"
                    @click="goDashboard()"
                    type="primary"
                    plain
                >
                    {{ $t('database.manage') }}
                </el-button>
            </template>
            <template #rightToolBar v-if="!isOnSetting">
                <el-select
                    v-model="currentDBName"
                    @change="changeDatabase()"
                    class="p-w-200 ml-5"
                    v-if="currentDB"
                    placement="bottom-end"
                >
                    <template #prefix>{{ $t('commons.table.type') }}</template>
                    <el-option-group :label="$t('commons.table.local')">
                        <div v-for="(item, index) in dbOptionsLocal" :key="index">
                            <el-option v-if="item.from === 'local'" :value="item.database" class="optionClass">
                                <span v-if="item.database.length < 25">{{ item.database }}</span>
                                <el-tooltip v-else :content="item.database" placement="top">
                                    <span>{{ item.database.substring(0, 25) }}...</span>
                                </el-tooltip>
                            </el-option>
                        </div>
                        <el-button link type="primary" class="jumpAdd" @click="goRouter('app')" icon="Position">
                            {{ $t('database.goInstall') }}
                        </el-button>
                    </el-option-group>
                    <el-option-group :label="$t('database.remote')">
                        <div v-for="(item, index) in dbOptionsRemote" :key="index">
                            <el-option v-if="item.from === 'remote'" :value="item.database" class="optionClass">
                                <span v-if="item.database.length < 25">{{ item.database }}</span>
                                <el-tooltip v-else :content="item.database" placement="top">
                                    <span>{{ item.database.substring(0, 25) }}...</span>
                                </el-tooltip>
                            </el-option>
                        </div>
                        <el-button link type="primary" class="jumpAdd" @click="goRouter('remote')" icon="Position">
                            {{ $t('database.createRemoteDB') }}
                        </el-button>
                    </el-option-group>
                </el-select>
            </template>
            <template #main v-if="!isOnSetting">
                <div v-if="currentDB && !isOnSetting" class="mt-5">
                    <el-empty
                        v-if="mongodbStatus !== 'Running'"
                        :image-size="80"
                        :style="{ height: `calc(100vh - ${loadHeight()})`, 'background-color': '#000' }"
                        :description="$t('commons.service.serviceNotStarted', ['MongoDB'])"
                    />
                    <el-result
                        v-else
                        icon="success"
                        :title="$t('commons.msg.operationSuccess')"
                        :sub-title="$t('database.manageHelper', ['MongoDB', 'Mongo Express'])"
                    >
                        <template #extra>
                            <el-button type="primary" @click="goDashboard()">{{ $t('database.manage') }}</el-button>
                        </template>
                    </el-result>
                </div>

                <div class="app-warn" v-if="isLoaded && dbOptionsLocal.length === 0 && dbOptionsRemote.length === 0">
                    <div class="flex flex-col gap-2 items-center justify-center w-full sm:flex-row">
                        <span>{{ $t('app.checkInstalledWarn', ['MongoDB']) }}</span>
                        <span @click="goRouter('app')" class="flex items-center justify-center gap-0.5">
                            <el-icon><Position /></el-icon>
                            {{ $t('database.goInstall') }}
                        </span>
                    </div>
                    <div>
                        <img src="@/assets/images/no_app.svg" />
                    </div>
                </div>
            </template>
        </LayoutContent>

        <Setting ref="settingRef" style="margin-top: 30px" />
        <Conn ref="connRef" />

        <PortJumpDialog ref="dialogPortJumpRef" />

        <DialogPro v-model="open" :title="$t('app.checkTitle')" size="small">
            <div class="flex justify-center items-center gap-2 flex-wrap">
                {{ $t('app.checkInstalledWarn', ['Mongo Express']) }}
                <el-link icon="Position" @click="getAppDetail('mongo-express')" type="primary">
                    {{ $t('database.goInstall') }}
                </el-link>
            </div>
            <template #footer>
                <span class="dialog-footer">
                    <el-button @click="open = false">{{ $t('commons.button.cancel') }}</el-button>
                </span>
            </template>
        </DialogPro>
    </div>
</template>

<script lang="ts" setup>
import Setting from '@/views/database/mongodb/setting/index.vue';
import Conn from '@/views/database/mongodb/conn/index.vue';
import AppStatus from '@/components/app-status/index.vue';
import PortJumpDialog from '@/components/port-jump/index.vue';
import { nextTick, onMounted, ref } from 'vue';
import { checkAppInstalled, getAppPort } from '@/api/modules/app';
import { GlobalStore } from '@/store';
import { listDatabases } from '@/api/modules/database';
import { Database } from '@/api/interface/database';
import { routerToName, routerToNameWithQuery } from '@/utils/router';
const globalStore = GlobalStore();

const loading = ref(false);

const settingRef = ref();
const isOnSetting = ref(false);
const mongodbIsExist = ref(false);
const mongodbStatus = ref();

const appStatusRef = ref();

const open = ref(false);
const dialogPortJumpRef = ref();
const mongoExpressPort = ref(0);

const appKey = ref('mongodb');
const appName = ref();

const isLoaded = ref(false);
const dbOptionsLocal = ref<Array<Database.DatabaseOption>>([]);
const dbOptionsRemote = ref<Array<Database.DatabaseOption>>([]);
const currentDB = ref<Database.DatabaseOption>();
const currentDBName = ref();

const onSetting = async () => {
    isOnSetting.value = true;
    settingRef.value!.acceptParams({ status: mongodbStatus.value, database: currentDBName.value, type: appKey.value });
};

const loadHeight = () => {
    return globalStore.openMenuTabs ? '470px' : '380px';
};

const getAppDetail = (key: string) => {
    routerToNameWithQuery('AppAll', { install: key });
};

const goRemoteDB = async () => {
    if (currentDB.value) {
        globalStore.setCurrentDB(currentDBName.value);
    }
    routerToName('MongoDB-Remote');
};

const connRef = ref();
const onLoadConn = async () => {
    connRef.value!.acceptParams({
        from: currentDB.value.from,
        type: currentDB.value.type,
        database: currentDBName.value,
    });
};

const goRouter = async (target: string) => {
    if (target === 'app') {
        routerToNameWithQuery('AppAll', { install: 'mongodb' });
        return;
    }
    routerToName('MongoDB-Remote');
};

const goDashboard = async () => {
    if (mongoExpressPort.value === 0) {
        open.value = true;
        return;
    }
    dialogPortJumpRef.value.acceptParams({ port: mongoExpressPort.value });
};

const changeDatabase = async () => {
    for (const item of dbOptionsLocal.value) {
        if (item.database == currentDBName.value) {
            currentDB.value = item;
            appKey.value = item.type;
            appName.value = item.database;
            appStatusRef.value?.onCheck(appKey.value, appName.value);
            checkStatus();
            return;
        }
    }
    for (const item of dbOptionsRemote.value) {
        if (item.database == currentDBName.value) {
            currentDB.value = item;
            break;
        }
    }
    checkStatus();
};

const loadDBOptions = async () => {
    try {
        const res = await listDatabases('mongodb');
        let datas = res.data || [];
        dbOptionsLocal.value = [];
        dbOptionsRemote.value = [];
        currentDBName.value = globalStore.currentDB;
        for (const item of datas) {
            if (currentDBName.value && item.database === currentDBName.value) {
                currentDB.value = item;
                if (item.from === 'local') {
                    appKey.value = item.type;
                    appName.value = item.database;
                }
            }
            if (item.from === 'local') {
                dbOptionsLocal.value.push(item);
            } else {
                dbOptionsRemote.value.push(item);
            }
        }
        if (currentDB.value) {
            checkStatus();
            return;
        }
        if (dbOptionsLocal.value.length !== 0) {
            currentDB.value = dbOptionsLocal.value[0];
            currentDBName.value = dbOptionsLocal.value[0].database;
            appKey.value = dbOptionsLocal.value[0].type;
            appName.value = dbOptionsLocal.value[0].database;
        }
        if (!currentDB.value && dbOptionsRemote.value.length !== 0) {
            currentDB.value = dbOptionsRemote.value[0];
            currentDBName.value = dbOptionsRemote.value[0].database;
        }
        if (currentDB.value) {
            checkStatus();
        }
    } finally {
        isLoaded.value = true;
    }
};

const checkStatus = async () => {
    loading.value = true;
    if (currentDB.value.from === 'remote') {
        loading.value = false;
        mongodbIsExist.value = true;
        mongodbStatus.value = 'Running';
        return;
    }
    await checkAppInstalled(currentDB.value.type, currentDBName.value)
        .then((res) => {
            mongodbIsExist.value = res.data.isExist;
            mongodbStatus.value = res.data.status;
            loading.value = false;
        })
        .catch(() => {
            loading.value = false;
        });
};

const loadMongoExpressPort = async () => {
    const res = await getAppPort('mongo-express', '');
    mongoExpressPort.value = res.data;
};

const onBefore = () => {
    isOnSetting.value = false;
    loading.value = true;
};

const onAfter = () => {
    nextTick(() => {
        loading.value = false;
        checkStatus();
    });
};

onMounted(() => {
    loadDBOptions();
    loadMongoExpressPort();
});
</script>

<style lang="scss" scoped>
.optionClass {
    width: 100%;
}
</style>
