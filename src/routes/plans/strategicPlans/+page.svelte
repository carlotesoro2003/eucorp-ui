<script lang="ts">
	import { supabase } from "$lib/supabaseClient";
	import { PlusCircle, Trash2, Save, AlertCircle, CheckCircle, Loader2, ChevronDown } from "lucide-svelte";
	import ObjectiveForm from "$lib/components/strategic-objectives/ObjectiveForm.svelte";
	import Toast from "$lib/components/strategic-objectives/Toast.svelte";

	interface StrategicGoal {
		id: number;
		name: string;
	}

	interface StrategicObjective {
		name: string;
		strategic_initiatives: string;
		kpi: string;
		persons_involved: string;
		target: string;
		eval_measures: string;
		strategic_goal_id: number;
		profile_id: string;
	}

	// State variables
	let strategicGoals: StrategicGoal[] = $state([]);
	let selectedGoal: number | null = $state(null);
	let objectives: StrategicObjective[] = $state([getEmptyObjective()]);
	let loading: boolean = $state(true);
	let submitting: boolean = $state(false);
	let successMessage: string | null = $state(null);
	let errorMessage: string | null = $state(null);
	let activeObjectiveIndex: number = $state(0);
	// Get current user profile id
	let profileId: string | null = $state(null);

	/** Get empty objective */
	function getEmptyObjective(): StrategicObjective {
		return {
			name: "",
			strategic_initiatives: "",
			kpi: "",
			persons_involved: "",
			target: "",
			eval_measures: "",
			strategic_goal_id: 0,
			profile_id: profileId || "", // Set profile_id from current user
		};
	}

	/** Get current user profile */
	const getCurrentUser = async () => {
		const {
			data: { user },
		} = await supabase.auth.getUser();
		if (user) {
			const { data: profile } = await supabase.from("profiles").select("id").eq("user_id", user.id).single();

			if (profile) {
				profileId = profile.id;
				// Refresh objectives with new profile_id
				objectives = objectives.map((obj) => ({ ...obj, profile_id: profile.id }));
			}
		}
	};

	/** Fetch strategic goals */
	const fetchStrategicGoals = async () => {
		try {
			const { data, error } = await supabase.from("strategic_goals").select("id, name");
			if (error) throw error;
			strategicGoals = data;
		} catch (error) {
			errorMessage = "Failed to fetch strategic goals.";
		} finally {
			loading = false;
		}
	};

	/** Add new objective */
	const addObjective = () => {
		objectives = [...objectives, getEmptyObjective()];
		activeObjectiveIndex = objectives.length - 1;
	};

	/** Remove objective */
	const removeObjective = (index: number) => {
		if (objectives.length > 1) {
			objectives = objectives.filter((_, i) => i !== index);
			if (activeObjectiveIndex >= objectives.length) {
				activeObjectiveIndex = objectives.length - 1;
			}
		}
	};

	/** Handle form submission */
	const handleSubmit = async () => {
		try {
			submitting = true;
			if (!selectedGoal) throw new Error("Please select a strategic goal");
			if (!profileId) throw new Error("No profile found. Please login again.");

			for (const obj of objectives) {
				if (!obj.name.trim() || !obj.strategic_initiatives.trim() || !obj.kpi.trim() || !obj.persons_involved.trim() || !obj.target.trim() || !obj.eval_measures.trim()) {
					throw new Error("Please fill in all fields for each objective");
				}
			}

			const objectivesWithProfile = objectives.map((obj) => ({
				...obj,
				strategic_goal_id: selectedGoal,
				profile_id: profileId,
			}));

			const { error } = await supabase.from("strategic_objectives").insert(objectivesWithProfile);
			if (error) throw error;

			successMessage = "Strategic Objectives added successfully!";
			setTimeout(() => {
				successMessage = null;
				objectives = [getEmptyObjective()];
				activeObjectiveIndex = 0;
			}, 2000);
		} catch (error) {
			errorMessage = (error as Error).message;
			setTimeout(() => {
				errorMessage = null;
			}, 3000);
		} finally {
			submitting = false;
		}
	};

	// Initialize data
	getCurrentUser();
	fetchStrategicGoals();
</script>

<div class="container mx-auto px-4 py-8">
	<div class="flex flex-col gap-6 max-w-[1000px] mx-auto">
		<div class="flex items-center justify-between">
			<h2 class="text-3xl font-bold">Create Strategic Objectives</h2>
			{#if loading}
				<div class="flex items-center gap-2 text-muted-foreground">
					<Loader2 class="w-5 h-5 animate-spin" />
					<span>Loading...</span>
				</div>
			{/if}
		</div>

		{#if successMessage}
			<Toast type="success" message={successMessage} />
		{/if}

		{#if errorMessage}
			<Toast type="error" message={errorMessage} />
		{/if}

		<div class="flex flex-col gap-2">
			<label for="strategic_goal" class="text-lg font-medium">Select Strategic Goal</label>
			<div class="relative">
				<select id="strategic_goal" bind:value={selectedGoal} class="select bg-card p-3 rounded-lg w-full max-w-md border border-border/50 focus:border-primary focus:ring-1 focus:ring-primary outline-none transition-colors appearance-none">
					<option value={null}>Select a goal</option>
					{#each strategicGoals as goal}
						<option value={goal.id}>{goal.name}</option>
					{/each}
				</select>
				<ChevronDown class="absolute right-3 top-1/2 -translate-y-1/2 w-5 h-5 text-muted-foreground pointer-events-none" />
			</div>
		</div>

		<div class="flex flex-col gap-4">
			<div class="flex items-center justify-between">
				<div class="flex gap-2">
					{#each objectives as _, index}
						<button onclick={() => (activeObjectiveIndex = index)} class="px-4 py-2 rounded-lg {activeObjectiveIndex === index ? 'bg-primary text-primary-foreground' : 'bg-secondary hover:bg-secondary/80'} transition-colors">
							Objective {index + 1}
						</button>
					{/each}
				</div>
				<button onclick={addObjective} class="flex items-center gap-2 bg-secondary text-foreground px-4 py-2 rounded-lg hover:bg-secondary/80 transition-colors">
					<PlusCircle class="w-5 h-5" />
					Add Objective
				</button>
			</div>

			<div class="bg-card border border-border rounded-lg p-6">
				<div class="flex justify-between items-center mb-6">
					<h3 class="text-xl font-semibold">Objective {activeObjectiveIndex + 1}</h3>
					{#if objectives.length > 1}
						<button onclick={() => removeObjective(activeObjectiveIndex)} class="flex items-center gap-1.5 bg-destructive/10 text-destructive px-3 py-1.5 rounded-lg hover:bg-destructive/20 transition-colors">
							<Trash2 class="w-4 h-4" />
							<span>Delete</span>
						</button>
					{/if}
				</div>
				<ObjectiveForm bind:objective={objectives[activeObjectiveIndex]} />
			</div>
		</div>

		<button onclick={handleSubmit} disabled={submitting} class="flex items-center gap-2 bg-primary text-primary-foreground px-4 py-2 rounded-lg hover:bg-primary/90 transition-colors disabled:opacity-50 disabled:cursor-not-allowed w-fit">
			{#if submitting}
				<Loader2 class="w-5 h-5 animate-spin" />
				<span>Submitting...</span>
			{:else}
				<Save class="w-5 h-5" />
				<span>Submit Objectives</span>
			{/if}
		</button>
	</div>
</div>
