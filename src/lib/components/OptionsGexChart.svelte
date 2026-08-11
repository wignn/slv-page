<script lang="ts">
	import type { OptionsGexItem, OptionsContract } from '$lib/types';

	interface Props {
		gexItems?: OptionsGexItem[];
		gex?: OptionsGexItem[];
		items?: OptionsGexItem[];
		contracts?: OptionsContract[];
		underlyingPrice?: number;
		height?: number;
	}

	let { gexItems, gex, items, contracts = [], underlyingPrice, height = 320 }: Props = $props();

	let rawItems = $derived.by(() => {
		const direct = gexItems ?? gex ?? items ?? [];
		if (direct.length > 0) return direct;

		if (contracts.length > 0) {
			const map = new Map<number, { strike: number; call_gex: number; put_gex: number; total_gex: number }>();
			for (const c of contracts) {
				const existing = map.get(c.strike) ?? { strike: c.strike, call_gex: 0, put_gex: 0, total_gex: 0 };
				const contractGex = c.gex ?? (c.gamma * c.open_interest * 100 * (underlyingPrice ?? 1));
				if (c.option_type === 'call') {
					existing.call_gex += Math.abs(contractGex);
				} else {
					existing.put_gex += -Math.abs(contractGex);
				}
				existing.total_gex = existing.call_gex + existing.put_gex;
				map.set(c.strike, existing);
			}
			return Array.from(map.values());
		}

		return [];
	});

	let sortedItems = $derived([...rawItems].sort((a, b) => a.strike - b.strike));

	let maxGex = $derived(
		Math.max(
			1,
			...sortedItems.map((item) =>
				Math.max(Math.abs(item.call_gex), Math.abs(item.put_gex), Math.abs(item.total_gex))
			)
		)
	);

	let totalCallGex = $derived(sortedItems.reduce((acc, curr) => acc + Math.abs(curr.call_gex), 0));
	let totalPutGex = $derived(sortedItems.reduce((acc, curr) => acc + -Math.abs(curr.put_gex), 0));
	let netGex = $derived(totalCallGex + totalPutGex);

	function formatGex(val: number): string {
		const prefix = val > 0 ? '+' : '';
		const formatted = new Intl.NumberFormat('en-US', {
			notation: 'compact',
			compactDisplay: 'short',
			maximumFractionDigits: 1
		}).format(val);
		return `${prefix}$${formatted.replace('$', '')}`;
	}

	function formatStrike(val: number): string {
		return val % 1 === 0 ? val.toFixed(0) : val.toFixed(1);
	}
</script>

<div class="flex flex-col rounded-lg border border-border bg-surface p-4 shadow-xs">
	<!-- Header -->
	<div class="flex flex-wrap items-center justify-between gap-2 border-b border-border pb-3">
		<div>
			<h3 class="text-sm font-semibold text-text">Gamma Exposure (GEX) by Strike</h3>
			<p class="text-xs text-text-muted">Strike distribution for calls vs puts</p>
		</div>

		<!-- Summary Badges & Legend -->
		<div class="flex flex-wrap items-center gap-3 text-xs">
			<div class="flex items-center gap-1.5">
				<span class="h-2.5 w-2.5 rounded-xs bg-green"></span>
				<span class="text-text-muted">Call GEX:</span>
				<span class="font-medium text-green">{formatGex(totalCallGex)}</span>
			</div>
			<div class="flex items-center gap-1.5">
				<span class="h-2.5 w-2.5 rounded-xs bg-red"></span>
				<span class="text-text-muted">Put GEX:</span>
				<span class="font-medium text-red">{formatGex(totalPutGex)}</span>
			</div>
			<div class="flex items-center gap-1.5 border-l border-border pl-3">
				<span class="text-text-muted">Net:</span>
				<span
					class="font-semibold"
					class:text-green={netGex > 0}
					class:text-red={netGex < 0}
					class:text-text={netGex === 0}
				>
					{formatGex(netGex)}
				</span>
			</div>
		</div>
	</div>

	<!-- Chart Area -->
	{#if sortedItems.length === 0}
		<div
			class="flex items-center justify-center text-sm text-text-muted"
			style="height: {height}px"
		>
			No GEX data available
		</div>
	{:else}
		<div class="relative mt-4 flex flex-col gap-1 overflow-x-auto" style="height: {height}px">
			<div class="flex h-full min-w-[600px] flex-1 items-end gap-1 pb-6 pt-4">
				{#each sortedItems as item (item.strike)}
					{@const callHeightPct = (Math.abs(item.call_gex) / maxGex) * 100}
					{@const putHeightPct = (Math.abs(item.put_gex) / maxGex) * 100}
					{@const isAtm =
						underlyingPrice !== undefined &&
						Math.abs(item.strike - underlyingPrice) ===
							Math.min(...sortedItems.map((i) => Math.abs(i.strike - underlyingPrice)))}

					<div class="group relative flex flex-1 flex-col items-center justify-end h-full">
						<!-- Hover Tooltip -->
						<div
							class="pointer-events-none absolute bottom-full mb-2 hidden z-10 rounded-md border border-border bg-surface-2 p-2 text-xs shadow-md group-hover:block"
						>
							<div class="font-bold text-text">Strike: ${formatStrike(item.strike)}</div>
							<div class="text-green">Call GEX: {formatGex(item.call_gex)}</div>
							<div class="text-red">Put GEX: {formatGex(item.put_gex)}</div>
							<div class="border-t border-border mt-1 pt-1 font-medium text-text">
								Total: {formatGex(item.total_gex)}
							</div>
						</div>

						<!-- Underlying ATM marker indicator -->
						{#if isAtm}
							<div
								class="absolute -top-3 left-1/2 -translate-x-1/2 rounded bg-accent px-1 text-[10px] font-bold text-surface whitespace-nowrap"
							>
								ATM ${underlyingPrice}
							</div>
						{/if}

						<!-- Bars Container -->
						<div class="flex w-full items-end justify-center gap-0.5" style="height: 80%">
							<!-- Call Bar -->
							<div
								class="w-1/2 rounded-t-xs bg-green/80 transition-all group-hover:bg-green"
								style="height: {callHeightPct}%"
							></div>
							<!-- Put Bar -->
							<div
								class="w-1/2 rounded-t-xs bg-red/80 transition-all group-hover:bg-red"
								style="height: {putHeightPct}%"
							></div>
						</div>

						<!-- X-Axis Strike Label -->
						<div
							class="mt-2 text-[10px] font-medium text-text-muted group-hover:text-text"
							class:font-bold={isAtm}
							class:text-accent={isAtm}
						>
							{formatStrike(item.strike)}
						</div>
					</div>
				{/each}
			</div>
		</div>
	{/if}
</div>
