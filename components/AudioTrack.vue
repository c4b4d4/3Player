<template>
	<Flex v-touch:tap="pick" gap="10px" class="item vcenter">
		<div class="thumbnail" :style="{'--image':'url('+(image||'')+')'}" />
		<Flex gap="0px" class="info" style="width:max-content; 	align-self: baseline;" :vertical="true">
			<div class="small bold" v-html="name" />
			<div class="small tiny" v-if="creator && creator.hash" v-html="creator.username||creator.hash" />
			<div class="tiny desc" v-if="description" v-html="description" />
		</Flex>
		<div class="flex_expand" />


		<Flex :vertical="true" gap="0px" v-if="buy" class="vcenter hcenter" style="width:max-width; margin-right:5px;" @clicked="">
			<Boton color="theme-action" @clicked="comprar" class="tiny botonero" label="collect" />
			<div class="comprar" v-if="buyLabel" v-html="buyLabel" />
		</Flex>
		<ReactionRow v-else class="like" :loading="!reactions" hideComments="true" :simple="true" :msg="false" v-model:value="reactions" :hash="hash" />
	</Flex>
</template>

<script>
	import {
		mix
	} from "@/components/mixin";
	import {
		app
	} from "@/services/app";
	import {
		modal
	} from "@/services/modals";
	import {
		request
	} from "@/services/request";
	import {
		account
	} from "@/modules/account-manager";
	import {
		waitFor,
		formatDate
	} from '@/base/shared';
	import {
		file
	} from "@/services/file";
	import {
		userAvatar,
		solify,
		currencySymbol
	} from "@/base/constants";
	import {
		nft
	} from "@/services/nft-manager";

	export default {
		mixins: [mix],
		data() {
			return {

				image: false,
			}
		},
		props: {
			reactions: {
				default: false
			},
			buy: {
				default: false
			},
			logged: {
				default: false
			},
			description: {
				default: false
			},
			creator: {
				default: false
			},
			listing: {
				default: false
			},
			name: {
				default: false
			},
			creators: {
				default: false
			},
			hash: {
				default: false
			},
			thumbnails: {
				default: false
			}
		},
		beforeUnmount() {
			if (this.image && this.image.includes("blob:")) {
				URL.revokeObjectURL(this.image);
			}
		},
		computed: {
			buyLabel() {
				if (this.listing && this.listing.listing_data) {
					return solify(this.listing.listing_data.price, this.listing.listing_data.treasury) + "" + currencySymbol(this.listing.listing_data.treasury, true);
				}
				return false;
			}
		},
		mounted() {
			this.loadImage();
		},
		methods: {
			async comprar() {

				if (!this.logged) {
					modal.new({
						id: "oops",
						props: {
							title: "Oops",
							message: "Connect a wallet in order to collect this NFT"
						}
					})
					return;
				}

				const good = await nft.buy({
					...this.listing
				}, {});

				if (good) {

					app.askSomething({
						title: translate("success"),
						message: translate("nft_bought"),
						vertical: true,
						buttons: [{
							action: "accept",
							color: "normal",
							label: translate("place_in_homebase")
						}, {
							label: translate("continue")
						}]
					}).then(async x => {
						await location.reload();
					})


				}
			},
			pick() {
				this.emit("clicked", this.hash);
			},
			async loadImage() {


				const tn = this.thumbnails && this.thumbnails[0];
				if (tn) {
					const f = await file.cacheFile(tn, {
						fast: true,
						compress: "gif",
						compressed: "gif"
					});
					if (f) {
						this.image = URL.createObjectURL(f);
					}

				}

				if (!this.image) this.image = "/assets/empty.webp";


			}
		}
	}
</script>

<style lang="scss" scoped>
	.desc {
		opacity: 0.5;
		overflow: hidden;
		text-overflow: ellipsis;
		display: -webkit-box;
		-webkit-line-clamp: 1;
		-webkit-box-orient: vertical;
		height: max-content;
	}

	.like {
		width: max-content;
		min-width: 30px;
	}

	.like :deep(img) {
		position: unset !important;
		margin-right: 6px !important;
		margin-top: -2px !important;
	}

	.like :deep(.reactionbtn) {
		min-height: unset !important;
		background: transparent !important;
		border: none !important;
		filter: brightness(0.9) invert(.7) sepia(1) hue-rotate(740deg) saturate(200%);
	}

	.item:hover .thumbnail {
		//transform: scale(1.05,1.05);
	}

	.item:hover {
		//	padding-top:9px;
		//	padding-bottom:11px;
		background-color: rgba(0, 0, 0, 0.1);

	}

	.botonero {
		border-radius: calc(var(--radius) / 2) !important;
		padding: 8px !important;
		align-self: end;
		margin-bottom: 3px;
		margin-top: 2px;
	}

	.comprar :deep(img) {
		margin-left: 4px;
		margin-bottom: -3px;
	}

	.comprar {
		font-size: 11px;
		align-self: end;
	}

	.info {
		max-width: calc(100% - 150px);
	}

	.item {
		cursor:#{$clickCursor};
		padding: 10px;
	}

	.thumbnail {
		width: 45px;
		height: 45px;
		min-width: 45px;
		border-radius: var(--radius);
		background-color: rgba(0, 0, 0, 0.1);
		background-image: var(--image);
		background-repeat: no-repeat;
		background-size: cover;
		background-position: center;

	}
</style>