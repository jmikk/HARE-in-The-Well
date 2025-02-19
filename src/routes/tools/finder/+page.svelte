<script lang="ts">
	import { onDestroy, onMount } from 'svelte';
	import { parser, sleep } from '$lib/globals';
	import { handleDownload } from '$lib/download';
	import { htmlContent } from '$lib/htmlContent';
	import InputCredentials from '$lib/component/InputCredentials.svelte';
	import Terminal from '$lib/component/Terminal.svelte';
	import Head from '$lib/component/Head.svelte';
	import { loadLocalStorage } from '$lib/loadLocalStorage';
	import Input from '$lib/component/Input.svelte';
	import Select from '$lib/component/Select.svelte';
	import Textarea from '$lib/component/Textarea.svelte';
	import type { Card } from '$lib/types';
	const abortController = new AbortController();
	let progress = "";
	let main = '';
	let puppets = '';
	let finderlist = '';
	let finderMode = 'Gift';
	let openNewLinkArr: Array<string> = [];
	let counter = 0;
	let junkHtml = '';
    let downloadable = false;
	let stoppable = false;
	let stopped = false;
	let password = "";
	let giftee = "";
	onMount(() => ({puppets, main, finderlist, finderMode, password, giftee} = loadLocalStorage(["stationPuppets", "stationMain", "stationFinderList", "stationFinderDefault", "stationPassword", "stationGiftee"])));
	onDestroy(() => abortController.abort());

	async function finder(main: string, puppets: string) {
		downloadable = false;
		stoppable = true;
		stopped = false;
		let puppetsList = puppets.split('\n');
		progress += "<p>Initiating Finder...</p>";
		const toFind = finderlist.split('\n');
		progress += `<p>Finding -> ${toFind.map((card) => card.trim()).join(', ')}</p>`;
		for (let i = 0; i < puppetsList.length; i++) {
			let currentNationXPin = ""
			let nation = puppetsList[i].toLowerCase().replaceAll(' ', '_');
			if (abortController.signal.aborted || stopped) {
				break;
			}
			try {
				await sleep(700);
				progress += `<p>Processing ${nation} ${i + 1}/${puppetsList.length} puppets</p>`;
				const response = await fetch(
					`https://www.nationstates.net/cgi-bin/api.cgi/?nationname=${nation}&q=cards+deck`,
					{
						headers: {
							'User-Agent': main
						}
					}
				);
				const xml = await response.text();
				const xmlDocument = parser.parse(xml);
				const cards: Array<Card> = xmlDocument.CARDS.DECK.CARD;
                const matches = finderlist.split('\n')
				if (cards) {
					for (let j = 0; j < cards.length; j++) {
						const id = cards[j].CARDID;
						const season = cards[j].SEASON
						if (matches.includes(String(id))) {
							progress += `<p class="text-green-400">Found S${season} ${id} on ${puppetsList[i]}</p>`;
							if (finderMode === "Gift") {
								let token = ""
								const prepare = await fetch(
									`https://www.nationstates.net/cgi-bin/api.cgi/?nation=${nation}&cardid=${id}&season=${season}&to=${giftee}&mode=prepare&c=giftcard`,
									{
										headers: {
											'User-Agent': main,
											'X-Password': currentNationXPin ? currentNationXPin : password
										}
									}
								);
								if (!currentNationXPin) currentNationXPin = prepare.headers.get('x-pin') || "";
								const text = await prepare.text()
								const xml = parser.parse(text)
								token = xml.NATION.SUCCESS
								
								const gift = await fetch(
									`https://www.nationstates.net/cgi-bin/api.cgi/?nation=${nation}&cardid=${id}&season=${season}&to=${giftee}&mode=execute&c=giftcard&token=${token}`,
									{
										headers: {
											'User-Agent': main,
											'X-Pin': currentNationXPin
										}
									}
								);
								if (gift.status === 200) {
									progress += `<p class="text-green-400">${nation} gifted ${id} to ${giftee}`;
								} else {
									progress += `<p class="text-red-400">${nation} failed to gift ${id} to ${giftee}`;
								}
							} else {
								progress += `<p class="text-green-400">${nation} owns ${id}!`;
								openNewLinkArr = [
									...openNewLinkArr,
									`https://www.nationstates.net/page=deck/container=${nation}/nation=${nation}/card=${id}/season=${season}/User_agent=${main}Script=Finder/Author_discord=scrambleds/Author_main_nation=Kractero/autoclose=1`
								];
								junkHtml += `<tr><td><p>${i + 1} of ${
									cards.length
								}</p></td><td><p><a target="_blank" href="https://www.nationstates.net/page=deck/container=${nation}/nation=${nation}/card=${id}/season=${season}/User_agent=${main}Script=Finder/Author_discord=scrambleds/Author_main_nation=Kractero/autoclose=1\n">Link to Card</a></p></td></tr>\n`;
							}
						}
					}
				} else {
					progress += `<p class="text-blue-400">It is likely ${nation} has 0 cards, skipping!`
				}
			} catch (err) {
				progress += `<p class="text-red-400">Error processing ${nation} with ${err}</p>`;
			}
		}
		progress += `<p>Finished processing</p>`;
		downloadable = true;
		stoppable = false;
	}
</script>

<Head title={"Hare - Finder"} description={"Find which of the specified nations have which of the specified cards."} />

<h1 class="text-4xl mb-2">Finder</h1>
<p class="text-xs mb-4">
	<a class="underline" href="https://github.com/Kractero/cards-utilities/blob/main/finder.py" target="_blank" rel="noreferrer noopener">
		Kractero
	</a>
<p class="mb-16">
	Find which of the specified nations have which of the specified cards.
</p>

<div class="lg:w-[1024px] lg:max-w-5xl flex flex-col lg:flex-row gap-8 break-normal">
	<form on:submit|preventDefault={() => finder(main, puppets)} class="flex flex-col gap-8">
		<InputCredentials bind:main bind:puppets authenticated={finderMode === "Gift" ? true : false} />
		{#if finderMode === "Gift"}
			<Input text={`Gift To`} bind:bindValue={giftee} forValue="giftee" required={true} />
		{/if}
		<Textarea text="Card IDs to Find" bind:bindValue={finderlist} forValue="find" required />
        <div class="flex flex-col lg:flex-row gap-4 justify-between max-w-lg">
            <label class="w-24" for="jdj">Finder Default Behavior</label>
			<Select bind:mode={finderMode} options={['Gift', 'Sell']} />
        </div>
		<div class="max-w-lg flex justify-center gap-2">
			<button
				type="submit"
				class="bg-green-500 rounded-md px-4 py-2 transition duration-300 hover:bg-green-300"
			>
				Start
			</button>
			<button
				disabled={progress.length === 0}
				type="button"
				on:click={() => {
					if (counter > openNewLinkArr.length - 1) {
						return;
					}
					window.open(openNewLinkArr[counter], '_blank');
					counter++;
				}}
				class="bg-green-500 rounded-md px-4 py-2 transition duration-300 hover:bg-green-300 disabled:opacity-20 disabled:hover:bg-green-500"
			>
				Open Available Link
			</button>
			<button disabled={!downloadable} type="button" on:click={() => handleDownload('html', htmlContent(junkHtml), 'finder')}
				class="bg-green-500 rounded-md px-4 py-2 transition duration-300 hover:bg-green-300 disabled:opacity-20 disabled:hover:bg-green-500">
				Download
			</button>
		</div>
	</form>
	<Terminal bind:progress={progress} />
</div>
