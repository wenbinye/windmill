<script lang="ts">
	import { type Job, JobService, type Flow, type RestartedFrom, type OpenFlow } from '$lib/gen'
	import { workspaceStore } from '$lib/stores'
	import { Badge, Button, Drawer, Popup } from './common'
	import { createEventDispatcher, getContext } from 'svelte'
	import type { FlowEditorContext } from './flows/types'
	import { runFlowPreview } from './flows/utils'
	import SchemaForm from './SchemaForm.svelte'
	import FlowStatusViewer from '../components/FlowStatusViewer.svelte'
	import FlowProgressBar from './flows/FlowProgressBar.svelte'
	import {
		AlertTriangle,
		ArrowRight,
		CornerDownLeft,
		Play,
		RefreshCw,
		X,
		ArrowLeftIcon
	} from 'lucide-svelte'
	import { emptyString, sendUserToast } from '$lib/utils'
	import DrawerContent from './common/drawer/DrawerContent.svelte'
	import SavedInputs from './SavedInputs.svelte'
	import { dfs } from './flows/dfs'
	import { sliceModules } from './flows/flowStateUtils'
	import CapturesInputs from '$lib/components/CapturesInputs.svelte'
	import ObjectViewer from './propertyPicker/ObjectViewer.svelte'

	export let previewMode: 'upTo' | 'whole'
	export let open: boolean

	export let jobId: string | undefined = undefined
	export let job: Job | undefined = undefined
	let selectedJobStep: string | undefined = undefined
	let branchOrIterationN: number = 0
	let restartBranchNames: [number, string][] = []

	let selectedJobStepIsTopLevel: boolean | undefined = undefined
	let selectedJobStepType: 'single' | 'forloop' | 'branchall' = 'single'

	let isRunning: boolean = false
	let jobProgressReset: () => void

	export function test() {
		runPreview($previewArgs, undefined)
	}

	const {
		selectedId,
		previewArgs,
		flowStateStore,
		flowStore,
		pathStore,
		initialPath,
		customUi,
		executionCount
	} = getContext<FlowEditorContext>('FlowEditorContext')
	const dispatch = createEventDispatcher()

	function extractFlow(previewMode: 'upTo' | 'whole'): OpenFlow {
		if (previewMode === 'whole') {
			return $flowStore
		} else {
			const flow: Flow = JSON.parse(JSON.stringify($flowStore))
			const idOrders = dfs(flow.value.modules, (x) => x.id)
			let upToIndex = idOrders.indexOf($selectedId)

			if (upToIndex != -1) {
				flow.value.modules = sliceModules(flow.value.modules, upToIndex, idOrders)
			}
			return flow
		}
	}

	let lastPreviewFlow: undefined | string = undefined
	export async function runPreview(
		args: Record<string, any>,
		restartedFrom: RestartedFrom | undefined
	) {
		try {
			lastPreviewFlow = JSON.stringify($flowStore)
			jobProgressReset()
			const newFlow = extractFlow(previewMode)
			jobId = await runFlowPreview(args, newFlow, $pathStore, restartedFrom)
			isRunning = true
		} catch (e) {
			sendUserToast('Could not run preview', true, undefined, e.toString())
			isRunning = false
			jobId = undefined
		}
	}

	function onKeyDown(event: KeyboardEvent) {
		if (open) {
			switch (event.key) {
				case 'Enter':
					if (event.ctrlKey || event.metaKey) {
						event.preventDefault()
						runPreview($previewArgs, undefined)
					}
					break
			}
		}
	}

	function onSelectedJobStepChange() {
		if (selectedJobStep !== undefined && job?.flow_status?.modules !== undefined) {
			selectedJobStepIsTopLevel =
				job?.flow_status?.modules.map((m) => m.id).indexOf(selectedJobStep) >= 0
			let moduleDefinition = job?.raw_flow?.modules.find((m) => m.id == selectedJobStep)
			if (moduleDefinition?.value.type == 'forloopflow') {
				selectedJobStepType = 'forloop'
			} else if (moduleDefinition?.value.type == 'branchall') {
				selectedJobStepType = 'branchall'
				moduleDefinition?.value.branches.forEach((branch, idx) => {
					restartBranchNames.push([
						idx,
						emptyString(branch.summary) ? `Branch #${idx}` : branch.summary!
					])
				})
			} else {
				selectedJobStepType = 'single'
			}
		}
	}

	$: if (job?.type === 'CompletedJob') {
		isRunning = false
	}

	$: selectedJobStep !== undefined && onSelectedJobStepChange()

	let inputLibraryDrawer: Drawer
	let captureLibraryDrawer: Drawer
	let renderCount: number = 0

	let selectedCapture: Record<string, any> | undefined = undefined
</script>

<svelte:window on:keydown={onKeyDown} />

<Drawer bind:this={inputLibraryDrawer}>
	<DrawerContent title="Input library {initialPath}" on:close={inputLibraryDrawer?.toggleDrawer}>
		<SavedInputs
			flowPath={initialPath}
			isValid={true}
			args={$previewArgs}
			on:selected_args={(e) => {
				$previewArgs = JSON.parse(JSON.stringify(e.detail))
				inputLibraryDrawer?.closeDrawer()
				renderCount++
			}}
		/>
	</DrawerContent>
</Drawer>

<Drawer bind:this={captureLibraryDrawer}>
	<DrawerContent
		title="Trigger captures library {initialPath}"
		on:close={captureLibraryDrawer?.toggleDrawer}
	>
		<div class="h-full flex flex-col gap-2">
			<CapturesInputs
				flowPath={$pathStore}
				headless={false}
				addButton={true}
				on:select={(e) => {
					selectedCapture = e.detail
				}}
				on:openTriggers={(e) => {
					dispatch('openTriggers', e.detail)
					captureLibraryDrawer?.closeDrawer()
				}}
			/>

			<div class="h-full overflow-hidden min-h-0 flex flex-col justify-between">
				<div class="w-full flex flex-col min-h-0 gap-2 px-2 py-2 grow">
					<div class="w-full flex flex-col">
						<Button
							color="blue"
							btnClasses="w-full"
							size="sm"
							spacingSize="xl"
							on:click={async () => {
								$previewArgs = JSON.parse(JSON.stringify(selectedCapture))
								captureLibraryDrawer?.closeDrawer()
								renderCount++
							}}
							disabled={!selectedCapture}
						>
							<ArrowLeftIcon class="w-4 h-4 mr-2" />
							Use input
						</Button>
					</div>
					<div class="w-full min-h-0 grow overflow-auto">
						{#if typeof selectedCapture == 'string' && selectedCapture == 'WINDMILL_TOO_BIG'}
							<div class="text-secondary mt-2">
								Payload too big to preview but can still be loaded</div
							>
						{:else if Object.keys(selectedCapture || {}).length > 0}
							<div class=" overflow-auto h-full p-2">
								<ObjectViewer json={selectedCapture} />
							</div>
						{:else}
							<div class="text-center text-tertiary">
								Select an Input to preview scripts arguments
							</div>
						{/if}
					</div>
				</div>
			</div>
		</div>
	</DrawerContent>
</Drawer>

<div class="flex flex-col space-y-2 h-screen bg-surface px-6 py-2 w-full" id="flow-preview-content">
	<div class="flex flex-row justify-between w-full items-center gap-x-2">
		<div class="w-8">
			<Button
				on:click={() => dispatch('close')}
				startIcon={{ icon: X }}
				iconOnly
				size="sm"
				color="light"
				btnClasses="hover:bg-surface-hover  bg-surface-secondaryw-8 h-8 rounded-full p-0"
			/>
		</div>

		{#if isRunning}
			<Button
				color="red"
				on:click={async () => {
					isRunning = false
					try {
						jobId &&
							(await JobService.cancelQueuedJob({
								workspace: $workspaceStore ?? '',
								id: jobId,
								requestBody: {}
							}))
					} catch {}
				}}
				size="sm"
				btnClasses="w-full max-w-lg"
				loading={true}
				clickableWhileLoading
			>
				Cancel
			</Button>
		{:else}
			<div class="flex flex-row gap-4">
				{#if jobId !== undefined && selectedJobStep !== undefined && selectedJobStepIsTopLevel}
					{#if selectedJobStepType == 'single'}
						<Button
							size="xs"
							color="light"
							variant="border"
							title={`Re-start this flow from step ${selectedJobStep} (included).`}
							on:click={() => {
								runPreview($previewArgs, {
									flow_job_id: jobId,
									step_id: selectedJobStep,
									branch_or_iteration_n: 0
								})
							}}
							startIcon={{ icon: Play }}
						>
							Re-start from
							<Badge baseClass="ml-1" color="indigo">
								{selectedJobStep}
							</Badge>
						</Button>
					{:else}
						<Popup floatingConfig={{ strategy: 'absolute', placement: 'bottom-start' }}>
							<svelte:fragment slot="button">
								<Button
									title={`Re-start this flow from step ${selectedJobStep} (included).`}
									variant="border"
									color="blue"
									startIcon={{ icon: RefreshCw }}
									on:click={() => {
										runPreview($previewArgs, {
											flow_job_id: jobId,
											step_id: selectedJobStep,
											branch_or_iteration_n: 0
										})
									}}
									nonCaptureEvent={true}
								>
									Re-start from
									<Badge baseClass="ml-1" color="indigo">
										{selectedJobStep}
									</Badge>
								</Button>
							</svelte:fragment>
							<label class="block text-primary">
								<div class="pb-1 text-sm text-secondary"
									>{selectedJobStepType == 'forloop' ? 'From iteration #:' : 'From branch:'}</div
								>
								<div class="flex w-full">
									{#if selectedJobStepType === 'forloop'}
										<input
											type="number"
											min="0"
											bind:value={branchOrIterationN}
											class="!w-32 grow"
											on:click|stopPropagation={() => {}}
										/>
									{:else}
										<select
											bind:value={branchOrIterationN}
											class="!w-32 grow"
											on:click|stopPropagation={() => {}}
										>
											{#each restartBranchNames as [branchIdx, branchName]}
												<option value={branchIdx}>{branchName}</option>
											{/each}
										</select>
									{/if}
									<Button
										size="xs"
										color="blue"
										buttonType="button"
										btnClasses="!p-1 !w-[34px] !ml-1"
										aria-label="Restart flow"
										on:click|once={() => {
											runPreview($previewArgs, {
												flow_job_id: jobId,
												step_id: selectedJobStep,
												branch_or_iteration_n: branchOrIterationN
											})
										}}
									>
										<ArrowRight size={18} />
									</Button>
								</div>
							</label>
						</Popup>
					{/if}
				{/if}
				<Button
					variant="contained"
					startIcon={{ icon: isRunning ? RefreshCw : Play }}
					color="dark"
					size="sm"
					btnClasses="w-full max-w-lg"
					on:click={() => runPreview($previewArgs, undefined)}
					id="flow-editor-test-flow-drawer"
					shortCut={{ Icon: CornerDownLeft }}
				>
					Test flow
				</Button>
			</div>
		{/if}
		<div class="flex gap-2">
			<Button
				btnClasses="h-full truncate"
				size="sm"
				variant="border"
				on:click={() => {
					captureLibraryDrawer?.openDrawer()
				}}>Trigger captures library</Button
			>
			{#if initialPath != ''}
				<Button
					btnClasses="h-full truncate"
					size="sm"
					variant="border"
					on:click={() => {
						inputLibraryDrawer?.openDrawer()
					}}>Past runs/Input library</Button
				>
			{/if}
		</div>
	</div>
	<div class="w-full flex flex-col gap-y-1">
		{#if lastPreviewFlow && JSON.stringify($flowStore) != lastPreviewFlow}
			<div class="pt-1">
				<div
					class="bg-orange-200 text-orange-600 border border-orange-600 p-2 flex items-center gap-2 rounded"
				>
					<AlertTriangle size={14} /> Flow changed since last preview
					<div class="flex" />
				</div>
			</div>
		{/if}
		<FlowProgressBar {job} bind:reset={jobProgressReset} />
	</div>

	<div class="overflow-y-auto grow flex flex-col pr-4">
		<div class="border-b">
			{#key renderCount}
				<SchemaForm
					noVariablePicker
					compact
					class="py-4 max-w-3xl"
					schema={$flowStore.schema}
					bind:args={$previewArgs}
				/>
			{/key}
		</div>
		<div class="pt-4 flex flex-col grow">
			{#if jobId}
				<FlowStatusViewer
					hideDownloadInGraph={customUi?.downloadLogs === false}
					wideResults
					{flowStateStore}
					{jobId}
					on:done={() => {
						console.log('done')
						$executionCount = $executionCount + 1
					}}
					on:jobsLoaded={({ detail }) => {
						job = detail
					}}
					bind:selectedJobStep
				/>
			{:else}
				<div class="italic text-tertiary h-full grow"> Flow status will be displayed here </div>
			{/if}
		</div>
	</div>
</div>
