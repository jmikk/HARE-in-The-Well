<script lang="ts">
	import Head from '$lib/component/Head.svelte';
	import Input from '$lib/component/Input.svelte';
	import Buttons from '$lib/component/Buttons.svelte';
	import Terminal from '$lib/component/Terminal.svelte';
	import { parseXML, parser } from '$lib/globals';
	import * as ExcelJS from 'exceljs';
	import type { NSRegion } from '$lib/types';

	let main = '';
	let stoppable = false;
	let stopped = false;
	let progress = '';
	let workbook: any;
	let downloadable = false;
	function sanitize(string: string) {
		try {
			if (['=', '+', '-', '@'].includes(string.charAt(0))) {
				return `'${string}`;
			} else {
				return string;
			}
		} catch (error) {
			return null;
		}
	}

	async function glass() {
		downloadable = false
		progress += `<p>Gathering founderless regions.</p>`
		const governorless = await parseXML(
			'https://www.nationstates.net/cgi-bin/api.cgi?q=regionsbytag;tags=governorless',
			main
		);
		const governorlessArr = governorless.WORLD.REGIONS.split(',');

		progress += `<p>Gathering passwordless regions.</p>`
		const passwordless = await parseXML(
			'https://www.nationstates.net/cgi-bin/api.cgi?q=regionsbytag;tags=-password',
			main
		);
		const passwordlessArr = passwordless.WORLD.REGIONS.split(',');

		const currentDate = new Date();
		const utcMinus7Date = new Date(currentDate.getTime() - 7 * 60 * 60 * 1000);
		utcMinus7Date.setDate(utcMinus7Date.getDate() - 1);
		const date = utcMinus7Date.toISOString().slice(0, 10);
		progress += `<p>Requesting ${date} regional dump.</p>`
		const regionRes = await fetch(`https://raw.githubusercontent.com/Kractero/region-xml-dump/main/data/${date}-Regions.xml`, {
			method: "GET"
		});
		const regionText = await regionRes.text()
		const regionXML = parser.parse(regionText)
		const regionList: Array<NSRegion> = regionXML.REGIONS.REGION;
		const regions = [];
		const names = regionList.map((region) => region.NAME);
		const numNations = regionList.map((region) => region.NUMNATIONS);
		const delVotes = regionList.map((region) => region.DELEGATEVOTES);
		const delAuth = regionList.map((region) => region.DELEGATEAUTH);
		const majorList = regionList.map((region) => region.LASTUPDATE);
		const wfelist = regionList.map((region) => region.FACTBOOK);
		const embassies: {[key: string]: Array<string>} = {};
		for (const region of regionList) {
			embassies[region.NAME] = Object.values(region.EMBASSIES);
		}
		for (let i = 0; i < names.length; i++) {
			const name = names[i];
			const nationCount = numNations[i];
			const delVotesText = delVotes[i];
			const authText = delAuth[i];
			const time = majorList[i];
			const wfe = sanitize(wfelist[i]);
			const embassiesList = embassies[name];

			const regionObject: { [key: string]: any } = {
				Regions: name,
				'Region Link': `https://www.nationstates.net/region=${name}`,
				'# Nations': nationCount,
				'Tot. Nations': 0,
				'Minor Upd. (est)': '',
				'Major Upd. (true)': '',
				'Del. Votes': delVotesText,
				'Del. Endos': delVotesText - 1,
				last_update: time,
				exec: authText[0] === 'X',
				Embassies: embassiesList ? embassiesList.join(',') : '',
				WFE: wfe
			};
			regions.push(regionObject);
		}

		let major = 5350;
		let minor = 3550;

		const lastRegion = regions[regions.length - 1];
		const firstRegion = regions[0];
		major = Number(lastRegion.last_update) - Number(firstRegion.last_update);

		const CumuNatList = [0];

		for (const region of regions) {
			region['Tot. Nations'] = CumuNatList[CumuNatList.length - 1];
			CumuNatList.push(CumuNatList[CumuNatList.length - 1] + Number(region['# Nations']));
		}

		const major_per_nation = major / CumuNatList[CumuNatList.length - 1];
		const minor_per_nation = minor / CumuNatList[CumuNatList.length - 1];

		for (const region of regions) {
			const tempminor = Math.floor(region['Tot. Nations'] * minor_per_nation);
			const tempmajor = Math.floor(region['Tot. Nations'] * major_per_nation);

			const ti_s = tempminor % 60;
			const ti_m = Math.floor((tempminor / 60) % 60);
			const ti_h = Math.floor(tempminor / 3600);

			region['Minor Upd. (est)'] = `${ti_h}:${ti_m}:${ti_s}`;

			const tm_s = tempmajor % 60;
			const tm_m = Math.floor((tempmajor / 60) % 60);
			const tm_h = Math.floor(tempmajor / 3600);

			region['Major Upd. (true)'] = `${tm_h}:${tm_m}:${tm_s}`;
		}

		workbook = new ExcelJS.Workbook();
		const worksheet = workbook.addWorksheet('Spyglass Timesheet');
		worksheet.columns = [
			{ header: 'Regions', key: 'region', width: 45 },
			{ header: 'Region Link', key: 'link' },
			{ header: '# Nations', key: 'numnations' },
			{ header: 'Tot. Nations', key: 'totnations' },
			{ header: 'Minor Upd. (est)', key: 'minor' },
			{ header: 'Major Upd. (true)', key: 'major' },
			{ header: 'Del. Votes', key: 'delvotes' },
			{ header: 'Del. Endos', key: 'delauth' },
			{ header: 'Embassies', key: 'embassies' },
			{ header: 'WFE', key: 'wfe' }
		];

		regions.forEach((region) => {
			const row = worksheet.addRow({
				region: region.Regions,
				link: region['Region Link'],
				numnations: region['# Nations'],
				totnations: region['Tot. Nations'],
				minor: region['Minor Upd. (est)'],
				major: region['Major Upd. (true)'],
				delvotes: region['Del. Votes'],
				delauth: region['Del. Endos'],
				embassies: region['Embassies'],
				wfe: region['WFE']
			});
			row.getCell(10).alignment = { horizontal: 'right' };
			if (region.exec && passwordlessArr.includes(region.Regions)) {
				row.getCell(1).fill = {
					type: 'pattern',
					pattern: 'solid',
					fgColor: { argb: 'FFFFFF00' }
				};
				row.getCell(1).value = `${region.Regions}~`;
				row.getCell(2).fill = {
					type: 'pattern',
					pattern: 'solid',
					fgColor: { argb: 'FFFFFF00' }
				};
			}
			if (passwordlessArr.includes(region.Regions) && governorlessArr.includes(region.Regions)) {
				row.getCell(1).fill = {
					type: 'pattern',
					pattern: 'solid',
					fgColor: { argb: 'FF00FF00' }
				};
				row.getCell(1).value = `${region.Regions}~`;
				row.getCell(2).fill = {
					type: 'pattern',
					pattern: 'solid',
					fgColor: { argb: 'FF00FF00' }
				};
			}
			if (!passwordlessArr.includes(region.Regions)) {
				row.getCell(1).fill = {
					type: 'pattern',
					pattern: 'solid',
					fgColor: { argb: 'FFFF0000' }
				};
				row.getCell(1).value = `${region.Regions}*`;
				row.getCell(2).fill = {
					type: 'pattern',
					pattern: 'solid',
					fgColor: { argb: 'FFFF0000' }
				};
			}
			if (region['Del. Votes'] === 0) {
				row.getCell(8).fill = {
					type: 'pattern',
					pattern: 'solid',
					fgColor: { argb: 'FFFF0000' }
				};
			}
		});
		worksheet.getCell('L1').value = 'World ';
		worksheet.getCell('M1').value = 'Data';
		worksheet.getCell('L2').value = 'Nations';
		worksheet.getCell('L3').value = 'Last Major';
		worksheet.getCell('L4').value = 'Secs/Nation';
		worksheet.getCell('L5').value = 'Nations/Sec';
		worksheet.getCell('L6').value = 'Last Minor';
		worksheet.getCell('L7').value = 'Secs/Nation';
		worksheet.getCell('L8').value = 'Nations/Sec';
		worksheet.getCell('L10').value = 'Glasses Version';
		worksheet.getCell('L11').value = 'Date Generated';
		worksheet.getCell('M2').value = CumuNatList[CumuNatList.length - 1];
		worksheet.getCell('M3').value = major;
		worksheet.getCell('M4').value = major / CumuNatList[CumuNatList.length - 1];
		worksheet.getCell('M5').value = 1 / (major / CumuNatList[CumuNatList.length - 1]);
		worksheet.getCell('M6').value = minor;
		worksheet.getCell('M7').value = minor_per_nation;
		worksheet.getCell('M8').value = 1 / minor_per_nation;
		worksheet.getCell('M10').value = '1.0';
		worksheet.getCell('M11').value = new Date().toISOString();
		progress += `<p class="text-green-400">Glasses finished processing!</p>`
		downloadable = true;
	}
</script>

<Head
	title={'Hare - Glasses'}
	description={'Specify a nation and get all the regionmates they are not endorsing.'}
/>

<h1 class="text-4xl mb-2">Glasses</h1>
<p class="text-xs mb-4">
	Original tool by <a href="https://github.com/Derpseh/Spyglass" target="_blank" rel="noreferrer noopener">the Spyglass Team</a>
	rewritten in JavaScript by Kractero
</p>
<p class="mb-16">Generate a spreadsheet that includes information on NationStates regions.</p>

<div class="lg:w-[1024px] lg:max-w-5xl flex flex-col lg:flex-row gap-8 break-normal">
	<form on:submit|preventDefault={() => glass()} class="flex flex-col gap-8">
		<Input text="User Agent" bind:bindValue={main} forValue="main" required={true} />
		<Buttons>
			<button
				disabled={!downloadable}
				type="button"
				on:click={async () => {
					const buffer = await workbook.xlsx.writeBuffer();
					const blob = new Blob([buffer], {
						type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
					});
					const blobUrl = URL.createObjectURL(blob);
					const downloadLink = document.createElement('a');
					downloadLink.href = blobUrl;
					downloadLink.download = 'Glasses.xlsx';
					downloadLink.click();
					URL.revokeObjectURL(blobUrl);
				}}
				class="bg-green-500 rounded-md px-4 py-2 transition duration-300 hover:bg-green-300 disabled:opacity-20 disabled:hover:bg-green-500"
			>
				Download
			</button>
		</Buttons>
	</form>
	<Terminal bind:progress />
</div>
