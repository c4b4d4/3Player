<template>
	<Flex gap="10px" v-if="loading" class="item vcenter hcenter">
		<CircleLoader class="loader" />
	</Flex>
	<Flex gap="0px" v-else class="item vcenter" style="--extra:0;">

		<RangeInput v-if="hash" ref="seeker" :hideInput="true" :value="Math.max(0,seeker)" :disabled="seeker==-1" @update:value="cambiaSeek" class="tiny full seeker" :min="0" step="0.01" :max="1" />

		<Flex v-if="showing_volume" class="volume_range">
			<RangeInput :hideInput="true" :vertical="true" :value="volume" @update:value="cambiaVolume" class="tiny full seeker" :min="0" step="1" :max="100" />
		</Flex>

		<Flex gap="10px" class="info sides" style="width:max-content">
			<div class="thumbnail" :style="{'--image':'url('+(image||'')+')'}" />
			<div class="artist_image" v-if="artist && artist.image" :style="{'--image':'url('+(artist.image||'')+')'}" />
			<Flex gap="0px" :vertical="true" class="hcenter" style="width:max-content">
				<div class="tiny name" v-html="name" />

				<div v-if="artist" class="tiny artist_name" style="opacity:0.5" v-html="artist.address || artist.username" />

			</Flex>
		</Flex>

		<div class="flex_expand" />
		<Flex style="width:40% !important;" class="vcenter hcenter">
			<div v-touch:tap="prev" class="prevSong pButton" :class="{hasPrev}" />

			<div v-touch:tap="pause" class="pauseSong pButton" v-if="playing" />
			<div v-touch:tap="play" class="playSong pButton" v-else />

			<div v-touch:tap="next" class="nextSong pButton" :class="{hasNext}" />
		</Flex>
		<div class="flex_expand" />

		<Flex class="sides">
			<div v-touch:tap="toggleVolume" :class="[getVolClass]" class="volume vButton" />
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
		sysendManager as sysend
	} from "@/services/sysend";
	import {
		userAvatar
	} from "@/base/constants";
	import {
		nft
	} from "@/services/nft-manager";

	export default {
		mixins: [mix],
		data() {
			return {
				seeker: 0,
				showing_volume: false,
				volume: 75,
				loading: false,
				playing: false,
				image: false,
				artist: false,
				name: "--"
			}
		},
		props: {
			hash: {
				default: false
			},
			hasNext: {
				default: false
			},
			hasPrev: {
				default: false
			}
		},
		computed: {
			getVolClass() {
				if (this.volume == 0) {
					return "volume_0";
				} else if (this.volume < 33) {
					return "volume_1";
				} else if (this.volume < 66) {
					return "volume_2";
				} else {
					return "volume_3"
				}
			}
		},
		beforeUnmount() {
			this.killVolumeTimer();
			if (this.image && this.image.includes("blob:")) {
				URL.revokeObjectURL(this.image);
			}
		},
		mounted() {


			sysend.attach(this, "musicplayer:play", () => {
				this.play();
			})

			sysend.attach(this, "musicplayer:pause", () => {
				this.pause();
			})

			sysend.attach(this, "musicplayer:next", () => {
				if (this.hasNext) this.next();
			})

			sysend.attach(this, "musicplayer:prev", () => {
				if (this.hasPrev) this.prev();
			})

			this.loadNFT();

			navigator.mediaSession.setActionHandler('play', () => {

				this.play();

			});
			navigator.mediaSession.setActionHandler('pause', () => {
				this.pause();
			});


			navigator.mediaSession.setActionHandler('previoustrack', () => {
				if (this.hasPrev) this.prev();
			});
			navigator.mediaSession.setActionHandler('nexttrack', () => {
				if (this.hasNext) this.next();
			});

			app.attach(this, "keyup", (e) => {

				if (e.code == "Space") {

					if (!this.playing) {
						this.play();
					} else {
						this.pause();
					}
				}
			})
		},
		watch: {
			hash() {
				this.seeker = 0;
				this.loadNFT();
			}
		},
		methods: {
			beat(x) {
				if (this.$refs.seeker) {
					this.$refs.seeker.$el.style.setProperty("--extra", (x * 2.5) + "");
				}
			},
			killVolumeTimer() {
				if (this.volume_timer) {
					clearTimeout(this.volume_timer);
					this.volume_timer = false;
				}
			},
			cambiaVolume(v) {
				this.volume = v;
				this.emit("volume", v);
				this.killVolumeTimer();
				this.volume_timer = setTimeout(() => {
					this.showing_volume = false;
				}, 1500)
			},
			cambiaSeek(v) {

				this.emit("seek", v);
			},
			toggleVolume() {
				if (app.isSafari()) {
					if (this.volume) {
						this.volume = 0;
						this.emit("mute", true);
					} else {
						this.volume = 100;
						this.emit("mute", false);
					}

					return;
				}
				this.showing_volume = !this.showing_volume;
				if (this.showing_volume) {
					this.killVolumeTimer();
					this.volume_timer = setTimeout(() => {
						this.showing_volume = false;
					}, 5000)
				}
			},
			pause() {
				this.playing = false;
				this.emit("pause")
			},
			progress(per) {
				this.seeker = per;
			},
			next() {
				this.loading = true;
				this.emit("next");
			},
			prev() {
				this.loading = true;
				this.emit("prev");
			},
			play(nuevo) {

				if (!this.hash) {
					this.loading = true;

					this.emit("next");
					return;
				}

				if (nuevo === true) this.seeker = 0;
				this.playing = true;

				if (nuevo === true) {
					this.emit("playing", this.hash);
				} else {
					this.emit("resume");
				}

				this.loading = false;
				this.emit("volume", this.volume);
			},
			async loadNFT() {
				if (!this.hash) {
					this.loaded = false;
					return;
				}
				this.loading = true;
				const tokendata = await nft.getNFT(this.hash);

				const img = tokendata.metadata.image;
				this.name = tokendata.chain.name || tokendata.metadata.name;

				const artist = tokendata.chain.data.creators[0].address;

				this.artist = false;
				nft.addressManager.getUsername(artist).then(data => {
					if (data) data.image = userAvatar(data.username);
					this.artist = data || {
						address: artist
					};

				});


				this.image = false;
				let type = "image/webp";
				if (img) {
					const f = await file.cacheFile(img, {
						fast: true,
						compress: "gif",
						compressed: "gif"
					});
					if (f) {
						type = f.type;
						this.image = URL.createObjectURL(f);
					}

				}

				if (!this.image) this.image = "/assets/empty.webp";




				this.loading = false;

			}
		}
	}
</script>

<style lang="scss" scoped>
	@import './shared.scss';

	.seeker :deep(input[type=range]::-webkit-slider-thumb) {
		transform: scale(calc(var(--extra) + 1), calc(var(--extra) + 1)) !important;

	}




	.sides:last-child {
		justify-content: end;
	}

	.info,
	.sides {
		width: 30%;
		min-width: 30%;
		max-width: 30%;
	}

	.info * {
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}


	.volume_range {
		position: absolute;
		right: 20px;
		width: 30px !important;
		background: #111;
		padding: 10px;
		border: solid 1px #000;
		border-radius: var(--radius);
		bottom: 55px;
	}



	.volume_range>.seeker {
		position: unset !important;
		left: unset;
		right: unset;
		top: unset;
		bottom: unset;
		filter: brightness(0.9) invert(0.7) sepia(1) hue-rotate(955deg) saturate(200%);
	}

	.seeker {
		position: absolute !important;
		left: 0px;
		right: 0px;
		width: 100%;
		top: -5px;
		background: transparent !important;
		filter: brightness(0.9) invert(.7) sepia(1) hue-rotate(740deg) saturate(200%);
	}

	.artist_image {
		width: 15px;
		height: 15px;
		background-image: var(--image);
		background-position: center;
		background-size: cover;
		position: absolute;
		border-radius: 2px;
		left: 23px;
		bottom: -1px;
	}

	.item {

		min-height: 66px;

		bottom: 0px;
		left: 0px;
		position: absolute;

		padding: 10px;
		border-top: solid 1px rgba(0, 0, 0, 0.35);
		background-color: #333;
	}

	.loader {
		width: 30px !important;
		height: 30px !important;
		--filtro: invert(var(--player-primary-invert));
		filter: var(--filtro);
	}
</style>