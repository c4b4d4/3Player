<template>
	<Flex class="playing">

		<div class="video_holder" v-if="video">
			<video ref="video" muted="true" playsinline="playsinline" autoplay loop="loop" :type="video.type" webkit-playsinline="webkit-playsinline" class="video" :src="video.url" />
		</div>

		<div v-else class="square_holder">
			<div v-for="(valor,key) in visible ? 0 : 3" class="square_song">
				<div v-if="key==1" class="signo">?</div>
			</div>
			<div v-touch:tap="select(key)" v-for="(valor,key) in visible" :style="{'--image':'url('+valor.image+')','--multiabs':Math.abs(valor.direction - selected), '--multi':(valor.direction - selected), '--dir':(valor.direction - selected) < 0 ? -1 : 1}" class="square_song real" :class="{selected:valor.direction==selected}">
			</div>
		</div>

		<slot name="default"></slot>

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
		file,
		interest
	} from "@/services/file";
	import {
		userAvatar
	} from "@/base/constants";
	import {
		nft
	} from "@/services/nft-manager";
	const videos = {};

	const imagenes = {};
	export default {
		mixins: [mix],
		data() {
			return {
				visible: false,
				selected: 0,
				video: false
			}
		},
		props: {

			list: {
				default: false
			},
			hash: {
				default: false
			}
		},
		watch: {
			list() {
				this.calcList();
			},
			hash() {
				this.calcList();
			}
		},
		mounted() {
			this.calcList();
		},
		methods: {
			select(key) {
				return (e) => {
					e.stopPropagation();
					e.preventDefault();
					this.emit("clicked", key);
				}
			},
			calcList() {
				const visa = {};

				const dif = Math.min(this.selected - 2, 0)

				let sela = 0;
				for (let i = 0; i < this.list.length; i++) {

					if (this.list[i].hash == this.hash) {
						sela = i;
						break;
					}
				}

				this.selected = sela;
				this.list.slice(Math.max(this.selected - 2, 0), Math.min(this.list.length, 5 + dif)).map((x, i) => {
					visa[x.hash] = x;
					visa[x.hash].direction = Math.max(this.selected - 2, 0) + i
				});

				const s = this.list[this.selected];


				if (s) {
					if (videos[s.hash]) {
						this.video = videos[s.hash];

					} else {
						const archivos = s.files;
						const v = interest(archivos, "poster_video");

						this.video = false;
						if (Array.isArray(v) && v[0]) {
							(async () => {
								const f = await file.cacheFile(v[0], {
									fast: true
								});
								videos[s.hash] = {
									url: URL.createObjectURL(f),
									type: f.type
								};
								this.video = videos[s.hash];
							})();
						}
					}

				} else {
					this.video = false;
				}

				if (Object.keys(visa).length > 0) {
					this.visible = visa;
				} else {
					this.visible = false;
				}


				const put = async (keya) => {

					if (keya == s.hash) {
						const artaddress = s.creators[0].address;
						const art = await nft.addressManager.getUsername(artaddress);

						const artist = art ? art.username : artaddress;

						const ima = imagenes[keya];

						app.setTitle({
							label: "🎶 " + s.name + " - " + artist,
							icon: false
						});
						file.iconify(ima.url).then(icon => {
							app.setTitle({
								label: "🎶 " + s.name + " - " + artist,
								icon
							});
						})

						if (ima.sizes) {

							navigator.mediaSession.metadata = new MediaMetadata({
								title: s.name,
								artist: artist,
								album: "3.land",
								//album: 'Playing at 3.land',
								artwork: [{
									src: ima.url,
									sizes: ima.sizes,
									type: ima.type
								}]
							});


						} else {
							const img2 = new Image();
							img2.onload = () => {

								ima.sizes = img2.naturalWidth + 'x' + img2.naturalHeight;

								navigator.mediaSession.metadata = new MediaMetadata({
									title: s.name,
									artist: artist,
									album: "3.land",
									//album: 'Playing at 3.land',
									artwork: [{
										src: ima.url,
										sizes: img2.naturalWidth + 'x' + img2.naturalHeight,
										type: ima.type
									}]
								});

							}
							img2.src = ima.url;

						}

					}
				}

				for (const key in this.visible) {
					if (imagenes[key]) {
						put(key);
						this.visible[key].image = imagenes[key].url;
					} else {
						(async () => {
							const tn = this.visible[key].thumbnails && this.visible[key].thumbnails[0];
							if (tn) {
								const f = await file.cacheFile(tn, {
									fast: true,
									compress: "gif",
									compressed: "gif"
								});
								if (f) {
									imagenes[key] = {
										url: URL.createObjectURL(f),
										type: f.type
									};
								} else {
									imagenes[key] = {
										url: "/assets/empty.webp",
										type: "image/webp"
									};
								}
							} else {
								imagenes[key] = {
									url: "/assets/empty.webp",
									type: "image/webp"
								};
							}
							put(key);
							this.visible[key].image = imagenes[key].url;
						})();

					}
				}
			}
		}
	}
</script>

<style lang="scss" scoped>
	.square_song:not(.real):not(:first-child):not(:last-child) {
		z-index: 2;
	}

	.square_song:first-child:not(.real) {
		z-index: 1;
		transform: translateX(-85%) scale(0.75, 0.75);
	}

	.square_song.real {
		cursor:#{$clickCursor};
		background-image: var(--image);
		background-position: center;
		background-repeat: no-repeat;
		background-size: cover;
	}

	.square_song.real:hover {
		width: 35%;
		padding-top: 35%;
	}

	.square_song:not(.selected) {
		--ancho: calc(0.75 / var(--multiabs));
		transform: translateX(calc(var(--multi) * 75% + (20% * var(--dir)) + var(--multi) / 2 * -56%)) scale(var(--ancho), var(--ancho));
		z-index: calc(10 - var(--multiabs));
	}

	.square_song.selected {
		transform: unset !important;
		z-index: 10;
	}

	.square_song:last-child:not(.real) {
		z-index: 1;
		transform: translateX(85%) scale(0.75, 0.75);
	}

	.signo {
		position: absolute;
		left: 0px;
		top: 0px;
		right: 0px;
		bottom: 0px;
		margin: auto;
		width: max-content;
		font-size: 32px;
		height: max-content;
		opacity: 0.75;
	}

	video.video {
		object-fit: cover !important;
		width: 100%;
		height: 100%;
		object-position: center;
	}

	.video_holder {
		width: 100%;
	}

	.square_song {
		position: absolute;
		left: 0px;
		right: 0px;
		margin: auto;
		bottom: 0px;
		top: 0px;
		width: 34%;
		padding-top: 34%;
		border: solid 1px rgba(0, 0, 0, 0.4);
		box-shadow: 0px 0px 10px rgba(0, 0, 0, 0.25);
		height: 0px;
		background: #161616;
		z-index: 2;
		border-radius: var(--radius);
	}
</style>