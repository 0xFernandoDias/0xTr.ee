<script context="module">
	/**
	 * @type {import('@sveltejs/kit').Load}
	 */
	export async function load({ fetch, params, url }) {
		const { pathname } = url;
		const { slug } = params;

		//check address is potential wallet path
		//TODO add more checking ;)
		//if (slug.startsWith('0x')) {
		const searchParams = url.searchParams;
		const tab = searchParams.get('tab');
		console.log('slug', slug);

		//send across tab path if defined
		return {
			props: {
				slug: '0x2953399124f0cbb46d2cbacd8a89cf0599974963_13881000456214464272594247052417607500385614301131248520949923274483803619329',
				tab: tab ? tab : 'Offers',
			},
		};
		//if it isn't 0x then 404 page..
		//} else {
		//	return {
		//		status: 404,
		//	};
		//}
	}
</script>

<script>
	//sveltkit
	import { goto } from '$app/navigation';

	// svelte
	import { onDestroy, onMount } from 'svelte';

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
	$: nftMeta = {};
	$: owner = '';
	$: nftImage = nftMeta.image ? `background-image:url("${nftMeta.image}")` : '';
	$: activity = [];

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
	$: latestBid = false;

	onMount(async () => {
		Moralis.start();
		Moralis.enableWeb3();
		const addressArr = slug.split('_');
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

		const getWalletTokenIdTransfers = await Moralis.Web3API.token.getWalletTokenIdTransfers(
			options,
		);
		console.log('getWalletTokenIdTransfers', getWalletTokenIdTransfers);
		activity = getWalletTokenIdTransfers.result;

		getLatestBid();
	});

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
	async function bids() {
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
</style>

<section style="transform: translate3d(0px, 0px, 0px);" id="XT-NFT" class="scrollable gpu_acc">
	<div style="flex:1;display:flex;flex-direction:column;">
		<article style="flex:1;" dir="auto">
			<header>
				<div class="profileBG" style="{nftImage}"></div>
				<div class="infoPanel">
					{#if nftMeta.name}
						<h4>{nftMeta.name}</h4>
						<p>{nftMeta.description}</p>
					{/if}
				</div>
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
				<br />
				<!--<button on:click="{bids}">bids</button>-->
			</header>
			<HeaderTabList
				on:tab="{(e) => {
					goto(e.detail.path, { replaceState: true, noscroll: true });
				}}"
				hasTabs="{hasTabs}"
				activeTab="{tab}" />

			<div style="width:100%; padding:0px 20px;">
				{#if tab === 'Offers'}
					Offers
				{:else if tab === 'History'}
					History
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
		</article>
	</div>
</section>
