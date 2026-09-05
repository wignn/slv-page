<script lang="ts">
	import type { OptionsSnapshot } from '$lib/types';

	interface Props {
		snapshot?: OptionsSnapshot | null;
		data?: OptionsSnapshot | null;
	}

	let { snapshot, data }: Props = $props();

	let currentSnapshot = $derived(snapshot ?? data ?? null);

	function formatCurrency(val: number | null | undefined): string {
		if (val === null || val === undefined || isNaN(val)) return '$0.00';
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

	function formatGex(val: number | null | undefined): string {
		if (val === null || val === undefined || isNaN(val)) return '$0.00';
		const prefix = val > 0 ? '+' : '';
		const formatted = new Intl.NumberFormat('en-US', {
			notation: 'compact',
			compactDisplay: 'short',
			maximumFractionDigits: 2
		}).format(val);
		return `${prefix}$${formatted.replace('$', '')}`;
	}

	function formatPercent(val: number | null | undefined): string {
		if (val === null || val === undefined || isNaN(val)) return 'N/A';
		const pct = val <= 1 ? val * 100 : val;
		return `${pct.toFixed(1)}%`;
	}
</script>

<div class="grid grid-cols-2 gap-3 sm:grid-cols-3 lg:grid-cols-6">
	<!-- Underlying Price -->
	<div class="flex flex-col rounded-lg border border-border bg-surface p-3.5 shadow-sm">
		<span class="text-xs font-medium tracking-wide text-text-muted">Underlying price</span>
		<div class="mt-2 text-lg font-bold tracking-tight text-text">
			{formatCurrency(currentSnapshot?.underlying_price)}
		</div>
		<div class="mt-1 text-xs text-text-dim">
			{currentSnapshot?.symbol ?? 'N/A'}
		</div>
	</div>

	<!-- Put / Call Ratio -->
	<div class="flex flex-col rounded-lg border border-border bg-surface p-3.5 shadow-sm">
		<span class="text-xs font-medium tracking-wide text-text-muted">Put / call ratio</span>
		<div class="mt-2 text-lg font-bold tracking-tight text-text">
			{currentSnapshot?.put_call_ratio !== undefined && currentSnapshot?.put_call_ratio !== null
				? currentSnapshot.put_call_ratio.toFixed(2)
				: '0.00'}
		</div>
		<div class="mt-1 text-xs text-text-dim">
			{#if (currentSnapshot?.put_call_ratio ?? 0) > 1}
				Bearish tilt
			{:else if (currentSnapshot?.put_call_ratio ?? 0) > 0}
				Bullish tilt
			{:else}
				Neutral
			{/if}
		</div>
	</div>

	<!-- Max Pain Strike -->
	<div class="flex flex-col rounded-lg border border-border bg-surface p-3.5 shadow-sm">
		<span class="text-xs font-medium tracking-wide text-text-muted">Max pain strike</span>
		<div class="mt-2 text-lg font-bold tracking-tight text-text">
			{formatCurrency(currentSnapshot?.max_pain_strike)}
		</div>
		<div class="mt-1 text-xs text-text-dim">Expiry concentration</div>
	</div>

	<!-- ATM IV -->
	<div class="flex flex-col rounded-lg border border-border bg-surface p-3.5 shadow-sm">
		<span class="text-xs font-medium tracking-wide text-text-muted">ATM IV</span>
		<div class="mt-2 text-lg font-bold tracking-tight text-text">
			{formatPercent(currentSnapshot?.iv_atm)}
		</div>
		<div class="mt-1 text-xs text-text-dim">Implied volatility</div>
	</div>

	<!-- Total OI & Volume -->
	<div class="flex flex-col rounded-lg border border-border bg-surface p-3.5 shadow-sm">
		<span class="text-xs font-medium tracking-wide text-text-muted">Total OI / volume</span>
		<div class="mt-2 text-lg font-bold tracking-tight text-text">
			{formatCompact(currentSnapshot?.total_open_interest)} / {formatCompact(
				currentSnapshot?.total_volume
			)}
		</div>
		<div class="mt-1 text-xs text-text-dim">Contracts</div>
	</div>

	<!-- Total Dollar GEX -->
	<div class="flex flex-col rounded-lg border border-border bg-surface p-3.5 shadow-sm">
		<span class="text-xs font-medium tracking-wide text-text-muted">Total dollar GEX</span>
		<div
			class="mt-2 text-lg font-bold tracking-tight"
			class:text-green={(currentSnapshot?.total_gex ?? 0) > 0}
			class:text-red={(currentSnapshot?.total_gex ?? 0) < 0}
			class:text-text={(currentSnapshot?.total_gex ?? 0) === 0}
		>
			{formatGex(currentSnapshot?.total_gex)}
		</div>
		<div class="mt-1 text-xs text-text-dim">Gamma exposure</div>
	</div>
</div>
