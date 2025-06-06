<template>
    <header title="Settings" />
    <main-container>
        <vee-form
            v-slot="{ errors, isSubmitting }"
            :validation-schema="schema"
            :initial-values="user"
            @submit="onSubmit"
        >
            <form-group
                title="Account"
                sub-title="Manage Your Account"
            >
                <PasswordField
                    name="currentPassword"
                    label="Current Password"
                    :error="errors.currentPassword"
                />

                <div>
                    <label for="name">Name</label>
                    <field
                        id="name"
                        type="text"
                        name="name"
                        autocomplete="name"
                        class="field"
                        :class="{ 'has-error': errors.name }"
                    />
                    <div class="field-error-message">
                        {{ errors.name }}
                    </div>
                </div>

                <div>
                    <label for="email">Email address</label>
                    <field
                        id="email"
                        type="text"
                        name="email"
                        autocomplete="email"
                        class="field"
                        :class="{ 'has-error': errors.email }"
                    />
                    <div class="field-error-message">
                        {{ errors.email }}
                    </div>
                </div>

                <PasswordField
                    name="newPassword"
                    label="New Password"
                    :error="errors.newPassword"
                />

                <label>Login using your GitHub account</label>

                <button 
                v-if="!connectedProviders.includes('github')"
                    type="button"
                    class="btn btn-primary"
                    @click="linkSocial('github')"
                >
                <icon-fa6-brands:github
                    class="icon"
                    aria-hidden="true"
                />
                Link GitHub
            </button>
            
                <button
                    v-else
                    type="button"
                    class="btn btn-danger"
                    @click="unlinkSocial('github')"
                >
                    <icon-fa6-brands:github
                        class="icon"
                        aria-hidden="true"
                    />
                    Unlink from GitHub
                </button>

            </form-group>
            

            <div class="form-submit">
                <button
                    :disabled="isSubmitting"
                    type="submit"
                    class="btn btn-primary"
                >
                    <icon-fa6-solid:floppy-disk
                        v-if="!isSubmitting"
                        class="icon"
                        aria-hidden="true"
                    />
                    <icon-line-md:loading-twotone-loop
                        v-else
                        class="icon"
                    />
                    Save
                </button>
            </div>
        </vee-form>

        <form-group title="Passkey Devices" class="form-group-table">
            <data-table
                v-if="passkeys"
                :columns="[
                    { key: 'name', name: 'Name', sortable: true },
                    { key: 'createdAt', name: 'Created At', sortable: true },
                ]"
                :data="passkeys"
            >
                <template #cell-actions="{ row }">
                    <button
                        type="button"
                        class="tooltip-position-left"
                        data-tooltip="Delete"
                        @click="removePasskey(row.id)"
                    >
                        <icon-fa6-solid:trash aria-hidden="true" class="icon icon-error" />
                    </button>
                </template>
            </data-table>
        </form-group>

        <div class="form-submit">
            <button
                type="button"
                class="btn btn-primary"
                @click="showPasskeyCreationModal = true"
            >
                <icon-material-symbols:passkey
                    class="icon icon-passkey"
                    aria-hidden="true"
                />
                Add a new Device
            </button>
        </div>

        <form-group title="Sessions" class="form-group-table">
            <data-table
                v-if="sessions && sessions.length"
                :columns="[
                    { key: 'userAgent', name: 'User Agent', sortable: true },
                    { key: 'createdAt', name: 'Created At', sortable: true },
                ]"
                :data="sessions"
            >
                <template #cell-actions="{ row }">
                    <button
                        v-if="row.token !== session.data?.session.token"
                        type="button"
                        class="tooltip-position-left"
                        data-tooltip="Delete"
                        @click="removeSession(row)"
                    >
                        <icon-fa6-solid:trash aria-hidden="true" class="icon icon-error" />
                    </button>
                </template>
            </data-table>
        </form-group>

        <form-group title="Deleting your Account">
                <p>
                    Once you delete your account, you will lose all data associated with it.
                    All owning organization will be also deleted with all shops associated.
                </p>

                <button
                    type="button"
                    class="btn btn-danger"
                    @click="showAccountDeletionModal = true"
                >
                    <icon-fa6-solid:trash class="icon icon-trash" />
                    Delete account
                </button>
        </form-group>

        <Modal
            :show="showAccountDeletionModal"
            @close="showAccountDeletionModal = false"
        >
            <template #icon>
                <icon-fa6-solid:triangle-exclamation
                    class="icon icon-error"
                    aria-hidden="true"
                />
            </template>

            <template #title>
                Deactivate account
            </template>

            <template #content>
                Are you sure you want to deactivate your account? All of your data will be permanently removed
                from our servers forever. This action cannot be undone.
            </template>
            
            <template #footer>
                <button
                    type="button"
                    class="btn btn-danger"
                    @click="deleteUser"
                >
                    Deactivate
                </button>

                <button
                    ref="cancelButtonRef"
                    type="button"
                    class="btn btn-cancel"
                    @click="showAccountDeletionModal = false"
                >
                    Cancel
                </button>
            </template>
        </Modal>

        <Modal :show="showPasskeyCreationModal"
            @close="showPasskeyCreationModal = false">
            <template #title>
                Add a Passkey Device
            </template>

            <template #icon>
                <icon-fa6-solid:key
                    class="icon icon-info"
                    aria-hidden="true"
                />
            </template>

            <template #content>
                Please give a name to your new Passkey Device.
                <field
                        v-model="passKeyName"
                        type="text"
                        name="name"
                        autocomplete="off"
                        class="field"
                    />
            </template>

            <template #footer>
                <button
                    type="button"
                    class="btn btn-primary"
                    @click="createPasskey"
                >
                    Create
                </button>

                <button
                    ref="cancelButtonRef"
                    type="button"
                    class="btn btn-cancel"
                    @click="showPasskeyCreationModal = false"
                >
                    Cancel
                </button>
            </template>
        </Modal>
    </main-container>
</template>

<script setup lang="ts">
import type { Passkey } from 'better-auth/plugins/passkey';
import { Field, Form as VeeForm } from 'vee-validate';
import { ref } from 'vue';
import * as Yup from 'yup';

import { useAlert } from '@/composables/useAlert';
import { authClient } from '@/helpers/auth-client';
import { trpcClient } from '@/helpers/trpc';
import type { Session } from 'better-auth/types';

const session = authClient.useSession();

const alert = useAlert();

const passKeyName = ref('');

const passkeys = ref<Passkey[] | null>([]);
const sessions = ref<Session[] | null>([]);
const connectedProviders = ref<string[]>([]);

authClient.passkey.listUserPasskeys().then((data) => {
    passkeys.value = data.data;
});

authClient.listSessions().then((data) => {
    sessions.value = data.data;
});

const user = {
    name: session.value.data?.user?.name ?? '',
    email: session.value.data?.user?.email ?? '',
    currentPassword: '',
    newPassword: '',
};

async function loadLinkedAccounts() {
    authClient.listAccounts().then((data) => {
        if (data.data) {
            connectedProviders.value = data.data?.map(
                (account) => account.provider,
            );
        }
    });
}
loadLinkedAccounts();

const showAccountDeletionModal = ref(false);
const showPasskeyCreationModal = ref(false);

const schema = Yup.object().shape({
    currentPassword: Yup.string().required('Current password is required'),
    email: Yup.string().email().required(),
    name: Yup.string().min(5, 'Name must be at least 5 characters'),
    newPassword: Yup.string()
        .transform((x) => (x === '' ? undefined : x))
        .min(8, 'Password must be at least 8 characters'),
});

async function onSubmit(values: Record<string, unknown>) {
    await authClient.changeEmail({
        newEmail: values.email as string,
    });

    if (values.displayName) {
        await authClient.updateUser({
            name: values.displayName as string,
        });
    }

    if (values.newPassword) {
        await authClient.changePassword({
            currentPassword: values.currentPassword as string,
            newPassword: values.newPassword as string,
            revokeOtherSessions: true,
        });
    }
}

async function deleteUser() {
    try {
        await trpcClient.account.deleteCurrentUser.mutate();
        await authClient.deleteUser();
    } catch (err) {
        alert.error(err instanceof Error ? err.message : String(err));
    }

    showAccountDeletionModal.value = false;
}

async function createPasskey() {
    if (!passKeyName.value) {
        alert.error('Please provide a name for the Passkey Device.');
        return;
    }

    await authClient.passkey.addPasskey({ name: passKeyName.value });
    authClient.passkey.listUserPasskeys().then((data) => {
        passkeys.value = data.data;
    });
    showPasskeyCreationModal.value = false;
}

async function removePasskey(id: string) {
    await authClient.passkey.deletePasskey({ id });
    authClient.passkey.listUserPasskeys().then((data) => {
        passkeys.value = data.data;
    });
}

async function removeSession(session: Session) {
    await authClient.revokeSession({ token: session.token });
    authClient.listSessions().then((data) => {
        sessions.value = data.data;
    });
}

async function linkSocial(provider: 'github') {
    try {
        await authClient.linkSocial({
            provider,
            callbackURL: window.location.href,
        });
    } catch (err) {
        alert.error(err instanceof Error ? err.message : String(err));
    }
}

async function unlinkSocial(providerId: string) {
    try {
        await authClient.unlinkAccount({ providerId });
        await loadLinkedAccounts();
        alert.success(`Successfully unlinked your account from ${providerId}`);
    } catch (err) {
        alert.error(err instanceof Error ? err.message : String(err));
    }
}
</script>
