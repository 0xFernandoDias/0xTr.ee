<script context="module">
	/**
	 * @type {import('@sveltejs/kit').Load}
	 */
	export async function load({ fetch, params, url }) {
		const { pathname } = url;
		const { slug } = params;

		//check address is potential wallet path
		//TODO add more checking ;)
		if (slug.startsWith('0x')) {
			const searchParams = url.searchParams;
			const tab = searchParams.get('tab');
			console.log('slug', slug);

			//send across tab path if defined
			return {
				props: {
					slug,
					tab: tab ? tab : 'Offers',
				},
			};
			//if it isn't 0x then 404 page..
		} else {
			return {
				status: 404,
			};
		}
	}
</script>

<script>
	//sveltkit
	import { goto } from '$app/navigation';

	// svelte
	import { onDestroy, onMount } from 'svelte';

	//card imgs
	import ogSquareImageSrc from '$lib/assets/home/home-open-graph-square.jpg';
	import ogImageSrc from '$lib/assets/home/home-open-graph.jpg';
	import twitterImageSrc from '$lib/assets/home/home-twitter.jpg';
	import featuredImageSrc from '$lib/assets/home/home.jpg';

	//img
	import SEO from '$lib/components/SEO/index.svelte';

	//conf
	import website from '$lib/config/website';

	//ABI
	import ABI from '$lib/abi/tree.json';

	//components
	import HeaderTabList from '$lib/components/HeaderTabList.svelte';
	import Button from '$lib/components/Button.svelte';
	import { bigBlue } from '$lib/components/conf/Buttons.js';

	//stores
	import { modal as sModal } from '../../stores/modal.js';
	import { bidding as sBidding } from '../../stores/bidding.js';
	import { user as sUser } from '../../stores/user.js';
	import { app as sApp } from '../../stores/app.js';
	import { nft as sNFT } from '../../stores/nft.js';

	export let tab, slug;

	let hasTabs = [
		{
			name: 'Offers',
			path: `/nft/${slug}?tab=Offers`,
		},
		{
			name: 'History',
			path: `/nft/${slug}?tab=History`,
		},
		{
			name: 'Activity',
			path: `/nft/${slug}?tab=Activity`,
		},
	];
	$: nftMeta = ($sNFT.id[slug])?$sNFT.id[slug]:{};
	$: owner = '';
	$: nftImage = nftMeta.image ? `background-image:url("${nftMeta.image}")` : '';
	$: activity = [];
	$: bids = [];

	$: if ($sBidding.waitingTransaction === $sBidding.tokenId) {
		updatingBid = true;
		console.log('................................loading..........');
		bidUpdating();
	}
	let updatingBid = false;
	$: if ($sBidding.waitingTransaction !== $sBidding.tokenId && updatingBid) {
		console.log('................................updating Bid..........');
		getbid();
	}

	function bidUpdating() {
		latestBid = false;
		showBiddingOptions = false;
		loadingTxt = 'Updating';
	}
	function getbid() {
		updatingBid = false;
		getLatestBid();
		showBiddingOptions = true;
	}
	//$: nftImage =
	//typeof nftImage.image !== 'undefined' ? `background-image:url("${nftImage.image}")` : '';

	//bid
	let showBiddingOptions = true;
	let bidLoading = true;
	let lastestBidderID = '';
	let loadingTxt = '';
	let web3Browser = false;
	$: latestBid = false;
	const addressArr = slug.split('_');
	let isMounted = false;
	let loading = true;

	$: if ((isMounted) && ($sApp.ready)) {
		refresh();
	}


	onMount(async () => {
		isMounted = true;
	});

	let runOnce = true;
	async function refresh() {
		if (runOnce) {
			runOnce = false;
			
			if (typeof window.ethereum !== 'undefined' || typeof window.web3 !== 'undefined') {
				await Moralis.enableWeb3();
			}
			console.log(addressArr);
			sBidding.updateVal('nftContract', addressArr[0]);
			sBidding.updateVal('tokenId', addressArr[1]);
			const options = { address: addressArr[0], token_id: addressArr[1], chain: 'mumbai' };
			//const metaData = await Moralis.Web3API.token.getNFTMetadata(options);
			//const tokenMetadata = await Moralis.Web3API.token.getTokenMetadata(options);
			const tokenIdMetadata = await Moralis.Web3API.token.getTokenIdMetadata(options);
			console.log('-------------', tokenIdMetadata);
			nftMeta = JSON.parse(tokenIdMetadata.metadata);
			owner = tokenIdMetadata.owner_of;
			console.log(nftMeta);
			sNFT.addNFT(slug,nftMeta);


			const getWalletTokenIdTransfers = await Moralis.Web3API.token.getWalletTokenIdTransfers(
				options,
			);
			console.log('getWalletTokenIdTransfers', getWalletTokenIdTransfers);
			activity = getWalletTokenIdTransfers.result;

			if (typeof window.ethereum !== 'undefined' || typeof window.web3 !== 'undefined') {
				web3Browser = true;
				getLatestBid();
			} else {
				web3Browser = false;
			}
			loading = false;
			getBids();
			
		}
	}

	/**
	 * getLatestBid
	 */
	async function getLatestBid() {
		console.log('[getLatestBid]');
		bidLoading = true;
		loadingTxt = 'Loading';
		//const web3 = await Moralis.enableWeb3();//do I need this?...
		const ethers = Moralis.web3Library;
		//const bigNumberPrice = activePrice);

		const options = {
			chain: $sBidding.chain, //'mumbai',
			contractAddress: $sBidding.treeContract,
			functionName: 'getBid',
			abi: ABI.abi,
			params: {
				_nftContract: $sBidding.nftContract,
				_tokenId: $sBidding.tokenId,
			},
		};

		const getBid = await Moralis.executeFunction(options);
		console.log('[getBid]', getBid);
		console.log(getBid.timestamp.toString());
		console.log(getBid.tokenId.toString());
		console.log(getBid.bidder);
		console.log(ethers.utils.formatEther(getBid.price.toString()));

		latestBid = ethers.utils.formatEther(getBid.price.toString());
		sBidding.updateVal('latestBidPrice', latestBid);
		sBidding.updateVal('buyerID', getBid.bidder);
		lastestBidderID = getBid.bidder;

		loadingTxt = '';
		bidLoading = false;
	}

	/**
	 * cancelLatestBid
	 */
	async function cancelLatestBid() {
		console.log('[cancelBid]');
		showBiddingOptions = false;
		bidLoading = true;
		loadingTxt = 'Canceling';
		//const web3 = await Moralis.enableWeb3();//do I need this?...
		const ethers = Moralis.web3Library;
		//const bigNumberPrice = activePrice);

		const options = {
			chain: $sBidding.chain, //'mumbai',
			contractAddress: $sBidding.treeContract,
			functionName: 'cancelBid',
			abi: ABI.abi,
			params: {
				_nftContract: $sBidding.nftContract,
				_tokenId: $sBidding.tokenId,
			},
		};

		const cancelBid = await Moralis.executeFunction(options);
		latestBid = false;
		console.log('[cancelBid]', cancelBid);
		await cancelBid.wait();
		loadingTxt = '';
		showBiddingOptions = true;
		getLatestBid();

		bidLoading = false;
	}
	/**
	 * rejectLatestBid
	 */
	async function rejectLatestBid() {
		console.log('[rejectLatestBid]');
		showBiddingOptions = false;
		bidLoading = true;
		loadingTxt = 'Canceling';
		//const web3 = await Moralis.enableWeb3();//do I need this?...
		const ethers = Moralis.web3Library;
		//const bigNumberPrice = activePrice);

		const options = {
			chain: $sBidding.chain, //'mumbai',
			contractAddress: $sBidding.treeContract,
			functionName: 'rejectBid',
			abi: ABI.abi,
			params: {
				_nftContract: $sBidding.nftContract,
				_tokenId: $sBidding.tokenId,
			},
		};

		const rejectBid = await Moralis.executeFunction(options);
		latestBid = false;
		console.log('[rejectBid]', rejectBid);
		await rejectBid.wait();
		loadingTxt = '';
		showBiddingOptions = true;
		getLatestBid();

		bidLoading = false;
	}

	/**
	 * bids
	 */
	async function bidsxOLD() {
		console.log('[bids]');
		//const web3 = await Moralis.enableWeb3();//do I need this?...
		const ethers = Moralis.web3Library;
		//const bigNumberPrice = activePrice);

		const options = {
			chain: $sBidding.chain, //'mumbai',
			contractAddress: $sBidding.treeContract,
			functionName: 'bids',
			abi: ABI.abi,
			params: {},
		};

		const getBids = await Moralis.executeFunction(options);
		console.log('[getBids]', getBids);
	}

	/**
	 * getBids
	 */
	 async function getBids() {
		console.log('[getBids]');
		let params = {
			nftContract:addressArr[0],
			tokenId:addressArr[1],
		};
		params = {
			nftContract:'0x2953399124f0cbb46d2cbacd8a89cf0599974963',
			tokenId:'13881000456214464272594247052417607500385614301131248520949923275583315247105'
		}
		let res = await Moralis.Cloud.run('getBids',params)

		console.log('[getBids]',res[0].attributes);
		bids = [res[0].attributes];
	}

	/**
	 * shareLink
	 */
	function shareLink() {
		if (
			/Android|webOS|iPhone|iPad|iPod|BlackBerry/i.test(navigator.userAgent) ||
			/Android|webOS|iPhone|iPad|iPod|BlackBerry/i.test(navigator.platform)
		) {
			// ...
			if (navigator.share) {
				navigator
					.share({
						title: nftMeta.name,
						url: `${nftMeta.name} ${window.location.href} via 0xtree_&related=0xtree_`,
					})
					.then(() => {
						console.log('Thanks for sharing!');
					})
					.catch(console.error);
			} else {
				window.open(
					`https://twitter.com/intent/tweet?text=${nftMeta.name}&url=${window.location.href}&via=0xtree_&related=0xtree_`,
				);
			}
		} else {
			window.open(
				`https://twitter.com/intent/tweet?text=${nftMeta.name}&url=${window.location.href}&via=0xtree_&related=0xtree_`,
			);
		}
	}

	const { author, siteUrl } = website;

	let title = 'Home';
	const breadcrumbs = [
		{
			name: 'Home',
			slug: '',
		},
	];

	let metadescription = '';
	const featuredImageAlt = '';
	const featuredImage = {
		url: featuredImageSrc,
		alt: featuredImageAlt,
		width: 672,
		height: 448,
		caption: 'Home page',
	};
	const ogImage = {
		url: ogImageSrc,
		alt: featuredImageAlt,
	};
	const ogSquareImage = {
		url: ogSquareImageSrc,
		alt: featuredImageAlt,
	};

	const twitterImage = {
		url: twitterImageSrc,
		alt: featuredImageAlt,
	};
	const entityMeta = {
		url: `${siteUrl}/`,
		faviconWidth: 512,
		faviconHeight: 512,
		caption: author,
	};
	const seoProps = {
		title,
		slug: '',
		entityMeta,
		datePublished: '2022-01-04T14:19:33.000+0100',
		lastUpdated: '2022-01-04T14:19:33.000+0100',
		breadcrumbs,
		metadescription,
		featuredImage,
		ogImage,
		ogSquareImage,
		twitterImage,
		playerURL: `https://www.0xtr.ee/player/nft?contract=${addressArr[0]}&tokenid=${addressArr[1]}`,
	};
</script>

<style lang="scss">
	section {
		display: flex;
		flex-direction: column;
		flex: 1;
	}
	.scrollable {
		overflow-y: scroll;
		overflow-x: hidden;
		-webkit-overflow-scrolling: touch;
		overscroll-behavior-y: none;
		position: fixed;
		top: 0px;
		left: 0px;
		right: 0px;
		bottom: 0px;
		background-color: var(--bg-baseWrap);
		padding-bottom: 80px;
	}

	article {
		display: flex;
		justify-content: start;
		flex-direction: column;
		margin: 0px;
		text-align: left;
		align-items: center;
	}
	header {
		width: 100%;
		position: relative;
		z-index: 10;
	}
	.profileBG {
		background: #fff;
		min-height: 160px;
		position: relative;
		width: 100%;
		background-repeat: no-repeat;
		background-position: center;
	}
	.infoPanel {
		padding: 0px 20px;
	}
	.infoPanel h4 {
		margin: 15px 0px 10px;
		color: #383c3a;
	}
	.activity b {
		font-size: 0.875em;
	}

	.hr {
		background: #f9fafc;
		border-radius: 50px;
		height: 4px;
		margin: 6px 0px;
	}
	hr {
		display: none;
	}
	.loader {
		min-width: 40px;
		height: 40px;
		background-color: rgb(20, 115, 23);
		background-image: url('/img/spinner_oval.svg');
		background-size: 24px;
		background-position: center;
		background-repeat: no-repeat;
		border-radius: 6px;
		transition: background 0.2s;
	}

	.latestBid {
		display: flex;
		border: solid 4px #eaeaea;
		border-radius: 100px;
		margin: 0px 20px 10px;
		overflow: hidden;
		background: #eaeaea;
	}
	.latestBid .txt {
		padding-left: 10px;
		display: flex;
		align-items: center;
		font-weight: bold;
	}

	.bidLoaded .loader {
		background-color: #fff;
		background-image: url('/img/ico_matic.svg');
	}

	.share {
		width: 40px;
		height: 40px;
		position: absolute;
		cursor: pointer;
		bottom: -20px;
		z-index: 10;
		right: 20px;
		background: #fff; /*#f5f7f5;*/
		border-radius: 50%;
		background-image: url('/img/ico_share.svg');
		background-repeat: no-repeat;
		background-size: 14px;
		background-position: center;
		border: solid 2px #eaeaea;
	}
	.web3InfoPanel,
	#deepLink {
		border-top: solid 8px #00c7b6;
		border-radius: 8px;
		padding: 16px 24px 24px 24px;
		margin: 20px;
		box-shadow: rgba(0, 0, 0, 0) 0px 0px 0px 0px, rgba(0, 0, 0, 0) 0px 0px 0px 0px,
			rgba(14, 30, 37, 0.12) 0px 2px 4px 0px;
	}
		#deepLink {
			display: flex;

		}

		#deepLink a {
			color: #0d1821;
			text-decoration: none;
			font-weight: bold;
			font-size: 1.1em;
		}
</style>

<SEO {...seoProps} />

<section style="transform: translate3d(0px, 0px, 0px);" id="XT-NFT" class="scrollable gpu_acc">
	<div style="flex:1;display:flex;flex-direction:column;">
		<article style="flex:1;" dir="auto">
			<header>
				<div class="profileBG" style="{nftImage}">
					<i on:click="{shareLink}" class="share"></i>
				</div>
				<div class="infoPanel">
					{#if nftMeta.name}
						<h4>{nftMeta.name}</h4>
						<p>{nftMeta.description}</p>
					{/if}
				</div>
				{#if (loading)}
					<div style="text-align:center;">
						
						<div class="latestBid" class:bidLoaded="{latestBid}">
							<div><div class="loader"></div></div>

							<div class="txt">
								Loading Bid Information...
							</div>
						</div>
					</div>
				{:else}
					{#if web3Browser}
						<div class="latestBid" class:bidLoaded="{latestBid}">
							<div><div class="loader"></div></div>

							<div class="txt">
								{#if latestBid}
									<b>Latest Bid</b>: {latestBid}
								{:else}
									{loadingTxt} Latest Bid
								{/if}
							</div>
						</div>
						{#if showBiddingOptions}
							<div style="display:flex;align-items:center;justif-content:center">
								{#if owner}
									{#if $sUser.ethAddress.toLowerCase() === owner.toLowerCase()}
										{#if latestBid > 0}
											<Button
												{...bigBlue}
												on:click="{() => {
													sModal.showModal({
														enable: 'true',
														title: 'Accept Offer',
														subHeader:
															'Great! We will handle the NFT swap and send MATIC and Tree tokens to your account!',
														tpl: 'recievedOffer',
														buttons: [
															{
																text: 'Accept Offer',
																action: 'acceptOffer',
															},
														],
													});
												}}">
												Accept Offer</Button>
											<Button {...bigBlue} on:click="{rejectLatestBid}">Decline Offer</Button>
										{/if}
									{:else}
										<Button
											{...bigBlue}
											on:click="{() => {
												//updatingBid = true;
												sModal.showModal({
													enable: 'true',
													title: 'Make an  Offer',
													subHeader:
														'By making an offer you will be sending funds into a holding area until offer has been accepted.',
													tpl: 'price',
													buttons: [
														{
															text: 'Make An Offer',
															action: 'makeOffer',
														},
													],
												});
											}}">
											{#if latestBid > 0}
												Increase Offer
											{:else}
												Make an Offer
											{/if}</Button>

										{#if lastestBidderID.toLowerCase() === $sUser.ethAddress.toLowerCase()}
											<Button {...bigBlue} on:click="{cancelLatestBid}">Cancel Bid</Button>
										{/if}
									{/if}
								{/if}
							</div>
						{/if}
					{:else}
						<div class="web3InfoPanel">
							To make an offer on this NFT you need metamask or a browser with web3 capabilties.
						</div>
						<dl id="deepLink">
							<dt><img width="40" src="/img/ico_metamask.svg" alt="metamask" /></dt>
							<dd><a target="_blank" id="deepLinkURL" href="https://metamask.app.link/dapp/0xtr.ee/nft/{addressArr[0]}_{addressArr[1]}">
								Launch Metamask Mobile</a>
							</dd>
						</dl>
					{/if}
				{/if}
				<br />
				<!--<button on:click="{bids}">bids</button>-->
			</header>
			
			
			{#if (!loading)}
			<HeaderTabList
				on:tab="{(e) => {
					goto(e.detail.path, { replaceState: true, noscroll: true });
				}}"
				hasTabs="{hasTabs}"
				activeTab="{tab}" />

			<div style="width:100%; padding:0px 20px;">
				{#if tab === 'Offers'}
					{#if (bids) && (bids.length > 0)}
					<div class="activity">
						<div class="hr"><hr /></div>
						{#each bids as bid}
						<div style="flex">
							<div><b>{bid.bidder}</b></div>
							<div>
								{#if window && window._ethers}
									<b>{window._ethers.utils.formatEther(bid.price)} MATIC</b><br />
								{/if}
							</div>
						</div>
						<div class="hr"><hr /></div>
							
						{/each}
					</div>
					{/if}
				{:else if tab === 'History'}
					<div style="text-align:center">Work in progress.. </div>
				{:else if tab === 'Activity'}
					<div class="activity">
						<div class="hr"><hr /></div>
						{#each activity as active}
							{#if active.from_address === active.to_address}
								<b>Sent the token same wallet id</b>
							{:else}
								<b>{active.from_address}</b> -> <b>{active.to_address}</b>
							{/if}
							<div class="hr"><hr /></div>
						{/each}
					</div>
				{/if}
			</div>
			{/if}
		</article>
	</div>
</section>
