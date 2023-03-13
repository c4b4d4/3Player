<template>
	<Flex class="player" gap="10px" :vertical="true">

		<Flex class="case" gap="0px" :vertical="true">

			<Flex class="header vcenter" gap="10px">
				<div class="logo" v-touch:tap="open3" style="background-image: url(/web/logo.svg);" />
				<div class="flex_expand" />
				<Slider :inline="true" class="tiny compact small full slider customScroll thinScroll" :options="categories" v-model:value="category" />
				<div class="flex_expand" />
				<Flex gap="10px" style="width:max-content !important;" v-touch:tap="openU" class="userclick vcenter hcenter">
					<div v-if="user.name" v-html="user.name" class="info username" />
					<div v-else class="loading username" />

					<div v-if="user.image" :style="{'--image':'url('+user.image+')'}" class="info userimage" />
					<div v-else class="loading userimage" />
				</Flex>
			</Flex>

			<Flex :vertical="true" gap="0px" class="section songs" v-if="(gotNFTs && category=='owned') || !logged">
				<PlayerCarrousel @clicked="carruselSelect" :list="allData" :hash="selected" class="playing" />
				<Flex :vertical="true" gap="0px" class="item_list">
					
					<Flex class="search_header" v-if="!anon && !previews">
						<Input v-model:value="extra['filter']" class="small" placeholder="search..." />
						<Exit v-if="filtering" @clicked="extra['filter'] = '';" class="cerrar" style="filter:invert(1)" />
					</Flex>

					<CenterBox v-if="filtering && nftData.length==0 && !anon">
						<div class="small" v-html="'nothing_found'" />
					</CenterBox>

					<CenterBox v-if="nftData.length==0 && !items.owned.processing && anon" style="	border-bottom: solid 1px rgba(0,0,0,0.2);">
						<Flex v-if="logged" :vertical="true" class="whmi" gap="10px" style="margin:auto; margin:40px 0px;">
							<div class="tiny" style="max-width:170px;"><span style="opacity:0.5; ">We couldn't find any Audio NFTs in your library</span> <span class="tiny bold">:-(</span></div>
						</Flex>

						<CenterBox style="margin-top:20px" v-if="!logged">
							<Flex gap="2px" :vertical="true" class="whmi vcenter hcenter">
								<div class="normal" v-html="d('account_required')" />
								<div style="margin-bottom:20px; opacity: 0.65;" class="tiny" v-html="d('account_required_msg')" />

								<Boton @clicked="conectar" class="small" color="theme-action" label="connect_wallet" />
							</Flex>
						</CenterBox>

						<div class="linea tiny full center" style="margin-bottom:45px; margin-top:35px;">
							<div class="line" />
							<div class="texto" v-html="logged ? 'but you can try these' : 'meanwhile try these songs'" />
						</div>
					</CenterBox>

					<!-- Infinite scroll list -->
					<InfiniteList v-if="showData.length>0" :square="true" :dimensions="[0,1]" @page="pager" @fit="newFit" :gap="[0,0]" :padding="[0,0]" style="width:100%" :max_size="[size_ref[1],size_ref[1]]" :direction="'y'" container="self" :loading="{loading:items.owned.processing, invert:true}" ref="list" :data="showData" />
						
				</Flex>

			</Flex>

			<CenterBox v-else-if="!gotNFTs">
				<CircleLoader class="loader" />
			</CenterBox>


			<PlayerTrack @mute="setMute" @volume="volumeTo" @seek="seekTo" @pause="pause" @resume="resume" v-if="gotNFTs || !logged" @playing="playing" :hasNext="hasNext" :hasPrev="hasPrev" ref="player" @next="nextSong" @prev="prevSong" :hash="selected" class="track" />

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
		file,
		interest
	} from "@/services/file";
	import {
		userAvatar
	} from "@/base/constants";
	import {
		nextTick
	} from "vue";
	import {
		nft
	} from "@/services/nft-manager";
	import {
		solanaGate
	} from "@/services/solana";

	import {
		musicPlayer
	} from "@/services/audio";

	import {
		generalBase
	} from "@/base";
	import {
		sysendManager as sysend
	} from "@/services/sysend";

	const size_ref = [60, 90];
	const audios = {};

	const categories = [{
			value: "owned",
			label: "songs"
		}
	]

	export default {
		mixins: [mix],

		data() {
			return {
				extras: [],
				logged: true,
				autoplay: false,
				playingSong: false,
				isPlaying: false,
				audio: {
					src: ""
				},
				selectedIndex: -1,
				selected: false,
				scrollPage: false,
				failed: [],
				items: {
					owned: {
						loaded: false,
						data: [],
						start: 8
					},
					artists: {
						loaded: false,
						data: [],
						start: 8
					}
				},
				size_ref,
				user: {},
				extra: {
					filter: "",
					audio: true,
					video: false
				},
				notLoaded: true,
				count: false,
				categories,
				category: "owned"
			}
		},
		props: {

		},
		watch: {
			finishedLoading(c) {
				if (this.notLoaded) {
					this.getListings();
					this.notLoaded = false;
				}
			},
			hasNext() {
				if (this.anon) return
				const copia = this.makePayload();
				copia.playing = this.isPlaying;
				sysend.broadcast("musicplayer:update", copia);
			}
		},
		computed: {
			page() {
				return this.scrollPage || -1;
			},
			gotNFTs() {
				return this.items.owned.loaded;
			},
			hasNext() {
				return this.allData.length > 0;
			},

			hasPrev() {
				return this.allData.length > 0;
			},
			finishedLoading() {
				return (this.gotNFTs && !this.items.owned.processing) || this.logged === false
			},
			filtering() {
				if ((this.extra.filter && this.extra.filter.length > 0)) {
					return true;
				}
				return false;
			},
			categorized() {

				let categoria = "owned";
				if (!categoria || !this.items || !this.items[categoria]) {
					return false;
				}

				let items = [...(this.items[categoria].data || [])];

				items = items.filter(x => {


					let show = false;
					let searchF = true;

					if (this.filtering) {

						const str = this.extra.filter && this.extra.filter.length > 0 && this.extra.filter.toLowerCase();


						if (str && str.length > 0) {

							if ((x.name || "").toLowerCase().indexOf(str) > -1 || (x.description || "").toLowerCase().indexOf(str) > -1 || (x.creator && ((x.creator.address || "").toLowerCase().indexOf(str) > -1 || (x.creator.username || "").toLowerCase().indexOf(str) > -1 || (x.creator.display || "").toLowerCase().indexOf(str) > -1))) {

								show = true;
								searchF = true;
							} else {
								searchF = false;
							}
						}

					}

					const has = {};
					if (this.extra["video"] || this.extra["audio"]) {
						show = false;
					}
					for (let i = 0; i < (x.files || []).length; i++) {
						const f = x.files[i];
						if (f.type.indexOf("audio") > -1) {
							has["audio"] = true;
						}
						if (f.type.indexOf("video") > -1) {
							has["video"] = true;
						}
					}
					if ((this.extra["audio"] && has["audio"]) || (this.extra["video"] && has["video"])) {
						show = true;
					}

					return show && searchF;

				})

				return items
			},
			anon() {
				return this.nftData.length == 0;
			},
			previews() {
				return true;
			},
			allData() {
				if (!this.anon && !this.previews) {
					return this.categorized || [];
				}

				const re = (this.categorized || []).slice(0)
				re.push(...this.extras)
				return re;

			},
			showData() {

				if (!this.anon && !this.previews) {
					return this.nftData || [];
				}

				const mas = [];
				if (this.previews) {
					mas.push({
						component: "TextC",
						size: [false, 100],
						props: {
							text: `<div class="linea tiny full center" style="margin:auto; position:absolute; left:0px; right:0px; top:0px; bottom:0px;"> <div class="line"></div><div class="texto alone">try some more tunes</div></div>`
						}
					});
				}

				const env = [...this.nftData, ...mas, ...this.extras.map((x, i) => {

					return {
						component: "AudioTrack",
						listen: {
							clicked: (y) => {
								this.selectedItem(x.hash);
							}
						},
						id: x.hash + (x.id || ""),
						props: {
							...x,
							logged: this.logged,
							buy: true,
							timed: true
						},
						size: [false, 65]
					};
				})];

				return env;

			},
			nftData() {
				const dto = (this.categorized || []).map((x, i) => {

					return {
						component: "AudioTrack",
						listen: {
							clicked: (y) => {

								this.selectedItem(x.hash);

							}
						},
						id: x.hash + (x.id || ""),
						props: {
							...x,
							timed: true
						},
						size: [false, 65]
					};
				})

				return dto;
			}
		},
		mounted() {
			
			sysend.attach(this, "musicplayer:status", () => {
				if (this.anon) return
				const copia = this.makePayload();
				copia.playing = this.isPlaying;
				sysend.broadcast("musicplayer:update", copia);
			})

			sysend.attach(this, "musicplayer:play:song", ({
				hash
			}) => {
				if (this.anon) return
				this.selectedItem(hash);
			})

			const word = "play:";
			if (location.hash && location.hash.includes(word)) {
				const iof = location.hash.indexOf(word) + word.length;
				const hash = location.hash.substring(iof, location.hash.length).split(";")[0]
				this.autoplay = {
					hash
				};
			}

			musicPlayer.start();
			(async () => {
				await this.loadUser();
				await this.loadNFTs();
			})();

			musicPlayer.attach(this, "talk", (por) => {
				this.$refs.player.beat(por);
			});
			musicPlayer.attach(this, "next", (por) => {

				this.nextSong();

			});

			musicPlayer.attach(this, "progress", (por) => {
				this.$refs.player.progress(por);
			});



		},
		methods: {
			async getListings() {

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
							this.gotNFT(true)({
								tokendata: y,
								listing: x
							})
						})
					})
				}

			},
			async conectar() {
				solanaGate.attach(this, "wallet:confirmed", "wallet:confirmed", () => {
					window.location.reload();
				})
				solanaGate.connect();
			},
			open3() {
				window.open("https://3.land", "_blank");
			},
			openU() {
				if (this.user.name) window.open("https://3.land/u/" + this.user.name, "_blank");
			},
			setMute(x) {
				musicPlayer.mute(x);
			},
			seekTo(x) {
				musicPlayer.seek(x);
			},
			resume() {
				musicPlayer.resume();
				if (this.anon) return
				const copia = this.makePayload();
				copia.playing = true;
				this.isPlaying = true;
				sysend.broadcast("musicplayer:update", copia);
			},
			pause() {
				musicPlayer.pause();
				if (this.anon) return
				const copia = this.makePayload();
				copia.playing = false;
				this.isPlaying = false;
				sysend.broadcast("musicplayer:update", copia);
			},
			makePayload() {
				const copia = {
					...this.playingSong
				};
				const cuales = ["hash", "name", "creator", "thumbnails"];
				for (const key in copia) {
					if (!cuales.includes(key)) {
						delete copia[key];
					} else {
						copia[key] = JSON.parse(JSON.stringify(copia[key]));
					}
				}
				return copia;
			},
			playing(hash) {
				const idx = this.calcIndex(hash);
				this.playingSong = this.allData[idx];
				const url = interest(this.playingSong.files, "audio");

				if (this.user.address) {
					if (!this.anon) {
						storage.setLocal({
							lastPlayed: this.playingSong
						}, this.user.address + "")
						const copia = this.makePayload();
						copia.playing = true;
						this.isPlaying = true;
						sysend.broadcast("musicplayer:update", copia);
					}
				}

				musicPlayer.play(url, this);
			},
			nextSong() {
				let n = this.selectedIndex + 1;

				if (!this.anon && this.previews) {
					if (this.selectedIndex < this.nftData.length && n >= this.nftData.length) {
						n = 0;
					}
				}

				if (!this.allData[n]) {
					this.selectedItem(this.allData[0].hash)
				} else {
					this.selectedItem(this.allData[n].hash)
				}

			},
			prevSong() {
				let n = this.selectedIndex - 1;

				if (!this.anon && this.previews) {
					if (this.selectedIndex < this.nftData.length && n < 0) {
						n = this.nftData.length - 1;
					}
				}

				if (!this.allData[n]) {
					this.selectedItem(this.allData[this.allData.length - 1].hash)
				} else {
					this.selectedItem(this.allData[n].hash)
				}
			},
			carruselSelect(x) {
				this.selectedItem(x);
			},
			selectedItem(x) {
				this.selected = x;
				nextTick(() => {
					this.$refs.player.play(true);

					this.selectedIndex = this.calcIndex();
				})
			},
			volumeTo(v) {
				musicPlayer.volume(v);
			},
			calcIndex(hash) {
				let idx = -1;
				for (let i = 0; i < this.allData.length; i++) {
					if (this.allData[i].hash == (hash || this.selected)) {
						idx = i;
						break;
					}
				}

				return idx;
			},

			gotNFT(anon) {
				return ({
					tokendata,
					listing
				}) => {


					this.items.owned.loaded = true;

					const v = tokendata;

					const pointer = (anon ? this.extras : this.items.owned.data);

					const id = pointer.length;

					const name = (v.chain && v.chain.data) ? v.chain.data.name : (v.metadata.name || "--");

					pointer.push({
						master: tokendata.master,
						hash: v.hash,
						creators: v.chain.data.creators,
						thumbnails: [v.metadata.image],
						minter: v.metadata.properties.minter || null,
						files: v.metadata.properties.files,
						name,
						description: v.metadata.description,
						listing: listing || false
					});


					const audio = interest(v.metadata.properties.files, "audio");
					if (audio) {
						nft.reactionsManager.getMyReactions(v.hash, "nft").then(async (x) => {
							pointer[id].reactions = x;
						})
						if (v.chain.data.creators) {
							const artist = v.chain.data.creators[0].address;
							nft.addressManager.getUsername(artist).then(data => {
								if (data) data.image = userAvatar(data.username);
								pointer[id].creator = data || {
									address: artist
								};
								this.playDefault(v.hash);
							});
						} else {
							this.playDefault(v.hash);
						}
					}

				}

			},
			playDefault(hash) {

				if (this.autoplay && this.autoplay.hash == hash) {
					location.hash = "";
					this.selectedItem(hash);
					this.autoplay = false;
				}
			},
			async loadNFTs() {

				const address = window.mimic || this.user.address;
				if (!address) {
					this.logged = false;
					return;
				}

				const starter = {
					start: this.items.owned.start
				};
				if (this.autoplay && this.autoplay.hash) {
					starter.first = [this.autoplay.hash];
				}
				const cacheTime = 3600 * 24 * 5;
				await nft.getNFTsFrom(address, 60 * 5, this.gotNFT(), () => {
					this.items.owned.loaded = true;
				}, (a) => {
					if (a.length == 0) this.items.owned.loaded = true;
					this.items.owned.qty = a.length;
					this.items.owned.hashes = [...a];
					this.items.owned.hashes.splice(0, this.items.owned.start);
					this.items.owned.og_hashes = a;
					this.ownedLoad(true);
				}, starter, cacheTime);
				this.ownedLoad();
			},
			newFit(x) {
				this.count = x;
				this.ownedLoad(true);
			},
			pager(pagina) {
				if (pagina.page) {
					const rpage = pagina.page.map(x => x);
					const idx = this.row ? 0 : 1;
					const flored = Math.floor(rpage[idx]);
					this.scrollPage = flored;
					if ((flored + 1) != this.items[this.category].page) {
						this.items[this.category].page = flored;
						this.ownedLoad(true);
					} else {
						this.items[this.category].page = flored;
					}
				} else {
					const flored = this.items[this.category].page != 0;
					this.scrollPage = 0;
					this.items[this.category].page = 0;
					if (flored) {
						this.ownedLoad(true);
					}
				}
			},

			ownedLoad(filter) {

				const address = this.user.address;
				if ((this.page > this.items.owned.max || filter || (this.page === -1 && this.items.owned.max == -1)) && this.items.owned.hashes) {
					if (!filter) this.items.owned.max = this.items.owned.page;

					if (this.items.owned.processing || (this.count && (!this.count[0] || !this.count[1]))) return;

					if (this.page === -1) {
						this.scrollPage = 1;
					}

					const cb = this.gotNFT();

					(async () => {

						this.items.owned.processing = true;


						const minimumView = () => {
							return (this.page * (this.count[1] || 1)) + ((this.count[1] || 1) * 1);
						}
						//const visibles = ()=>(this.page*this.count[0]);


						const temporal = {};

						while ( this.nftData.length < minimumView() && this.items.owned.hashes.length > 0 ) {

							const t = this.items.owned.hashes.splice(0, 1)[0];
							const mt = t.mint.toBase58 ? t.mint.toBase58() : t.mint;
							temporal[mt] = nft.getNFT(mt, false).then((tokendata) => {
								if (tokendata) {
									cb({
										hash: mt,
										tokendata
									});
								} else {
									this.failed.push(mt);
								}
								delete temporal[mt];
							});

							if (Object.keys(temporal).length > 10) {
								await waitFor(() => {
									return Object.keys(temporal).length < 2;
								})
							}
						}

						this.items.owned.processing = false;

					})();

				}

			},
			async loadUser() {
				const ses = await account.getActiveSession();
				let address = ses && ses.profile && ses.profile.address;
				let username = address && ses.profile.username;
				if (address) {
					const avi = userAvatar(username);
					const f = await file.cacheFile(avi, {
						fast: true
					})
					let image = false;
					if (f) image = URL.createObjectURL(f);
					this.user = {
						name: username,
						address,
						image
					};
				}
			}

		}
</script>
<style scoped lang="scss">
	.player :deep(.linea .line) {
		position: absolute;
		left: 0px;
		right: 0px;
		top: 0px;
		bottom: 0px;
		width: calc(100% - 50px);
		height: 1px;
		background: #FFF;
		content: '';
		z-index: 0;
		opacity: 0.1;
		margin: auto;
	}

	.player :deep(.texto.alone) {
		height: max-content !important;
		position: absolute !important;
		margin: auto !important;
		left: 0px;
		top: 0px;
		right: 0px;
		background: #202020 !important;
		bottom: 0px;
	}

	.player :deep(.linea .texto) {
		position: relative;
		z-index: 1;
		width: max-content;
		background: #222;
		color: #FFF;
		margin: auto;
		padding: 0px 30px;
	}

	.player :deep(.linea) {
		position: relative;
	}

	.slider :deep(.item) {
		//	padding: 7px !important;
	}

	.header {
		padding: 5px;
		background: rgba(0, 0, 0, 0.25);
		height: 50px !important;
		min-height: 50px;
		border-bottom: solid 1px rgba(0, 0, 0, 0.175);
	}

	.nolog {
		position: absolute;
		left: 0px;
		top: 0px;
		right: 0px;
		z-index: 10;
		background: rgba(0, 0, 0, 0.75);
	}

	.userimage {
		width: 30px;
		height: 30px;
		min-width: 30px;
		border-radius: var(--radius);
		background-color: rgba(0, 0, 0, 0.25);
		background-image: var(--image);
		background-position: center;
		background-size: cover;
	}

	@media (max-width:599px) {
		.item_list {
			height: calc(100% - 258px) !important;
		}

		.playing {
			height: 50vw !important;
			min-height: 50vw !important;
		}

		.slider {
			right: unset !important;
			left: 45px !important;
		}
	}

	@media (min-width:600px) {
		.item_list {
			height: calc(100% - 279px) !important;
		}

		.playing {
			height: 32% !important;
			min-height: 32% !important;
		}

		.player .case {
			max-width: 500px;
			max-height: 700px;
			height: calc(100vh - 50px) !important;
			width: calc(100vw - 50px) !important;
			border-radius: var(--radius);
			box-shadow: 0px 0px 25px rgba(0, 0, 0, 0.5);
		}
	}

	.player {
		--player-primary: #111;
		--player-primary-contrast: #FFF;
		--player-primary-invert-contrast: 0;
		--player-primary-invert: 1;
		overflow: hidden;
	}

	.loader {
		width: 30px !important;
		height: 30px !important;
		--filtro: invert(var(--player-primary-invert));
		filter: var(--filtro);
	}

	.logo:hover,
	.userclick:hover {
		transform: translateY(-1px);
	}

	.logo,
	.userclick {
		cursor:#{$clickCursor};
	}

	.logo {

		background-size: 40%;
		background-repeat: no-repeat;
		background-position: center;
		width: 35px;
		height: 35px;
		--filtro: invert(var(--player-primary-invert));
		filter: var(--filtro);
		min-width: 35px;
	}



	.item_list {

		overflow-y: auto;
		overflow-y: overlay;
		border-top: solid 1px rgba(0, 0, 0, 0.225);
	}

	.list :deep(.card) {
		border-bottom: solid 1px rgba(0, 0, 0, 0.225);
	}

	.list :deep(.card:not(.odd)) {
		background: rgba(0, 0, 0, 0.0625);

	}


	.slider :deep(input:not(.selected) + .item > .label) {
		color: var(--theme-primary-contrast);
	}

	.section {
		height: calc(100% - 50px) !important;
		width: 100% !important;
		position: absolute !important;
		top: 50px;
		left: 0px;
		right: 0px;
		bottom: 0px;
		margin: auto;
	}

	.search_header {
		padding: 10px;
		border-bottom: solid 1px rgba(0, 0, 0, 0.25);
		z-index: 10000;
		position: sticky !important;
		background: #222;
		top: 0px;

	}

	.search_header input {
		background: rgba(0, 0, 0, 0.25) !important;
		border: solid 1px rgba(0, 0, 0, 0.25) !important;
		height: 40px;
		color: #FFF !important;
		outline: none !important;
	}

	.slider {
		width: 100%;
		max-width: 100px;
		position: absolute;
		left: 0px;
		right: 0px;
		background-color: rgba(0, 0, 0, 0.1) !important;
		bottom: 0px;
		--theme-primary: #000;
		--theme-primary-contrast: #FFF;
		top: 0px;
		color: var(--theme-primary);
		height: 35px;
		margin: auto;
	}

	.cerrar {
		position: absolute;
		right: 20px;
		top: 0px;
		bottom: 0px;
		margin: auto;
		opacity: 0.25;
	}

	.playing {

		position: relative;
		background-color: rgba(0, 0, 0, 0.1);
		max-height: 300px;
	}

	.player .case {

		background: #222;
		height: var(--window-height);
		width: 100vw;
		margin: auto;
		overflow: hidden;
	}

	.player {
		padding-top: var(--top);
		padding-bottom: var(--bottom);
		background: var(--player-primary);
		color: var(--player-primary-contrast);
		height: var(--window-height) !important;
		width: 100vw !important;
	}
</style>