<template>
    <b-modal
        id="modalDecodePrivateKey"
        @hidden="$emit('init')"
        @show="onShow()"
        no-close-on-esc
        no-close-on-backdrop
        hide-header-close
        centered
        scrollable
        title="Reclaim ownership over your assets"
    >
        <div class="w-100 text-center" v-if="busy">
            <b-spinner variant="dark" />
        </div>
        <template v-else>
            <b-alert show variant="danger" v-if="error">
                {{ error }}
            </b-alert>

            <p>
                Your new address:<br />
                <code>{{ account.address }}</code>
            </p>
            <p>
                You might have pending withdrawals in your emporary wallet. Please provide your password to transfer the
                ownership of your assets to your new wallet.
            </p>
            <form @submit.prevent="onSubmit()" id="formPassword">
                <b-form-input
                    autofocus
                    size="lg"
                    v-model="password"
                    type="password"
                    placeholder="Enter your password"
                />
            </form>
        </template>
        <template v-slot:modal-footer>
            <b-button class="mt-3 rounded-pill" block variant="primary" form="formPassword" type="submit">
                Transfer ownership
            </b-button>
        </template>
    </b-modal>
</template>

<script lang="ts">
import Web3 from 'web3';
import { UserProfile } from '@/store/modules/account';
import { BLink, BAlert, BButton, BSpinner, BModal, BFormInput } from 'bootstrap-vue';
import { Component, Vue } from 'vue-property-decorator';
import { mapGetters } from 'vuex';
import { isPrivateKey, NetworkProvider, signCall } from '@/utils/network';
import { Account } from 'web3-core/types/index';
import { decryptString } from '@/utils/decrypt';
import { IMemberships, Membership } from '@/store/modules/memberships';
import { TNetworks } from '@/store/modules/network';

@Component({
    name: 'ModalDecodePrivateKey',
    components: {
        'b-form-input': BFormInput,
        'b-alert': BAlert,
        'b-link': BLink,
        'b-modal': BModal,
        'b-spinner': BSpinner,
        'b-button': BButton,
    },
    computed: mapGetters({
        profile: 'account/profile',
        privateKey: 'account/privateKey',
        memberships: 'memberships/all',
    }),
})
export default class ModalDecodePrivateKey extends Vue {
    busy = true;
    error = '';
    password = '';
    decryptedPrivateKey = '';

    tempAccount: Account | null = null;
    account: Account | null = null;
    web3!: Web3;

    // getters
    profile!: UserProfile;
    privateKey!: string;
    memberships!: IMemberships;
    networks!: TNetworks;

    onShow() {
        this.web3 = this.networks[NetworkProvider.Main];
        this.account = this.web3.eth.accounts.privateKeyToAccount(this.privateKey) as any;
        this.busy = false;
    }

    async onSubmit() {
        this.busy = true;

        try {
            const tempPrivateKey = decryptString(this.profile.privateKey, this.password);

            if (isPrivateKey(tempPrivateKey)) {
                this.tempAccount = this.web3.eth.accounts.privateKeyToAccount(tempPrivateKey);
            } else {
                throw new Error('Not a valid key');
            }

            await this.$store.dispatch('memberships/getAll');

            for (const id in this.memberships) {
                await this.transferOwnership(this.memberships[id]);
            }

            await this.$store.dispatch('account/getProfile');

            if (this.profile.address !== this.tempAccount?.address) {
                this.$bvModal.hide('modalDecodePrivateKey');
            } else {
                throw new Error('Account not patched');
            }
        } catch (error) {
            this.error = (error as Error).message;
        } finally {
            this.busy = false;
        }
    }

    async transferOwnership(membership: Membership) {
        try {
            if (this.tempAccount && this.account) {
                await this.$store.dispatch('network/setNetwork', {
                    npid: membership.network,
                    privateKey: this.privateKey,
                });

                const calldata = await signCall(
                    this.web3,
                    membership.poolAddress,
                    'upgradeAddress',
                    [this.tempAccount.address, this.account.address],
                    this.tempAccount.privateKey,
                );

                if (!calldata.error) {
                    const r = await this.$store.dispatch('assetpools/upgradeAddress', {
                        poolAddress: membership.poolAddress,
                        newAddress: this.account.address,
                        data: calldata,
                    });

                    if (r.error) {
                        throw new Error('Upgrading address for pool failed');
                    }
                }
            }
        } catch (error) {
            this.error = String(error);
        }
    }
}
</script>
