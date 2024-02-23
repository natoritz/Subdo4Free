<template>
    <el-dialog
        :title="`${modeLocale} Record`"
        :modelValue="dialogVisible"
        class="container"
        @update:modelValue="handleClose"
        width="unset"
    >
        <div>
            <el-form
                ref="ruleFormRef"
                label-width="120px"
                :model="model"
                :rules="validationRules"
            >
                <el-form-item
                    prop="name"
                >
                    <template #label>
                        <span>
                            <FontAwesomeIcon
                                icon="file"
                            /> name
                        </span>
                    </template>
                    <el-input v-model="model.name" />
                </el-form-item>
                <el-form-item
                    prop="content"
                >

                    <template #label>
                        <span>
                            <FontAwesomeIcon
                                icon="font"
                            /> Target
                        </span>
                    </template>
                    <el-input v-model="model.content" />
                </el-form-item>
                <el-form-item
                    v-if="requirePriority"
                    prop="priority"
                >
                    <template #label>
                        <span>
                            <FontAwesomeIcon
                                icon="sort-numeric-up"
                            /> priority
                        </span>
                    </template>
                    <el-input-number v-model="model.priority" />
                </el-form-item>
                <el-form-item
                    prop="type"
                >
                    <template #label>
                        <span>
                            <FontAwesomeIcon
                                icon="globe-asia"
                            /> type
                        </span>
                    </template>
                    <el-select
                        v-model="model.type"
                        placeholder="选择"
                    >
                        <el-option
                            v-for="(item, index) in dnsTypes"
                            :key="index"
                            :label="item"
                            :value="item"
                        />
                    </el-select>
                </el-form-item>
                <el-form-item
                    prop="ttl"
                >
                    <template #label>
                        <span>
                            <FontAwesomeIcon
                                icon="clock"
                            /> TTL
                        </span>
                    </template>
                    <div style="display: grid; grid-row-gap: 10px;">
                        <el-input-number
                            v-model="model.ttl"
                            :disabled="model.autoTTL"
                        />

                        <el-switch
                            v-model="model.autoTTL"
                            active-text="automatic TTL"
                        />
                    </div>
                </el-form-item>
                <el-form-item
                    prop="proxied"
                >
                    <template #label>
                        <span>
                            <FontAwesomeIcon
                                icon="cloud"
                            /> acting
                        </span>
                    </template>
                    <el-checkbox v-model="model.proxied">
                        Cloudflare CDN
                    </el-checkbox>
                </el-form-item>
            </el-form>
        </div>
        <template #footer>
            <span class="dialog-footer">
                <el-button
                    type="primary"
                    :loading="isLoading"
                    @click="createRecord"
                >keep</el-button>
                <el-button @click="handleClose">Cancel</el-button>
            </span>
        </template>
    </el-dialog>
</template>

<script lang="ts" setup>
import { ref, reactive, Ref, computed } from 'vue'
import { ElMessage } from 'element-plus'
// import { Vue, Component, Prop } from 'vue-property-decorator'
import { DnsRecordTypeEnum } from '@/api/enum'
import { createDnsRecord, putRecord } from '@/api/dns'
import { DnsRecordType, CloudflareDnsRecord } from '@/api'
import type { FormInstance, FormRules } from 'element-plus'

import { FontAwesomeIcon } from '@fortawesome/vue-fontawesome'
const ruleFormRef = ref<FormInstance>()
const emits = defineEmits<{
  (event: 'close'): void
  (event: 'success'): void
}>()


const props = withDefaults(defineProps<{
    zone: string
}>(), {

})

type RecordModalType = {
    name: string
    content: string
    type: DnsRecordType
    ttl: number
    autoTTL?: boolean
    proxied: boolean
    priority?: number
}

const DefaultModalValue: RecordModalType = {
    name: '',
    content: '',
    type: 'A' as DnsRecordType,
    ttl: 300,
    autoTTL: true,
    proxied: false,
    priority: 0,
}

const isLoading = ref(false)
const dialogVisible = ref(false)
const record: Ref<CloudflareDnsRecord | null> = ref(null)

const model = ref(DefaultModalValue)

const requirePriority = computed(() => {
    return model.value.type === 'MX' || model.value.type === 'SRV'
})

const dnsTypes = computed(() => {
    return Object.keys(DnsRecordTypeEnum)
})

const isEditMode = computed(() => {
    return !!record.value
})

const modeLocale = computed(() => {
    return isEditMode.value ? 'Revise' : 'Newly built'
})

function display() {
    dialogVisible.value = true
}

function handleClose() {
    dialogVisible.value = false
    emits('close')
}


const validationRules = reactive<FormRules>({
    name: [{ required: true, message: 'Please enter the record name', trigger: 'blur' } ],
    content: [{ required: true, message: 'Please enter the record value', trigger: 'blur' } ],
    ttl: [
        { required: true, message: 'Please enter the record TTL', trigger: 'blur' },
        { type: 'number', message: 'TTL Need a number', trigger: 'blur' },
    ],
    type: [{ required: true, message: 'Please enter the record type', trigger: 'blur' } ],
})

function setRecord(r: CloudflareDnsRecord) {
    record.value = r
}

function reset() {
    model.value = Object.assign({}, DefaultModalValue)
    record.value = null
}

function setModel(m: Partial<RecordModalType>) {
    model.value = Object.assign(model.value, m)
}

async function createRecord() {
    const valid = await ruleFormRef.value?.validate()
    if (!valid) {
        return
    }

    isLoading.value = true

    const submit = async (doc: RecordModalType) => {
        if (isEditMode.value) {
            if (!record.value) {
                return
            }
            return await putRecord(record.value, doc)
        } else {
            return await createDnsRecord(props.zone, doc)
        }
    }

    const requestBody: RecordModalType = {
        name: model.value.name,
        content: model.value.content,
        type: model.value.type,
        ttl: (model.value.autoTTL) ? 1 : model.value.ttl,
        proxied: model.value.proxied,
    }

    if (requirePriority.value) {
        requestBody.priority = model.value.priority
    }

    const res = await submit(requestBody)

    if (!res) {
        emits('success')
        ElMessage(`${modeLocale.value}Success, being refreshed`)
        handleClose()
    } else {
        ElMessage({
            type: 'error',
            message: `${modeLocale.value}Failure, message: ${res}`,
        })
    }

    isLoading.value = false
}

defineExpose({
    display,
    reset,
    setModel,
    setRecord,
})
</script>
