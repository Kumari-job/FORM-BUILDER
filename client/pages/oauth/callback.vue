<template>
   <template>
  <div class="flex flex-grow mt-6 mb-10">
    <div class="w-full md:w-2/3 md:mx-auto md:max-w-md px-4">
        <div class="m-10" v-if="loading">
            <h3 class="my-6 text-center">
              Please wait...
            </h3>
            <Loader class="h-6 w-6 mx-auto m-10" />
        </div>
        <div class="m-6 flex flex-col items-center space-y-4" v-else>
            <p class="text-center"> Unable to sign it at the moment. </p>
            <v-button
              :to="{ name: 'login' }"
            >Back to login</v-button>
        </div>
    </div>
  </div>
</template>
</template>

<script setup>
const router = useRouter()
const route = useRoute()
const authStore = useAuthStore()
const workspacesStore = useWorkspacesStore()
const formsStore = useFormsStore()
const logEvent = useAmplitude().logEvent
const loading = ref(true);

definePageMeta({
    alias: [
        '/oauth/:provider/callback'
    ]
})

function handleCallback() {

    const code = route.query.code
    const provider = route.params.provider
    opnFetch(`/oauth/${provider}/callback`, {
        method: 'POST',
        params: {
            code
        }
    }).then(async (data) => {
        authStore.setToken(data.token)
        const [userDataResponse, workspacesResponse] = await Promise.all([
            opnFetch("user"),
            fetchAllWorkspaces(),
        ])
        authStore.setUser(userDataResponse)
        workspacesStore.set(workspacesResponse.data.value)

        // Load forms
        formsStore.loadAll(workspacesStore.currentId)
        if (!data.new_user) {
            logEvent("login", { source: provider })
            try {
                useGtm().trackEvent({
                    event: 'login',
                    source: provider
                })
            } catch (error) {
                console.error(error)
            }
            router.push({ name: "home" })
            return
        } else {
            logEvent("register", { source: provider })
            router.push({ name: "forms-create" })
            useAlert().success("Success! You're now registered with your Google account! Welcome to OpnForm.")
            try {
                useGtm().trackEvent({
                    event: 'register',
                    source: provider
                })
            } catch (error) {
                console.error(error)
                useAlert().error(error)
            }
        }
    }).catch(error => {
        useAlert().error(error.response._data.message)
        loading.value = false;
    })
}
onMounted(() => {
    handleCallback()
})

</script>