<script lang='ts'>
	import { AnimCore, type JSONData } from 'src/storage/AnimCore'
	import { ErrorMsg, arrayMove, camelToSpaces, i18n, nonNullable } from 'src/utils'
	import { flip } from 'svelte/animate'
	import type { Writable } from 'svelte/store'
	import { slide } from 'svelte/transition'
	import Separator from './Separator.svelte'

	export let key: string
	export let value: JSONData[string]
	export let flag: Writable<JSONData>
	export let disabled: boolean = false

	let isReference = typeof value === 'string'
	$: isReference = typeof value === 'string'
	let hidden = window.pf2eGraphics.liveSettings.dev

	function convertToReference() {
		if (!isReference) {
			foundry.applications.api.DialogV2.confirm({
				content: i18n('confirmConvert'),
				rejectClose: false,
				modal: true,
			}).then((x) => {
				if (!x) return
				value = ''
			})
		} else {
			value = [AnimCore.CONST.TEMPLATE_ANIMATION()]
		}
	}

	function bindJSON(event: Event, obj: Record<string, any>, prop: string) {
		try {
			obj[prop] = JSON.parse((event.target as HTMLInputElement).value)
			$flag = $flag
		} catch {
			throw new ErrorMsg('Invalid JSON!')
		}
	}

	function addNewAnimation() {
		if (typeof value === 'string')
			throw new ErrorMsg('Trying to push a new animation into a reference!')

		value = [...value, (AnimCore.CONST.TEMPLATE_ANIMATION())]
	}

	function remove() {
		foundry.applications.api.DialogV2.confirm({
			content: i18n('areYouSureM8'),
			rejectClose: false,
			modal: true,
		}).then((x) => {
			if (!x) return
			// @ts-ignore Explicit Deletion
			$flag[key] = null
		})
	}

	function dragStart(event: DragEvent, index: Number) {
		const data = { indexFrom: index }
		event.dataTransfer?.setData('text/plain', JSON.stringify(data))
	}

	function drop(event: DragEvent, indexTo: number) {
		event?.preventDefault()
		const json = event.dataTransfer?.getData('text/plain')
		const { indexFrom } = JSON.parse(String(json))

		if (disabled || typeof value === 'string' || !nonNullable(indexFrom) || indexFrom === indexTo) return

		value = arrayMove(value, indexFrom, indexTo)
	}

	// @ts-expect-error TODO: Yes it does.
	const dbEntries = window.Sequencer.Database.entries
</script>

{#if nonNullable(value)}
	<section class='
		border border-solid border-slate-600 rounded-sm
		bg-slate-300 bg-opacity-50
		w-full p-2
	'>
		<header class='flex text-lg items-center'>
			<!-- svelte-ignore a11y-click-events-have-key-events -->
			<i
				class='fa fa-chevron-{!hidden ? 'down' : 'up'} pr-1'
				on:click={() => hidden = !hidden}
				role='button'
				tabindex='-1'
			/>
			<div>{key}</div>
			<div class='flex flex-grow justify-end gap-1'>
				{#if isReference}
					<input
						{disabled}
						class='
							mx-1 h-full
							text-sm truncate
							w-40 focus:w-1/2 transition-all
						'
						type='text'
						bind:value
						list='options'
					/>
					<datalist id='options'>
						{#each AnimCore.keys as key}
							<option value={key}>{key}</option>
						{/each}
					</datalist>
				{/if}
				<div class='flex'>
					<button
						{disabled}
						class='
							pseud-button
							fa{isReference ? '' : '-regular'} fa-clone
						'
						data-tooltip='pf2e-graphics.reference'
						on:click={convertToReference}
						tabindex='-1'
					/>
					{#if !isReference}
						<button
							{disabled}
							class='
								pseud-button
								fas fa-add
							'
							data-tooltip='pf2e-graphics.addAnim'
							on:click={addNewAnimation}
							tabindex='-1'
						/>
					{/if}
					<button
						{disabled}
						class='
							pseud-button
							fa fa-trash-can
						'
						data-tooltip='pf2e-graphics.reference'
						on:click={remove}
						tabindex='-1'
					/>
				</div>
			</div>
		</header>
		{#if !isReference && typeof value !== 'string' && hidden}
			<div transition:slide class='[&>*]:bg-slate-300 [&>*]:bg-opacity-80 [&>*]:rounded-sm'>
				<hr />
				{#if !value.length}
					<div class='text-center w-full opacity-50'>
						<i>There's nothing here. This may be done on purpose to disable a preset animation.</i>
					</div>
				{:else}
					{#each value as ani, index (ani)}
						<section
							animate:flip={{ duration: 250 }}
							class='flex flex-col gap-1.5 overflow-clip p-1'
							role='group'
							draggable={Boolean(value.length - 1)}
							on:dragstart={event => dragStart(event, index)}
							on:drop={event => drop(event, index)}
						>
							{(ani.options ??= {}) && ''}
							<header class='flex justify-between gap-1.5'>
								<!-- Preset -->
								<label class='flex items-center gap-2'>
									<span class='mr-1 text-nowrap'> {i18n('editor.preset')} </span>
									<select bind:value={ani.preset} {disabled}>
										{#each AnimCore.CONST.PRESETS as preset}
											<option value={preset}>{camelToSpaces(preset).titleCase()}</option>
										{/each}
									</select>
								</label>

								<!-- Trigger -->
								<label class='flex items-center gap-2'>
									<span class='mr-1 text-nowrap'> {i18n('editor.trigger')} </span>
									<select bind:value={ani.trigger} {disabled}>
										{#each AnimCore.CONST.TRIGGERS as trigger}
											<option value={trigger}>{i18n(`triggers.${trigger}`)}</option>
										{/each}
									</select>
								</label>

								<!-- Predicate -->
								<label class='flex items-center gap-2'>
									<span class='mr-1 text-nowrap'> {i18n('editor.predicate')} </span>
									<input
										{disabled}
										type='text'
										value={JSON.stringify(ani.predicate || [])}
										on:change={e => bindJSON(e, ani, 'predicate')}
									/>
								</label>
							</header>

							<!-- Image File -->
							<label class='flex items-center gap-1 col-span-4'>
								<span class='mr-1 text-nowrap'>{i18n('editor.imageFile')}</span>
								<input
									{disabled}
									type='text'
									placeholder={window.Sequencer.Helpers.random_array_element(dbEntries.jb2a).dbPath}
									bind:value={ani.file}
								/>
								<button
									class='fas fa-database w-min leading-6'
									data-tooltip='SEQUENCER.SidebarButtons.Database'
									on:click={() => window.Sequencer.DatabaseViewer.show()}
								/>
							</label>

							{(ani.options.sound ??= {}) && ''}
							{#if Array.isArray(ani.options.sound)}
								<div class='p-1 text-center text-opacity-50 text-slate-600 border border-solid rounded-sm'><i>This animation contains multiple sounds which are modifiable only through JSON editor.</i></div>
							{:else}
								<!-- Sound File -->
								<label class='flex items-center gap-1 col-span-4'>
									<span class='mr-1 text-nowrap'>{i18n('editor.soundFile')}</span>
									<input
										{disabled}
										type='text'
										placeholder={window.Sequencer.Helpers.random_array_element(dbEntries['pf2e-graphics']).dbPath}
										bind:value={ani.options.sound.file}
									/>
									<button
										class='fas fa-database w-min leading-6'
										data-tooltip='SEQUENCER.SidebarButtons.Database'
										on:click={() => window.Sequencer.DatabaseViewer.show()}
									/>
								</label>
							{/if}

							<Separator>
								Advanced Options
							</Separator>
							<!-- Booleans -->
							<div class='flex justify-between'>
								<label class='flex items-center gap-1 text-nowrap'>
									<span>Persistent</span>
									<input {disabled} type='checkbox' bind:checked={ani.options.persist} />
								</label>
								<label class='flex items-center gap-1 text-nowrap'>
									<span>Masked</span>
									<input {disabled} type='checkbox' bind:checked={ani.options.mask} />
								</label>
								<label class='flex items-center gap-1 text-nowrap'>
									<span>Tie To Documents</span>
									<input {disabled} type='checkbox' bind:checked={ani.options.tieToDocuments} />
								</label>
							</div>

							<!-- Delays -->
							<div class='flex justify-between gap-2'>
								<label class='flex items-center gap-1 text-nowrap'>
									<span>Wait Until Finished</span>
									<input {disabled} type='number'
										placeholder='-1000 (ms)'
										bind:value={ani.options.waitUntilFinished}
									/>
								</label>
								<label class='flex items-center gap-1 text-nowrap'>
									<span>Wait</span>
									<input {disabled} type='number'
										placeholder='1000 (ms)'
										bind:value={ani.options.wait}
									/>
								</label>
								<label class='flex items-center gap-1 text-nowrap'>
									<span>Delay</span>
									<input {disabled} type='number'
										placeholder='1000 (ms)'
										bind:value={ani.options.delay}
									/>
								</label>
							</div>

							{#if index !== value.length - 1}
								<hr />
							{/if}
						</section>
					{/each}
				{/if}
			</div>
		{/if}
	</section>
{/if}

<style lang='postcss'>
	.pseud-button {
		@apply p-1 border border-solid rounded-sm transition-colors size-8;
	}
</style>
