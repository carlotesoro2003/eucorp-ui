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

  onMount(async () => {
    await fetchCurrentUserRole();
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

    if (userRole === "admin") {
      adminName = `${profile.first_name} ${profile.last_name}`;
    }
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

  const fetchVPAndPresidentNames = async () => {
    const { data, error } = await supabase
      .from("profiles")
      .select("role, first_name, last_name")
      .in("role", ["vice_president", "president"]);

    if (error) {
      console.error("Error fetching VP and President names:", error);
      return;
    }

    const vp = data.find((profile) => profile.role === "vice_president");
    const president = data.find((profile) => profile.role === "president");

    vicePresidentName = vp ? `${vp.first_name} ${vp.last_name}` : "N/A";
    presidentName = president ? `${president.first_name} ${president.last_name}` : "N/A";
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
    // Logic to open and populate edit modal (not included)
    console.log("Editing Opportunity:", opportunity);
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
  const title = "Opportunities Report";
  const columns = [
    "User Name",
    "Opportunity Statement",
    "Planned Actions",
    "KPI",
    "Key Persons",
    "Target Output",
    "Budget",
    "Department",
  ];
  const rows = displayedOpportunities.map((opportunity) => [
    opportunity.user_name || "Unknown",
    opportunity.opt_statement,
    opportunity.planned_actions,
    opportunity.kpi,
    opportunity.key_persons,
    opportunity.target_output,
    opportunity.budget.toFixed(2), // Ensure budget is properly formatted
    opportunity.department_name || "Unknown",
  ]);

  // Add the title
  doc.setFontSize(14);
  doc.text(title, 14, 15);

  // Add the table
  autoTable(doc, {
    head: [columns],
    body: rows,
    startY: 25, // Ensure proper spacing
    theme: "grid",
    styles: { fontSize: 10 },
    headStyles: { fillColor: [41, 128, 185] }, // Custom color for table header
  });

  // Prepare space for signatures
  const pageHeight = doc.internal.pageSize.height;
  const signatureStartY = pageHeight - 30;

  // Ensure the first opportunity exists to extract user and department details
  const firstOpportunity = displayedOpportunities[0];
  const userName = firstOpportunity?.user_name || "Unknown";
  const departmentName = firstOpportunity?.department_name || "Unknown";

  // User Signature
  doc.text(`${userName} (sgnd)`, 14, signatureStartY - 5);
  doc.text("_________________________", 14, signatureStartY);
  doc.text(`${departmentName} Department Head`, 14, signatureStartY + 5);

  // Corporate Planning Officer
  doc.text(`${adminName || "N/A"} (sgnd)`, 100, signatureStartY - 5);
  doc.text("_________________________", 100, signatureStartY);
  doc.text("Corporate Planning Officer", 100, signatureStartY + 5);

  // Vice President
  const vpSigned = displayedOpportunities.some((opportunity) => opportunity.is_approved_vp);
  doc.text(
    vpSigned ? `${vicePresidentName || "N/A"} (sgnd)` : `${vicePresidentName || "N/A"}`,
    180,
    signatureStartY - 5
  );
  doc.text("_________________________", 180, signatureStartY);
  doc.text("Vice President", 180, signatureStartY + 5);

  // President
  const presidentSigned = displayedOpportunities.some(
    (opportunity) => opportunity.is_approved_president
  );
  doc.text(
    presidentSigned ? `${presidentName || "N/A"} (sgnd)` : `${presidentName || "N/A"}`,
    260,
    signatureStartY - 5
  );
  doc.text("_________________________", 260, signatureStartY);
  doc.text("President", 260, signatureStartY + 5);

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
