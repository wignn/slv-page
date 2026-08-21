<script lang="ts">
	import { marketStore } from '$lib/stores/websocket.svelte';
	import type { PriceData } from '$lib/types';

	import { getLocalLogo } from '$lib/logo';

	interface Props {
		selected: string;
		onselect: (symbol: string) => void;
	}
	let { selected, onselect }: Props = $props();

	let allPrices: PriceData[] = $derived(marketStore.prices);
	const primarySymbols = [
		'IHSG',
		'SPX',
		'XAUUSD',
		'WTI',
		'BTCUSDT',
		'DXY',
		'ETHUSDT',
		'EURUSD',
		'GBPUSD',
		'USDJPY',
		'BBCA',
		'BBRI',
		'BMRI',
		'AAPL',
		'NVDA'
	];
	let livePrices: PriceData[] = $derived.by(() => {
		const bySymbol = new Map(allPrices.map((price) => [price.symbol.toUpperCase(), price]));
		return primarySymbols.map((symbol) => bySymbol.get(symbol) ?? placeholderPrice(symbol));
	});

	function placeholderPrice(symbol: string): PriceData {
		const assetType = symbol.endsWith('USDT')
			? 'crypto'
			: symbol === 'SPX' || symbol === 'DXY'
				? 'index'
				: symbol === 'XAUUSD' || symbol === 'WTI'
					? 'commodity'
					: symbol === 'IHSG'
						? 'index'
						: ['BBCA', 'BBRI', 'BMRI'].includes(symbol)
							? 'stock'
							: 'forex';
		return {
			symbol,
			price: 0,
			bid: null,
			ask: null,
			volume: null,
			source: 'placeholder',
			asset_type: assetType,
			received_at: null,
			direction: 'none',
			prev_price: 0,
			updated_at: 0
		};
	}

	function getSymbolDetails(itemOrSymbol: PriceData | string) {
		const symbol = typeof itemOrSymbol === 'string' ? itemOrSymbol : itemOrSymbol.symbol;
		const category =
			typeof itemOrSymbol === 'string'
				? getAssetCategory({ symbol, asset_type: '', price: 0 } as PriceData)
				: getAssetCategory(itemOrSymbol);
		const sym = symbol.toUpperCase();
		let name = sym;
		let badge = sym.substring(0, 4);
		let badgeColor = 'bg-accent/10 text-accent border border-accent/20';
		let unit =
			category === 'forex'
				? 'RATE'
				: category === 'stocks'
					? 'EQTY'
					: category === 'indices'
						? 'IDX'
						: 'USD';
		let format = (val: number) => formatPrice(val, category, sym);
		let logo = { type: 'svg', url: '' };
		let displaySymbol = sym;

		if (sym === 'BTCUSDT') {
			name = 'Bitcoin';
			badge = 'BTC';
			badgeColor = 'bg-[#F7931A]/10 text-[#F7931A] border border-[#F7931A]/20';
			unit = 'USD';
			format = (val: number) =>
				val.toLocaleString(undefined, { minimumFractionDigits: 0, maximumFractionDigits: 0 });
			logo = { type: 'img', url: 'https://assets.coincap.io/assets/icons/btc@2x.png' };
		} else if (sym === 'ETHUSDT') {
			name = 'Ethereum';
			badge = 'ETH';
			badgeColor = 'bg-[#627EEA]/10 text-[#627EEA] border border-[#627EEA]/20';
			unit = 'USD';
			format = (val: number) =>
				val.toLocaleString(undefined, { minimumFractionDigits: 0, maximumFractionDigits: 0 });
			logo = { type: 'img', url: 'https://assets.coincap.io/assets/icons/eth@2x.png' };
		} else if (sym === 'SOLUSDT') {
			name = 'Solana';
			badge = 'SOL';
			badgeColor = 'bg-[#14F195]/10 text-[#14F195] border border-[#14F195]/20';
			unit = 'USD';
			format = (val: number) => val.toFixed(2);
			logo = { type: 'img', url: 'https://assets.coincap.io/assets/icons/sol@2x.png' };
		} else if (sym === 'BNBUSDT') {
			name = 'BNB';
			badge = 'BNB';
			badgeColor = 'bg-[#F3BA2F]/10 text-[#F3BA2F] border border-[#F3BA2F]/20';
			unit = 'USD';
			format = (val: number) => val.toFixed(1);
			logo = { type: 'img', url: 'https://assets.coincap.io/assets/icons/bnb@2x.png' };
		} else if (sym === 'PAXGUSDT') {
			name = 'PAX Gold';
			badge = 'PAXG';
			displaySymbol = 'PAXG';
			badgeColor = 'bg-[#D4AF37]/10 text-[#D4AF37] border border-[#D4AF37]/20';
			unit = 'USD';
			format = (val: number) =>
				val.toLocaleString(undefined, { minimumFractionDigits: 1, maximumFractionDigits: 1 });
			logo = { type: 'svg', url: '' };
		} else if (sym === 'EURUSD') {
			name = 'Euro / USD';
			badge = 'EUR';
			badgeColor = 'bg-[#003399]/10 text-[#003399] border border-[#003399]/20';
			unit = 'RATE';
			format = (val: number) => val.toFixed(5);
			logo = { type: 'img', url: 'https://flagcdn.com/w80/eu.png' };
		} else if (sym === 'GBPUSD') {
			name = 'GBP / USD';
			badge = 'GBP';
			badgeColor = 'bg-[#C8102E]/10 text-[#C8102E] border border-[#C8102E]/20';
			unit = 'RATE';
			format = (val: number) => val.toFixed(5);
			logo = { type: 'img', url: 'https://flagcdn.com/w80/gb.png' };
		} else if (sym === 'USDJPY') {
			name = 'USD / Yen';
			badge = 'JPY';
			badgeColor = 'bg-[#BC002D]/10 text-[#BC002D] border border-[#BC002D]/20';
			unit = 'JPY';
			format = (val: number) => val.toFixed(3);
			logo = { type: 'img', url: 'https://flagcdn.com/w80/jp.png' };
		} else if (sym === 'XAUUSD') {
			name = 'Gold / USD';
			badge = 'GOLD';
			badgeColor = 'bg-[#FFD700]/10 text-[#FFD700] border border-[#FFD700]/20';
			unit = 'USD';
			format = (val: number) =>
				val.toLocaleString(undefined, { minimumFractionDigits: 1, maximumFractionDigits: 1 });
		} else if (sym === 'SPX') {
			name = 'S&P 500 Index';
			badge = 'SPX';
			badgeColor = 'bg-blue-600/10 text-blue-600 border border-blue-600/20';
			unit = 'USD';
			format = (val: number) =>
				val.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 });
		} else if (sym === 'IHSG') {
			name = 'Jakarta Composite / IHSG';
			badge = 'IHSG';
			badgeColor = 'bg-red-600/10 text-red-500 border border-red-600/20';
			unit = 'INDEX';
			format = (val: number) =>
				val.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 });
		} else if (sym === 'DXY') {
			name = 'US Dollar Index';
			badge = 'DXY';
			badgeColor = 'bg-emerald-600/10 text-emerald-600 border border-[#10B981]/20';
			unit = 'RATE';
			format = (val: number) => val.toFixed(3);
		} else if (sym === 'WTI') {
			name = 'WTI Crude Oil';
			badge = 'WTI';
			badgeColor = 'bg-orange-500/10 text-orange-500 border border-orange-500/20';
			unit = 'USD';
			format = (val: number) => val.toFixed(2);
		} else if (category === 'stocks') {
			name = `${sym} Equity`;
			badge = sym.slice(0, 4);
			badgeColor = 'bg-cyan-500/10 text-cyan-400 border border-cyan-500/20';
			unit = 'USD';
		} else if (category === 'indices') {
			name = `${sym} Index`;
			badge = sym.slice(0, 5);
			badgeColor = 'bg-violet-500/10 text-violet-400 border border-violet-500/20';
			unit = 'INDEX';
		} else if (category === 'forex') {
			name = sym.length === 6 ? `${sym.slice(0, 3)} / ${sym.slice(3)}` : sym;
			badge = sym.slice(0, 3);
			badgeColor = 'bg-emerald-500/10 text-emerald-400 border border-emerald-500/20';
			unit = 'RATE';
		}

		// Prioritize local logo over any fallback URL defined above
		const localLogoUrl = getLocalLogo(sym);
		if (localLogoUrl) {
			logo = { type: 'img', url: localLogoUrl };
		}

		return { name, badge, badgeColor, unit, format, logo, displaySymbol };
	}

	function getAssetCategory(item: PriceData): string {
		const assetType = (item.asset_type ?? '').toLowerCase();
		if (['stock', 'stocks', 'equity', 'saham'].includes(assetType)) return 'stocks';
		if (['forex', 'fx', 'currency'].includes(assetType)) return 'forex';
		if (['index', 'indices', 'global_index'].includes(assetType)) return 'indices';
		if (['crypto', 'cryptocurrency'].includes(assetType)) return 'crypto';
		if (['commodity', 'commodities', 'metal', 'energy'].includes(assetType)) return 'commodities';

		const sym = item.symbol.toUpperCase();
		if (sym.endsWith('USDT')) return 'crypto';
		if (sym === 'XAUUSD' || sym.startsWith('XAU')) return 'commodities';
		if (/^[A-Z]{6}$/.test(sym)) return 'forex';
		return 'stocks';
	}

	function formatPrice(val: number, category: string, symbol: string): string {
		if (category === 'crypto')
			return val.toLocaleString(undefined, { minimumFractionDigits: 0, maximumFractionDigits: 2 });
		if (category === 'stocks' || category === 'indices')
			return val.toLocaleString(undefined, { minimumFractionDigits: 0, maximumFractionDigits: 2 });
		if (category === 'forex') {
			if (symbol.includes('JPY')) return val.toFixed(3);
			if (symbol.includes('IDR'))
				return val.toLocaleString(undefined, { maximumFractionDigits: 2 });
			return val.toFixed(5);
		}
		return val.toLocaleString(undefined, { minimumFractionDigits: 1, maximumFractionDigits: 2 });
	}

	let flashMap = $state<Map<string, 'up' | 'down'>>(new Map());
	const activeTimeouts = new Map<string, ReturnType<typeof setTimeout>>();
	let initialPrices = $state<Map<string, number>>(new Map());

	$effect(() => {
		for (const p of allPrices) {
			if (!initialPrices.has(p.symbol) && p.price > 0) {
				// Base price for percentage calculation
				initialPrices.set(p.symbol, p.price);
			}
		}
	});

	type FreshnessState = 'live' | 'stale' | 'closed' | 'unknown';

	function hasValidPrice(p: PriceData): boolean {
		return p.source !== 'placeholder' && p.price > 0;
	}

	function priceTimestamp(p: PriceData): number {
		if (p.received_at) {
			const parsed = Date.parse(p.received_at);
			if (!Number.isNaN(parsed)) return parsed;
		}
		return p.updated_at;
	}

	function getFreshness(p: PriceData): { state: FreshnessState; label: string; className: string } {
		if (!hasValidPrice(p))
			return {
				state: 'unknown',
				label: 'NO DATA',
				className: 'bg-surface-2 text-text-dim border-border'
			};
		const ts = priceTimestamp(p);
		if (!ts)
			return {
				state: 'unknown',
				label: 'NO DATA',
				className: 'bg-surface-2 text-text-dim border-border'
			};

		const session = p.session;
		const ageMs = Date.now() - ts;
		const isCrypto = p.symbol.toUpperCase().endsWith('USDT') || p.asset_type === 'crypto';
		const freshMs = isCrypto ? 15 * 60_000 : 5 * 60_000;
		if (session?.is_open && ageMs <= freshMs)
			return { state: 'live', label: 'LIVE', className: 'bg-green/10 text-green border-green/20' };
		if (session && !session.is_open)
			return {
				state: 'closed',
				label: session.state === 'break' ? 'BREAK' : 'CLOSED',
				className: 'bg-surface-2 text-text-dim border-border'
			};
		if (ageMs <= freshMs)
			return { state: 'live', label: 'LIVE', className: 'bg-green/10 text-green border-green/20' };
		return {
			state: 'stale',
			label: isCrypto ? 'FEED LAG' : 'STALE',
			className: 'bg-yellow-500/10 text-yellow-600 border-yellow-500/20'
		};
	}

	function getPercentChange(p: PriceData): { value: number; string: string } {
		if (!hasValidPrice(p)) return { value: 0, string: '--' };
		const base = initialPrices.get(p.symbol) ?? p.price;
		if (base === 0) return { value: 0, string: '0.00%' };
		const pct = ((p.price - base) / base) * 100;
		const sign = pct >= 0 ? '+' : '';
		return {
			value: pct,
			string: `${sign}${pct.toFixed(2)}%`
		};
	}

	$effect(() => {
		const now = Date.now();
		for (const p of allPrices) {
			if (p.direction !== 'none' && now - p.updated_at < 1000) {
				if (activeTimeouts.has(p.symbol) && flashMap.get(p.symbol) === p.direction) {
					continue;
				}

				if (activeTimeouts.has(p.symbol)) {
					clearTimeout(activeTimeouts.get(p.symbol));
				}

				flashMap.set(p.symbol, p.direction);

				const timeout = setTimeout(() => {
					flashMap.delete(p.symbol);
					activeTimeouts.delete(p.symbol);
				}, 600);

				activeTimeouts.set(p.symbol, timeout);
			}
		}

		return () => {
			for (const timeout of activeTimeouts.values()) {
				clearTimeout(timeout);
			}
			activeTimeouts.clear();
		};
	});
</script>

<div
	class="flex h-full flex-col overflow-hidden rounded-lg border border-border bg-surface shadow-sm"
>
	<div
		class="flex h-11 shrink-0 items-center justify-between border-b border-border bg-surface px-3"
	>
		<div>
			<h2 class="text-xs font-black tracking-tight text-text">Live Instruments</h2>
			<p class="font-mono text-[9px] font-bold text-text-dim uppercase">Watchlist</p>
		</div>
		<div class="flex items-center gap-1.5 text-[10px] text-text-dim">
			<span
				class="inline-block h-1.5 w-1.5 rounded-full {allPrices.length > 0
					? 'animate-pulse bg-green'
					: 'bg-red'}"
			></span>
			<span class="font-mono font-bold"
				>{livePrices.filter(hasValidPrice).length}/{primarySymbols.length}</span
			>
		</div>
	</div>

	<div class="divide-y divide-border bg-surface">
		{#each livePrices as p (p.symbol)}
			{@const details = getSymbolDetails(p)}
			{@const pct = getPercentChange(p)}
			{@const flash = flashMap.get(p.symbol)}
			{@const isSelected = selected === p.symbol}
			{@const validPrice = hasValidPrice(p)}
			{@const freshness = getFreshness(p)}

			<button
				onclick={() => onselect(p.symbol)}
				class="group relative flex h-14 w-full cursor-pointer items-center gap-2 overflow-hidden px-3 text-left transition-all duration-150 focus:outline-none
					{isSelected
					? 'bg-accent/7 shadow-[inset_2px_0_0_var(--color-accent)] dark:bg-accent/10'
					: 'bg-surface hover:bg-surface-2/45'}
					{flash === 'up' ? 'flash-green' : flash === 'down' ? 'flash-red' : ''}"
			>
				<div class="flex w-8 shrink-0 items-center justify-center">
					{#if details.logo.type === 'img'}
						<div
							class="flex h-5.5 w-5.5 shrink-0 items-center justify-center overflow-hidden rounded-full border border-border bg-surface-2"
						>
							<img
								src={details.logo.url}
								alt={details.badge}
								class="h-full w-full rounded-full object-cover"
							/>
						</div>
					{:else}
						<div
							class="flex h-5.5 w-5.5 shrink-0 items-center justify-center rounded-full border border-accent/20 bg-accent/10"
						>
							{#if p.symbol.toUpperCase() === 'XAUUSD'}
								<svg
									class="h-3.5 w-3.5"
									viewBox="0 0 64 64"
									fill="none"
									xmlns="http://www.w3.org/2000/svg"
								>
									<path
										d="M16 28 L48 28 L40 18 L24 18 Z"
										fill="#FFD700"
										stroke="#DAA520"
										stroke-width="2"
									/>
									<path
										d="M8 46 L36 46 L30 36 L14 36 Z"
										fill="#FFD700"
										stroke="#DAA520"
										stroke-width="2"
									/>
									<path
										d="M28 46 L56 46 L50 36 L34 36 Z"
										fill="#FFD700"
										stroke="#DAA520"
										stroke-width="2"
									/>
								</svg>
							{:else if p.symbol.toUpperCase() === 'SPX'}
								<svg
									class="h-3.5 w-3.5 text-blue-600"
									viewBox="0 0 24 24"
									fill="none"
									stroke="currentColor"
									stroke-width="2.5"
									stroke-linecap="round"
									stroke-linejoin="round"
								>
									<polyline points="22 7 13.5 15.5 8.5 10.5 2 17"></polyline>
									<polyline points="16 7 22 7 22 13"></polyline>
								</svg>
							{:else}
								<svg
									class="h-3.5 w-3.5 text-emerald-600"
									viewBox="0 0 24 24"
									fill="none"
									stroke="currentColor"
									stroke-width="2.5"
									stroke-linecap="round"
									stroke-linejoin="round"
								>
									<polyline points="22 7 13.5 15.5 8.5 10.5 2 17"></polyline>
									<polyline points="16 7 22 7 22 13"></polyline>
								</svg>
							{/if}
						</div>
					{/if}
				</div>

				<div class="min-w-0 flex-1">
					<div class="flex items-center gap-1.5">
						<span class="font-mono text-xs font-black tracking-tight text-text"
							>{details.displaySymbol}</span
						>
						<span
							class="rounded border px-1 py-0.5 font-mono text-[7px] font-bold {freshness.className}"
							>{freshness.label}</span
						>
					</div>
					<p class="truncate text-[10px] font-semibold text-text-dim">{details.name}</p>
				</div>

				<div class="shrink-0 text-right">
					<div class="font-mono text-sm font-black text-text">
						{validPrice ? details.format(p.price) : '--'}
					</div>
					<div
						class="flex items-center justify-end gap-1 font-mono text-[10px] font-bold {pct.value >=
						0
							? 'text-green'
							: 'text-red'}"
					>
						<span>{validPrice ? (pct.value >= 0 ? '▲' : '▼') : '■'}</span>
						<span>{pct.string}</span>
					</div>
				</div>
			</button>
		{/each}
	</div>
</div>
