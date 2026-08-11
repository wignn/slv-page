<script lang="ts">
	import type { OptionsContract } from '$lib/types';
	import { Calendar, Search } from 'lucide-svelte';

	interface Props {
		contracts?: OptionsContract[];
		chain?: OptionsContract[];
		underlyingPrice?: number;
	}

	let { contracts, chain, underlyingPrice }: Props = $props();

	let rawContracts = $derived(contracts ?? chain ?? []);

	let selectedExpiration = $state<string>('all');
	let selectedType = $state<'all' | 'call' | 'put'>('all');
	let searchQuery = $state<string>('');

	function getExpiration(c: any): string {
		return c.expiration_date ?? c.expiration ?? c.expire_date ?? c.exp_date ?? '-';
	}
	function getType(c: any): string {
		return (c.option_type ?? c.type ?? c.side ?? c.optionType ?? '').toLowerCase();
	}
	function getStrike(c: any): number {
		return typeof c.strike === 'number' ? c.strike : parseFloat(c.strike ?? c.strike_price ?? 0);
	}
	function getMark(c: any): number | null {
		return c.mark_price ?? c.mark ?? c.price ?? c.last_price ?? null;
	}
	function getIv(c: any): number | null {
		return c.implied_volatility ?? c.iv ?? c.impliedVolatility ?? null;
	}
	function getOi(c: any): number | null {
		return c.open_interest ?? c.oi ?? c.openInterest ?? null;
	}
	function getVol(c: any): number | null {
		return c.volume ?? c.vol ?? null;
	}
	function getGex(c: any): number {
		return c.gex ?? ((c.gamma ?? 0) * (getOi(c) ?? 0) * 100 * (underlyingPrice ?? 1));
	}

	// Unique expiration dates sorted ascending
	let expirationDates = $derived(
		Array.from(new Set(rawContracts.map(getExpiration).filter((d) => d !== '-'))).sort(
			(a, b) => new Date(a).getTime() - new Date(b).getTime()
		)
	);

	// Filtered contracts
	let filteredContracts = $derived(
		rawContracts.filter((c) => {
			const exp = getExpiration(c);
			const optType = getType(c);
			const strike = getStrike(c);
			const symbolStr = (c.contract_symbol ?? c.symbol ?? '').toLowerCase();

			if (selectedExpiration !== 'all' && exp !== selectedExpiration) {
				return false;
			}
			if (selectedType !== 'all' && optType !== selectedType) {
				return false;
			}
			if (searchQuery.trim() !== '') {
				const q = searchQuery.toLowerCase();
				const strikeStr = strike.toString();
				if (!strikeStr.includes(q) && !symbolStr.includes(q)) {
					return false;
				}
			}
			return true;
		})
	);

	// Sorted contracts by strike then option_type
	let sortedContracts = $derived(
		[...filteredContracts].sort((a, b) => {
			const sA = getStrike(a);
			const sB = getStrike(b);
			if (sA !== sB) return sA - sB;
			return getType(a).localeCompare(getType(b));
		})
	);

	function formatCurrency(val: number | null | undefined): string {
		if (val === null || val === undefined || isNaN(val)) return '-';
		return new Intl.NumberFormat('en-US', {
			style: 'currency',
			currency: 'USD',
			minimumFractionDigits: 2,
			maximumFractionDigits: 2
		}).format(val);
	}

	function formatCompact(val: number | null | undefined): string {
		if (val === null || val === undefined || isNaN(val)) return '0';
		return new Intl.NumberFormat('en-US', {
			notation: 'compact',
			compactDisplay: 'short',
			maximumFractionDigits: 1
		}).format(val);
	}

	function formatGreek(val: number | null | undefined): string {
		if (val === null || val === undefined || isNaN(val)) return '-';
		return val.toFixed(3);
	}

	function formatIv(val: number | null | undefined): string {
		if (val === null || val === undefined || isNaN(val)) return '-';
		const pct = val <= 1 ? val * 100 : val;
		return `${pct.toFixed(1)}%`;
	}

	function isItm(contract: any, undPrice?: number): boolean {
		if (undPrice === undefined) return false;
		const s = getStrike(contract);
		const t = getType(contract);
		if (t === 'call') {
			return undPrice > s;
		}
		if (t === 'put') {
			return undPrice < s;
		}
		return false;
	}
</script>

<div class="flex flex-col rounded-lg border border-border bg-surface p-4 shadow-xs">
	<!-- Header & Filters -->
	<div class="flex flex-wrap items-center justify-between gap-3 border-b border-border pb-3">
		<div>
			<h3 class="text-sm font-semibold text-text">Option Chain & Greeks</h3>
			<p class="text-xs text-text-muted">Filtered by expiration and strike</p>
		</div>

		<!-- Filter Bar -->
		<div class="flex flex-wrap items-center gap-2">
			<!-- Search -->
			<div class="relative flex items-center">
				<Search class="absolute left-2.5 h-3.5 w-3.5 text-text-muted" />
				<input
					type="text"
					placeholder="Search strike..."
					bind:value={searchQuery}
					class="h-8 w-32 rounded-md border border-border bg-surface-2 pl-8 pr-2.5 text-xs text-text placeholder-text-muted focus:border-accent focus:outline-none"
				/>
			</div>

			<!-- Type Filter -->
			<select
				bind:value={selectedType}
				class="h-8 rounded-md border border-border bg-surface-2 px-2 text-xs text-text focus:border-accent focus:outline-none"
			>
				<option value="all">All Types</option>
				<option value="call">Calls Only</option>
				<option value="put">Puts Only</option>
			</select>

			<!-- Expiration Dropdown -->
			<div class="relative flex items-center">
				<Calendar class="absolute left-2.5 h-3.5 w-3.5 pointer-events-none text-text-muted" />
				<select
					bind:value={selectedExpiration}
					class="h-8 rounded-md border border-border bg-surface-2 pl-8 pr-2.5 text-xs text-text focus:border-accent focus:outline-none"
				>
					<option value="all">All Expirations ({expirationDates.length})</option>
					{#each expirationDates as expDate}
						<option value={expDate}>{expDate}</option>
					{/each}
				</select>
			</div>
		</div>
	</div>

	<!-- Table Area -->
	<div class="mt-3 overflow-x-auto">
		<table class="w-full text-left text-xs">
			<thead class="border-b border-border bg-surface-2 font-medium text-text-muted">
				<tr>
					<th class="px-3 py-2">Strike</th>
					<th class="px-3 py-2">Type</th>
					<th class="px-3 py-2">Expiry</th>
					<th class="px-3 py-2 text-right">Mark</th>
					<th class="px-3 py-2 text-right">Bid / Ask</th>
					<th class="px-3 py-2 text-right">IV</th>
					<th class="px-3 py-2 text-right">Delta</th>
					<th class="px-3 py-2 text-right">Gamma</th>
					<th class="px-3 py-2 text-right">Theta</th>
					<th class="px-3 py-2 text-right">Vega</th>
					<th class="px-3 py-2 text-right">OI</th>
					<th class="px-3 py-2 text-right">Volume</th>
					<th class="px-3 py-2 text-right">GEX</th>
				</tr>
			</thead>
			<tbody class="divide-y divide-border/60">
				{#if sortedContracts.length === 0}
					<tr>
						<td colspan="13" class="px-3 py-8 text-center text-text-muted">
							No option contracts match selected filters.
						</td>
					</tr>
				{:else}
					{#each sortedContracts as contract (contract.contract_symbol || `${contract.strike}-${contract.option_type}-${contract.expiration_date}`)}
						{@const isCall = getType(contract) === 'call'}
						{@const itm = isItm(contract, underlyingPrice)}
						<tr
							class="transition-colors hover:bg-surface-2/60 {itm && isCall ? 'bg-green/5' : itm && !isCall ? 'bg-red/5' : ''}"
						>
							<td class="px-3 py-2 font-semibold text-text">
								${contract.strike}
							</td>
							<td class="px-3 py-2">
								<span
									class="inline-block rounded px-1.5 py-0.5 text-[10px] font-bold uppercase {isCall ? 'bg-green/15 text-green' : 'bg-red/15 text-red'}"
								>
									{contract.option_type}
								</span>
							</td>
							<td class="px-3 py-2 text-text-muted">{contract.expiration_date}</td>
							<td class="px-3 py-2 text-right font-medium text-text">
								{formatCurrency(contract.mark_price)}
							</td>
							<td class="px-3 py-2 text-right text-text-muted">
								{formatCurrency(contract.bid)} / {formatCurrency(contract.ask)}
							</td>
							<td class="px-3 py-2 text-right text-text">{formatIv(contract.implied_volatility)}</td>
							<td class="px-3 py-2 text-right text-text">{formatGreek(contract.delta)}</td>
							<td class="px-3 py-2 text-right text-text">{formatGreek(contract.gamma)}</td>
							<td class="px-3 py-2 text-right text-text">{formatGreek(contract.theta)}</td>
							<td class="px-3 py-2 text-right text-text">{formatGreek(contract.vega)}</td>
							<td class="px-3 py-2 text-right text-text-muted">{formatCompact(contract.open_interest)}</td>
							<td class="px-3 py-2 text-right text-text-muted">{formatCompact(contract.volume)}</td>
							<td
								class="px-3 py-2 text-right font-medium"
								class:text-green={contract.gex > 0}
								class:text-red={contract.gex < 0}
								class:text-text-muted={contract.gex === 0}
							>
								{formatCompact(contract.gex)}
							</td>
						</tr>
					{/each}
				{/if}
			</tbody>
		</table>
	</div>
</div>
