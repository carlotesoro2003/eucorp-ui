<script lang="ts">
	import type { StrategicObjective } from "$lib/components/strategic-objectives/types";
    
	let { objective = $bindable({} as StrategicObjective) }: { objective: StrategicObjective } = $props();

	const fields = [
		{ id: "name", label: "Objective Name", placeholder: "Enter objective name" },
		{ id: "strategic_initiatives", label: "Strategic Initiatives", placeholder: "Enter strategic initiatives" },
		{ id: "kpi", label: "KPI", placeholder: "Enter KPI" },
		{ id: "persons_involved", label: "Persons Involved", placeholder: "Enter persons involved" },
		{ id: "target", label: "Target", placeholder: "Enter target" },
		{ id: "eval_measures", label: "Evaluation Measures", placeholder: "Enter evaluation measures" },
	];
</script>

<div class="grid gap-6">
	{#each fields as field}
		<div class="flex flex-col gap-2">
			<label for={field.id} class="font-medium">{field.label}</label>
			<textarea id={field.id} bind:value={objective[field.id as keyof StrategicObjective]} class="bg-background rounded-lg p-3 resize-none border border-border/50 focus:border-primary focus:ring-1 focus:ring-primary outline-none transition-colors min-h-[100px]" placeholder={field.placeholder}></textarea>
		</div>
	{/each}
</div>
