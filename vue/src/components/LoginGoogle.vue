<template>
	<div class="field">
		<div class="control is-expanded">
			<GoogleLogin class="button is-grey big-button outlined-button is-thick transition-faster" v-if="clientId"
				:params="{ clientId }" :onSuccess="onLogin" :onFailure="onError" data-cy="googleButton">
				<span class="icon img">
					<img src="@/assets/img/google_logo.svg" alt="Google Logo" />
				</span>
				<span v-if="signIn">{{  $t('auth.SIGN_UP_GOOGLE') }} </span>
				<span v-else-if="unlock">{{  $t('auth.UNLOCK_GOOGLE') }} </span>
				<span v-else-if="update">{{  $t('auth.UPDATE_GOOGLE') }} </span>
				<span v-else>{{  $t('auth.LOG_IN_GOOGLE') }} </span>
			</GoogleLogin>
		</div>

	</div>
</template>

<script>
import GoogleLogin from 'vue-google-login';
import { sha256 } from '../utils/cryptoFunctions';

import Component, { mixins } from 'vue-class-component';
import { Authenticated, Global } from '../mixins/mixins';
import { Emit, Prop, Watch } from 'vue-property-decorator';

@Component({
	components: {
		GoogleLogin
	}
})
export default class AddRecoveryGoogle extends mixins(Global, Authenticated) {
	hasRecoveryMethod = false;
	clientId = process.env.VUE_APP_GOOGLE_APP_ID;
	recoveryTypeId = 3;

	@Prop({ default: false })
	signIn;

	@Prop({ default: false })
	unlock;

	@Prop({ default: false })
	update;	

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
		const userID = googleUser.getId();
		const key = this.clientId + userID;
		const token = (googleUser.Cc || googleUser.Bc).id_token
		this.processMethod({
			success: true,
			userID, key, token, recoveryTypeId: this.recoveryTypeId, email: googleUser.getBasicProfile().getEmail()
		});
		return;
	}

}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.google-icon {
	color: #fc6404;
}
</style>
