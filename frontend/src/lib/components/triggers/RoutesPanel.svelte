<script lang="ts">
	import { userStore, workspaceStore } from '$lib/stores'
	import { HttpTriggerService, type HttpTrigger } from '$lib/gen'
	import Skeleton from '../common/skeleton/Skeleton.svelte'
	import RouteEditor from './RouteEditor.svelte'
	import { canWrite } from '$lib/utils'
	import Alert from '../common/alert/Alert.svelte'
	import type { TriggerContext } from '../triggers'
	import { getContext, onMount } from 'svelte'
	import Section from '$lib/components/Section.svelte'
	import TriggersEditorSection from './TriggersEditorSection.svelte'
	import Description from '../Description.svelte'

	export let isFlow: boolean
	export let path: string
	export let newItem: boolean = false
	export let isEditor: boolean = false
	export let canHavePreprocessor: boolean = false
	export let hasPreprocessor: boolean = false

	let routeEditor: RouteEditor

	$: path && loadTriggers()
	const { triggersCount, selectedTrigger, defaultValues } =
		getContext<TriggerContext>('TriggerContext')

	onMount(() => {
		if (
			defaultValues &&
			$selectedTrigger === 'routes' &&
			Object.keys($defaultValues ?? {}).length > 0
		) {
			routeEditor.openNew(isFlow, path, $defaultValues)
			defaultValues.set(undefined)
		}
	})

	let httpTriggers: (HttpTrigger & { canWrite: boolean })[] | undefined = undefined
	export async function loadTriggers() {
		try {
			httpTriggers = (
				await HttpTriggerService.listHttpTriggers({
					workspace: $workspaceStore ?? '',
					path,
					isFlow
				})
			).map((x) => {
				return { canWrite: canWrite(x.path, x.extra_perms!, $userStore), ...x }
			})
			$triggersCount = { ...($triggersCount ?? {}), http_routes_count: httpTriggers?.length }
		} catch (e) {
			console.error('impossible to load http routes', e)
		}
	}
</script>

<RouteEditor
	on:update={() => {
		loadTriggers()
	}}
	bind:this={routeEditor}
/>

<div class="flex flex-col gap-8">
	<Description link="https://www.windmill.dev/docs/core_concepts/http_routing">
		Routes expose your scripts and flows as HTTP endpoints. Each route can be configured with a
		specific HTTP method and path.
	</Description>
	<TriggersEditorSection
		on:saveTrigger={(e) => {
			routeEditor?.openNew(isFlow, path, e.detail.config)
		}}
		on:applyArgs
		on:addPreprocessor
		on:updateSchema
		on:testWithArgs
		cloudDisabled={false}
		triggerType="http"
		{isFlow}
		{path}
		{isEditor}
		{canHavePreprocessor}
		{hasPreprocessor}
		{newItem}
	/>
	{#if !newItem}
		<Section label="Routes">
			{#if !$userStore?.is_admin && !$userStore?.is_super_admin}
				<Alert title="Only workspace admins can create routes" type="warning" size="xs" />
			{/if}

			{#if httpTriggers}
				{#if httpTriggers.length == 0}
					<div class="text-xs text-secondary text-center"> No http routes </div>
				{:else}
					<div class="flex flex-col divide-y pt-2">
						{#each httpTriggers as httpTriggers (httpTriggers.path)}
							<div class="grid grid-cols-5 text-2xs items-center py-2">
								<div class="col-span-2 truncate">{httpTriggers.path}</div>
								<div class="col-span-2 truncate">
									{httpTriggers.http_method.toUpperCase()} /{httpTriggers.route_path}
								</div>
								<div class="flex justify-end">
									<button
										on:click={() => routeEditor?.openEdit(httpTriggers.path, isFlow)}
										class="px-2"
									>
										{#if httpTriggers.canWrite}
											Edit
										{:else}
											View
										{/if}
									</button>
								</div>
							</div>
						{/each}
					</div>
				{/if}
			{:else}
				<Skeleton layout={[[8]]} />
			{/if}
		</Section>
	{/if}
</div>
