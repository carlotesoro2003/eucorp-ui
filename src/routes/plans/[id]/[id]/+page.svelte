<script lang="ts">
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabaseClient";
  import { page } from "$app/stores";
  import jsPDF from "jspdf";
  import autoTable from "jspdf-autotable";

  interface ActionPlan {
    id: number;
    actions_taken: string;
    kpi: string;
    target_output: string;
    key_person_responsible: string;
    created_at: string;
    objective_id: number;
    is_approved: boolean;
    profile_id: string;
  }

  interface StrategicObjective {
    id: number;
    name: string;
  }

  interface StrategicGoal {
    id: number;
    name: string;
  }

  let actionPlans: ActionPlan[] = [];
  let displayedActionPlans: ActionPlan[] = [];
  let filterType: "all" | "approved" | "notApproved" = "all";

  let objective: StrategicObjective | null = null;
  let strategicGoal: StrategicGoal | null = null;
  let objective_id: number | null = null;
  let isLoading = true;

  // Fetch data on page load
  onMount(async () => {
    const { params } = $page;
    objective_id = parseInt(params.id);

    if (objective_id) {
      try {
        // Fetch action plans
        const { data: plansData, error: plansError } = await supabase
          .from("action_plans")
          .select("*")
          .eq("objective_id", objective_id);

        if (plansError) {
          console.error("Error fetching action plans:", plansError);
        } else {
          actionPlans = plansData;
          applyFilter(); // Apply filter after fetching action plans
        }

        // Fetch objective
        const { data: objectiveData, error: objectiveError } = await supabase
          .from("strategic_objectives")
          .select("id, name, strategic_goal_id")
          .eq("id", objective_id)
          .single();

        if (objectiveError) {
          console.error("Error fetching objective:", objectiveError);
        } else {
          objective = objectiveData;

          // Fetch strategic goal
          const { data: goalData, error: goalError } = await supabase
            .from("strategic_goals")
            .select("id, name")
            .eq("id", objectiveData.strategic_goal_id)
            .single();

          if (goalError) {
            console.error("Error fetching strategic goal:", goalError);
          } else {
            strategicGoal = goalData;
          }
        }
      } catch (error) {
        console.error("Error:", error);
      } finally {
        isLoading = false;
      }
    } else {
      console.error("Objective ID is missing.");
      isLoading = false;
    }
  });

  const applyFilter = () => {
    if (filterType === "approved") {
      displayedActionPlans = actionPlans.filter((plan) => plan.is_approved);
    } else if (filterType === "notApproved") {
      displayedActionPlans = actionPlans.filter((plan) => !plan.is_approved);
    } else {
      displayedActionPlans = [...actionPlans];
    }
  };

  const approveActionPlan = async (planId: number) => {
    try {
      const { data: planData, error: fetchError } = await supabase
        .from("action_plans")
        .select("profile_id")
        .eq("id", planId)
        .single();

      if (fetchError || !planData) {
        console.error("Error fetching profile_id from action_plan:", fetchError);
        return;
      }

      const planProfileId = planData.profile_id;

      const { error: updateError } = await supabase
        .from("action_plans")
        .update({ is_approved: true })
        .eq("id", planId);

      if (updateError) {
        console.error("Error approving action plan:", updateError);
        return;
      }

      const { error: insertError } = await supabase.from("plan_monitoring").insert({
        action_plan_id: planId,
        profile_id: planProfileId,
      });

      if (insertError) {
        console.error("Error inserting into plan_monitoring:", insertError);
      } else {
        actionPlans = actionPlans.map((plan) =>
          plan.id === planId ? { ...plan, is_approved: true } : plan
        );
        applyFilter();
      }
    } catch (error) {
      console.error("Error approving action plan:", error);
    }
  };

  const deleteActionPlan = async (planId: number) => {
    try {
      const { error } = await supabase.from("action_plans").delete().eq("id", planId);
      if (error) {
        console.error("Error deleting action plan:", error);
      } else {
        actionPlans = actionPlans.filter((plan) => plan.id !== planId);
        applyFilter();
      }
    } catch (error) {
      console.error("Error deleting action plan:", error);
    }
  };

  const exportToPDF = () => {
    const doc = new jsPDF("landscape");
    const title = `Action Plans for Objective: ${objective?.name || "N/A"}`;
    const goalName = strategicGoal
      ? `Strategic Goal: ${strategicGoal.name}`
      : "No Strategic Goal Assigned";

    doc.setFontSize(10);
    doc.text(title, 14, 15);
    doc.text(goalName, 14, 25);

    const columns = [
      "Actions Taken",
      "KPI",
      "Target Output",
      "Key Person Responsible",
      "Approved",
    ];
    const rows = displayedActionPlans.map((plan) => [
      plan.actions_taken,
      plan.kpi,
      plan.target_output,
      plan.key_person_responsible,
      plan.is_approved ? "Yes" : "No",
    ]);

    autoTable(doc, {
      head: [columns],
      body: rows,
      startY: 35,
      theme: "grid",
      styles: { fontSize: 10 },
      headStyles: { fillColor: [41, 128, 185] },
    });

    doc.save("ActionPlans.pdf");
  };
</script>


<div class="min-h-screen p-8">
  <h1 class="text-2xl font-bold mb-4">Action Plans</h1>

  <!-- Filters -->
  <div class="mb-4 flex gap-4">
    <button class="btn btn-primary" on:click={() => { filterType = "all"; applyFilter(); }}>
      All
    </button>
    <button class="btn btn-secondary" on:click={() => { filterType = "approved"; applyFilter(); }}>
      Approved
    </button>
    <button class="btn btn-error" on:click={() => { filterType = "notApproved"; applyFilter(); }}>
      Not Approved
    </button>
  </div>

  <!-- Export to PDF -->
  {#if filterType === "approved" && displayedActionPlans.length > 0}
    <button class="btn btn-secondary mb-4" on:click={exportToPDF}>
      Export Approved Plans to PDF
    </button>
  {/if}

  <!-- Table for Action Plans -->
  {#if isLoading}
    <div>Loading action plans...</div>
  {:else if displayedActionPlans.length > 0}
    <div class="overflow-x-auto">
      <table class="table table-zebra w-full shadow-lg rounded-lg">
        <thead>
          <tr>
            <th>Actions Taken</th>
            <th>KPI</th>
            <th>Target Output</th>
            <th>Key Person Responsible</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {#each displayedActionPlans as plan}
            <tr>
              <td>{plan.actions_taken}</td>
              <td>{plan.kpi}</td>
              <td>{plan.target_output}</td>
              <td>{plan.key_person_responsible}</td>
              <td class="flex space-x-2">
                <button class="btn btn-sm btn-success text-white" on:click={() => approveActionPlan(plan.id)} disabled={plan.is_approved}>
                  {plan.is_approved ? "Approved" : "Approve"}
                </button>
                <button class="btn btn-sm btn-error text-white" on:click={() => deleteActionPlan(plan.id)}>
                  Delete
                </button>
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  {:else}
    <div>No action plans found for this objective.</div>
  {/if}
</div>
