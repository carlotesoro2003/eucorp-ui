<script lang="ts">
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabaseClient";
  import jsPDF from "jspdf";
  import autoTable from "jspdf-autotable";
  import { page } from "$app/stores";

  interface ActionPlan {
    id: number;
    actions_taken: string;
    kpi: string;
    target_output: string;
    key_person_responsible: string;
    created_at: string;
    objective_id: number;
    is_approved: boolean;
    profile_id: string | null;
    department_name: string | null; 
    user_name: string | null; 
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
  let departments: { id: string; name: string }[] = [];
  let selectedDepartment: string | "all" = "all";
  let filterType: "all" | "approved" | "notApproved" = "all";

  let objective: StrategicObjective | null = null;
  let strategicGoal: StrategicGoal | null = null;
  let objective_id: number | null = null;
  let isLoading = true;

  let adminName: string | null = null;
  let vicePresidentName: string | null = null;
  let presidentName: string | null = null;

  // Fetch data on page load
  onMount(async () => {
    const { params } = $page;
    objective_id = parseInt(params.id);

    if (objective_id) {
      try {
        // Fetch all action plans with user and department details
        const { data: plansData, error: plansError } = await supabase
          .from("action_plans")
          .select(`
            *,
            profiles (
              first_name,
              last_name,
              departments (name)
            ),
            department_id
          `);

        if (plansError) {
          console.error("Error fetching action plans:", plansError);
        } else {
          actionPlans = await Promise.all(plansData.map(async (plan: any) => ({
            id: plan.id,
            actions_taken: plan.actions_taken,
            kpi: plan.kpi,
            target_output: plan.target_output,
            key_person_responsible: plan.key_person_responsible,
            created_at: plan.created_at,
            objective_id: plan.objective_id,
            is_approved: plan.is_approved,
            profile_id: plan.profile_id,
            department_name:
              plan.profiles?.departments?.name ||
              (await fetchDepartmentName(plan.department_id)) ||
              "Unknown",
            user_name: plan.profiles
              ? `${plan.profiles?.first_name || ""} ${plan.profiles?.last_name || ""}`
              : "Unknown",
          })));
          displayedActionPlans = [...actionPlans];
        }

        // Fetch departments for filtering
        const { data: departmentData, error: departmentError } = await supabase
          .from("departments")
          .select("id, name");

        if (departmentError) {
          console.error("Error fetching departments:", departmentError);
        } else {
          departments = departmentData;
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

        await fetchAdminName();
        await fetchVPAndPresidentNames();
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

  const fetchDepartmentName = async (departmentId: string) => {
    try {
      const { data, error } = await supabase
        .from("departments")
        .select("name")
        .eq("id", departmentId)
        .single();

      if (error) {
        console.error("Error fetching department name:", error);
        return null;
      }
      return data?.name || null;
    } catch (error) {
      console.error("Error fetching department name:", error);
      return null;
    }
  };

  const applyFilters = () => {
    let filteredPlans = [...actionPlans];

    if (selectedDepartment !== "all") {
      filteredPlans = filteredPlans.filter(
        (plan) => plan.department_name === selectedDepartment
      );
    }

    if (filterType === "approved") {
      filteredPlans = filteredPlans.filter((plan) => plan.is_approved);
    } else if (filterType === "notApproved") {
      filteredPlans = filteredPlans.filter((plan) => !plan.is_approved);
    }

    displayedActionPlans = filteredPlans;
  };

  const approveActionPlan = async (planId: number) => {
    try {
      const { data, error } = await supabase
        .from("action_plans")
        .update({ is_approved: true })
        .eq("id", planId);

      if (error) {
        console.error("Error approving action plan:", error);
      } else {
        const planIndex = actionPlans.findIndex((plan) => plan.id === planId);
        actionPlans[planIndex].is_approved = true;
        displayedActionPlans = [...actionPlans];
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
        displayedActionPlans = [...actionPlans];
      }
    } catch (error) {
      console.error("Error deleting action plan:", error);
    }
  };

  const fetchAdminName = async () => {
    try {
      const {
        data: { session },
      } = await supabase.auth.getSession();

      if (!session || !session.user) return;

      const { data: profileData, error } = await supabase
        .from("profiles")
        .select("first_name, last_name")
        .eq("id", session.user.id)
        .single();

      if (error) {
        console.error("Error fetching admin name:", error);
      } else if (profileData) {
        adminName = `${profileData.first_name} ${profileData.last_name}`;
      }
    } catch (error) {
      console.error("Error fetching admin details:", error);
    }
  };

  const fetchVPAndPresidentNames = async () => {
    try {
      const { data, error } = await supabase
        .from("profiles")
        .select("first_name, last_name, role")
        .in("role", ["vice_president", "president"]);

      if (error) {
        console.error("Error fetching VP and President names:", error);
        return;
      }

      const vp = data?.find((user) => user.role === "vice_president");
      const president = data?.find((user) => user.role === "president");

      vicePresidentName = vp ? `${vp.first_name} ${vp.last_name}` : "N/A";
      presidentName = president ? `${president.first_name} ${president.last_name}` : "N/A";
    } catch (error) {
      console.error("Error fetching VP and President details:", error);
    }
  };

  const exportToPDF = () => {
    const userName = actionPlans[0]?.user_name || "Unknown";
    const departmentName = actionPlans[0]?.department_name || "Unknown";

    const doc = new jsPDF("landscape");
    const title = `${departmentName} Action Plans for Objective: ${objective?.name || "N/A"}`;
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
      "Department",
    ];
    const rows = displayedActionPlans.map((plan) => [
      plan.actions_taken,
      plan.kpi,
      plan.target_output,
      plan.key_person_responsible,
    ]);

    autoTable(doc, {
      head: [columns],
      body: rows,
      startY: 35,
      theme: "grid",
      styles: { fontSize: 10 },
      headStyles: { fillColor: [41, 128, 185] },
    });

    const pageHeight = doc.internal.pageSize.height;
    const signatureStartY = pageHeight - 30;

    // User Name
    doc.text(`${userName} (sgnd)`, 14, signatureStartY - 5);
    doc.text("_________________________", 14, signatureStartY);
    doc.text(`${departmentName} Department Head`, 14, signatureStartY + 5);

    // Admin
    doc.text(`${adminName || "N/A"} (sgnd)`, 100, signatureStartY - 5);
    doc.text("_________________________", 100, signatureStartY);
    doc.text("Corporate Planning Officer", 100, signatureStartY + 5);

    // Vice President
    doc.text(`${vicePresidentName || "N/A"}`, 180, signatureStartY - 5);
    doc.text("_________________________", 180, signatureStartY);
    doc.text("Vice President", 180, signatureStartY + 5);

    // President
    doc.text(`${presidentName || "N/A"}`, 260, signatureStartY - 5);
    doc.text("_________________________", 260, signatureStartY);
    doc.text("President", 260, signatureStartY + 5);

    doc.save("ActionPlans.pdf");
  };
</script>

<div class="min-h-screen p-8">
  <h1 class="text-2xl font-bold mb-4">Action Plans</h1>

  <!-- Department Filter -->
  <div class="mb-4 flex gap-4">
    <select
      class="select select-bordered"
      bind:value={selectedDepartment}
      on:change={applyFilters}
    >
      <option value="all">All Departments</option>
      {#each departments as department}
        <option value={department.name}>{department.name}</option>
      {/each}
    </select>

    <button class="btn btn-primary" on:click={() => { filterType = "all"; applyFilters(); }}>
      All
    </button>
    <button class="btn btn-secondary" on:click={() => { filterType = "approved"; applyFilters(); }}>
      Approved
    </button>
    <button class="btn btn-error" on:click={() => { filterType = "notApproved"; applyFilters(); }}>
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
      <table class="table w-full shadow-lg rounded-lg">
        <thead>
          <tr>
            <th>Department</th>
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
              <td>{plan.department_name}</td>
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
