<template>
	<div class="container">
		<vue-recaptcha
			ref="recaptcha"
			size="invisible"
			:sitekey="recaptchaSiteKey"
			:load-recaptcha-script="true"
			@verify="onCaptchaVerified"
			@error="onCaptchaError"
			@expired="onCaptchaExpired"
			@render="onCaptchaLoaded"
			style="display: none"
		/>
		<img
			v-if="(store.twoFaRequired.email || store.twoFaRequired.needConfirmation) && !store.twoFaRequired.authenticator"
			src="@/assets/img/email_verification.svg"
			alt="Email 2FA image"
			class="mb-3"
		/>
		<img v-if="store.twoFaRequired.authenticator" src="@/assets/img/authenticator.svg" alt="Phone authenticator image" class="mb-3" />
		<h2 data-cy="verificationTitle" class="title">{{ $t('settings.2_STEP_VERIFICATION') }}</h2>
		<p v-if="(store.twoFaRequired.email || store.twoFaRequired.needConfirmation) && !store.twoFaRequired.authenticator" class="subtitle">
			{{ $t('2fa.ENTER_EMAIL_CODE') }}
		</p>
		<p v-if="store.twoFaRequired.authenticator && !store.twoFaRequired.email && !store.twoFaRequired.needConfirmation" class="subtitle">
			{{ $t('2fa.ENTER_AUTH_CODE') }}
		</p>
		<p v-if="store.twoFaRequired.email && store.twoFaRequired.authenticator" class="subtitle">
			{{ $t('2fa.ENTER_BOTH_CODES') }}
		</p>
		<form v-on:submit.prevent="validateCode" novalidate>
			<div class="field" v-if="store.twoFaRequired.email || store.twoFaRequired.needConfirmation">
				<label class="label">{{ $t('2fa.EMAIL_CODE') }}</label>
				<div class="control">
					<input
						type="number"
						inputmode="numeric"
						min="100000"
						max="999999"
						class="input"
						name="emailCode"
						id="emailCode"
						data-cy="emailCode"
						v-model="emailCode"
						ref="email_code"
						@keypress="handleKeyPress" 
					/>
				</div>
			</div>
			<div class="field" v-if="store.twoFaRequired.authenticator">
				<label class="label">{{ $t('2fa.AUTH_CODE') }}</label>
				<div class="control">
					<input
						type="number"
						inputmode="numeric"
						class="input"
						name="authenticatorCode"
						id="authenticatorCode"
						data-cy="authenticatorCode"
						ref="auth_code"
						v-model="authenticatorCode"
						@keypress="handleKeyPress" 
					/>
				</div>
			</div>

			<div class="error" v-if="logonError">
				<p>⚠️ <span v-html="logonError"></span></p>
			</div>

			<button class="button is-green big-button is-login transition-faster mt-5" type="submit" data-cy="unlock">
				<span class="text">{{ $t('common.SUBMIT') }}</span>
			</button>

			<button v-on:click="logout()" tag="button" class="button is-ghost is-blue big-button medium-text transition-faster">
				<span class="text">{{ $t('common.CANCEL') }}</span>
			</button>
		</form>

		<p class="mt-5 transition-faster">
			{{ $t('2fa.HAVING_PROBLEMS') }}
			<a
				href="https://support.morpher.com/en/article/2fa-2-step-verification-troubleshooting-ejmssf/"
				target="__blank"
				class="login-router"
				>{{ $t('2fa.2_STEP_SUPPORT') }}</a
			>
		</p>
	</div>
</template>

<script lang="ts">
import Component, { mixins } from 'vue-class-component';
import { Global } from '../mixins/mixins';
import { Watch } from 'vue-property-decorator';
import { getDictionaryValue } from '../utils/dictionary';
import { Recaptcha } from '../mixins/recaptcha';

@Component
export default class TwoFA extends mixins(Global, Recaptcha) {
	// Component properties
	emailCode = '';
	authenticatorCode = '';
	showRecovery = false;
	logonError = '';

	async mounted() {
		window.setTimeout(() => {
			const email: any = this.$refs.email_code;
			const auth: any = this.$refs.auth_code;
			if (email) email.focus();
			else if (auth) auth.focus();
		}, 100);

		if (this.isIframe()) {
			if (this.store.connection && this.store.connection !== null) {
				const connection:any = await this.store.connection.promise;

				connection.on2FA();
			}
		}

		this.executeHiddenLogin()
		//
	}

	@Watch('store.hiddenLogin')
	onPropertyChanged(value: any) {
		this.executeHiddenLogin()
	}

	@Watch('authenticatorCode')
	authenticatorCodeChanged() {
		if (this.authenticatorCode.length === 6) {
			this.validateCode();
		}
	}

	executeHiddenLogin() {
		try {
			
			 if (this.store.hiddenLogin) {

				this.emailCode = this.store.hiddenLogin.twoFACode
				this.validateCode();

			 }
			} catch (err) {
			console.log('error processing hidden login', err)
			
		}
		
		

	}			 

	/**
	 * Process email 2fa authentication
	 */
	async validateCode() {
		if (!this.recaptchaToken && (!localStorage.getItem('recaptcha_date') || Number(localStorage.getItem('recaptcha_date')) < Date.now() - (1000 * 60 * 8))) return this.executeRecaptcha(this.validateCode);

		// block if 2fa validation is already executing
		if (this.store.loading) {
			return;
		}
		this.logonError = '';
		this.showSpinner(this.$t('loader.VALIDATING_CODE').toString());
		this.unlock2FA({ email2FA: this.emailCode, authenticator2FA: this.authenticatorCode, recaptchaToken: this.recaptchaToken })
			.then((nextroute) => {
				this.hideSpinner();
				this.router.push(nextroute).catch(() => undefined);
			})
			.catch((error) => {
				this.hideSpinner();
				if (error.error === 'RECAPTCHA_REQUIRED') {
					this.executeRecaptcha(this.validateCode);
					return;
				}

				if (error && error.toString() === 'TypeError: Failed to fetch') {
					this.showNetworkError(true);
				}

				if (error.toString() === 'invalid password') {
					this.store.status = 'invalid password';
					this.router.push('/login').catch(() => undefined);
				}
				this.logonError = getDictionaryValue(error.toString());
			});
	}

	logout() {
		this.logoutWallet();
		//this.router.push('/login').catch(() => undefined);;
	}

	handleKeyPress(e: any) {
		const key = e.which || e.charCode || e.keyCode || 0;

		if (key === 13) {
			this.validateCode();
		}
	}
}
</script>
