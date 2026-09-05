<script lang="ts">
	import { RefreshCw, AlertCircle, Activity, Database, TrendingUp } from 'lucide-svelte';
	import { fetchMacroData, macroState } from '$lib/stores/macro';
	import {
		FEAR_GREED_COMPONENTS,
		YIELD_CONTEXT_TENORS,
		YIELD_CURVE_TENORS,
		type FearGreedComponent,
		type RateObservation
	} from '$lib/types';

	const TENOR_LABELS: Record<string, string> = {
		'3M': '3 month',
		'2Y': '2 year',
		'5Y': '5 year',
		'10Y': '10 year',
		'30Y': '30 year',
		'10Y_REAL': '10Y real',
		'10Y_BREAKEVEN': '10Y breakeven'
	};

	const COMPONENT_LABELS: Record<string, string> = {
		momentum: 'Momentum',
		volatility: 'Volatility',
		safe_haven: 'Safe haven',
		news_risk: 'News risk',
		positioning: 'Positioning'
	};

	const COMPONENT_WEIGHTS: Record<string, number> = {
		momentum: 0.25,
		volatility: 0.2,
		safe_haven: 0.2,
		news_risk: 0.2,
		positioning: 0.15
	};

	let state = $derived($macroState);
	let curvePoints = $derived.by(() => {
		const points = state.yieldCurve?.points ?? [];
		return YIELD_CURVE_TENORS.map((tenor) => points.find((point) => point.tenor === tenor)).filter(
			(point): point is RateObservation => point !== undefined
		);
	});
	let contextPoints = $derived.by(() => {
		const points = state.yieldCurve?.points ?? [];
		return YIELD_CONTEXT_TENORS.map((tenor) =>
			points.find((point) => point.tenor === tenor)
		).filter((point): point is RateObservation => point !== undefined);
	});
	let spreadPoints = $derived(state.yieldCurve?.spreads ?? []);
	let curveMin = $derived(
		curvePoints.length ? Math.min(...curvePoints.map((point) => point.value)) : 0
	);
	let curveMax = $derived(
		curvePoints.length ? Math.max(...curvePoints.map((point) => point.value)) : 1
	);
	let curveRange = $derived(Math.max(curveMax - curveMin, 0.5));
	let curveShape = $derived.by(() => {
		const short = curvePoints.find((point) => point.tenor === '3M')?.value;
		const long = curvePoints.find((point) => point.tenor === '10Y')?.value;
		if (short === undefined || long === undefined) return 'unknown';
		return long >= short ? 'normal' : 'inverted';
	});

	let fearRecord = $derived(state.fearGreed);
	let fearComponents = $derived(state.fearGreedComponents?.components ?? []);
	let fearScore = $derived(state.fearGreedComponents?.score ?? fearRecord?.score ?? null);
	let fearLabel = $derived(state.fearGreedComponents?.label ?? fearRecord?.label ?? 'neutral');
	let availableComponentCount = $derived.by(() => {
		const statuses = fearRecord?.source_status ?? {};
		const listed =
			fearComponents.length > 0
				? fearComponents
				: FEAR_GREED_COMPONENTS.map((name) => ({
						name,
						score: fearRecord?.components[name] ?? null,
						base_weight: COMPONENT_WEIGHTS[name] ?? 0,
						status:
							statuses[name] ?? (fearRecord?.components[name] !== undefined ? 'ok' : 'missing'),
						description: ''
					}));
		return listed.filter((component) => component.status === 'ok' && component.score !== null)
			.length;
	});
	let hasMeaningfulFearSignal = $derived(availableComponentCount > 0);
	let historyValues = $derived(state.fearGreedHistory?.data ?? []);
	let historyPath = $derived.by(() => {
		if (historyValues.length < 2) return '';
		return historyValues
			.map((item, index) => {
				const x = 8 + (index / (historyValues.length - 1)) * 184;
				const y = 58 - (item.score / 100) * 48;
				return `${index === 0 ? 'M' : 'L'} ${x.toFixed(1)} ${y.toFixed(1)}`;
			})
			.join(' ');
	});

	function pointX(index: number, count: number): number {
		return count <= 1 ? 360 : 40 + (index / (count - 1)) * 640;
	}

	function pointY(value: number): number {
		return 218 - ((value - curveMin) / curveRange) * 170;
	}

	let curvePath = $derived.by(() =>
		curvePoints
			.map(
				(point, index) =>
					`${index === 0 ? 'M' : 'L'} ${pointX(index, curvePoints.length).toFixed(1)} ${pointY(point.value).toFixed(1)}`
			)
			.join(' ')
	);

	function formatPercent(value: number | null | undefined, digits = 2): string {
		return value === null || value === undefined || !Number.isFinite(value)
			? '—'
			: `${value.toFixed(digits)}%`;
	}

	function formatSpread(value: number | null | undefined): string {
		if (value === null || value === undefined || !Number.isFinite(value)) return '—';
		return `${value > 0 ? '+' : ''}${value.toFixed(2)} pp`;
	}

	function spreadTone(value: number): string {
		return value < 0 ? 'text-red' : value > 0 ? 'text-green' : 'text-text-muted';
	}

	function formatDate(value: string | null | undefined, includeTime = false): string {
		if (!value) return 'Date unavailable';
		const parsed = new Date(value);
		if (Number.isNaN(parsed.getTime())) return 'Date unavailable';
		const options: Intl.DateTimeFormatOptions = {
			timeZone: 'UTC',
			month: 'short',
			day: 'numeric',
			year: 'numeric'
		};
		return new Intl.DateTimeFormat('en-US', {
			...options,
			...(includeTime ? { hour: 'numeric', minute: '2-digit' } : {})
		}).format(parsed);
	}

	function displayLabel(value: string): string {
		return value.replaceAll('_', ' ');
	}

	function scoreColor(score: number | null): string {
		if (score === null) return 'text-text-dim';
		if (score < 25) return 'text-red';
		if (score < 45) return 'text-red/80';
		if (score < 55) return 'text-text-muted';
		return score < 75 ? 'text-green/80' : 'text-green';
	}

	function componentRows(): FearGreedComponent[] {
		const byName = new Map(fearComponents.map((component) => [component.name, component]));
		const statuses = fearRecord?.source_status ?? {};
		return FEAR_GREED_COMPONENTS.map(
			(name) =>
				byName.get(name) ?? {
					name,
					score: fearRecord?.components[name] ?? null,
					base_weight: COMPONENT_WEIGHTS[name] ?? 0,
					status: statuses[name] ?? (fearRecord?.components[name] !== undefined ? 'ok' : 'missing'),
					description: ''
				}
		);
	}
</script>

<div class="flex flex-col gap-5">
	<div class="flex flex-col justify-between gap-3 sm:flex-row sm:items-end">
		<div>
			<div class="flex items-center gap-3">
				<h2 class="text-2xl font-black tracking-tight text-text">Macro signals</h2>
				<span
					class="rounded-md border border-accent/20 bg-accent/5 px-2.5 py-1 font-mono text-[11px] font-bold tracking-wide text-accent uppercase"
					>Rates + risk</span
				>
			</div>
			<p class="mt-1 max-w-2xl text-sm text-text-muted">
				A daily read on the US Treasury curve and ATLSD's composite market mood.
			</p>
		</div>
		<div class="flex items-center gap-3 text-xs text-text-dim">
			{#if state.loading && (state.yieldCurve || state.fearGreed)}
				<span class="flex items-center gap-1.5"
					><RefreshCw class="h-3.5 w-3.5 animate-spin" />Refreshing</span
				>
			{/if}
			<button
				class="inline-flex items-center gap-1.5 rounded-md border border-border bg-surface px-2.5 py-1.5 font-semibold text-text-muted transition-colors hover:bg-surface-2 hover:text-text disabled:cursor-not-allowed disabled:opacity-50"
				onclick={() => fetchMacroData()}
				disabled={state.loading}
			>
				<RefreshCw class="h-3.5 w-3.5" /> Refresh
			</button>
		</div>
	</div>

	{#if state.loading && !state.yieldCurve && !state.fearGreed}
		<div class="grid gap-5 xl:grid-cols-[minmax(0,1.3fr)_minmax(360px,0.7fr)]">
			{#each [1, 2] as _}
				<div class="h-[430px] animate-pulse rounded-xl border border-border/80 bg-surface"></div>
			{/each}
		</div>
	{:else}
		<div class="grid gap-5 xl:grid-cols-[minmax(0,1.3fr)_minmax(360px,0.7fr)]">
			<section class="overflow-hidden rounded-xl border border-border/80 bg-surface shadow-sm">
				<div
					class="flex flex-wrap items-start justify-between gap-3 border-b border-border/70 px-5 py-4"
				>
					<div>
						<div class="flex items-center gap-2">
							<TrendingUp class="h-4 w-4 text-blue" />
							<h3 class="text-sm font-black tracking-wide text-text uppercase">
								US Treasury yield curve
							</h3>
						</div>
						<p class="mt-1 text-xs text-text-muted">
							Nominal yields by maturity · source: {state.yieldCurve?.source ?? 'FRED'}
						</p>
					</div>
					<div class="flex items-center gap-2">
						{#if state.yieldCurve?.stale}<span
								class="rounded border border-amber/30 bg-amber/10 px-2 py-1 font-mono text-[10px] font-bold tracking-wide text-amber uppercase"
								>Stale dates</span
							>{/if}
						{#if curveShape === 'normal'}<span
								class="rounded border border-green/20 bg-green/10 px-2 py-1 font-mono text-[10px] font-bold tracking-wide text-green uppercase"
								>Normal slope</span
							>{:else if curveShape === 'inverted'}<span
								class="rounded border border-red/20 bg-red/10 px-2 py-1 font-mono text-[10px] font-bold tracking-wide text-red uppercase"
								>Inverted slope</span
							>{/if}
					</div>
				</div>

				{#if state.errors.yieldCurve && !state.yieldCurve}
					<div class="m-5 rounded-lg border border-red/20 bg-red/10 p-4 text-sm text-red">
						<div class="flex items-center gap-2 font-bold">
							<AlertCircle class="h-4 w-4" /> Yield data unavailable
						</div>
						<p class="mt-1 text-xs text-red/80">{state.errors.yieldCurve}</p>
						<button
							class="mt-3 rounded-md border border-red/30 px-3 py-1.5 text-xs font-bold hover:bg-red/10"
							onclick={() => fetchMacroData()}>Try again</button
						>
					</div>
				{:else if curvePoints.length === 0}
					<div
						class="m-5 flex min-h-[300px] flex-col items-center justify-center rounded-lg border border-dashed border-border bg-surface-2/30 p-6 text-center"
					>
						<Database class="h-7 w-7 text-text-dim" />
						<p class="mt-3 text-sm font-bold text-text-muted">No yield curve data yet</p>
						<p class="mt-1 max-w-sm text-xs text-text-dim">
							FRED has not returned a Treasury snapshot for this environment.
						</p>
					</div>
				{:else}
					<div class="px-4 pt-4 sm:px-5">
						<div class="overflow-hidden rounded-lg border border-border/70 bg-bg/30">
							<svg
								viewBox="0 0 720 260"
								class="h-[250px] w-full"
								role="img"
								aria-label="US Treasury nominal yield curve"
								><line
									x1="40"
									y1="218"
									x2="680"
									y2="218"
									stroke="currentColor"
									stroke-opacity="0.22"
								/><line
									x1="40"
									y1="48"
									x2="680"
									y2="48"
									stroke="currentColor"
									stroke-opacity="0.08"
								/><line
									x1="40"
									y1="133"
									x2="680"
									y2="133"
									stroke="currentColor"
									stroke-opacity="0.08"
								/><text x="8" y="53" fill="currentColor" opacity="0.5" font-size="10"
									>{curveMax.toFixed(1)}%</text
								><text x="8" y="138" fill="currentColor" opacity="0.5" font-size="10"
									>{(curveMin + curveRange / 2).toFixed(1)}%</text
								><text x="8" y="223" fill="currentColor" opacity="0.5" font-size="10"
									>{curveMin.toFixed(1)}%</text
								><path
									d={curvePath}
									fill="none"
									stroke="var(--color-accent)"
									stroke-width="3"
									stroke-linecap="round"
									stroke-linejoin="round"
								/>{#each curvePoints as point, index (point.tenor)}<circle
										cx={pointX(index, curvePoints.length)}
										cy={pointY(point.value)}
										r="5"
										fill="var(--color-surface)"
										stroke="var(--color-accent)"
										stroke-width="3"
										><title>{TENOR_LABELS[point.tenor]}: {formatPercent(point.value)}</title
										></circle
									><text
										x={pointX(index, curvePoints.length)}
										y="244"
										text-anchor="middle"
										fill="currentColor"
										opacity="0.65"
										font-size="10">{point.tenor}</text
									>{/each}</svg
							>
						</div>
					</div>

					<div class="grid grid-cols-2 gap-2 px-4 py-4 sm:grid-cols-5 sm:px-5">
						{#each curvePoints as point (point.tenor)}<div
								class="rounded-lg border border-border/70 bg-surface-2/40 px-3 py-2.5"
							>
								<p class="text-[10px] font-bold tracking-wide text-text-dim uppercase">
									{point.tenor}
								</p>
								<p class="mt-1 font-mono text-sm font-black text-text">
									{formatPercent(point.value)}
								</p>
								<p class="mt-0.5 text-[10px] text-text-dim">{formatDate(point.date)}</p>
							</div>{/each}
					</div>

					<div class="border-t border-border/70 px-4 py-4 sm:px-5">
						<div class="mb-3 flex items-center justify-between">
							<div>
								<p class="text-xs font-black tracking-wide text-text uppercase">Curve context</p>
								<p class="mt-0.5 text-[11px] text-text-dim">
									Real yield and inflation expectations are shown separately.
								</p>
							</div>
							<span class="font-mono text-[10px] text-text-dim"
								>{formatDate(state.yieldCurve?.date)}</span
							>
						</div>
						<div class="grid gap-2 sm:grid-cols-2">
							{#each contextPoints as point (point.tenor)}<div
									class="flex items-center justify-between rounded-lg border border-border/70 bg-surface-2/30 px-3 py-2"
								>
									<span class="text-xs font-semibold text-text-muted"
										>{TENOR_LABELS[point.tenor]}</span
									><span class="font-mono text-sm font-bold text-text"
										>{formatPercent(point.value)}</span
									>
								</div>{/each}{#if contextPoints.length === 0}<p class="text-xs text-text-dim">
									No real yield or breakeven observations available.
								</p>{/if}
						</div>
						<div class="mt-4 grid gap-2 sm:grid-cols-2" aria-label="Yield spreads">
							{#each spreadPoints as spread (spread.spread)}<div
									class="rounded-lg border border-border/70 bg-surface-2/30 px-3 py-2"
								>
									<div class="flex items-center justify-between gap-3">
										<span class="text-xs font-semibold text-text-muted">{spread.spread}</span><span
											class="font-mono text-sm font-bold {spreadTone(spread.value)}"
											>{formatSpread(spread.value)}</span
										>
									</div>
									<p class="mt-1 text-[10px] text-text-dim">
										{spread.value < 0
											? 'Inverted · long yield below short yield'
											: 'Positive spread'}
									</p>
								</div>{/each}{#if spreadPoints.length === 0}<p class="text-xs text-text-dim">
									No spread observations available.
								</p>{/if}
						</div>
					</div>
				{/if}
				{#if state.errors.yieldCurve && state.yieldCurve}<p
						class="border-t border-amber/20 bg-amber/5 px-5 py-2.5 text-xs text-amber"
					>
						{state.errors.yieldCurve}
					</p>{/if}
			</section>

			<section class="overflow-hidden rounded-xl border border-border/80 bg-surface shadow-sm">
				<div class="border-b border-border/70 px-5 py-4">
					<div class="flex items-center justify-between gap-3">
						<div class="flex items-center gap-2">
							<Activity class="h-4 w-4 text-accent" />
							<h3 class="text-sm font-black tracking-wide text-text uppercase">
								Official Fear &amp; Greed
							</h3>
						</div>
						<span
							class="rounded border border-border bg-surface-2 px-2 py-1 font-mono text-[9px] font-bold tracking-wide text-text-dim uppercase"
							>ATLSD composite</span
						>
					</div>
					<p class="mt-1 text-xs text-text-muted">
						A 0–100 score built from five market conditions.
					</p>
				</div>
				{#if state.errors.fearGreed && !fearRecord}
					<div class="m-5 rounded-lg border border-red/20 bg-red/10 p-4 text-sm text-red">
						<div class="flex items-center gap-2 font-bold">
							<AlertCircle class="h-4 w-4" /> Fear &amp; Greed unavailable
						</div>
						<p class="mt-1 text-xs text-red/80">{state.errors.fearGreed}</p>
						<button
							class="mt-3 rounded-md border border-red/30 px-3 py-1.5 text-xs font-bold hover:bg-red/10"
							onclick={() => fetchMacroData()}>Try again</button
						>
					</div>
				{:else if !fearRecord && fearScore === null}
					<div
						class="m-5 flex min-h-[300px] flex-col items-center justify-center rounded-lg border border-dashed border-border bg-surface-2/30 p-6 text-center"
					>
						<Activity class="h-7 w-7 text-text-dim" />
						<p class="mt-3 text-sm font-bold text-text-muted">Awaiting market mood data</p>
						<p class="mt-1 text-xs text-text-dim">
							The composite will appear when its source signals are ready.
						</p>
					</div>
				{:else}
					{@const score = fearScore ?? 50}
					<div class="flex flex-col items-center px-5 pt-5">
						<div class="relative h-[142px] w-[250px]">
							<svg
								viewBox="0 0 250 142"
								class="h-full w-full overflow-visible"
								role="img"
								aria-label={`Fear and Greed score ${Math.round(score)} out of 100`}
								><defs
									><linearGradient id="macro-fear-gradient" x1="0%" y1="0%" x2="100%" y2="0%"
										><stop offset="0%" stop-color="var(--color-red, #c85550)" /><stop
											offset="50%"
											stop-color="var(--color-text-dim, #8e8a81)"
										/><stop
											offset="100%"
											stop-color="var(--color-green, #2f8f64)"
										/></linearGradient
									></defs
								><path
									d="M 28 124 A 97 97 0 0 1 222 124"
									fill="none"
									stroke="var(--color-surface-2, #eeece5)"
									stroke-width="14"
									stroke-linecap="round"
								/><path
									d="M 28 124 A 97 97 0 0 1 222 124"
									fill="none"
									stroke="url(#macro-fear-gradient)"
									stroke-width="14"
									stroke-linecap="round"
								/><g
									style={`transform: rotate(${(score / 100) * 180 - 90}deg); transform-origin: 125px 124px; transition: transform 700ms cubic-bezier(0.16, 1, 0.3, 1);`}
									><path d="M 123 124 L 125 31 L 127 124 Z" fill="var(--color-text, #171716)" /></g
								><circle cx="125" cy="124" r="8" fill="var(--color-text, #171716)" /><circle
									cx="125"
									cy="124"
									r="3"
									fill="var(--color-surface, #fffefa)"
								/></svg
							>
							<div
								class="absolute inset-x-0 bottom-0 flex justify-between px-5 font-mono text-[9px] font-bold tracking-wide text-text-dim uppercase"
							>
								<span>Fear</span><span>Neutral</span><span>Greed</span>
							</div>
						</div>
						<div class="-mt-1 text-center">
							<p class={`text-2xl font-black tracking-tight uppercase ${scoreColor(score)}`}>
								{displayLabel(fearLabel)}
							</p>
							<p class="mt-1 font-mono text-sm font-bold text-text">
								{Math.round(score)}<span class="text-text-dim"> / 100</span>
							</p>
							<p class="mt-1 text-[10px] text-text-dim">
								As of {formatDate(state.fearGreedComponents?.updated_at ?? fearRecord?.date)}
							</p>
						</div>
					</div>
					<div class="mx-5 mt-4 rounded-lg border border-border/70 bg-surface-2/30 p-3">
						<div class="flex items-center justify-between gap-3">
							<p class="text-xs font-black tracking-wide text-text uppercase">Signal coverage</p>
							<span
								class={`font-mono text-xs font-bold ${hasMeaningfulFearSignal ? 'text-green' : 'text-amber'}`}
								>{availableComponentCount}/5 sources ready</span
							>
						</div>
						<div class="mt-2 h-1.5 overflow-hidden rounded-full bg-border/70">
							<div
								class="h-full rounded-full bg-accent transition-all"
								style={`width: ${(availableComponentCount / 5) * 100}%`}
							></div>
						</div>
						<p class="mt-2 text-[10px] leading-4 text-text-dim">
							{hasMeaningfulFearSignal
								? 'Missing sources are excluded and remaining weights are rebalanced.'
								: 'The neutral fallback is not a directional signal until at least one source is ready.'}
						</p>
					</div>
					<div class="mt-4 border-t border-border/70 px-5 pt-4 pb-5">
						<div class="mb-3 flex items-center justify-between">
							<p class="text-xs font-black tracking-wide text-text uppercase">Components</p>
							{#if historyValues.length > 1}<svg
									viewBox="0 0 200 66"
									class="h-12 w-[150px]"
									role="img"
									aria-label="Fear and Greed history sparkline"
									><line
										x1="8"
										y1="34"
										x2="192"
										y2="34"
										stroke="currentColor"
										stroke-opacity="0.12"
									/><path
										d={historyPath}
										fill="none"
										stroke="var(--color-accent)"
										stroke-width="2"
										stroke-linecap="round"
										stroke-linejoin="round"
									/></svg
								>{/if}
						</div>
						<div class="flex flex-col gap-3">
							{#each componentRows() as component (component.name)}<div
									class:opacity-50={component.score === null}
								>
									<div class="flex items-center justify-between gap-3 text-xs">
										<span class="font-semibold text-text-muted"
											>{COMPONENT_LABELS[component.name] ?? displayLabel(component.name)}</span
										><span class="font-mono font-bold {scoreColor(component.score)}"
											>{component.score === null ? 'N/A' : component.score.toFixed(0)}
											<span class="font-normal text-text-dim"
												>· {(component.base_weight * 100).toFixed(0)}%</span
											></span
										>
									</div>
									<div class="mt-1.5 h-1.5 overflow-hidden rounded-full bg-border/60">
										{#if component.score !== null}<div
												class="h-full rounded-full bg-accent"
												style={`width: ${component.score}%`}
											></div>{/if}
									</div>
								</div>{/each}
						</div>
					</div>
				{/if}
				{#if state.errors.fearGreed && fearRecord}<p
						class="border-t border-amber/20 bg-amber/5 px-5 py-2.5 text-xs text-amber"
					>
						{state.errors.fearGreed}
					</p>{/if}
			</section>
		</div>
	{/if}

	{#if state.lastFetchedAt}<p class="text-right font-mono text-[10px] text-text-dim">
			Updated {formatDate(state.lastFetchedAt, true)} UTC · backend data is refreshed periodically
		</p>{/if}
</div>
