<script lang="ts">
	import { InputService, type RunnableType, type Job } from '$lib/gen/index.js'
	import { workspaceStore } from '$lib/stores.js'
	import { sendUserToast } from '$lib/utils.js'
	import JobSchemaPicker from '$lib/components/schema/JobSchemaPicker.svelte'
	import RunningJobSchemaPicker from '$lib/components/schema/RunningJobSchemaPicker.svelte'
	import { createEventDispatcher, onDestroy } from 'svelte'
	import JobLoader from './runs/JobLoader.svelte'
	import { DataTable } from '$lib/components/table'
	import { clickOutside } from '$lib/utils'
	import InfiniteList from './InfiniteList.svelte'

	export let scriptHash: string | null = null
	export let scriptPath: string | null = null
	export let flowPath: string | null = null

	const dispatch = createEventDispatcher()

	let jobs: Job[] = []
	let loading: boolean = false
	let hasMoreCurrentRuns = false
	let page = 1
	let infiniteList: InfiniteList | undefined = undefined
	let loadInputsPageFn: ((page: number, perPage: number) => Promise<any>) | undefined = undefined

	let runnableType: RunnableType | undefined = undefined
	$: runnableType = scriptHash
		? 'ScriptHash'
		: scriptPath
		? 'ScriptPath'
		: flowPath
		? 'FlowPath'
		: undefined

	let runnableId: string | undefined = undefined
	function initLoadInputs() {
		runnableId = scriptHash || scriptPath || flowPath || undefined

		loadInputsPageFn = async (page: number, perPage: number) => {
			const inputs = await InputService.getInputHistory({
				workspace: $workspaceStore!,
				runnableId,
				runnableType,
				page,
				perPage,
				includePreview: true
			})

			const inputsWithPayload = await Promise.all(
				inputs.map(async (input) => {
					const payloadData = await loadArgsFromHistory(input.id, undefined, false)
					return {
						...input,
						payloadData
					}
				})
			)
			return inputsWithPayload
		}
		infiniteList?.setLoader(loadInputsPageFn)
	}

	function handleSelected(data: any) {
		if (selected === data.id) {
			selected = undefined
			dispatch('select', undefined)
			return
		}
		selected = data.id
		dispatch('select', data.payloadData)
	}

	let selected: string | undefined = undefined

	onDestroy(() => {
		selected = undefined
		dispatch('select', undefined)
	})

	async function getPropPickerElements(): Promise<HTMLElement[]> {
		return Array.from(
			document.querySelectorAll('[data-schema-picker], [data-schema-picker] *')
		) as HTMLElement[]
	}

	async function loadArgsFromHistory(
		id: string | undefined,
		input: boolean | undefined,
		allowLarge: boolean
	): Promise<any> {
		if (!id) return
		const payloadData = await InputService.getArgsFromHistoryOrSavedInput({
			jobOrInputId: id,
			workspace: $workspaceStore!,
			input,
			allowLarge
		})
		return payloadData
	}

	function handleKeydown(event: KeyboardEvent) {
		if (event.key === 'Escape' && selected) {
			selected = undefined
			dispatch('select', undefined)
		}
	}

	function handleError(e: { type: string; error: any }) {
		if (e.type === 'load') {
			sendUserToast(`Failed to load input history: ${e.error}`, true)
		}
	}

	let jobHovered: string | undefined = undefined

	function refresh() {
		if (infiniteList) {
			infiniteList.loadData('refresh')
		}
	}

	$: !loading && refresh()
	$: $workspaceStore && (scriptHash || scriptPath || flowPath) && infiniteList && initLoadInputs()
</script>

<svelte:window on:keydown={handleKeydown} />

{#if runnableId}
	<JobLoader
		bind:jobs
		path={runnableId}
		isSkipped={false}
		jobKindsCat="all"
		user={null}
		label={null}
		folder={null}
		concurrencyKey={null}
		tag={null}
		success="running"
		argFilter={undefined}
		bind:loading
		syncQueuedRunsCount={false}
		refreshRate={10000}
		computeMinAndMax={undefined}
		perPage={5}
	/>
{/if}

<div
	class="h-full w-full flex flex-col gap-4"
	use:clickOutside={{ capture: false, exclude: getPropPickerElements }}
	on:click_outside={() => {
		if (selected) {
			selected = undefined
			dispatch('select', undefined)
		}
	}}
>
	<div class="grow-0">
		<DataTable size="xs" bind:currentPage={page} hasMore={hasMoreCurrentRuns} tableFixed={true}>
			{#if loading && (jobs == undefined || jobs?.length == 0)}
				<div class="text-center text-tertiary text-xs py-2">Loading current runs...</div>
			{:else if jobs?.length > 0}
				<colgroup>
					<col class="w-8" />
					<col class="w-20" />
					<col />
				</colgroup>

				<tbody class="w-full overflow-y-auto">
					{#each jobs as job (job.id)}
						<RunningJobSchemaPicker
							{job}
							selected={selected === job.id}
							hovering={jobHovered === job.id}
							on:select={(e) => handleSelected(e.detail)}
						/>
					{/each}
					{#if jobs?.length == 5}
						<div class="text-left text-tertiary text-xs"
							>... there may be more runs not displayed here as the limit is 5</div
						>
					{/if}
				</tbody>
			{:else}
				<div class="text-center text-tertiary text-xs py-2">No job currently running</div>
			{/if}
		</DataTable>
	</div>

	<div class="min-h-0 grow">
		<InfiniteList
			bind:this={infiniteList}
			selectedItemId={selected}
			on:error={(e) => handleError(e.detail)}
			on:select={(e) => handleSelected(e.detail)}
		>
			<svelte:fragment slot="columns">
				<colgroup>
					<col class="w-8" />
					<col class="w-20" />
					<col />
				</colgroup>
			</svelte:fragment>
			<svelte:fragment let:item let:hover>
				<JobSchemaPicker
					job={item}
					selected={selected === item.id}
					hovering={hover}
					payloadData={item.payloadData}
				/>
			</svelte:fragment>
			<svelte:fragment slot="empty">
				<div class="text-center text-tertiary text-xs py-2">No previous Runs</div>
			</svelte:fragment>
		</InfiniteList>
	</div>
</div>
