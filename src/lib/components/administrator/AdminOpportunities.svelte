<script lang="ts">
  import { onMount } from "svelte";
  import { supabase } from "$lib/supabaseClient";
  import jsPDF from "jspdf";
  import autoTable from "jspdf-autotable";

  interface Opportunity {
    id: number;
    opt_statement: string;
    planned_actions: string;
    kpi: string;
    key_persons: string;
    target_output: string;
    budget: number;
    is_approved: boolean;
    is_approved_vp: boolean;
    is_approved_president: boolean;
    profile_id: string;
    department_name: string | null;
    user_name: string | null;
  }

  let opportunities: Opportunity[] = [];
  let displayedOpportunities: Opportunity[] = [];
  let departments: { id: string; name: string }[] = [];
  let selectedDepartment: string | "all" = "all";

  let isLoading = false;
  let isSaving = false;
  let isDeleting = false;
  let isApproving = false;

  let userRole: string | null = null;
  let adminName: string | null = null;
  let vicePresidentName: string | null = null;
  let presidentName: string | null = null;

  let isEditModalOpen = false;
  let currentOpportunity: Opportunity | null = null;


  onMount(async () => {
    await fetchCurrentUserRole();
    await fetchAdminName();
    await fetchOpportunities();
    await fetchDepartments();
    await fetchVPAndPresidentNames();
  });

  const fetchCurrentUserRole = async () => {
    const {
      data: { session },
    } = await supabase.auth.getSession();

    if (!session || !session.user) return;

    const { data: profile, error } = await supabase
      .from("profiles")
      .select("role, first_name, last_name")
      .eq("id", session.user.id)
      .single();

    if (error) {
      console.error("Error fetching user role:", error);
      return;
    }

    userRole = profile.role;
  };

  const fetchOpportunities = async () => {
    isLoading = true;

    const { data, error } = await supabase
      .from("opportunities")
      .select(`
        *,
        profiles (
          first_name,
          last_name,
          departments (name)
        )
      `);

    if (error) {
      console.error("Error fetching opportunities:", error);
      return;
    }

    opportunities = data.map((opportunity: any) => ({
      ...opportunity,
      department_name: opportunity.profiles?.departments?.name || "N/A",
      user_name: `${opportunity.profiles?.first_name || ""} ${opportunity.profiles?.last_name || ""}`,
    }));

    applyFilters();
    isLoading = false;
  };

  const fetchDepartments = async () => {
    const { data, error } = await supabase
      .from("departments")
      .select("id, name");

    if (error) {
      console.error("Error fetching departments:", error);
      return;
    }

    departments = data;
  };

  const fetchAdminName = async () => {
      try {
        const { data, error } = await supabase
          .from("profiles")
          .select("first_name, last_name")
          .eq("role", "admin");

        if (error || !data) {
          console.error("Error fetching admin names:", error || "No data found");
          adminName = "N/A"; // Fallback if no admins are found
        } else {
          // Combine all admin names into a single string
          adminName = data.map((admin) => `${admin.first_name} ${admin.last_name}`).join(", ");
        }
      } catch (error) {
        console.error("Error fetching admin details:", error);
        adminName = "N/A";
      }
  };


    const fetchVPAndPresidentNames = async () => {
      try {
        const { data, error } = await supabase
          .from("profiles")
          .select("first_name, last_name, role")
          .in("role", ["vice_president", "president"]);

        if (error || !data) {
          console.error("Error fetching VP and President names:", error || "No data found");
          vicePresidentName = "N/A";
          presidentName = "N/A";
          return;
        }

        const vp = data.find((user) => user.role === "vice_president");
        const president = data.find((user) => user.role === "president");

        vicePresidentName = vp ? `${vp.first_name} ${vp.last_name}` : "N/A";
        presidentName = president ? `${president.first_name} ${president.last_name}` : "N/A";
      } catch (error) {
        console.error("Error fetching VP and President details:", error);
        vicePresidentName = "N/A";
        presidentName = "N/A";
      }
    };

  const applyFilters = () => {
    let filteredOpportunities = [...opportunities];

    if (selectedDepartment !== "all") {
      filteredOpportunities = filteredOpportunities.filter(
        (opportunity) => opportunity.department_name === selectedDepartment
      );
    }

    displayedOpportunities = filteredOpportunities;
  };

  const editOpportunity = (opportunity: Opportunity) => {
    currentOpportunity = { ...opportunity }; // Clone the opportunity to avoid direct mutation
    isEditModalOpen = true;
  };

  const saveOpportunity = async () => {
    if (!currentOpportunity) return;

    isSaving = true;
    const { error } = await supabase
      .from("opportunities")
      .update({
        opt_statement: currentOpportunity.opt_statement,
        planned_actions: currentOpportunity.planned_actions,
        kpi: currentOpportunity.kpi,
        key_persons: currentOpportunity.key_persons,
        target_output: currentOpportunity.target_output,
        budget: currentOpportunity.budget,
      })
      .eq("id", currentOpportunity.id);

    if (error) {
      console.error("Error updating opportunity:", error);
    } else {
      await fetchOpportunities(); // Refresh the list of opportunities
      isEditModalOpen = false;
      currentOpportunity = null;
    }

    isSaving = false;
  };



  const deleteOpportunity = async (id: number) => {
    isDeleting = true;

    const { error } = await supabase.from("opportunities").delete().eq("id", id);

    if (error) {
      console.error("Error deleting opportunity:", error);
    } else {
      await fetchOpportunities();
    }

    isDeleting = false;
  };

  const approveOpportunity = async (id: number) => {
    isApproving = true;

    let updateField = {};
    if (userRole === "admin") {
      updateField = { is_approved: true };
    } else if (userRole === "vice_president") {
      updateField = { is_approved_vp: true };
    } else if (userRole === "president") {
      updateField = { is_approved_president: true };
    }

    const { data, error } = await supabase
      .from("opportunities")
      .update(updateField)
      .eq("id", id)
      .select();

    if (error) {
      console.error("Error approving opportunity:", error);
    }

    if (userRole === "president" && data && data.length > 0) {
      const opportunity = data[0];
      const { error: monitoringError } = await supabase.from("opt_monitoring").insert({
        opt_id: opportunity.id,
        profile_id: opportunity.profile_id,
        department_id: opportunity.department_id,
      });
      if (monitoringError) {
        console.error("Error adding to opportunity monitoring:", monitoringError);
      }
    }

    await fetchOpportunities();
    isApproving = false;
  };

  const exportToPDF = () => {
  const doc = new jsPDF("landscape");

  // Set font to Times New Roman (or similar serif font)
  doc.setFont("times", "normal");

  // Title and Header Information
  doc.setFontSize(12);
  doc.text("MANUEL S. ENVERGA UNIVERSITY FOUNDATION", 14, 10);
  doc.text("Opportunities Report", 14, 16);
  doc.setFontSize(10);
  doc.text("SY 2022-2023", 14, 22);

  // Table Columns
  const columns = [
    "Opportunity Statement",
    "Planned Actions",
    "KPI",
    "Key Persons",
    "Target Output",
    "Budget",
    "Department",
  ];

  // Table Rows
  const rows = displayedOpportunities.map((opportunity) => [
    opportunity.opt_statement,
    opportunity.planned_actions,
    opportunity.kpi,
    opportunity.key_persons,
    opportunity.target_output,
    opportunity.budget.toFixed(2), // Ensure budget is properly formatted
    opportunity.department_name || "Unknown",
  ]);

  // Add the table using autoTable
  autoTable(doc, {
    head: [columns],
    body: rows,
    startY: 28, // Position below the header
    theme: "grid",
    styles: {
      font: "times", // Set font for the table
      fontSize: 10, // Standard font size
    },
    headStyles: { fillColor: [41, 128, 185] }, // Custom color for header background
  });

  // Signature Section
  const pageHeight = doc.internal.pageSize.height;
  const signatureStartY = pageHeight - 40; // Adjust for footer space

  // Positions for signatures
  const columnWidth = doc.internal.pageSize.width / 4;
  const positions = [14, columnWidth, columnWidth * 2, columnWidth * 3];

  // Add Signatures
  doc.setFontSize(10);

  // Department Head
  const firstOpportunity = displayedOpportunities[0];
  const userName = firstOpportunity?.user_name || "Unknown";
  const departmentName = firstOpportunity?.department_name || "Unknown";

  doc.text(`${userName} (sgnd)`, positions[0], signatureStartY - 5);
  doc.text("_________________________", positions[0], signatureStartY);
  doc.text(`${departmentName} Department Head`, positions[0], signatureStartY + 5);

  // Corporate Planning Officer
  doc.text(`${adminName || "N/A"} (sgnd)`, positions[1], signatureStartY - 5);
  doc.text("_________________________", positions[1], signatureStartY);
  doc.text("Corporate Planning Officer", positions[1], signatureStartY + 5);

  // Vice President
  doc.text(`${vicePresidentName || "N/A"} (sgnd)`, positions[2], signatureStartY - 5);
  doc.text("_________________________", positions[2], signatureStartY);
  doc.text("Vice President", positions[2], signatureStartY + 5);

  // President
  doc.text(`${presidentName || "N/A"} (sgnd)`, positions[3], signatureStartY - 5);
  doc.text("_________________________", positions[3], signatureStartY);
  doc.text("President", positions[3], signatureStartY + 5);

  // Save the PDF file
  doc.save("Opportunities_Report.pdf");
};

</script>

<div class="container mx-auto p-6">
  <h2 class="text-2xl font-bold mb-6">Opportunities</h2>

  <div class="flex gap-4 mb-4">
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
    <button class="btn btn-secondary" on:click={exportToPDF}>
      Export to PDF
    </button>
  </div>

  {#if isLoading}
    <div>Loading opportunities...</div>
  {:else if displayedOpportunities.length === 0}
    <div>No opportunities available.</div>
  {:else}
    <table class="table w-full">
      <thead>
        <tr>
          <th>Opportunity Statement</th>
          <th>Planned Actions</th>
          <th>KPI</th>
          <th>Key Persons</th>
          <th>Target Output</th>
          <th>Budget</th>
          <th>Department</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        {#each displayedOpportunities as opportunity}
          <tr>
            <td>{opportunity.opt_statement}</td>
            <td>{opportunity.planned_actions}</td>
            <td>{opportunity.kpi}</td>
            <td>{opportunity.key_persons}</td>
            <td>{opportunity.target_output}</td>
            <td>{opportunity.budget}</td>
            <td>{opportunity.department_name}</td>
            <td class="flex gap-2">
              <button
                class="btn btn-sm btn-success"
                on:click={() => approveOpportunity(opportunity.id)}
                disabled={
                  isApproving ||
                  (userRole === "admin" && opportunity.is_approved) ||
                  (userRole === "vice_president" &&
                    (!opportunity.is_approved || opportunity.is_approved_vp)) ||
                  (userRole === "president" &&
                    (!opportunity.is_approved_vp || opportunity.is_approved_president))
                }
              >
                {isApproving
                  ? "Processing..."
                  : userRole === "admin"
                  ? opportunity.is_approved
                    ? "Admin Approved"
                    : "Approve as Admin"
                  : userRole === "vice_president"
                  ? opportunity.is_approved_vp
                    ? "VP Approved"
                    : "Approve as VP"
                  : userRole === "president"
                  ? opportunity.is_approved_president
                    ? "President Approved"
                    : "Approve as President"
                  : "Approve"}
              </button>
              <button
                class="btn btn-sm btn-primary"
                on:click={() => editOpportunity(opportunity)}
              >
                Edit
              </button>
              <button
                class="btn btn-sm btn-error"
                on:click={() => deleteOpportunity(opportunity.id)}
              >
                Delete
              </button>
            </td>
          </tr>
        {/each}
      </tbody>
    </table>
  {/if}
</div>


<!--Edit Modal-->
{#if isEditModalOpen}
  <div class="modal modal-open">
    <div class="modal-box">
      <h3 class="font-bold text-lg">Edit Opportunity</h3>

      <div class="form-control mb-4">
        <label class="label" for="opt_statement">Opportunity Statement</label>
        {#if currentOpportunity}
          <textarea
            id="opt_statement"
            bind:value={currentOpportunity.opt_statement}
            class="textarea textarea-bordered"
          ></textarea>
        {/if}
      </div>

      <div class="form-control mb-4">
        <label class="label" for="planned_actions">Planned Actions</label>
        {#if currentOpportunity}
          <textarea
            id="planned_actions"
            bind:value={currentOpportunity.planned_actions}
            class="textarea textarea-bordered"
          ></textarea>
        {/if}
      </div>

      <div class="form-control mb-4">
        <label class="label" for="kpi">KPI</label>
        {#if currentOpportunity}
          <textarea
            bind:value={currentOpportunity.kpi}
            class="textarea textarea-bordered"
          ></textarea>
        {/if}
      </div>

      <div class="form-control mb-4">
        <label class="label" for="key_persons">Key Persons</label>
        {#if currentOpportunity}
          <textarea
            bind:value={currentOpportunity.key_persons}
            class="textarea textarea-bordered"
          ></textarea>
        {/if}
      </div>

      <div class="form-control mb-4">
        <label class="label" for="target_output">Target Output</label>
        {#if currentOpportunity}
          <textarea
            bind:value={currentOpportunity.target_output}
            class="textarea textarea-bordered"
          ></textarea>
        {/if}
      </div>

      <div class="form-control mb-4">
        <label class="label" for="budget">Budget</label>
        {#if currentOpportunity}
        <input
          type="number"
          bind:value={currentOpportunity.budget}
          class="input input-bordered"
        />
        {/if}
      </div>

      <div class="modal-action">
        <button class="btn btn-primary" on:click={saveOpportunity} disabled={isSaving}>
          {isSaving ? "Saving..." : "Save"}
        </button>
        <button class="btn" on:click={() => (isEditModalOpen = false)}>
          Cancel
        </button>
      </div>
    </div>
  </div>
{/if}
