<template>
    <el-card class="box-card">
        <template #header>
            <div class="clearfix">
                <span>Log in</span>
            </div>
        </template>
        <el-form
            ref="form"
            :model="model"
            label-width="120px"
        >
            <el-form-item
                label="Token"
                prop="token"
            >
                <el-input v-model="model.token" />
            </el-form-item>
            <el-divider>or</el-divider>
            <el-form-item
                label="email address"
                prop="email"
            >
                <el-input v-model="model.email" />
            </el-form-item>
            <el-form-item
                label="Global Token"
                prop="globalToken"
            >
                <el-input v-model="model.globalToken" />
            </el-form-item>

            <el-form-item label="Remember the voucher">
                <el-switch v-model="model.save" />
            </el-form-item>
            <el-form-item>
                <el-button
                    type="primary"
                    :loading="isLoading"
                    @click="submit"
                >
                    submit
                </el-button>
                <el-button @click="reset">
                    Repossess
                </el-button>
            </el-form-item>
        </el-form>
    </el-card>
</template>

<script lang="ts" setup>
import { reactive, ref, computed } from 'vue'
import { useUserStore } from '@/store/user'
import { ElMessage } from 'element-plus'
import { useRouter } from 'vue-router'
import { userTokenVerify, userEmailVerify } from '@/api/account'

const model = reactive({
    token: '',
    save: false,
    email: '',
    globalToken: '',
})

const isLoading = ref(false)

const useToken = computed(() => {
    if (model.token.length === 40) {
        return true
    }

    return false
})

const router = useRouter()
const userStore = useUserStore()

if (!userStore.saveToken) {
    userStore.logout()
}

model.token = userStore.token
model.save = userStore.saveToken
model.email = userStore.email
model.globalToken = userStore.globalToken

if (model.save) {
    submit()
}


function onLoginSuccessful() {
    ElMessage('Log in successfully, and the management page is about to jump to the management page')

    setTimeout(() => {
        router.push({ name: 'ZoneList' })
    }, 1000)
}

async function submit() {
    isLoading.value = true
    if (useToken.value) {
        const status = await userTokenVerify(model.token)

        if (status) {
            userStore.loginByToken(model)
            onLoginSuccessful()
        } else {
            ElMessage({
                type: 'error',
                message: 'Token error',
            })
        }
    } else {
        const status = await userEmailVerify(model.email, model.globalToken)

        if (status) {
            userStore.loginByEmail(model)
            onLoginSuccessful()
        } else {
            ElMessage({
                type: 'error',
                message: 'Email or global key error',
            })
        }
    }

    isLoading.value = false
}

function reset() {
    model.token = ''
    model.save = false
}
</script>

<style lang="scss" scoped>
    .login-panel {
        margin: auto;
        margin-top: 20px;
        max-width: 400px;
    }
</style>
