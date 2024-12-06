<script lang="ts">
  import { supabase } from "$lib/supabaseClient";
  import { onMount } from "svelte";
  import jsPDF from "jspdf";
  import autoTable from "jspdf-autotable";

  interface Risk {
    id: string;
    rrn: string;
    risk_statement: string;
    classification: number | null;
    actions: string;
    key_persons: string;
    budget: number;
    profile_id: string;
    is_approved: boolean;
  }

  interface Classification {
    id: number;
    name: string;
  }

  interface LikelihoodRating {
    id: number;
    name: string;
    symbol: string;
  }

  interface Severity {
    id: number;
    name: string;
    value: number;
  }

  interface RiskControlRating {
    id: number;
    name: string;
    symbol: string;
  }

  interface RiskMonitoringRating {
    id: number;
    status: string;
  }

  interface RiskAssessment {
    id: number;
    lr: number | null;
    s: number | null;
    rcr: number | null;
    rmr: number | null;
    risk_id: string;
  }

  let risks: Risk[] = [];
  let classification: Classification[] = [];
  let likelihoodRating: LikelihoodRating[] = [];
  let severity: Severity[] = [];
  let riskControlRating: RiskControlRating[] = [];
  let riskMonitoringRating: RiskMonitoringRating[] = [];
  let riskAssessment: RiskAssessment[] = [];
  let isLoading = true;
  let loadingRiskAssessments = true;

  let deletingRiskId: string | null = null;

  let successMessage: string | null = null;
  let errorMessage: string | null = null;

  const fetchRisks = async () => {
    isLoading = true;
    try {
      const { data, error } = await supabase
        .from("risks")
        .select("*")
        .order("rrn", { ascending: true });

      if (error) throw error;

      risks = data.sort((a, b) => {
        const getNumericPart = (rrn: string) => {
          const match = rrn.match(/(\d+)$/);
          return match ? parseInt(match[0], 10) : 0;
        };
        return getNumericPart(a.rrn) - getNumericPart(b.rrn);
      });
    } catch (error) {
      console.error("Error fetching risks:", error);
      errorMessage = "Failed to fetch risks.";
    } finally {
      isLoading = false;
    }
  };

  const fetchClassification = async () => {
    try {
      const { data, error } = await supabase.from("classification").select("*");
      if (error) throw error;
      classification = data;
    } catch (error) {
      console.error("Error fetching classification:", error);
      errorMessage = "Failed to fetch classification.";
    }
  };

  const fetchRiskAssessment = async () => {
    loadingRiskAssessments = true;
    try {
      const { data, error } = await supabase
        .from("risk_assessment")
        .select("*");
      if (error) throw error;
      riskAssessment = data;
    } catch (error) {
      console.error("Error fetching risk assessment:", error);
      errorMessage = "Failed to fetch risk assessment.";
    } finally {
      loadingRiskAssessments = false;
    }
  };

  const fetchLikelihoodRating = async () => {
    const { data, error } = await supabase
      .from("likelihood_rating")
      .select("*");
    if (!error) likelihoodRating = data;
  };

  const fetchSeverity = async () => {
    const { data, error } = await supabase.from("severity").select("*");
    if (!error) severity = data;
  };

  const fetchRiskControlRating = async () => {
    const { data, error } = await supabase
      .from("risk_control_rating")
      .select("*");
    if (!error) riskControlRating = data;
  };

  const fetchRiskMonitoringRating = async () => {
    const { data, error } = await supabase
      .from("risk_monitoring_rating")
      .select("*");
    if (!error) riskMonitoringRating = data;
  };

  // Helper functions to retrieve names from IDs with fallbacks
  const getLikelihoodRatingName = (id: number | null) =>
    likelihoodRating.find((item) => item.id === id)?.symbol || "Not Available";

  const getSeverityName = (id: number | null) =>
    severity.find((item) => item.id === id)?.value || "Not Available";

  const getRiskControlRatingName = (id: number | null) =>
    riskControlRating.find((item) => item.id === id)?.symbol || "Not Available";

  const getRiskMonitoringRatingStatus = (id: number | null) =>
    riskMonitoringRating.find((item) => item.id === id)?.status ||
    "Not Available";

  const getRiskAssessmentForRisk = (riskId: string) =>
    riskAssessment.filter((assessment) => assessment.risk_id === riskId);

  const exportToPDF = () => {
    const doc = new jsPDF("landscape");

    doc.setFontSize(12);
    doc.text("Risk and Risk Assessment Data", 14, 15);

    autoTable(doc, {
      startY: 20,
      head: [
        [
          "RRN",
          "Risk Statement",
          "Classification",
          "Likelihood",
          "Severity",
          "Control Rating",
          "Monitoring Rating",
        ],
      ],
      body: risks.flatMap((risk) => {
        const assessments = getRiskAssessmentForRisk(risk.id);
        if (assessments.length === 0) {
          return [
            [
              risk.rrn,
              risk.risk_statement,
              classification.find((cls) => cls.id === risk.classification)
                ?.name || "N/A",
              "No Data",
              "No Data",
              "No Data",
              "No Data",
            ],
          ];
        }

        return assessments.map((assessment) => [
          risk.rrn,
          risk.risk_statement,
          classification.find((cls) => cls.id === risk.classification)?.name ||
            "N/A",
          getLikelihoodRatingName(assessment.lr),
          getSeverityName(assessment.s),
          getRiskControlRatingName(assessment.rcr),
          getRiskMonitoringRatingStatus(assessment.rmr),
        ]);
      }),
    });

    doc.save("Risks_Assessment_Data.pdf");
  };

  const deleteRisk = async (riskId: string) => {
    deletingRiskId = riskId;
    try {
      const { error } = await supabase.from("risks").delete().eq("id", riskId);

      if (error) {
        errorMessage = "Failed to delete risk.";
        return;
      }

      risks = risks.filter((risk) => risk.id !== riskId);
      successMessage = "Risk deleted successfully!";
    } catch (error) {
      errorMessage = "An unexpected error occurred.";
    } finally {
      deletingRiskId = null;
    }
  };

  // Function to approve a risk
  const approveRisk = async (riskId: string) => {
    try {
      // Fetch the risk details
      const { data: risk, error: fetchError } = await supabase
        .from("risks")
        .select("id, rrn, risk_statement, profile_id")
        .eq("id", riskId)
        .single();

      if (fetchError || !risk) {
        throw new Error("Failed to fetch risk details.");
      }

      // Fetch the department_id from the profile
      const { data: profile, error: profileError } = await supabase
        .from("profiles")
        .select("department_id")
        .eq("id", risk.profile_id)
        .single();

      if (profileError || !profile) {
        throw new Error("Failed to fetch profile details.");
      }

      const departmentId = profile.department_id;

      // Fetch the related risk assessment details
      const { data: riskAssessment, error: assessmentError } = await supabase
        .from("risk_assessment")
        .select("*")
        .eq("risk_id", riskId);

      if (assessmentError || !riskAssessment || riskAssessment.length === 0) {
        throw new Error("Failed to fetch risk assessment details.");
      }

      // Insert into the risk_monitoring table
      const { error: insertError } = await supabase
        .from("risk_monitoring")
        .insert({
          risk_id: riskId,
          department_id: departmentId,
          likelihood_rating_id: riskAssessment[0].lr, // Likelihood Rating ID
          severity_id: riskAssessment[0].s, // Severity ID
          control_rating_id: riskAssessment[0].rcr, // Control Rating ID
          monitoring_rating_id: riskAssessment[0].rmr, // Monitoring Rating ID
        });

      if (insertError) {
        throw new Error("Failed to insert into risk_monitoring table.");
      }

      // Update the risk as approved
      const { error: updateError } = await supabase
        .from("risks")
        .update({ is_approved: true })
        .eq("id", riskId);

      if (updateError) {
        throw new Error("Failed to update risk approval status.");
      }

      // Update local risks state
      risks = risks.map((r) =>
        r.id === riskId ? { ...r, is_approved: true } : r
      );

      successMessage = `Risk approved successfully with risk assessment included!`;
    } catch (error) {
      console.error("Error approving risk:", error);
      errorMessage = "Failed to approve risk.";
    }
  };

  // Fetch all data on mount
  onMount(() => {
    Promise.all([
      fetchRisks(),
      fetchClassification(),
      fetchLikelihoodRating(),
      fetchSeverity(),
      fetchRiskControlRating(),
      fetchRiskMonitoringRating(),
      fetchRiskAssessment(),
    ]).catch((error) => {
      console.error("Error fetching data:", error);
      errorMessage = "Failed to fetch some data.";
    });
  });
</script>

<div class="container mx-auto my-6">
  <div class="mb-4">
    <button class="btn btn-primary" on:click={exportToPDF}>
      Export to PDF
    </button>
  </div>

  {#if isLoading}
    <div class="text-center text-xl">
      <span class="loading loading-spinner loading-md"></span>
    </div>
  {:else}
    {#if errorMessage}
      <div class="alert alert-error">{errorMessage}</div>
    {/if}

    {#if successMessage}
      <div class="alert alert-success">{successMessage}</div>
    {/if}

    <div class="overflow-x-auto">
      <table class="table w-full">
        <thead>
          <tr>
            <th>RRN</th>
            <th>Risk Statement</th>
            <th>Classification</th>
            <th>Actions</th>
            <th>Key Persons</th>
            <th>Budget</th>
            <th>Risk Assessments</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          {#each risks as risk}
            <tr>
              <td>{risk.rrn}</td>
              <td>{risk.risk_statement}</td>
              <td>
                {classification.find((cls) => cls.id === risk.classification)
                  ?.name || "N/A"}
              </td>
              <td>{risk.actions}</td>
              <td>{risk.key_persons}</td>
              <td>${risk.budget}</td>
              <td>
                {#if loadingRiskAssessments}
                  Loading...
                {:else}
                  {#each getRiskAssessmentForRisk(risk.id) as assessment}
                    <div>
                      <strong>Likelihood:</strong>
                      {getLikelihoodRatingName(assessment.lr)}
                      <br />
                      <strong>Severity:</strong>
                      {getSeverityName(assessment.s)}
                      <br />
                      <strong>Control Rating:</strong>
                      {getRiskControlRatingName(assessment.rcr)}
                      <br />
                      <strong>Monitoring Rating:</strong>
                      {getRiskMonitoringRatingStatus(assessment.rmr)}
                    </div>
                  {/each}
                  {#if getRiskAssessmentForRisk(risk.id).length === 0}
                    No assessment data available.
                  {/if}
                {/if}
              </td>
              <td>
                <button
                  class="btn btn-sm btn-success"
                  on:click={() => approveRisk(risk.id)}
                  disabled={risk.is_approved}
                >
                  {risk.is_approved ? "Approved" : "Approve Risk"}
                </button>
              </td>
              <td>
                <button
                  class="btn btn-error btn-sm"
                  on:click={() => deleteRisk(risk.id)}
                  disabled={deletingRiskId === risk.id}
                >
                  {#if deletingRiskId === risk.id}
                    Deleting...
                  {:else}
                    Delete
                  {/if}
                </button>
              </td>
            </tr>
          {/each}
        </tbody>
      </table>
    </div>
  {/if}
</div>
