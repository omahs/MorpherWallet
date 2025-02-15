<template>
	<div class="field">
		<div class="control is-expanded" v-if="!hasRecoveryMethod">
			<GoogleLogin
				class="button is-grey big-button outlined-button is-thick transition-faster"
				:params="{ clientId }"
				:onSuccess="onLogin"
				:onFailure="onError"
				data-cy="googleButton"
			>
				<span class="icon img">
					<img src="@/assets/img/google_logo.svg" alt="Google Logo" />
				</span>
				<span>Google</span>
			</GoogleLogin>
		</div>
		<div v-if="hasRecoveryMethod" class="has-text-centered">
			<div class="control is-expanded" v-if="hasRecoveryMethod">
				<GoogleLogin
					class="button is-danger big-button is-thick transition-faster"
					:params="{ clientId }"
					:onSuccess="onDelete"
					:onFailure="onError"
				>
					<span class="icon img">
						<img src="@/assets/img/google_logo_white.svg" alt="Google Logo" />
					</span>
					<span>{{ $t('recovery.REVOKE_ACCESS') }}</span>
				</GoogleLogin>
			</div>
			<div class="recovery-active is-text-small">
				<span class="icon">
					<i class="fas fa-check-circle"></i>
				</span>
				{{
					$t('recovery.RECOVERY_ACTIVE', {
						currentMethod: 'Google'
					})
				}}
			</div>
		</div>
	</div>
</template>

<script>
import GoogleLogin from 'vue-google-login';
import { sha256 } from './../utils/cryptoFunctions';

import Component, { mixins } from 'vue-class-component';
import { Authenticated, Global } from '../mixins/mixins';
import { Emit } from 'vue-property-decorator';

@Component({
	components: {
		GoogleLogin
	}
})
export default class AddRecoveryGoogle extends mixins(Global, Authenticated) {
	hasRecoveryMethod = false;
	clientId = process.env.VUE_APP_GOOGLE_APP_ID;
	recoveryTypeId = 3;

	@Emit('processMethod')
	processMethod(data) {
		return data;
	}

	async mounted() {
		this.hasRecoveryMethod = await this.hasRecovery(this.recoveryTypeId);
	}

	onError(error) {
		let errorMessage = error.error || error.err || error.message || JSON.stringify(error)
		this.logSentryError('addGoogleRecovery', errorMessage, {
			hasRecoveryMethod: this.hasRecoveryMethod,
			clientId: this.clientId,
			recoveryTypeId: this.recoveryTypeId
		});
		let errorText = error.error || error.err || 'Google login Error';

		if (String(errorText.toLowerCase()).includes('script not loaded correctly')) {
			errorText = 'google_script_blocked';
		}

		this.processMethod({
			success: false,
			error: errorText
		});
	}

	async onLogin(googleUser) {
		this.showSpinner(this.$t('loader.SAVING_KEYSTORE_RECOVERY'));
		const userID = googleUser.getId();
		const key = await sha256(this.clientId + userID);
		const token = (googleUser.Cc || googleUser.Bc).id_token
		
		this.addRecoveryMethod({ key, password: userID, recoveryTypeId: this.recoveryTypeId, token, email: googleUser.getBasicProfile().getEmail(), currentRecoveryTypeId: this.store.recoveryTypeId })
			.then(async () => {
				if (this.$gtag && window.gtag)
					window.gtag('event', 'add_recovery', {
						method: 'google'
					});

				googleUser.disconnect();
				this.showSpinnerThenAutohide(this.$t('loader.SAVED_KEYSTORE_SUCCESSFULLY'));
				this.hasRecoveryMethod = await this.hasRecovery(this.recoveryTypeId);
				this.processMethod({
					success: true,
					method: 'Google',
					enabled: true,
					erorr: ''
				});
			})
			.catch((error) => {
				let errorMessage = error.error || error.err || error.message || JSON.stringify(error)
				console.log('onLogin error', errorMessage)				
				this.logSentryError('addGoogleRecovery', errorMessage, {
					hasRecoveryMethod: this.hasRecoveryMethod,
					clientId: this.clientId,
					recoveryTypeId: this.recoveryTypeId,
					googleUser
				});
				this.showSpinnerThenAutohide(this.$t('loader.SAVED_KEYSTORE_ERROR'));
				this.processMethod({
					success: false,
					method: 'Google',
					enabled: true,
					erorr: ''
				});
			});
	}

	async onDelete(googleUser) {
		this.showSpinner(this.$t('loader.DELETING_KEYSTORE_RECOVERY'));
		const key = await sha256(this.clientId + userID);
		const userID = googleUser.getId();
		const token = (googleUser.Cc || googleUser.Bc).id_token

		this.resetRecoveryMethod({ key, recoveryTypeId: this.recoveryTypeId, token })
			.then(async () => {
				googleUser.disconnect();
				this.showSpinnerThenAutohide(this.$t('loader.DELETED_KEYSTORE_SUCCESSFULLY'));
				this.hasRecoveryMethod = false;
				this.processMethod({
					success: true,
					method: 'Google',
					enabled: false,
					erorr: ''
				});
			})
			.catch((error) => {
				let errorMessage = error.error || error.err || error.message || JSON.stringify(error)
				console.log('onDelete error', errorMessage)					
				this.logSentryError('deleteGoogleRecovery', errorMessage, {
					hasRecoveryMethod: this.hasRecoveryMethod,
					clientId: this.clientId,
					recoveryTypeId: this.recoveryTypeId,
					googleUser
				});
				this.showSpinnerThenAutohide(this.$t('common.ERROR_FIND_USER'));
				this.processMethod({
					success: false,
					method: 'Google',
					enabled: false,
					erorr: ''
				});
			});
	}
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.google-icon {
	color: #fc6404;
}
</style>
