<script>
	import { createEventDispatcher, onMount } from 'svelte';
	import { fly } from 'svelte/transition';
	import AppData from '../stores/AppData.js';

	const dispatch = createEventDispatcher();

	let showCookieConsent;

	onMount(async () => {
		showCookieConsent = !$AppData.dismissCookieConsent;
	});

	function closeCookieConsent() {
		showCookieConsent = false;
		$AppData.dismissCookieConsent = true;
		dispatch('routeEvent', {action: 'saveData'});
	}
</script>

{#if showCookieConsent}
	<div class="cookieConsentContainer" transition:fly="{{ y: 100, duration: 600 }}">
		<div class="cookieConsentText">AFKBenchmarks only stores information that is strictly necessary for the functionality of the application and the saving of preferences on your device. No tracking, marketing, or statistics data is stored or collected.</div>
		<button type="button" class="closeButton" on:click={closeCookieConsent}>Accept All</button>
	</div>
{/if}

<style lang="scss">
	.cookieConsentContainer {
		background-color: var(--appBGColor);
		bottom: 20px;
		border-radius: 10px;
		box-shadow: var(--neu-large-ni-BGColor-shadow);
		color: var(--appColorBlack);
		height: fit-content;
		left: 5%;
		padding: 10px;
		position: absolute;
		text-align: center;
		width: 90%;
		z-index: 5;
		.cookieConsentText {
			padding: 10px;
		}
		.closeButton {
			background-color: var(--appBGColor);
			border: none;
			border-radius: 10px;
			box-shadow: var(--neu-med-i-BGColor-shadow);
			color: var(--appColorPrimary);
			cursor: pointer;
			font-size: 1rem;
			font-weight: bold;
			margin-top: 20px;
			outline: none;
			padding: 10px;
			text-align: center;
			transition: all 0.2s;
			width: 140px;
		}
	}
	@media only screen and (min-width: 767px) {
		.cookieConsentContainer {
			align-items: center;
			display: flex;
			text-align: left;
			.closeButton {
				flex-shrink: 0;
				margin-top: 0;
				margin-left: auto;
				&:hover {
					background: var(--neu-convex-BGColor-wide-bg);
				}
			}
		}
	}
</style>
