<template>
    <van-pull-refresh v-model="loading" :disabled="disablePull" @refresh="onRefresh" :pulling-text="$t('common.pulling-text')"
                      :loosing-text="$t('common.loosing-text')" :loading-text="$t('common.loading-text')">
        <!-- cell-group version -->
        <van-cell-group :title="$t('dashboard.version')" inset>
            <van-cell :title="$t('dashboard.version-module')" title-style="max-width:35%;" :value="version"
                      url="https://github.com/Asterisk4Magisk/Xray4Magisk" clickable/>
            <van-cell :title="$t('dashboard.version-dashboard')" title-style="max-width:35%;" :value="$t('common.dashboard-version')"
                      url="https://github.com/Asterisk4Magisk/xray4magisk-ksu-webui" clickable/>
        </van-cell-group>
        <!-- cell-group status -->
        <van-cell-group :title="$t('dashboard.status')" inset>
            <van-cell :title="$t('dashboard.status-core-type')" title-style="max-width:35%;" :value="status.coreType"/>
            <van-popover :actions="serviceCmd" @select="execServiceCmd" placement="bottom-end">
                <template #reference>
                    <van-cell :title="$t('dashboard.status-core-status')" title-style="max-width:35%;" :value="coreStatus" clickable/>
                    <div/>
                </template>
            </van-popover>
            <van-cell v-show="running" :title="$t('dashboard.status-core-pid')" title-style="max-width:35%;" :value="status.pid"/>
            <van-popover :actions="proxyCmd" @select="execProxyCmd" placement="bottom-end">
                <template #reference>
                    <van-cell :title="$t('dashboard.status-method')" title-style="max-width:35%;" :value="status.method" clickable/>
                </template>
            </van-popover>
        </van-cell-group>
        <!-- cell-group tool -->
        <van-cell-group :title="$t('dashboard.tool')" inset>
            <van-space/>
            <van-row :gutter="[0, 11]" justify="space-around">
                <van-col span="11">
                    <van-button plain hairline type="default" block @click="execUpdateCmd('core')">
                        {{ $t('dashboard.tool-update-core') }}
                    </van-button>
                </van-col>
                <van-col span="11">
                    <van-button plain hairline type="default" block @click="execUpdateCmd('adghome')">
                        {{ $t('dashboard.tool-update-adghome') }}
                    </van-button>
                </van-col>
                <van-col v-if="status.coreType==='v2ray'||status.coreType==='xray'" span="11">
                    <van-button plain hairline type="default" block @click="execUpdateCmd('geodata')">
                        {{ $t('dashboard.tool-update-geodata') }}
                    </van-button>
                </van-col>
                <van-col span="11">
                    <van-button plain hairline type="default" block @click="execUpdateCmd('subscribe')">
                        {{ $t('dashboard.tool-update-subscribe') }}
                    </van-button>
                </van-col>
                <van-col v-if="status.coreType==='mihomo'" span="11">
                    <van-button plain hairline type="default" block @click="execUpdateCmd('yacd-meta')">
                        {{ $t('dashboard.tool-update-yacd-meta') }}
                    </van-button>
                </van-col>
                <van-col v-if="status.coreType==='mihomo'" span="11">
                    <van-button plain hairline type="default" block @click="execUpdateCmd('metacubexd')">
                        {{ $t('dashboard.tool-update-metacubexd') }}
                    </van-button>
                </van-col>
                <van-col span="11">
                    <van-button plain hairline type="default" block @click="execUpdateCmd('tun2socks')">
                        {{ $t('dashboard.tool-update-tun2socks') }}
                    </van-button>
                </van-col>
                <van-col v-if="status.coreType==='mihomo'" span="11">
                    <van-button plain hairline type="default" block @click="switchClicked(false)">
                        {{ $t('dashboard.tool-switch') }}
                    </van-button>
                </van-col>
            </van-row>
            <van-space/>
        </van-cell-group>
        <!-- switch chooser -->
        <van-popup v-model:show="switchChooser" round :style="{ width: '90%' ,maxHeight:'85%'}" @closed="refresh">
            <van-radio-group v-model="switchChooserChecked">
                <van-cell-group inset>
                    <van-cell v-for="(item, idx) in switchResult.result" :title="item" clickable
                              @click="switchChecked(switchCustom,idx)">
                        <template #right-icon>
                            <van-radio :name="idx.toString()"/>
                        </template>
                    </van-cell>
                </van-cell-group>
            </van-radio-group>
        </van-popup>
        <!-- stdout receiver -->
        <StdoutReceiver v-model:show="receiver" v-model:stdout="stdout" @closed="refresh"/>
    </van-pull-refresh>
</template>

<script setup>
import {ref} from 'vue';
import {callApi, execCmd, execXrayHelperCmd, readFile, saveFile} from "./tools.js";
import i18n from "./locales/i18n.js"
import StdoutReceiver from "./components/StdoutReceiver.vue";

defineProps(["theme"]);
const MODULE_PROP = "/data/adb/modules/xray4magisk/module.prop"
const loading = ref(false)
const disablePull = ref(false)
const running = ref(false)
const receiver = ref(false)
const switchCustom = ref(false)
const switchChooser = ref(false)
const switchChooserChecked = ref("")
const switchResult = ref({result: []})
const stdout = ref(i18n.global.t('common.waiting-text'))
const version = ref("")
const status = ref({api: "", coreType: "xray", pid: "", method: "", dataDir: ""})
const coreStatus = ref(i18n.global.t('dashboard.status-core-status-stopped'))

const serviceCmd = [
    {text: 'start', value: 'start'},
    {text: 'stop', value: 'stop'},
    {text: 'restart', value: 'restart'},
    {text: 'status', value: 'status'},
]
const execServiceCmd = (operation) => {
    receiver.value = true
    disablePull.value = true
    setTimeout(() => {
        execXrayHelperCmd("service " + operation.value).then(value => {
            if(operation.value==="start"||operation.value==="restart"){
                execXrayHelperCmd("proxy refresh").then(value2 => {
                    stdout.value=value+'\n'+value2;
                })
            }else if (operation.value==="stop"){
                execXrayHelperCmd("proxy disable").then(value2 => {
                    stdout.value=value+'\n'+value2;
                })
            }
            stdout.value = value
        })
    }, 300)
}
const proxyCmd = [
    {text: 'enable', value: 'enable'},
    {text: 'disable', value: 'disable'},
    {text: 'refresh', value: 'refresh'},
]
const execProxyCmd = (operation) => {
    receiver.value = true
    disablePull.value = true
    setTimeout(() => {
        execXrayHelperCmd("proxy " + operation.value).then(value => {
            stdout.value = value
        })
    }, 300)
}
const execUpdateCmd = (comp) => {
    receiver.value = true
    disablePull.value = true
    setTimeout(() => {
        execXrayHelperCmd("update " + comp).then(value => {
            stdout.value = value
        })
    }, 300)
}
const switchClicked = async (custom) => {
    disablePull.value = true
    switchChooser.value = true
    switchCustom.value = custom
    switchResult.value = custom ? await callApi(`get switch custom`) : await callApi(`get switch`)
    let idx = custom ? localStorage.getItem('switchCustomIdx') : localStorage.getItem('switchIdx')
    if (typeof idx != "undefined" && idx != null) {
        switchChooserChecked.value = idx
    } else {
        switchChooserChecked.value = ""
    }
}
const switchChecked = (custom, idx) => {
    let api = custom ? `set switch custom ${idx}` : `set switch ${idx}`
    callApi(api).then(value => {
        switchChooser.value = false
        if (custom) {
            localStorage.setItem('switchCustomIdx', idx)
            localStorage.removeItem('switchIdx')
        } else {
            localStorage.setItem('switchIdx', idx)
            localStorage.removeItem('switchCustomIdx')
        }
        if (value.ok) {
            showToast(i18n.global.t('dashboard.tool-switch-success'))
        } else {
            showToast(i18n.global.t('dashboard.tool-switch-failed'))
        }
    })
}
const refresh = () => {
    version.value = ""
    status.value = {api: "", coreType: "xray", pid: "", method: "", dataDir: ""}
    running.value = false
    coreStatus.value = i18n.global.t('dashboard.status-core-status-stopped')
    stdout.value = i18n.global.t('common.waiting-text')
    disablePull.value = false
    initVersion()
    initStatus()
}
const onRefresh = () => {
    setTimeout(() => {
        refresh()
        loading.value = false
    }, 500)
};
const initVersion = () => {
    execCmd("grep version= " + MODULE_PROP).then(value => {
        version.value = value.split('=')[1]
    })
}
const initStatus = () => {
    callApi("get status").then(value => {
        status.value = value
        if (status.value.pid.length > 0) {
            running.value = true
            coreStatus.value = i18n.global.t('dashboard.status-core-status-running')
        }
    }).catch(ex => {
        showToast(i18n.global.t('dashboard.status-get-failed') + ex)
    })
}
initVersion()
initStatus()
</script>

<style>
</style>
