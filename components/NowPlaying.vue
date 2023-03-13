<template>
	<Flex gap="10px" class="item vcenter">


		<Flex v-touch:tap="openPlayer" gap="0px" style="padding:10px 11px; height:100%;" v-if="!played || !played.hash" class="vcenter hcenter">

			<Flex class="background">


				<div class="thumbnail prev1 prevs" :style="{'--image':'url('+(previews[0] || '')+')'}" />
				<div class="thumbnail prevs prev2" :style="{'--image':'url('+(previews[1] || '')+')'}" />
				<div class="thumbnail prevs prev3" :style="{'--image':'url('+(previews[2] || '')+')'}" />


			</Flex>

			<Flex gap="3px" :vertical="true" class="click vcenter hcenter">
				<Boton @clicked="openPlayer" class="tiny bold texto" color="theme-action" style="margin-left: auto; border-radius:5px; padding:7px;" label="Try 3 Player!" />
				<div class="small tiny texto" style="margin-left: auto;" v-html="'Play music from your wallet'" />
			</Flex>



		</Flex>

		<Flex gap="0px" style="padding:10px 11px" v-else class="vcenter hcenter">



			<Flex gap="10px" class="vcenter hcenter click" v-touch:tap="open" style="width:60%; min-width:60%;">
				<div class="thumbnail" :style="{'--image':'url('+image+')'}" />
				<Flex gap="0px" style="align-self:start;" :vertical="true">
					<div class="small bold" v-html="played.name" />
					<div class="small tiny artist" v-if="played.creator && played.creator.hash" v-html="played.creator.username||played.creator.hash" />
				</Flex>

			</Flex>

			<Flex gap="0px" style="min-width:40%; width:40%; 	justify-content: end;" class=" vcenter">
				<div v-touch:tap="prev" class="prevSong pButton" :class="{hasPrev:loaded}" />

				<div v-touch:tap="pause" class="pauseSong pButton" v-if="playing" />
				<div v-touch:tap="play" class="playSong pButton" v-else />

				<div v-touch:tap="next" class="nextSong pButton" :class="{hasNext:loaded}" />


			</Flex>

		</Flex>
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
		storage
	} from "@/services/storage";
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
		userAvatar
	} from "@/base/constants";
	import {
		nft
	} from "@/services/nft-manager";
	import {
		sysendManager as sysend
	} from "@/services/sysend";

	export default {
		mixins: [mix],
		data() {
			return {
				address: false,
				image: false,
				played: false,
				loaded: false,
				playing: false,
				previews: []
			}
		},
		props: {

		},
		beforeUnmount() {
			if (this.image) URL.revokeObjectURL(this.image);
		},
		mounted() {
			this.start();
			(async () => {
				const url = request.buildUrl({
					path: "/auction/listing/sales"
				});

				const uploaded = await request.request(this, {
					url,
					method: "POST",
					stack: true,
					cache: {
						id: "random_song",
						seamless: true,
						time: 3600 * 3
					},
					args: {
						random: 5,
						object_type: "audio"
					}
				});

				if (uploaded && uploaded.data && uploaded.data.list) {
					this.previews = [];

					uploaded.data.list.forEach(x => {

						nft.getNFT(x.mint).then(async (y) => {
							const image = y && y.metadata && y.metadata.image;
							const f = await file.cacheFile(image, {
								fast: true,
								compress: "gif",
								compressed: "gif"
							});
							if (f) {
								this.previews.push(URL.createObjectURL(f));
							}
						})

					})
				}
			})();


			sysend.attach(this, "musicplayer:update", (x) => {
				this.loaded = true;
				this.reload(x);

				app.registerPopup({
					id: "musicplayer"
				}, this.onPopClose);
			});

			this.sendSysend("musicplayer:status");

		},
		methods: {
			onPopClose() {
				this.loaded = false;
				if (this.played) this.played.playing = false;
				this.start();
			},
			async sendSysend(que, data) {
				const ack = await sysend.broadcast(que, data || null, {
					timeout: 1000
				});
				if (!ack) {
					this.loaded = false;
					if (this.played) this.played.playing = false;
					this.start();
				}
			},
			openPlayer(play) {

				if (this.played && play) {
					app.openPopup({
						path: "/music#play:" + this.played.hash,
						id: "musicplayer",
						size: {
							width: 400,
							height: 700
						}
					}, this.onPopClose);
				} else {
					app.openPopup({
						path: "/music",
						id: "musicplayer",
						size: {
							width: 400,
							height: 700
						}
					}, this.onPopClose);
				}
			},
			open() {
				const pop = app.getPopup({
					id: "musicplayer"
				});
				if (pop) {
					pop.focus();
				} else {
					this.loaded = false;
					if (this.played) this.playing = false;
					this.openPlayer();
				}
			},
			next() {
				this.sendSysend("musicplayer:next");
			},
			prev() {
				this.sendSysend("musicplayer:prev");
			},
			pause() {
				this.sendSysend("musicplayer:pause");
			},
			play() {


				if (!this.loaded) {
					this.openPlayer(true);
				} else {
					if (this.played) {
						this.sendSysend("musicplayer:play:song", {
							hash: this.played.hash
						});
					} else {
						this.sendSysend("musicplayer:play");
					}
				}



			},
			async reload(resp) {

				if (resp) {
					this.played = {
						...(this.played || {}),
						...resp
					};
					this.playing = this.played && this.played.playing;

					const url = this.played.thumbnails && this.played.thumbnails[0]

					if (url) {
						const f = await file.cacheFile(url, {
							fast: true,
							compress: "gif",
							compressed: "gif"
						})
						if (f) {
							if (this.image) URL.revokeObjectURL(this.image);

							this.image = URL.createObjectURL(f);
						}
					}
					return
				}
				this.played = false;

			},
			async start() {
				const ses = await account.getActiveSession();
				if (ses && ses.profile && ses.profile.address) {
					this.address = window.mimic || ses.profile.address;
					let respa = false;
					if (this.address) {
						const resp = await storage.getLocal(["lastPlayed"], this.address);
						if (resp && resp.lastPlayed) {
							respa = resp.lastPlayed;
						}
					}
					this.reload(respa);
				}
			}
		}
	}
</script>

<style lang="scss" scoped>
	.thumbnail.prevs {
		background-color: #DDD;
		border: solid 1px rgba(0, 0, 0, 0.75);
	}

	.thumbnail.prev1,
	.thumbnail.prev2,
	.thumbnail.prev3 {
		position: absolute;
		border: solid 1px rgba(0, 0, 0, 0.075);

	}

	.thumbnail.prev3 {
		left: 48px;
		z-index: 0;
		transform: scale(0.65, 0.65);
	}

	.thumbnail.prev1 {
		left: 0px;
		top: -1px !important;
		z-index: 2;
	}

	.thumbnail.prev2 {
		left: 26px;
		z-index: 1;
		transform: scale(0.85, 0.85);
	}

	.loading {

		height: 15px;
		background: rgba(0, 0, 0, 0.1);
		border-radius: calc(var(--radius) / 2);

	}

	.name.loading {
		width: 30%;
		margin-top: 4px;
	}

	.artist.loading {
		width: 15%;
		margin-top: 7px;
	}

	.clica:hover .texto {
		transform: translateY(-1px);
	}

	.clica {
		cursor:#{$clickCursor};
	}

	.background {
		position: absolute;
		left: 0px;
		top: 0px;
		bottom: 0px;
		right: 0px;
		width: calc(100% - 18px);
		height: calc(100% - 15px);
		margin: auto;
	}

	.artist:not(.loading) {
		max-width: 75px;
		text-overflow: ellipsis;
		overflow: hidden;
		white-space: nowrap;
	}

	.click {
		cursor:#{$clickCursor};
	}

	@import './shared.scss';

	.thumbnail {
		width: 45px !important;
		min-width: 45px !important;
		height: 45px !important;
	}

	.pButton {
		width: 30px !important;
		min-width: 30px !important;
		height: 30px !important;
	}

	.background {
		pointer-events: none;

		top: 6px;
	}

	.item {
		margin: -6px;


		border-radius: calc(var(--radius) - 0px);
		height: calc(100% + 12px) !important;
		width: calc(100% + 12px) !important;
	}
</style>